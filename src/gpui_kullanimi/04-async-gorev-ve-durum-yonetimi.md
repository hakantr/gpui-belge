# Async, Görev ve Durum Yönetimi

---

## Async İşler

GPUI'de async işler, bağlam üzerinden başlatılan `Task` handle'larıyla yönetilir. Üç temel kalıp vardır: ön plan görevi, o anki entity'ye bağlı görev ve pencereyi de hesaba katan görev. Ön plan görevi UI iş parçacığı ile aynı çalıştırıcı üzerinde koşar; diğer iki kalıp entity ya da pencere yaşam döngüsünü hesaba katar. Her biri biraz farklı bir closure imzasıyla yazılır.

**Ön plan görevi.** En sade biçimdir; entity veya pencere bağlamı gerektirmez:

```rust
cx.spawn(async move |cx| {
    cx.update(|cx| {
        // App verisini güncelle
    })
})
.detach();
```

**Entity'ye bağlı görev.** Closure başlangıçta o anki entity'nin `WeakEntity` handle'ını verir. Böylece async beklemeler sırasında entity'nin düşmüş olma ihtimali `Result` üzerinden görünür hale gelir:

```rust
cx.spawn(async move |this, cx| {
    cx.background_executor().timer(Duration::from_millis(200)).await;
    this.update(cx, |this, cx| {
        this.ready = true;
        cx.notify();
    })?;
    Ok::<(), anyhow::Error>(())
})
.detach_and_log_err(cx);
```

**Pencereye bağlı görev.** Pencere bağlamını da async tarafa taşır; `Window` metotları `cx.update_in` ile birlikte kullanılabilir:

```rust
cx.spawn_in(window, async move |this, cx| {
    this.update_in(cx, |this, window, cx| {
        window.activate_window();
        cx.notify();
    })?;
    Ok::<(), anyhow::Error>(())
})
.detach_and_log_err(cx);
```

`Context::spawn` ve `spawn_in` imzaları `AsyncFnOnce(WeakEntity<T>, &mut AsyncApp/AsyncWindowContext)` ister; bu nedenle closure `async move |this, cx| { ... }` biçiminde yazılır. Closure'ın gövdesi `Result` döndürüyorsa derleyicinin tipini çıkarması için ya örnekteki gibi `Ok::<(), anyhow::Error>(())` ile son ifade tipinin açıkça yazılması, ya da en üstte `let _: Result<_, anyhow::Error> = ...` kalıbının kullanılması gerekir.

`window.to_async(cx)` doğrudan bir `AsyncWindowContext` üretir. Geri çağrı dışına taşınacak pencereye bağlı async yardımcılar yazılırken bu yol tercih edilir. Günlük entity ve view kodunda ise `cx.spawn_in(window, ...)`, daha güvenli ve okunaklı bir sarmalayıcıdır.

**Arka plan iş parçacığı.** CPU yoğun iş, ön plan çalıştırıcısını bloklamasın diye ayrı bir çalıştırıcıya verilir. Sonuç hazır olduğunda ön plan görevi ile UI'ya geri taşınır:

```rust
let task = cx.background_spawn(async move {
    expensive_work().await
});

cx.spawn(async move |cx| {
    let result = task.await;
    cx.update(|cx| {
        // sonucu UI'a taşı
    })
})
.detach();
```

**Testlerde zamanlayıcı.** Test ortamında zamanın kontrol altında olması için GPUI'nın kendi zamanlayıcısı kullanılır:

- GPUI testlerinde `smol::Timer::after(...)` yerine `cx.background_executor().timer(duration).await` çağrılır. Bu sayede zamanı elle ilerletme ve `run_until_parked()` ile senkron çalışma mümkün olur.
- `run_until_parked()` ile uyum sağlandığında GPUI çalıştırıcısının zamanlayıcısı tercih edilir; bu ikili, deterministik test akışının temelidir.

## Çalıştırıcı, Öncelik, Timeout ve Test Zamanı

GPUI'da iş iki çalıştırıcı arasında bölünür. Ön plan çalıştırıcısı UI iş parçacığı üzerinde çalışır; arka plan çalıştırıcısı ise zamanlayıcı (`scheduler`) ve iş parçacığı havuzu (`thread pool`) üzerinde çalışır. Bu ayrım yalnızca performans için değildir. Bir `await` noktasından sonra hangi bağlamın tutulabileceğini de belirler. Ön plan tarafında `App` veya `Window`'a güvenli dönüş noktaları vardır; arka plan tarafında bunlar yoktur.

**Temel tipler.** Sistem aşağıdaki parçalardan oluşur:

- `BackgroundExecutor`: ana metotları `spawn`, `spawn_with_priority`, `timer`, `scoped`, `scoped_priority`'dir; test desteğiyle birlikte `advance_clock`, `run_until_parked` ve `simulate_random_delay` da kullanılabilir.
- `ForegroundExecutor`: future'ları ana iş parçacığına yerleştirir; `spawn`, `spawn_with_priority`, ek olarak senkron köprü için `block_on` ve `block_with_timeout` sağlar.
- `Priority`: dört seviyesi vardır — `RealtimeAudio`, `High`, `Medium`, `Low`. `RealtimeAudio` ayrı bir iş parçacığı ister; UI dışı ses gibi çok sınırlı işler dışında kullanılması önerilmez.
- `Task<T>`: `await` edilebilir handle'dır. Değer elden çıktığında içindeki iş iptal olur; tamamlanmasının istendiği durumlarda `await` edilir, struct alanında saklanır veya `detach()` / `detach_and_log_err(cx)` ile bırakılır.
- `FutureExt::with_timeout(duration, executor)`: bir future'ı çalıştırıcının zamanlayıcısı ile yarıştırır ve sonucu `Result<T, Timeout>` olarak verir.

**Timeout örneği.** Tipik bir kullanım, uzun süren bir arka plan işine süre sınırı koymaktır:

```rust
let executor = cx.background_executor().clone();
let task = cx.background_spawn(async move {
    parse_large_file(path).await
});

let result = task
    .with_timeout(Duration::from_secs(5), &executor)
    .await?;
```

**Ön plan önceliği.** Aciliyet seviyeleri ön plan tarafında yoklama (`polling`) sırasını belirler; öncelikli işler çalıştırıcı kuyruğunda öne geçer:

```rust
cx.spawn_with_priority(Priority::High, async move |cx| {
    cx.update(|cx| {
        cx.refresh_windows();
    });
})
.detach();
```

**Update anlamı.** `AsyncApp::update(|cx| ...)` doğrudan `R` döndürür; entity'lerin aksine başarısız olabilen bir çağrı değildir, dolayısıyla `?` ile yayılması gerekmez. Pencere içi async çalışmada `AsyncWindowContext::update(|window, cx| ...) -> Result<R>` veya `Entity::update(cx, ...)` başarısız olabilen varyantları kullanılır.

**Async bağlam kestirme yüzeyi.** Async bağlamlar üzerinde tekrar tekrar ihtiyaç duyulan birkaç kestirme metot vardır:

- `AsyncApp::refresh()`, tüm pencereler için yeniden çizim planlar; async akıştan yeniden çizim tetiklemek için `update(|cx| cx.refresh_windows())` yazmaya gerek bırakmaz.
- `AsyncApp::background_executor()` ve `foreground_executor()`, çalıştırıcı handle'larını döndürür. Zamanlayıcı, timeout veya iç içe `spawn` gerektiğinde buradan alınır.
- `AsyncApp::subscribe(&entity, ...)`, `open_window(options, ...)`, `spawn(...)`, `has_global::<G>()`, `read_global`, `try_read_global`, `read_default_global`, `update_global` ve `on_drop(&weak, ...)`, `await` edilebilir app görevlerinde aynı ön plan verisine güvenli dönüş noktalarıdır.
- `AsyncWindowContext::window_handle()`, bağlı pencereyi verir. `update(|window, cx| ...)` yalnızca pencere verisini güncellerken, `update_root(|root, window, cx| ...)` root `AnyView` de gerektiğinde kullanılır.
- `AsyncWindowContext::on_next_frame(...)`, `read_global`, `update_global`, `spawn(...)` ve `prompt(...)`, pencereye bağlı async işlerde "pencere kapanmış olabilir" durumunu `Result` veya yedek `receiver` üzerinden yönetir.

**Entity ve pencereye bağlı öncelikli spawn.** Öncelikli işlerin daha tipli sürümleri için de yardımcılar mevcuttur:

- `cx.spawn_in_with_priority(priority, window, async move |weak, cx| { ... })`, o anki entity'nin `WeakEntity<T>` handle'ını ve `AsyncWindowContext` bağlamını verir.
- `window.spawn_with_priority(priority, cx, async move |cx| { ... })`, pencere handle'ına bağlı ama entity'siz async iş için uygundur.
- Öncelik yalnızca ön plan çalıştırıcısının kuyruğunda yoklama önceliği sağlar; uzun CPU işi hâlâ `background_spawn` tarafına taşınmalıdır.

**Hazır değer.** Bir hesap sonucu zaten eldeyse ek bir `Task` açmadan doğrudan `Task::ready` ile döndürülebilir. Bu, çağıran kodun imzasını her iki yolda da `Task` olarak tutarlı bırakır:

```rust
fn cached_or_async(cached: Option<Data>, cx: &App) -> Task<anyhow::Result<Data>> {
    if let Some(data) = cached {
        Task::ready(Ok(data))
    } else {
        cx.background_spawn(async move { load_data().await })
    }
}
```

**Test zamanı.** Test ortamında zamanlama davranışını anlamak için birkaç noktanın bilinmesi gerekir:

- `cx.background_executor().timer(duration).await` GPUI zamanlayıcısına bağlıdır; `smol::Timer::after` ise GPUI `run_until_parked()` ile uyumsuz kalabilir, bu nedenle testlerde tercih edilmez.
- `advance_clock(duration)` yalnızca sahte saati (`fake clock`) ilerletir; ilerletilen süreyle gelinen noktada hazır olan işleri çalıştırmak için ayrıca `run_until_parked()` çağrısı gereklidir.
- `allow_parking()`, bekleyen görev varken `parked` olmayı testte bilerek kabul etmek için kullanılır; üretim akışlarına taşınması doğru değildir.
- `block_with_timeout` zaman aşımı olduğunda future'ı geri verir; bu davranış, işi iptal etmek veya daha sonra yeniden yoklamak konusundaki kararı çağırana bırakır.
- `PriorityQueueSender<T>` / `PriorityQueueReceiver<T>` yalnızca Windows, Linux ve wasm `cfg`'lerinde yeniden dışa aktarılır. `send(priority, item)`, `try_pop()`, `pop()`, `try_iter()` ve `iter()` metotları high, medium ve low kuyruklarını ağırlıklı seçimle tüketir; `Priority::RealtimeAudio` bu kuyruğa girmez.

## Task, TaskExt ve Async Hata Yönetimi

`Task<T>` GPUI'ın temel async handle'ıdır. Yardımcı trait `TaskExt` (`crates/gpui/src/executor.rs:33+`) `Task<Result<T, E>>` tipleri üzerine ek metotlar bindirir:

```rust
pub trait TaskExt<T, E> {
    fn detach_and_log_err(self, cx: &App);
    fn detach_and_log_err_with_backtrace(self, cx: &App);
}
```

`detach_and_log_err`, görevi arka plana atar ve hata oluşması durumunda hatayı `log::error!("...: {err}")` biçiminde loglar. `_with_backtrace` varyantı aynı işi `{:?}` biçimiyle yapar; bu sayede `anyhow::Error` gibi geri izleme (`backtrace`) taşıyan tipler tam çağrı yığınıyla loglanır.

**Pratik akış.** Tipik bir async UI iş parçası, ağdan veri çekmek ve sonra view verisini güncellemekten oluşur:

```rust
cx.spawn_in(window, async move |this, cx| {
    let data = http_client.get(url).await?;
    this.update_in(cx, |this, window, cx| {
        this.apply(data, window, cx);
    })?;
    Ok::<(), anyhow::Error>(())
})
.detach_and_log_err(cx);
```

**Detach varyantları.** Görevi bırakırken niyete göre üç farklı yardımcı vardır:

- `task.detach()` — hata loglanmaz, sessizce yutulur. Yalnızca UI'da gösterilemeyen ve sonucu kaybolsa sorun olmayacak işler için uygundur.
- `task.detach_and_log_err(cx)` — standart akıştır; üretim kodunda hata yönetiminin varsayılan yolu olarak tercih edilir.
- `task.detach_and_prompt_err(prompt_label, window, cx, |err, window, cx| ...)` — workspace UI'sında kullanılan ek bir yardımcıdır (workspace crate'inde tanımlıdır); hatayı modal bir prompt ile kullanıcıya gösterir.

**Yazarken kararlar.** Bir görevin imzada nasıl görüneceğini seçerken şu pratik kurallar işe yarar:

- Async sonuç çağıran koda dönmeliyse metot `Task<R>` döndürmeli ve çağıran kod bunu `await` etmelidir. Sonucu struct alanında saklamak ise ileride alan elden çıkarsa iptal davranışı verir.
- Çağıran kod sonucu beklemeyecekse görevi geri döndürmek gereksizdir; doğrudan `detach_and_log_err(cx)` çağırmak niyeti daha açık gösterir.
- `Result`'ı log'a düşürmemek için elle `if let Err(e) = task.await { ... }` yazmak gereksizdir; `detach_and_log_err` zaten `track_caller` ile log konumunu kayda alır.

**Tuzaklar.** Bu API'lerle ilgili sık yapılan hatalar şunlardır:

- `Result` tipinin `E` argümanı `Display + Debug` istemelidir; `anyhow::Error` ve standart özel hata tipleri otomatik uyar.
- Görevler `Vec<Task<()>>` içinde toplandığında elden çıkma sırası sürpriz olabilir; iptalin amaçlanmadığı tipik akışlarda `detach()` daha açık bir niyet bildirir.
- `cx.spawn_in(window, ...)`, `Window` düştüğünde görevi otomatik iptal etmez; `WeakEntity` üzerinden `update` veya `update_in` çağrısı `Result` döndüğünden bu dönüş, erken çıkış sinyali olarak ele alınmalıdır.

## Uygulama Geneli Veri, Observe ve Event

GPUI'da uygulama genelindeki paylaşılan veri üç ana mekanizmayla yönetilir: `Global` (uygulama geneli veri), observe (veri değişimini dinlemek) ve event (tipli mesaj yayma). Üçü de bağlam üzerinden çağrılır ve genellikle birlikte kullanılır.

**Uygulama geneli veri.** Uygulama ömrü boyunca tek nüsha tutulan bir kaynak için `Global` trait'i uygulanır ve `cx.set_global` ile yerleştirilir:

```rust
struct MyGlobal(State);
impl Global for MyGlobal {}

cx.set_global(MyGlobal(state));

cx.update_global::<MyGlobal, _>(|global, cx| {
    global.0.changed = true;
});

let value = cx.read_global::<MyGlobal, _>(|global, _| global.0.clone());
```

**Observe.** Bir entity'nin `cx.notify()` çağırması bütün gözlemcileri tetikler. Tipik kullanım, türetilmiş veriyi kaynağa bağlamaktır:

```rust
subscriptions.push(cx.observe(&other, |this, other, cx| {
    this.copy = other.read(cx).value;
    cx.notify();
}));
```

**Event.** Tipli mesaj yayma için entity `EventEmitter<E>` uygular; yayılan olay abonelere ulaşır:

```rust
struct Saved;
impl EventEmitter<Saved> for Document {}

cx.emit(Saved);

subscriptions.push(cx.subscribe(&document, |this, document, _: &Saved, cx| {
    this.last_saved = Some(document.entity_id());
    cx.notify();
}));
```

**Pencere bazlı gözlem.** Pencerenin kendisine ait değişimler için ayrı bir gözlemci ailesi vardır; her biri ilgili pencere değişiminde tetiklenir:

- `cx.observe_window_bounds(window, ...)`
- `cx.observe_window_activation(window, ...)`
- `cx.observe_window_appearance(window, ...)`
- `cx.observe_button_layout_changed(window, ...)`
- `cx.observe_pending_input(window, ...)`
- `cx.observe_keystrokes(...)`

## Uygulama Geneli Veri Yardımcıları ve `cx.defer`

`App` üzerinde bulunan `Global` yardımcıları rehberin farklı bölümlerinde parça parça geçer; burada tek bir listede toplanır. Aynı kategori altında "global'i değiştir" ve "etki döngüsünü (`effect cycle`) yönet" başlıklarını birlikte ele almak en sık sorulan iki konuyu yan yana getirir.

- `cx.set_global<T: Global>(value)` — var olanı ezer; yoksa kurar.
- `cx.global<T>() -> &T` — var olduğundan emin olunan çağrı noktalarında kullanılır; yoksa `panic` üretir.
- `cx.global_mut<T>()` — aynısının değiştirilebilir karşılığıdır.
- `cx.default_global<T: Default>() -> &mut T` — global yoksa varsayılan değerle bir nesne oluşturur, varsa mevcut global'i değiştirilebilir biçimde verir.
- `cx.has_global<T>() -> bool` — okuma öncesi varlık kontrolü.
- `cx.try_global<T>() -> Option<&T>` — `null` olabilen okuma.
- `cx.update_global<T, R>(|g, cx| ...) -> R` — kapsamlı güncelleme.
- `cx.read_global<T, R>(|g, cx| ...) -> R` — kapsamlı okuma.
- `cx.remove_global<T>() -> T` — nesneyi geri alır; tekrar ayarlanmadığı sürece global yok sayılır.
- `cx.observe_global<T>(|cx| ...) -> Subscription` — global her bildirim gönderdiğinde geri çağrıyı tetikler.

**Etki döngüsü yönetimi.** GPUI içinde veri değişimleri "etki döngüsü" denilen turlar hâlinde işlenir; bir güncellemenin içinden başka bir güncelleme başlatmak yerine işleri ertelemek genellikle daha güvenlidir:

- `cx.defer(|cx| ...)`, mevcut etki döngüsü bittiğinde çalışır. İç içe güncellemeleri kırmak veya entity'leri yığına geri vermek için idealdir.
- `Context<T>::defer_in(window, |this, window, cx| ...)`, aynı erteleme davranışının pencereye bağlı varyantıdır.
- `window.defer(cx, |window, cx| ...)`, doğrudan pencere bağlamından ertelemek için kullanılır.
- `window.refresh()`, pencereyi bir sonraki ekran karesinde yeniden çizim için işaretler.
- `cx.refresh_windows()`, tüm pencereler için aynı işi yapar.

**Tuzaklar.** `Global`'ler ve `defer` kullanımında dikkat edilmesi gerekenler:

- `cx.global<T>()` ve `cx.global_mut<T>()` global yoksa `panic` üretir; kurulum kontrolünün yapılmadığı çağrı noktalarında `try_global` veya `has_global` tercih edilmelidir.
- `update_global` sırasında aynı global yeniden güncellenirse `panic` verir; iç içe çağrılar varsa erteleme için `defer` güvenli yoldur.
- `Subscription` `detach()` edilmezse sahibinin düşmesiyle iptal olur; uzun yaşaması gereken gözlemciler bir struct alanında saklanmalıdır.

## Subscription Yaşam Döngüsü

`crates/gpui/src/subscription.rs`.

`Subscription`, opak (`opaque`) bir tiptir; elden çıktığında içindeki geri çağrı kaydı silinir. Bu davranış üç farklı kullanım desenine yol açar; aralarındaki seçim abonelik kaydının ne kadar yaşaması gerektiğine bağlıdır:

```rust
// 1. Alanda sakla
struct View { _subs: Vec<Subscription> }
// new(): self._subs.push(cx.subscribe(...));

// 2. Detach (geri çağrı view ömrü boyunca yaşar)
cx.subscribe(&entity, |...| { ... }).detach();

// 3. Geçici kapsam (elden çıkınca abonelik kalkar)
let _sub = cx.observe(&entity, |...| { ... });
// _sub düştüğünde geri çağrı kaldırılır
```

**Abonelik üreten yöntemler.** `Context<T>` üzerinde abonelik üreten temel metotlar şunlardır:

- `cx.observe(entity, f)` — entity `cx.notify()` çağırdığında tetiklenir.
- `cx.subscribe(entity, f)` — `EventEmitter<E>` olayları için.
- `cx.observe_global::<G>(f)` — uygulama geneli veri değiştiğinde.
- `cx.observe_release(entity, f)` — entity elden çıktığında.
- `cx.on_focus(handle, window, f)`, `cx.on_blur(...)`, `cx.on_focus_in(...)`, `cx.on_focus_lost(window, f)`. Alt öğenin odak kaybı için düşük seviyeli `window.on_focus_out(handle, cx, f)` kullanılır.
- `cx.observe_window_bounds`, `cx.observe_window_activation`, `cx.observe_window_appearance`, `cx.observe_button_layout_changed`, `cx.observe_pending_input`, `cx.observe_keystrokes`.

**Tuzaklar.** Abonelik kullanımında dikkat edilecek noktalar:

- `detach()`, uzun yaşayan bir geri çağrıyı view ömründen koparır; view düştükten sonra geri çağrı hâlâ çalışıyorsa veri erişimi için `WeakEntity` ile koruma şart olur.
- Birden çok abonelik birbirini etkiliyorsa elden çıkma sırasına davranış bağlamak hatalıdır; açık bir kapatma metodu veya tek sahipli struct kullanmak güvenli yoldur.
- `observe` geri çağrısının içinden entity'yi güncellemek `panic` verir; bunun yerine `cx.spawn(..)` ile async akışa taşınır veya `cx.defer(|cx| ...)` ile sonraki etki döngüsüne ertelenir.

## Pencere Bağlı Gözlemci, Release ve Odak Yardımcı Desenleri

Standart `observe`, `subscribe` ve `on_release` geri çağrıları yalnızca `App` veya `Context<T>` verir. Buna karşılık UI katmanında çoğu iş pencereye de ihtiyaç duyduğu için GPUI, aynı desenlerin pencereye duyarlı varyantlarını ayrıca sağlar. Aşağıdaki başlıklar bu yardımcı ailelerini ve aralarındaki seçimi açar.

**Yeni entity gözleme.** Belirli bir tipin oluşturulduğu anda kanca (`hook`) takmak gerektiğinde `observe_new` kullanılır:

```rust
cx.observe_new(|workspace: &mut Workspace, window, cx| {
    if let Some(window) = window {
        workspace.install_window_hooks(window, cx);
    }
}).detach();
```

- `App::observe_new<T>(|state, Option<&mut Window>, &mut Context<T>| ...)`, belirli türde bir entity oluşturulduğunda çalışır. Entity bir pencere içinde yaratıldıysa geri çağrıya `Some(window)` gelir; başsız ya da uygulama seviyesindeki yaratımda `None` gelebilir.
- Zed `zed.rs`, `toast_layer`, `theme_preview`, `telemetry_log`, `move_to_applications` gibi modüllerde workspace, project veya editor yaratıldığında uygulama geneli kanca takmak için bu deseni kullanır.
- Dönen `Subscription` saklanmalı ya da uygulama ömrü boyunca gerekiyorsa `detach()` edilmelidir.

**Pencere bağlamıyla observe ve subscribe.** İçinde pencere bağlamı gerektiren abonelikler ayrı bir yardımcı ailesiyle yapılır:

```rust
self._observe_active_pane = cx.observe_in(active_pane, window, |this, pane, window, cx| {
    this.sync_from_pane(&pane, window, cx);
});

self._subscription = cx.subscribe_in(&modal, window, |this, modal, event, window, cx| {
    this.handle_modal_event(modal, event, window, cx);
});
```

- `Context<T>::observe_in(&Entity<V>, window, |this, Entity<V>, window, cx| ...)`, gözlenen entity `cx.notify()` yaptığında o anki entity'yi pencere bağlamı eşliğinde günceller.
- `Context<T>::subscribe_in(&Entity<Emitter>, window, |this, emitter, event, window, cx| ...)`, `EventEmitter` olaylarını pencere bağlamıyla işler.
- `Context<T>::observe_self(|this, cx| ...)`, o anki entity `cx.notify()` yaptığında kendi üzerinde geri çağrıyı çalıştırır; türetilmiş ya da önbelleklenmiş veriyi tek noktada tutmak için kullanılabilir.
- `Context<T>::subscribe_self::<Evt>(|this, event, cx| ...)`, o anki entity'nin kendi yaydığı olayı dinler. Bu desen dikkatli kullanılmalıdır; çoğu durumda olayı yayan kod yolunda veriyi doğrudan güncellemek daha açıktır.
- `Context<T>::observe_global_in::<G>(window, |this, window, cx| ...)`, uygulama geneli veri bildirim gönderdiğinde o anki entity'yi pencere bağlamıyla günceller. Pencere geçici olarak güncelleme yığınından alınmış veya kapanmışsa bildirim atlanır, gözlemci canlı kalır.
- Bu API'ler `ensure_window(observer_id, window.handle.id)` çağırır; entity'nin hangi pencereye bağlı çalışacağını GPUI'a kaydeder. Aynı entity birden fazla pencerede kullanılacaksa hangi pencerenin bağlandığı bilinçli şekilde düşünülmelidir.

**Release gözleme.** Bir entity yok edilirken yapılacak temizlikler için serbest bırakma (`release`) geri çağrıları vardır:

- `App::observe_release(&entity, |state, cx| ...)` — entity'nin son güçlü handle'ı düştükten sonra, veri elden çıkmadan hemen önce çalışır.
- `App::observe_release_in(&entity, window, |state, window, cx| ...)` — aynı geri çağrıyı pencere handle'ı üzerinden çalıştırır; pencere kapanmışsa güncelleme başarısız olur ve geri çağrı atlanabilir.
- `Context<T>::on_release_in(window, |this, window, cx| ...)` — o anki entity'nin serbest bırakılmasını pencereyle birlikte gözler.
- `Context<T>::observe_release_in(&other, window, |this, other, window, cx| ...)` — başka bir entity serbest bırakılırken gözlemci entity'yi de günceller.

**Odak yardımcıları.** Odak akışına müdahale için tipli yardımcılar mevcuttur:

- `cx.focus_view(&entity, window)` — `Focusable` uygulayan başka bir view'a odağı taşır.
- `cx.focus_self(window)` — o anki entity `Focusable` ise odağı kendine taşır. İçeride `window.defer(...)` kullanır; bu nedenle render ya da action geri çağrısı içinde çağrıldığında odak değişimi etki döngüsünün sonunda uygulanır.
- `window.disable_focus()` — pencerenin odağını sıfırlar ve ardından `focus_enabled` bayrağını `false` yapar. Tersine çeviren bir API yoktur; yani çağrıldıktan sonra `focus_next` / `focus_prev` / `focus(...)` çağrıları sessizce işlem yapmaz. Uygulama bileşenlerinde genellikle gerekmez; sadece pencere ömrü boyunca klavye odağını kalıcı kapatmak isteyen ender durumlarda kullanılır.

**Tuzaklar.** Bu yardımcı aileleri kullanılırken gözden kaçabilecek noktalar:

- `observe_new` geri çağrısında `window`'un her zaman var olduğu varsayılmamalıdır; başsız test ve uygulama seviyesindeki entity yaratımı `None` üretebilir.
- Pencereye bağlı abonelik bir struct alanında saklanmazsa geri çağrı hemen düşer.
- `focus_self` ertelemeli çalıştığı için hemen sonraki satırda odak değişmiş gibi okumak yanıltıcıdır; sonucu sonraki etki veya ekran karesi akışında gözlemek gerekir.

## Entity Reservation ve Çift Yönlü Referans

`crates/gpui/src/app/async_context.rs:43+` ve `app.rs::reserve_entity`/`insert_entity`.

Bazen bir entity oluşturulurken başka bir entity'nin kimliğini veya zayıf handle'ını önceden bilmek gerekir; en tipik örnek `Workspace` ve `Pane` ikilisidir. Bunu kuvvetli referans döngüsü kurmadan yapmak için `Reservation` deseni mevcuttur. Önce kimlik rezerve edilir, ardından entity bu rezervasyona yerleştirilir:

```rust
let pane_reservation: Reservation<Pane> = cx.reserve_entity();
let pane_id = pane_reservation.entity_id();

let workspace = cx.new(|cx| {
    Workspace::with_pane_id(pane_id, cx)
});

let pane = cx.insert_entity(pane_reservation, |cx| {
    Pane::new(workspace.downgrade(), cx)
});
```

**`Reservation<T>` yüzeyi.** Rezervasyon nesnesi ile iki temel iş yapılır:

- `entity_id()` — entity henüz oluşturulmadan kimliğini verir.
- `cx.insert_entity(reservation, build)` — rezervasyonu doldurur ve `Entity<T>` döndürür.
- Doldurulmadan elden çıkarıldığında rezervasyon iptal olur.

**Deseni.** Alt entity, üst öğeye `WeakEntity` ile bağlanır; rezervasyon sayesinde üst öğeyi oluştururken alt öğenin handle'ının önceden bilinmesi gereken durumlarda da döngü oluşmaz. Aynı API `AsyncApp` üzerinden de çağrılabilir.

**Tuzaklar.** Rezervasyon kullanılırken karşılaşılabilecek hata desenleri:

- Rezervasyon kullanmadan iki `Entity<T>` birbirine güçlü sahiplikle bağlandığında, hiçbir handle düşmediği için bellek sızıntısı oluşur.
- `insert_entity` çağrılmadan rezervasyon elden çıkarıldığında entity hiç oluşturulmamış sayılır; daha önce `entity_id()` ile yayılmış olan kimlik artık geçersizdir.
- `cx.new`, bir güncellemenin ortasında rezervasyonu da doldurabilir; iç içe güncelleme yasakları rezervasyon için de aynen geçerlidir.

## Entity Release, Temizlik ve Sızıntı Tespiti

Entity handle'ları referans sayısı (`ref-count`) mantığıyla yaşar; son güçlü `Entity<T>` handle'ı düştüğünde entity serbest bırakılır. `WeakEntity<T>` ise bu davranışı engellemez, yalnızca canlıyken zayıf erişim sağlar.

**Temizlik API'leri.** Serbest bırakma anında yapılacak işler için birkaç farklı geri çağrı biçimi vardır:

- `cx.on_release(|this, cx| ...)` — mevcut entity serbest bırakılırken çalışır.
- `App::observe_release(&entity, |entity, cx| ...)` — uygulama bağlamından başka bir entity'nin serbest bırakılmasını izler.
- `Context<T>::observe_release(&entity, |this, entity, cx| ...)` — view verisi ile birlikte başka bir entity'nin serbest bırakılmasını izler.
- `window.observe_release(&entity, cx, |entity, window, cx| ...)` — serbest bırakma sırasında pencere bağlamı gerektiğinde kullanılır.
- `cx.on_drop(...)` / `AsyncApp::on_drop(...)` — Rust kapsamı düştüğünde entity güncellemek için ertelenmiş bir geri çağrı üretir; entity zaten düşmüşse güncelleme başarısız olabilir.

**Örnek.** Önbellek serbest bırakılmasını gözlemleyen bir view tipik bir desendir:

```rust
struct Preview {
    cache: Entity<RetainAllImageCache>,
    cache_released: bool,
    _subscriptions: Vec<Subscription>,
}

impl Preview {
    fn new(cx: &mut Context<Self>) -> Self {
        let cache = RetainAllImageCache::new(cx);
        let subscription = cx.observe_release(&cache, |this, _cache, cx| {
            this.cache_released = true;
            cx.notify();
        });

        Self {
            cache,
            cache_released: false,
            _subscriptions: vec![subscription],
        }
    }
}
```

**Sızıntı kontrolü.** Test ve özellik bayrağı altında entity sızıntısı izlenebilir:

```rust
let snapshot = cx.leak_detector_snapshot();
// Test gövdesi.
cx.assert_no_new_leaks(&snapshot);
```

**Tuzaklar.** Temizlik ve serbest bırakma çalışmasında dikkat edilmesi gerekenler:

- `Subscription` saklanmadığında hemen düşer ve dinleyici iptal edilir.
- Karşılıklı `Entity<T>` alanları döngü üretir; bir tarafın `WeakEntity<T>` olması gereklidir.
- Serbest bırakma geri çağrısı içinde uzun async iş başlatılacaksa entity verisinin kapanmakta olduğu varsayılmalı; gerekli veri geri çağrı başında kopyalanmalıdır.
- `WeakEntity::update` ve `read_with` her zaman `Result` döndürür; entity düşmüş olabileceği için hata görünür biçimde ele alınmalıdır.

## Entity Tip Soyutlaması, Geri Çağrı Adaptörleri ve View Önbelleği

Bu bölüm GPUI çekirdeğinde public olan ama günlük kullanımda kolay atlanan küçük API yüzeylerini toplar. Entity'nin tipli ve tipsiz varyantları, view önbellek mekanizması, geri çağrı adaptörleri ve düşük seviyeli kimlik tipleri burada ele alınır.

#### Entity ve WeakEntity Tam Yüzeyi

`Entity<T>` güçlü handle'dır ve şu metotları sağlar:

- `entity.entity_id() -> EntityId`
- `entity.downgrade() -> WeakEntity<T>`
- `entity.into_any() -> AnyEntity`
- `entity.read(cx: &App) -> &T`
- `entity.read_with(cx, |state, cx| ...) -> R`
- `entity.update(cx, |state, cx| ...) -> R`
- `entity.update_in(visual_cx, |state, window, cx| ...) -> C::Result<R>`
- `entity.as_mut(cx) -> GpuiBorrow<T>` — değiştirilebilir ödünç verir; ödünç düşerken entity bildirim alır.
- `entity.write(cx, value)` — veriyi tamamen değiştirir ve `cx.notify()` çağırır.

`Context<T>`, o anki entity için aynı kimlik ve handle yüzeyini sağlar:

- `cx.entity_id() -> EntityId`
- `cx.entity() -> Entity<T>` — o anki entity hâlâ canlı olmak zorunda olduğu için güçlü handle döndürür.
- `cx.weak_entity() -> WeakEntity<T>` — async görev, dinleyici veya döngüsel sahiplik riski olan alanlarda saklanacak handle budur.

**Kimlik dönüşümleri.** GPUI çalışma zamanı kimlikleri `u64` olarak dışarı verilebilir:

- `EntityId::as_u64()` ve `EntityId::as_non_zero_u64()`, FFI, telemetri veya hata ayıklama eşlemesi anahtarı gibi tipli id sınırının dışına çıkılan yerlerde kullanılır.
- `WindowId::as_u64()` aynı işi pencere kimliği için yapar. Bu değerler iş alanı (`domain`) kimliği veya kalıcı workspace serileştirme anahtarı olarak kullanılmamalıdır; GPUI çalışma zamanı kimliği olmaları kalıcılık garantisi vermez.

`WeakEntity<T>` zayıf handle'dır:

- `weak.upgrade() -> Option<Entity<T>>`
- `weak.update(cx, |state, cx| ...) -> Result<R>`
- `weak.read_with(cx, |state, cx| ...) -> Result<R>`
- `weak.update_in(cx, |state, window, cx| ...) -> Result<R>` — entity'nin o anki penceresini `App::with_window(entity_id, ...)` üzerinden bulur; entity düşmüşse veya o anki pencere yoksa hata döner.
- `WeakEntity::new_invalid()`, hiçbir zaman yükseltilemeyen nöbetçi (`sentinel`) handle üretir; opsiyon yerine "geçersiz ama tipli handle" gereken yerlerde kullanılır.

`AnyEntity` ve `AnyWeakEntity`, heterojen koleksiyonlar içindir:

- `AnyEntity::{entity_id, entity_type, downgrade, downcast::<T>}`
- `AnyWeakEntity::{entity_id, is_upgradable, upgrade, new_invalid}`
- `AnyWeakEntity::assert_released()` yalnız `test` veya `leak-detection` özelliği altında vardır; güçlü handle sızıntısını yakalamak için kullanılır.

**Kural.** Plugin, dock veya workspace gibi heterojen koleksiyon sınırı yoksa tipli `Entity<T>` veya `WeakEntity<T>` tercih edilir. `AnyEntity`, alta dönüştürme (`downcast`) zorunluluğu getirir; yanlış tipte `downcast::<T>()` çağrısı entity'yi `Err(AnyEntity)` olarak geri verir.

##### Deref ile gizlenmiş yüzey (tipli handle üzerinden tipsiz metot)

`Entity<T>` ve `WeakEntity<T>`, `#[derive(Deref, DerefMut)]` ile içlerindeki tipsiz handle'a deref eder (`crates/gpui/src/app/entity_map.rs:413` ve `:739`):

```rust
#[derive(Deref, DerefMut)]
pub struct Entity<T>     { any_entity: AnyEntity,         entity_type: PhantomData<_> }
#[derive(Deref, DerefMut)]
pub struct WeakEntity<T> { any_entity: AnyWeakEntity,     entity_type: PhantomData<_> }
```

Bu yüzden `AnyEntity` ve `AnyWeakEntity` üzerindeki bazı metotlar tipli handle'da metot çözümlemesi ile çağrılabilir. "Sahip yalnız tipsiz tiptir" yanılgısı buradan doğar. Doğru ayrım `Sahip::metot -> dönüş tipi` çiftiyle yapılır; metot adı tek başına yeterli değildir.

| Sahip | Metot | Dönüş | Erişim |
|---|---|---|---|
| `Entity<T>` | `entity_id()` | `EntityId` | Doğrudan (`AnyEntity::entity_id` ile aynı değeri okur; doğrudan çağrı kazanır). |
| `Entity<T>` | `downgrade()` | `WeakEntity<T>` | Doğrudan; aynı adlı `AnyEntity::downgrade -> AnyWeakEntity` gölgelenir. |
| `Entity<T>` | `into_any()` | `AnyEntity` | Doğrudan; `self`'i tüketir. |
| `Entity<T>` | `read(&App)` | `&T` | Doğrudan. |
| `Entity<T>` | `read_with(cx, \|&T, &App\| ...)` | `R` | Doğrudan. |
| `Entity<T>` | `update(cx, \|&mut T, &mut Context<T>\| ...)` | `R` | Doğrudan. |
| `Entity<T>` | `update_in(visual_cx, \|&mut T, &mut Window, &mut Context<T>\| ...)` | `C::Result<R>` | Doğrudan. |
| `Entity<T>` | `as_mut(&mut cx)` | `GpuiBorrow<T>` | Doğrudan; düşerken `cx.notify()`. |
| `Entity<T>` | `write(&mut cx, value)` | `()` | Doğrudan; veriyi değiştirir ve bildirim gönderir. |
| `Entity<T>` deref | `entity_type()` | `TypeId` | Sahip `AnyEntity`; `Entity`'de doğrudan karşılığı yoktur, deref ile çağrılır. |
| `Entity<T>` yalnız deref özel durum | `downcast::<U>()` | `Result<Entity<U>, AnyEntity>` | Sahip `AnyEntity::downcast(self)` — **`self`'i tüketir**. Otomatik deref tüketen metoda uygulanmadığı için `entity.downcast::<U>()` doğrudan derlenmez; `entity.into_any().downcast::<U>()` yazılmalıdır. |
| `AnyEntity` | `entity_id()` | `EntityId` | Doğrudan. |
| `AnyEntity` | `entity_type()` | `TypeId` | Doğrudan. |
| `AnyEntity` | `downgrade()` | `AnyWeakEntity` | Doğrudan. |
| `AnyEntity` | `downcast::<U>()` | `Result<Entity<U>, AnyEntity>` | Doğrudan; `self`'i tüketir. |

`WeakEntity<T>` tarafında ad çakışmaları yüzünden tablo daha da dikkatli okunmalıdır; doğrudan ve yalnız deref ile gelen metotlar aynı isimle farklı imza taşır:

| Sahip | Metot | Dönüş | Erişim |
|---|---|---|---|
| `WeakEntity<T>` | `upgrade()` | `Option<Entity<T>>` | Doğrudan; aynı adlı `AnyWeakEntity::upgrade -> Option<AnyEntity>` gölgelenir. |
| `WeakEntity<T>` | `update(cx, \|&mut T, &mut Context<T>\| ...)` | `Result<R>` | Doğrudan. |
| `WeakEntity<T>` | `update_in(cx, \|&mut T, &mut Window, &mut Context<T>\| ...)` | `Result<R>` | Doğrudan; entity'nin son render edildiği pencereyi `App::with_window` ile bulur. |
| `WeakEntity<T>` | `read_with(cx, \|&T, &App\| ...)` | `Result<R>` | Doğrudan. |
| `WeakEntity<T>` | `new_invalid()` | `Self` | Doğrudan; aynı adlı `AnyWeakEntity::new_invalid -> AnyWeakEntity` gölgelenir. |
| `WeakEntity<T>` deref | `entity_id()` | `EntityId` | Sahip `AnyWeakEntity`; deref ile çağrılır. |
| `WeakEntity<T>` deref | `is_upgradable()` | `bool` | Sahip `AnyWeakEntity`; deref ile çağrılır. |
| `WeakEntity<T>` deref | `assert_released()` | `()` | Sahip `AnyWeakEntity`; yalnız `test` veya `leak-detection`. |
| `AnyWeakEntity` | `entity_id()` | `EntityId` | Doğrudan. |
| `AnyWeakEntity` | `is_upgradable()` | `bool` | Doğrudan. |
| `AnyWeakEntity` | `upgrade()` | `Option<AnyEntity>` | Doğrudan — tipli handle üzerinden çağrılırsa `WeakEntity::upgrade` kazanır. |
| `AnyWeakEntity` | `new_invalid()` | `Self` | Doğrudan. |
| `AnyWeakEntity` | `assert_released()` | `()` | Doğrudan; `test` veya `leak-detection`. |

**Pratik sonuç.** `weak.entity_id()` çağrısı, tipli handle'da deref üzerinden `AnyWeakEntity::entity_id` metoduna iner. Buna karşılık `weak.upgrade()` çağrısında tipli metot kazanır ve sonuç `Option<Entity<T>>` olur. Aynı kod parçasında ikisi yan yana görünebilir ama sahipleri farklıdır; ayrım yine `Sahip::metot -> dönüş` çiftiyle yapılır.

#### AnyView, AnyWeakView ve EmptyView

`AnyView`, `Render` uygulayan bir `Entity<V>` için element olarak kullanılabilen, tipi silinmiş view handle'ıdır. Bu sayede farklı view tipleri tek bir element yuvasına yerleştirilebilir:

```rust
let view: AnyView = pane.clone().into();
div().child(view.clone());
```

**Önemli metotlar.** Tipi silinmiş view ile çalışırken sık kullanılan API'ler şunlardır:

- `AnyView::from(entity)` veya `entity.into_element()` — tipli view'u, tipi silinmiş element hâline getirir.
- `any_view.downcast::<T>() -> Result<Entity<T>, AnyView>` — tipli handle'a geri dönmek için kullanılır.
- `any_view.downgrade() -> AnyWeakView` ve `AnyWeakView::upgrade() -> Option<AnyView>` — zayıf handle dönüşümleri.
- `any_view.entity_id()` ve `entity_type()` — hata ayıklama ve kayıt defteri mantığında kullanılır.
- `EmptyView` — hiçbir şey render etmeyen `Render` view'udur; yer tutucu amaçlı kullanılır.

**Önbelleklenmiş view.** `AnyView::cached(style_refinement)`, pahalı bir alt öğe view'unun render'ını önbelleğe almak için kullanılır:

```rust
div().child(
    AnyView::from(pane.clone())
        .cached(StyleRefinement::default().v_flex().size_full()),
)
```

Önbellek, view `cx.notify()` çağırmadığı sürece önceki yerleşim, çizim hazırlığı ve çizim aralıklarını yeniden kullanır. `Window::refresh()` çağrısı önbelleği atlatır; inspector seçimi açıkken de hitbox'ların eksiksiz olabilmesi için önbellek devre dışı kalır. Önbellek anahtarı; sınırları (`bounds`), aktif `ContentMask` ve aktif `TextStyle`'ı içerir. Bu nedenle `cached(...)` çağrısında verilen root `StyleRefinement`, view'un gerçek root yerleşim stiliyle uyumlu olmalıdır; yanlış refinement yerleşimi bayat veya hatalı gösterir.

#### Geri Çağrı Adaptörleri

Çoğu element geri çağrısı view verisini doğrudan almaz:

```rust
Fn(&Event, &mut Window, &mut App)
```

Geri çağrıdan view verisine geri dönmek için uygun adaptör seçilir:

- `cx.listener(|this, event, window, cx| ...)` — `Fn(&Event, &mut Window, &mut App)` üretir. İçeride o anki entity'nin `WeakEntity` handle'ı kullanılır; entity düşmüşse geri çağrı sessizce işlem yapmaz.
- `cx.processor(|this, event, window, cx| -> R { ... })` — `Fn(Event, &mut Window, &mut App) -> R` üretir. Olayı sahiplenir ve dönüş değeri gerektiğinde tercih edilir.
- `window.listener_for(&entity, |state, event, window, cx| ...)` — o anki `Context<T>` dışında, elde tipli `Entity<T>` varken dinleyici üretir.
- `window.handler_for(&entity, |state, window, cx| ...)` — olay parametresi olmayan `Fn(&mut Window, &mut App)` dinleyicisi üretir.

`cx.listener` dışındaki adaptörler, yeniden kullanılabilir bileşenler veya pencere seviyesindeki yardımcılar yazılırken devreye girer. Dinleyici içinde veri değiştiğinde `cx.notify()` çağırmak yine çağıranın sorumluluğundadır; adaptör bunu otomatik yapmaz.

#### `FocusHandle` Zayıf Handle ve Dispatch

`FocusHandle` yalnızca odak vermek için değildir; zayıf handle ve yönlendirme kontrolü de sunar:

- `focus_handle.downgrade() -> WeakFocusHandle`
- `WeakFocusHandle::upgrade() -> Option<FocusHandle>`
- `focus_handle.contains(&other, window) -> bool` — son render edilen ekran karesindeki odak ağacı ilişkisini kontrol eder.
- `focus_handle.dispatch_action(&action, window, cx)` — yönlendirmeyi odaktaki düğüm yerine belirli bir odak handle'ının düğümünden başlatır.

`contains_focused(window, cx)` "ben veya altımdaki bir düğüm odakta mı?" sorusuna, `within_focused` ise "ben odaktaki düğümün içinde miyim?" sorusuna cevap verir. `within_focused` imzasında `cx: &mut App` vardır; çünkü yönlendirme ve odak yolu hesaplarında uygulama verisiyle çalışır.

#### `ElementId` Tam Varyantları

Stabil element verisi için kullanılan `ElementId` varyantları şunlardır:

- `View(EntityId)`, `Integer(u64)`, `Name(SharedString)`, `Uuid(Uuid)`
- `FocusHandle(FocusId)`, `NamedInteger(SharedString, u64)`, `Path(Arc<Path>)`
- `CodeLocation(Location<'static>)`, `NamedChild(Arc<ElementId>, SharedString)`
- `OpaqueId([u8; 20])`

`ElementId::named_usize(name, usize)`, `NamedInteger` üretir. Pratik seçim şöyledir: hata ayıklama seçicisi veya metin tabanlı ID gerektiğinde `Name`; liste satırı gibi tekrar eden yapılarda `NamedInteger`; metin tutamacı (`text anchor`) gibi byte seviyesinde kimliklerde `OpaqueId` tercih edilir.

---
