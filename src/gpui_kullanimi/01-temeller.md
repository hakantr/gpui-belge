# Temeller

---

## Büyük Resim

GPUI, birbirinin üzerine kurulan üç katmandan oluşur. Her katman bir altındakine güvenir ve bir üstündekine daha sade bir arayüz sunar. Böylece uygulama kodunda hangi sorunun nerede çözülmesi gerektiği daha kolay görülür.

1. **Platform katmanı.** İşletim sistemine doğrudan dokunan kısımdır. macOS, Windows, Linux, web ve test ortamları aynı arayüzün arkasına alınır. Uygulama kodu "pencere aç", "girdi al", "ekrana çiz" gibi istekleri ortak bir sözleşmeyle anlatır. Bu sözleşmeyi `Platform` ve `PlatformWindow` trait'leri taşır. Pencere oluşturma, ekran listesi, pano (clipboard), sürükle-bırak, ses ve dosya seçici gibi platforma özgü yetenekler bu iki trait üzerinden açılır. Her hedef için ayrı bir gerçekleştirme vardır, ama üstteki uygulama kodu tek bir API ile konuşur.
2. **Uygulama/durum katmanı.** Uygulamanın yaşam döngüsü ve bellekteki tüm durumu burada tutulur. Çekirdek tipler birbirine sıkıca bağlıdır. `Application` süreç başlangıcını ve olay döngüsünü (event loop) yönetir. `App` global durumun ana erişim noktasıdır. `Context<T>`, belirli bir varlık güncellenirken `App`'in üstüne eklenen daha geniş bir bağlamdır. `Entity<T>` ve `WeakEntity<T>`, heap üzerinde tutulan durum kutularına güçlü ve zayıf erişim sağlar. `Task` arka plan işlerini, `Subscription` ise olay dinleme aboneliklerini drop edildiğinde otomatik temizleyen sahiplik araçlarıdır. `Global`'lar uygulama ömrü boyunca tek nüsha duran kaynaklar içindir. Event sistemi de varlıklar arasında tipli mesajlaşmayı kurar.
3. **Render/element katmanı.** Ekrandaki ağacı üretip çizen kısımdır. Burada iki ana rol vardır. `Render` trait'i, durum sahibi entity'lerin her frame'de yeni bir element ağacı üretmesini sağlar. `RenderOnce` ve `IntoElement` ise yeniden kullanılabilir, durumsuz bileşenleri tanımlar. `Element` trait'i layout + paint sözleşmesinin kendisidir. `div`, `canvas`, `list`, `uniform_list`, `img`, `svg`, `anchored` ve `surface` bu trait'in hazır gerçekleştirmeleridir. Üstüne `Styled` ve `InteractiveElement` fluent API'leri eklenir. Flex/grid stil zinciri, renkler, tıklama, sürükleme, focus ve scroll davranışı bu zincirlerden okunur.

Zed bu üç katmanın üstüne kendi tasarım sistemini koyar. Bunlar GPUI'nin parçası değil, GPUI üzerine yazılmış son kullanıcı bileşenleridir:

- `crates/ui` — Button, Icon, Label, Modal, ContextMenu, Tooltip, Tab, Table, Toggle ve benzeri yeniden kullanılan bileşenleri barındırır. Tutarlı bir görsel dil ve davranış kalıbı sağlar; uygulama içindeki ekranlar bu kitten bileşen alarak yapılır.
- `crates/platform_title_bar` — platforma göre pencere kontrol butonlarını ve başlık çubuğu davranışını çizer. Linux ve Windows tarafında "client-side decoration" gerektiğinde başlık çubuğu da bu paket tarafından üretilir.
- `crates/workspace` — ana çalışma alanını, client-side decoration gölgesini, pencere köşelerindeki resize bölgelerini ve pencere içeriğini tek bir bütün halinde birleştirir. Uygulamanın iskeleti, panellerin yerleşimi ve pencere kromu burada toplanır.

Kısacası alttan yukarıya doğru sıralama "platform → durum → çizim" şeklindedir. Zed bu temelin üstüne kendi UI bileşen setini ekler ve uygulamanın tanıdık görünümünü buradan kurar. İlerleyen bölümler önce bu üç katmanı açar, son bölümler ise Zed'in üst tabakasına döner.

## GPUI Kavram Sözlüğü: Temel Kavramlara Giriş

Bu bölüm bir ezber tablosu değildir. GPUI'yi ilk kez okurken asıl zor olan şey, aynı anda birkaç farklı "dünya" ile karşılaşmaktır: çalışan uygulama, pencere, kalıcı durum, her frame yeniden kurulan element ağacı, async işler ve kullanıcı girdisi. Aşağıdaki sözlük bu dünyaları birbirinden ayırmak için vardır.

En kısa zihinsel model şudur:

1. `Application` programı başlatır.
2. `App` çalışan uygulamanın ana belleği ve servis kapısıdır.
3. `App`, `Entity<T>` adı verilen kalıcı durum nesneleri oluşturur.
4. Bir pencerenin root'u genellikle `Render` implement eden bir `Entity<V>` olur.
5. `Render::render`, o anki state'i geçici bir element ağacına çevirir.
6. `Element` ağacı layout, prepaint ve paint aşamalarından geçerek ekrana çizilir.

Yani GPUI'de ekran doğrudan "nesnelerden" oluşmaz. Kalıcı olan çoğunlukla `Entity<T>` içindeki state'tir; ekrandaki element ağacı ise her render'da yeniden üretilen geçici bir tariftir. Bu ayrımı erken oturtmak, sonraki bölümlerdeki neredeyse tüm API kararlarını anlamayı kolaylaştırır.

### Uygulama ve Durum

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| `Application` | Programın dış kabı | `main` tarafında platformu, asset kaynağını ve event loop'u kurar. Uygulama hazır olduğunda `run` callback'i içinde size `&mut App` verir. | Günlük UI kodunda çok sık görünmez; daha çok başlangıç ve konfigürasyon tipidir. |
| `App` | Çalışan uygulamanın merkezi | Global state'e, pencerelere, entity haritasına, keymap'e, executor'lara, platform servislerine ve asset sistemine erişim sağlar. | Kodda çoğu zaman `cx` adıyla görülür. Her `cx` aynı şey değildir; bazen `App`, bazen `Context<T>` olabilir. |
| `Global` | App ömrü boyunca tek nüsha kaynak | Tema store'u, ayar store'u veya uygulama geneli servisler gibi her yerden okunması gereken state için kullanılır. | Bir view'a ait state'i `Global` yapmak genellikle yanlıştır; `Global` gerçekten uygulama geneli olmalıdır. |
| `Entity<T>` | Tipli, kalıcı state kutusu | `T` değerini GPUI'nin entity haritasında tutar. Okuma ve güncelleme `entity.read(cx)` ve `entity.update(cx, ...)` üzerinden yapılır. | `Entity<T>` ekrandaki element değildir; ekrana ne çizileceğini belirleyen state'i tutar. |
| `WeakEntity<T>` | Entity'yi hayatta tutmayan referans | Async işler, abonelikler veya karşılıklı referanslarda sahiplik döngüsü kurmadan entity'ye dönmeyi sağlar. | `upgrade` veya `update` başarısız olabilir; çünkü entity kapanmış veya drop edilmiş olabilir. |
| `Context<T>` | Bir entity güncellenirken kullanılan bağlam | `Entity<T>` update/render edilirken `App` yeteneklerine ek olarak `cx.notify()`, `cx.emit(...)`, `cx.observe(...)`, `cx.subscribe(...)`, `cx.spawn(...)` gibi entity'ye özel metotları açar. | State render çıktısını etkiliyorsa update sonunda çoğunlukla `cx.notify()` gerekir. |
| `EventEmitter<E>` | Entity'nin tipli olay yayınlaması | Bir entity'nin dışarıya `E` tipinde olaylar yayabileceğini belirtir. Başka kodlar bu olaylara subscribe olabilir. | Bu, DOM event sistemi değildir; entity'ler arası tipli mesajlaşmadır. |
| `Subscription` | Abonelik sahipliği | Observe, subscribe veya listener kayıtlarının yaşam süresini taşır. `Subscription` drop edilince abonelik de kalkar. | Struct alanında tutulmazsa abonelik hemen düşer ve dinleyici çalışmaz. |
| `Task<T>` | İptal edilebilir async iş | Foreground/background executor'da çalışan future'ı temsil eder. | `Task` drop edilirse iş iptal edilir. Uzun sürecekse alan olarak saklanır ya da bilinçli şekilde detach edilir. |
| `AsyncApp` | `await` sonrasına taşınabilen app bağlamı | Async blok içinde `App`'e güvenli şekilde geri dönmeyi sağlar. | `await` sırasında pencere veya entity kapanmış olabilir; bu yüzden async bağlamdaki erişimler çoğu zaman hata döndürebilir. |

### Pencere ve Kullanıcı Girdisi

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| `Window` | Tek pencerenin canlı bağlamı | Focus, cursor, bounds, IME, prompt, hitbox, scroll, action dispatch, refresh ve düşük seviyeli paint işlemlerini yönetir. | `App` uygulama geneline bakar; `Window` yalnızca o pencereye ait bilgiyi taşır. |
| `WindowHandle<V>` | Root view tipini bilen pencere referansı | Açılmış pencerenin root entity'sini tipli şekilde okumak veya güncellemek için kullanılır. | Handle pencereyi temsil eder; doğrudan çizim yapmaz. Çizim yine root view'un `Render` çıktısından gelir. |
| `FocusHandle` | Focus kimliği | Bir view veya element grubunun klavye focus'una katılmasını sağlar. `track_focus` ve tab navigasyonu bunun etrafında çalışır. | Focus state'i element ağacı her frame yenilense bile bu handle sayesinde izlenebilir. |
| `Hitbox` | Mouse ile test edilen alan | Prepaint aşamasında kaydedilen dikdörtgen veya bölge üzerinden hover, click, drag gibi davranışların hedefini belirler. | Görsel olarak çizilmiş olmak tek başına tıklanabilir olmak demek değildir; hitbox gerekir. |
| `ScrollHandle` | Paylaşılan scroll durumu | Bir scroll alanının pozisyonunu ve scroll davranışını frame'ler arasında korur. | Scroll state'i kaybolmasın istiyorsanız stabil element id'leri ve handle sahipliği önemlidir. |
| `Action` | Komut mesajı | Menü, keybinding, komut paleti veya dispatch tree üzerinden taşınan kullanıcı niyetini temsil eder. | Action "şu tuşa basıldı" değil, "şu komut istendi" bilgisidir. |
| `Keymap` | Kısayol eşleme tablosu | Tuş kombinasyonlarını aktif bağlama göre action'lara bağlar. | Aynı tuş farklı key context'lerde farklı action'a gidebilir. Bu yüzden focus ve key context birlikte düşünülür. |

### Render ve Element Modeli

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| View | Render edilebilen kalıcı state | GPUI'de "view" çoğu zaman `Render` implement eden ve `Entity<V>` içinde tutulan bir Rust tipidir. | View ayrı bir widget sınıfı değildir; state ile render sözleşmesinin birleşimidir. |
| Root view | Pencerenin en üst view'u | Bir pencerenin render ağacı root view'dan başlar. Pencere açılırken bu root genellikle `Entity<V>` olarak oluşturulur. | Pencerenin sahibi root view'dur; altındaki elementler her frame yeniden kurulur. |
| `Render` | Stateful view sözleşmesi | Bir `Entity<V>`'nin ekrana nasıl görüneceğini üretir. `render(&mut self, window, cx)` her render döngüsünde element ağacı döndürür. | `Render` implement eden şey genellikle kalıcı view state'idir. |
| `RenderOnce` | Tek seferlik bileşen tarifi | State'i kendi içinde kalıcı olmayan, veriden element üreten küçük bileşenler için kullanılır. Zed UI bileşenlerinde sık görülür. | `self` tüketilir; kalıcı state tutmak için değil, tekrar kullanılabilir element tarifi yazmak içindir. |
| `IntoElement` | Element'e dönüşebilme | Bir değerin GPUI element ağacına katılabileceğini söyler. `String`, `div()`, Zed UI bileşenleri veya özel bileşenler bu yolla child olabilir. | Çoğu API `impl IntoElement` alır; bu yüzden farklı görünen birçok şey aynı `.child(...)` çağrısına girebilir. |
| `Element` | Layout + paint işi yapan node | `request_layout`, `prepaint` ve `paint` aşamalarını tanımlar. `div`, `canvas`, `list`, `img`, `svg` gibi hazır elementler bunun implementasyonlarıdır. | Element ağacı kalıcı değildir; frame sonunda düşer ve sonraki render'da yeniden kurulur. |
| `div()` | Temel container | Flex/grid layout, spacing, border, background, child ekleme ve interaktif davranışların çoğu burada başlar. | Web'deki `div` gibi düşünmek yardımcıdır, ama GPUI'nin Rust trait zinciriyle çalışır. |
| `Styled` | Stil zinciri | `.flex()`, `.p_2()`, `.bg(...)`, `.text_color(...)`, `.rounded_sm()` gibi fluent style metotlarını açar. | Stil metotları element tarifi oluşturur; kalıcı state saklamaz. |
| `InteractiveElement` | Etkileşim zinciri | `.on_click(...)`, `.on_mouse_down(...)`, `.on_action(...)`, `.track_focus(...)`, `.key_context(...)` gibi kullanıcı girdisi metotlarını açar. | Event dinlemek için element'in ilgili hitbox/focus/dispatch bağlamına doğru bağlanması gerekir. |
| `Animation` | Zaman tabanlı geçiş | Süre ve easing bilgisiyle değerleri frame'ler arasında interpolate eder. | Animasyonun devam etmesi için pencerenin yeni frame istemesi gerekir. |

### Görsel Veri, Ölçü ve Asset

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| `Pixels` | Mantıksal piksel | Boyut, konum, padding ve bounds değerlerinde kullanılan ana ölçü birimidir. | Fiziksel ekran pikseliyle bire bir aynı olmak zorunda değildir; scale factor devrededir. |
| `Hsla` / `Rgba` | Renk tipleri | UI renklerini HSLA veya RGBA uzayında taşır. | Zed tarafında çoğu renk doğrudan sabit değil, tema üzerinden gelir. |
| `Background` | Dolgu tanımı | Düz renk, gradient veya pattern gibi arka plan dolgularını temsil eder. | Renk ile background aynı şey değildir; background daha geniş bir dolgu tarifidir. |
| `AssetSource` | Asset byte kaynağı | SVG, image, font veya paketlenmiş dosya gibi varlıkların nereden okunacağını uygulamaya söyler. | Başlangıçta `Application` üzerinde kurulur; elementler asset isterken bu kaynağa dayanır. |

### Hangi Kavramı Ne Zaman Aramalıyım?

- "Bu veri ekranda değişince yeniden çizilsin" diyorsanız `Entity<T>`, `Context<T>` ve `cx.notify()` üçlüsüne bakın.
- "Bu iş bir pencerenin focus'u, cursor'u, bounds'u veya çizim fazı ile ilgili" diyorsanız `Window` tarafındasınız.
- "Bu şey ekranda nasıl görünüyor?" sorusu `Render`, `RenderOnce`, `IntoElement`, `Element` ve `Styled` zincirine çıkar.
- "Kullanıcı bir komut verdi" diyorsanız `Action`, `Keymap`, focus ve dispatch context'i birlikte değerlendirilir.
- "Async iş bitince hâlâ aynı view var mı?" sorusu `Task<T>`, `WeakEntity<T>` ve `AsyncApp` ile ilgilidir.
- "Bu state her yerden okunmalı mı, yoksa tek view'a mı ait?" ayrımı `Global` ile `Entity<T>` arasındaki ana seçimdir.

Zed'in `crates/ui` içindeki `Button`, `Icon`, `Label`, `Modal`, `Tooltip` gibi bileşenleri bu çekirdek kavramların üstüne kuruludur. GPUI size state, pencere, element, input ve çizim altyapısını verir; Zed UI ise bu altyapıyı kullanarak ürün içinde tekrar edilen hazır bileşenleri sağlar.

---
