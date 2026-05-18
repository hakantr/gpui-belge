# Bağlamlar ve Pencere Handle'ları

---

## Temel Bağlamlar

GPUI'de neredeyse her iş bir bağlam (context) üzerinden yapılır. Kodda bu bağlam genellikle `cx` adıyla görünür. Bağlam, o anda hangi katmandan konuşulduğunu ve nelere erişilebildiğini belirler. Birden fazla bağlam tipi vardır ve her birinin sorumluluğu farklıdır:

- **`App`**: uygulamanın kök bağlamıdır. Global durum, açık pencerelerin listesi, platform servisleri, keymap, global'ler, yeni entity oluşturma ve pencere açma gibi süreç ömrü boyu geçerli işler buradan yapılır.
- **`Context<T>`**: belirli bir `Entity<T>` güncellenirken karşılaşılan bağlamdır. `App` üzerine deref eder, yani `App`'in tüm metotlarına buradan da ulaşılabilir; ek olarak `cx.notify()`, `cx.emit(...)`, `cx.listener(...)`, `cx.observe(...)`, `cx.subscribe(...)`, `cx.spawn(...)` gibi entity'ye özel API'leri açar.
- **`Window`**: tek bir pencereye özgü durum ve davranıştır. Klavye odağı, imleç, pencere boyutu, yeniden boyutlandırma, başlık, arka plan görünümü, komut yönlendirme (`action dispatch`), IME, prompt ve platform pencere işlemleri bu bağlam üzerinden yürür.
- **`AsyncApp` / `AsyncWindowContext`**: bir `await` noktasının ötesine kadar taşınabilen async bağlamlardır. Bekleyiş sırasında entity veya pencere kapanmış olabileceği için bu bağlamlarda yapılan erişimler `Result` döner; yani başarısız olabilir.
- **`TestAppContext` / `VisualTestContext`**: testlerde simülasyon, zamanlayıcı kontrolü ve görsel doğrulama için ayrılmış bağlamlardır; üretim akışlarında kullanılmaz.

**Entity kullanımı.** Bir entity hem okunabilir hem güncellenebilir. İki işlem de bağlam üzerinden yapılır:

```rust
let varlik = baglam.new(|baglam| Durum::new(baglam));

let deger = varlik.read(baglam).deger;

varlik.update(baglam, |durum, baglam| {
    durum.deger += 1;
    baglam.notify();
});

let zayif = varlik.downgrade();
zayif.update(baglam, |durum, baglam| {
    durum.deger += 1;
    baglam.notify();
})?;
```

**Kurallar.** Bağlam kullanımında dikkat edilmesi gereken ana noktalar şunlardır:

- Çizim çıktısını etkileyen bir veri değiştiğinde `cx.notify()` çağrılır. Aksi halde view'da yeni veriye rağmen ekran yenilenmez.
- Bir entity güncellenirken aynı entity yeniden update'e sokulmaz; bu durum yeniden girişe (`reentrancy`) yol açtığı için `panic` üretebilir.
- Uzun yaşayan async işlerde `Entity<T>` yerine `WeakEntity<T>` yakalanır; bu sayede iş bitmeden entity elden çıktığında döngüsel sahiplik oluşmaz.
- Bir `Task` değeri elden çıktığında içindeki iş iptal olur. Bu nedenle ya `await` edilir, ya struct'ın bir alanında saklanır, ya da `detach()` / `detach_and_log_err(cx)` ile bağımsız bırakılır.

## WindowHandle, AnyWindowHandle ve VisualContext

`open_window` ve test yardımcıları, açılan pencereyi temsil eden tipli bir `WindowHandle<V>` döndürür. Bu handle, kök view'un tipini derleme zamanında taşır. Buna karşılık `AnyWindowHandle` kök tipini çalışma zamanında tutar ve gerektiğinde alta dönüştürülebilir (`downcast`).

İki handle arasında doğrudan bir Deref ilişkisi vardır: `WindowHandle<V>` `#[derive(Deref, DerefMut)]` sayesinde içindeki `AnyWindowHandle` değerine deref olur. Bu yüzden bazı metotlar tipli handle üzerinden çağrılabilir gibi görünür, oysa o metotların asıl sahibi `AnyWindowHandle`'dır. API yüzeyi okunurken `Owner::method -> dönüş tipi -> hata davranışı` üçlüsünü birlikte düşünmek gerekir. Yalnız metot adına bakmak burada kolayca yanıltır.

Aynı Deref kalıbı GPUI'de başka handle ve olay ailelerinde de görülür:

- `Entity<T>: Deref<Target = AnyEntity>` ve `WeakEntity<T>: Deref<Target = AnyWeakEntity>` — tipli entity handle'ları tipsiz handle'a deref eder. Detay "Entity Tip Soyutlaması, Geri Çağrı Adaptörleri ve View Önbelleği" başlığında işlenir.
- `ModifiersChangedEvent`, `ScrollWheelEvent`, `PinchEvent`, `MouseExitEvent`: `Deref<Target = Modifiers>` — bu dört olay Modifiers metotlarını doğrudan açar (`event.secondary()`, `event.modified()`). Buna karşılık `MouseDownEvent`, `MouseUpEvent`, `MouseMoveEvent` Deref *etmez*; yalnızca `modifiers` alanını alan olarak verir. Detay "CursorStyle, FontWeight ve Sabit Enum Tabloları" başlığındadır.
- `Context<'a, T>: Deref<Target = App>` — `cx.theme()`, `cx.refresh_windows()` gibi App metotları Context üzerinden de çağrılabilir (bkz. "Temel Bağlamlar").

Bu Deref kalıbında metot adı tek başına yeterli değildir. Aynı isim tipli ve tipsiz sahiplerde (`owner`) farklı dönüş tipleriyle bulunabilir.

**`WindowHandle<V>`.** Tipli handle kök view'a doğrudan tipli erişim sağlar:

```rust
tutamac.update(baglam, |kok: &mut Workspace, pencere, baglam| {
    kok.focus_active_pane(pencere, baglam);
})?;

let kok_ref: &Workspace = tutamac.read(baglam)?;
let baslik = tutamac.read_with(baglam, |kok, baglam| kok.title(baglam))?;
let varlik = tutamac.entity(baglam)?;
// WindowHandle::is_active `Option<bool>` döner; pencere kapanmış/geçici
// olarak ödünç alınmışsa `None`. Tipik kullanım:
let aktif_mi: Option<bool> = tutamac.is_active(baglam);

// `window_id()` sahip olarak `AnyWindowHandle` metodudur; fakat
// `WindowHandle<V>: Deref<Target = AnyWindowHandle>` olduğu için bu çağrı
// metot çözümleme ile çalışır:
let id = tutamac.window_id();

// Tipsiz tutamac saklamak veya AnyWindowHandle API'sini açık göstermek
// gerektiğinde dönüşüm bilinçli yapılır:
let tipsiz: AnyWindowHandle = tutamac.into();
let ayni_id = tipsiz.window_id();
```

**`AnyWindowHandle`.** Tipi silinmiş handle ise jenerik kodlarda ve çalışma zamanı `downcast` senaryolarında kullanılır:

```rust
if let Some(calisma_alani) = tipsiz_tutamac.downcast::<Workspace>() {
    calisma_alani.update(baglam, |calisma_alani, pencere, baglam| {
        calisma_alani.activate(pencere, baglam);
    })?;
}

tipsiz_tutamac.update(baglam, |kok_gorunum, pencere, _baglam| {
    let kok_varlik_id = kok_gorunum.entity_id();
    pencere.refresh();
    (kok_varlik_id, pencere.is_window_active())
})?;

let baslik = tipsiz_tutamac.read::<Workspace, _, _>(baglam, |calisma_alani, baglam| {
    calisma_alani.read(baglam).title(baglam)
})?;
```

**Tam sahip/metot yüzeyi.** Aşağıdaki tablo her metodun asıl sahibini, dönüş tipini ve hata davranışını birlikte gösterir. Böylece tipli handle üzerinde "hangi çağrı bu tipe ait, hangisi deref ile geliyor" ayrımı netleşir:

| Sahip | Metot | Dönüş | Not |
|---|---|---|---|
| `WindowHandle<V>` | `new(id)` | `Self` | Kök tipini çalışma zamanında doğrulamaz; id + `TypeId::of::<V>()` saklar. |
| `WindowHandle<V>` | `root(cx)` | `Result<Entity<V>>` | Sadece `test` veya `test-support`; kök tipi uyuşmazsa ya da pencere kapalıysa hata. |
| `WindowHandle<V>` | `update(cx, \|&mut V, &mut Window, &mut Context<V>\| ...)` | `Result<R>` | Tipli kök view'u günceller. |
| `WindowHandle<V>` | `read(&App)` | `Result<&V>` | Kısa süreli değiştirilemez ödünç alma; kapalı veya ödünç verilmiş pencere hata verir. |
| `WindowHandle<V>` | `read_with(cx, \|&V, &App\| ...)` | `Result<R>` | Geri çağrı içinde güvenli okuma. |
| `WindowHandle<V>` | `entity(cx)` | `Result<Entity<V>>` | Kök entity handle'ını döndürür. |
| `WindowHandle<V>` | `is_active(&mut App)` | `Option<bool>` | Kapalı veya ödünç verilmiş pencere için `None` döner. |
| `WindowHandle<V>` deref | `window_id()` | `WindowId` | Asıl sahip `AnyWindowHandle`; deref sayesinde `handle.window_id()` çalışır. |
| `WindowHandle<V>` deref | `downcast<T>()` | `Option<WindowHandle<T>>` | Asıl sahip `AnyWindowHandle`; tipli handle üzerinde çoğu zaman gereksizdir. |
| `AnyWindowHandle` | `window_id()` | `WindowId` | Pencere kimliği. |
| `AnyWindowHandle` | `downcast<T>()` | `Option<WindowHandle<T>>` | `TypeId` eşleşmezse `None`. |
| `AnyWindowHandle` | `update(cx, \|AnyView, &mut Window, &mut App\| ...)` | `Result<R>` | Kök tipi bilinmez; geri çağrı `AnyView` alır. |
| `AnyWindowHandle` | `read::<T, _, _>(cx, \|Entity<T>, &App\| ...)` | `Result<R>` | Önce `downcast` yapar, sonra tipli entity okutur. |

**Context trait'leri.** Farklı bağlamlar arasında ortak metot setleri trait'ler aracılığıyla paylaşılır:

- `AppContext`: `new`, `reserve_entity`, `insert_entity`, `update_entity`, `read_entity`, `update_window`, `with_window`, `read_window`, `background_spawn`, `read_global` gibi temel App davranışlarını sağlar.
- `VisualContext`: pencereye bağlı bağlamlara (örneğin `Window`+`App` çiftine) `window_handle`, `update_window_entity`, `new_window_entity`, `replace_root_view`, `focus` metotlarını ekler.
- `BorrowAppContext`: `App`, `Context<T>`, async ve test context'leri gibi farklı bağlamlar arasında çalışan yardımcı fonksiyonlar için ortak global API yüzeyidir.

**Pencerede kök değiştirme.** `window.replace_root(cx, |window, cx| NewRoot::new(window, cx))` çağrısı, mevcut pencerenin kök entity'sini yeni bir `Render` view ile değiştirir. Async ve test bağlamlarında aynı işlem `replace_root_view` yardımcısı üzerinden yapılır. Bu kalıp, yeni bir pencere açmadan kök akışını değiştirmek için kullanılır. Yine de değiştirilen köke ait abonelikler ve `Task` sahiplikleri ayrıca düşünülmelidir; aksi halde geride kalan dinleyiciler veya işler bağlamını kaybetmiş halde çalışmaya devam edebilir.

**`with_window` kullanımı.** `with_window(entity_id, ...)` çağrısı, verilen entity'nin en son çizildiği pencereyi bulur. Aynı entity birden fazla pencerede çiziliyorsa bu API bilinçli bir "o anki pencere" kısayolu olarak çalışır; belirli bir pencere hedefleniyorsa o pencerenin `WindowHandle`'ı doğrudan saklanmalıdır.

---
