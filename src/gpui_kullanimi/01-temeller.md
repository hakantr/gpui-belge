# Temeller

---

## Büyük Resim

GPUI, birbirinin üzerine kurulan üç katmandan oluşur. Her katman bir altındakine güvenir ve bir üstündekine daha sade bir arayüz sunar. Böylece uygulama kodunda hangi sorunun nerede çözülmesi gerektiği daha kolay görülür.

1. **Platform katmanı.** İşletim sistemine doğrudan dokunan kısımdır. macOS, Windows, Linux, web ve test ortamları aynı arayüzün arkasına alınır. Uygulama kodu "pencere aç", "girdi al", "ekrana çiz" gibi istekleri ortak bir sözleşmeyle anlatır. Bu sözleşmeyi `Platform` ve `PlatformWindow` trait'leri taşır. Pencere oluşturma, ekran listesi, pano (clipboard), sürükle-bırak, ses ve dosya seçici gibi platforma özgü yetenekler bu iki trait üzerinden açılır. Her hedef için ayrı bir gerçekleştirme vardır, ama üstteki uygulama kodu tek bir API ile konuşur.
2. **Uygulama/durum katmanı.** Uygulamanın yaşam döngüsü ve bellekteki tüm durumu burada tutulur. Çekirdek tipler birbirine sıkıca bağlıdır. `Application` süreç başlangıcını ve olay döngüsünü (event loop) yönetir. `App` uygulama genelindeki duruma erişilen ana kapıdır. `Context<T>`, belirli bir varlık güncellenirken `App`'in üstüne eklenen daha geniş bir bağlamdır. `Entity<T>` ve `WeakEntity<T>`, heap üzerinde tutulan durum kutularına güçlü ve zayıf erişim sağlar. `Task` arka plan işlerini, `Subscription` ise olay dinleme aboneliklerini değer elden çıktığında otomatik temizleyen sahiplik araçlarıdır. `Global`'lar uygulama ömrü boyunca tek nüsha duran kaynaklar içindir. Event sistemi de varlıklar arasında tipli mesajlaşmayı kurar.
3. **Render/element katmanı.** Ekrandaki ağacı üretip çizen kısımdır. Burada iki ana rol vardır. `Render` trait'i, kendi verisini taşıyan entity'lerin her ekran karesinde yeni bir element ağacı üretmesini sağlar. `RenderOnce` ve `IntoElement` ise yeniden kullanılabilir, kendi kalıcı verisini taşımayan bileşenleri tanımlar. `Element` trait'i yerleşim + çizim sözleşmesinin kendisidir. `div`, `canvas`, `list`, `uniform_list`, `img`, `svg`, `anchored` ve `surface` bu trait'in hazır gerçekleştirmeleridir. Üstüne `Styled` ve `InteractiveElement` zincirleri eklenir. Flex/grid stil zinciri, renkler, tıklama, sürükleme, klavye odağı ve scroll davranışı bu zincirlerden okunur.

Zed bu üç katmanın üstüne kendi tasarım sistemini koyar. Bunlar GPUI'nin parçası değil, GPUI üzerine yazılmış son kullanıcı bileşenleridir:

- `crates/ui` — Button, Icon, Label, Modal, ContextMenu, Tooltip, Tab, Table, Toggle ve benzeri yeniden kullanılan bileşenleri barındırır. Tutarlı bir görsel dil ve davranış kalıbı sağlar; uygulama içindeki ekranlar bu kitten bileşen alarak yapılır.
- `crates/platform_title_bar` — platforma göre pencere kontrol butonlarını ve başlık çubuğu davranışını çizer. Linux ve Windows tarafında "client-side decoration" gerektiğinde başlık çubuğu da bu paket tarafından üretilir.
- `crates/workspace` — ana çalışma alanını, client-side decoration gölgesini, pencere köşelerindeki resize bölgelerini ve pencere içeriğini tek bir bütün halinde birleştirir. Uygulamanın iskeleti, panellerin yerleşimi ve pencere kromu burada toplanır.

Kısacası alttan yukarıya doğru sıralama "platform → durum → çizim" şeklindedir. Zed bu temelin üstüne kendi UI bileşen setini ekler ve uygulamanın tanıdık görünümünü buradan kurar. İlerleyen bölümler önce bu üç katmanı açar, son bölümler ise Zed'in üst tabakasına döner.

## GPUI Kavram Sözlüğü: Temel Kavramlara Giriş

Bu bölüm bir ezber tablosu değildir. GPUI'yi ilk kez okurken asıl zor olan şey, aynı anda birkaç farklı "dünya" ile karşılaşmaktır: çalışan uygulama, pencere, bellekte tutulan veri, her ekran karesinde yeniden kurulan element ağacı, async işler ve kullanıcı girdisi. Aşağıdaki sözlük bu dünyaları birbirinden ayırmak için vardır.

En kısa zihinsel model şudur:

1. `Application` programı başlatır.
2. `App` çalışan uygulamanın ana belleği ve servis kapısıdır.
3. `App`, `Entity<T>` adı verilen kalıcı veri nesneleri oluşturur.
4. Bir pencerenin başlangıç view'u, yani root view, genellikle `Render` trait'ini uygulayan bir `Entity<V>` olur.
5. `Render::render`, o anki veriyi geçici bir element ağacına çevirir.
6. `Element` ağacı yerleşim, çizim hazırlığı ve çizim aşamalarından geçerek ekrana basılır.

Yani GPUI'de ekranda gördüğünüz şeyler doğrudan bellekte duran nesneler değildir. Bellekte kalıcı olan çoğunlukla `Entity<T>` içindeki veridir; ekrandaki element ağacı ise her render'da yeniden üretilen geçici bir tariftir. Bu ayrımı erken oturtmak, sonraki bölümlerdeki neredeyse tüm API kararlarını anlamayı kolaylaştırır.

### Uygulama ve Durum

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| `Application` | Programın dış kabı | `main` tarafında platformu, asset kaynağını ve event loop'u kurar. Uygulama hazır olduğunda `run` callback'i içinde size `&mut App` verir. | Günlük UI kodunda çok sık görünmez; daha çok uygulamanın başlangıç ayarlarında görülür. |
| `App` | Çalışan uygulamanın merkezi | Uygulama genelindeki verilere, pencerelere, entity listesine, kısayol tablosuna, async çalıştırıcılara, platform servislerine ve asset sistemine erişim sağlar. | Kodda çoğu zaman `cx` adıyla görülür. Her `cx` aynı şey değildir; bazen yalnızca `App`, bazen de bir entity'ye bağlı `Context<T>` olabilir. |
| `Global` | Her yerden erişilen ortak veri | Tema, ayarlar, uygulama oturumu veya tüm pencerelerin paylaşması gereken servisler gibi uygulama açık kaldığı sürece tek kopya durması gereken veriler için kullanılır. | Sadece tek bir panelin kullandığı arama metni, seçili satır, açık/kapalı durumu gibi bilgiler burada tutulmaz; bunlar o paneli yöneten Rust struct'ının alanlarında tutulur. Bu struct genellikle `Entity<T>` içinde yaşar. |
| `Entity<T>` | Tipli, kalıcı veri kaydı | `T` değerini GPUI'nin entity listesinde tutar. Okuma ve güncelleme `entity.read(cx)` ve `entity.update(cx, ...)` üzerinden yapılır. | `Entity<T>` ekrandaki kutu veya buton değildir; ekrana ne çizileceğini belirleyen veriyi tutar. |
| `WeakEntity<T>` | Entity'yi hayatta tutmayan referans | Async işler, abonelikler veya iki nesnenin birbirini işaret ettiği durumlarda entity'ye geri dönmek için kullanılır; ama entity'yi tek başına canlı tutmaz. | `upgrade` veya `update` başarısız olabilir; çünkü kullanıcı ilgili pencereyi kapatmış ve entity artık silinmiş olabilir. |
| `Context<T>` | Bir entity üzerinde çalışırken gelen bağlam | `Entity<T>` güncellenirken veya render edilirken `App` yeteneklerine ek olarak `cx.notify()`, `cx.emit(...)`, `cx.observe(...)`, `cx.subscribe(...)`, `cx.spawn(...)` gibi o entity'ye özel metotları açar. | Değişen veri ekrandaki sonucu değiştiriyorsa update sonunda çoğunlukla `cx.notify()` çağrılır. |
| `EventEmitter<E>` | Entity'nin olay duyurması | Bir entity'nin dışarıya `E` tipinde olaylar yayabileceğini belirtir. Başka kodlar bu olayları dinleyebilir. | Bu, tarayıcıdaki DOM event sistemi gibi element tıklaması değildir; entity'ler arasında tipli mesajlaşmadır. |
| `Subscription` | Dinleyici kaydının ömrü | Observe, subscribe veya listener kayıtlarının ne kadar süre yaşayacağını taşır. `Subscription` elden çıkınca dinleme de biter. | Oluşturduğunuz `Subscription`'ı bir değişkende veya entity alanında saklamazsanız dinleyici hemen kapanır. |
| `Task<T>` | İptal edilebilir async iş | Foreground/background executor'da çalışan future'ı temsil eder. | `Task` değeri elden çıkarsa iş iptal edilir. İşin devam etmesi gerekiyorsa task bir alanda saklanır ya da bilinçli şekilde bağımsız bırakılır. |
| `AsyncApp` | `await` sonrasına taşınabilen app bağlamı | Async blok içinde `App`'e güvenli şekilde geri dönmeyi sağlar. | `await` sırasında pencere veya entity kapanmış olabilir; bu yüzden async bağlamdaki erişimler çoğu zaman hata döndürebilir. |

### Pencere ve Kullanıcı Girdisi

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| `Window` | Tek pencerenin canlı bağlamı | Klavye odağı, imleç, pencere boyutu, IME, prompt, tıklama alanları, scroll, komut yönlendirme, yenileme ve düşük seviyeli çizim işlemlerini yönetir. | `App` uygulama geneline bakar; `Window` yalnızca o pencereye ait bilgiyi taşır. |
| `WindowHandle<V>` | Pencereyi sonradan bulmaya yarayan referans | Açılmış pencerenin en üst view'unu doğru Rust tipiyle okumak veya güncellemek için kullanılır. | Bu değer pencereye ulaşma yoludur; kendi başına çizim yapmaz. Çizim yine root view'un `Render` çıktısından gelir. |
| `FocusHandle` | Klavye odağı kimliği | Bir view veya element grubunun klavye odağına katılmasını sağlar. `track_focus` ve tab navigasyonu bunun etrafında çalışır. | Element ağacı her render'da yeniden kurulur; hangi parçanın odakta olduğu bu referans sayesinde takip edilir. |
| `Hitbox` | Mouse ile test edilen alan | Prepaint aşamasında kaydedilen dikdörtgen veya bölge üzerinden hover, click, drag gibi davranışların hedefini belirler. | Görsel olarak çizilmiş olmak tek başına tıklanabilir olmak demek değildir; hitbox gerekir. |
| `ScrollHandle` | Scroll pozisyonunu tutan referans | Bir scroll alanının konumunu ve scroll davranışını ekran kareleri arasında korur. | Scroll konumu kaybolmasın istiyorsanız aynı alan için stabil element id'si kullanılır ve `ScrollHandle` uygun yerde saklanır. |
| `Action` | Kullanıcı komutu | Menü, kısayol veya komut paleti üzerinden gelen "kaydet", "sekmeyi kapat", "satırı seç" gibi niyeti temsil eder. | Action "şu tuşa basıldı" değil, "şu komut istendi" bilgisidir. |
| `Keymap` | Kısayol eşleme tablosu | Tuş kombinasyonlarını aktif bağlama göre action'lara bağlar. | Aynı tuş editörde başka, terminalde başka komut çalıştırabilir. Bu yüzden klavye odağı ve `key_context` birlikte düşünülür. |

### Render ve Element Modeli

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| View | Ekran parçasını yöneten Rust tipi | GPUI'de "view" çoğu zaman `Render` trait'ini uygulayan ve `Entity<V>` içinde tutulan bir Rust tipidir. Örneğin bir panelin seçili satırı veya açık menüsü bu tipin alanlarında durabilir. | View ayrı bir widget sınıfı değildir; veriyi tutan Rust tipi ile ekrana çizme metodunun birleşimidir. |
| Root view | Pencerenin en üst view'u | Bir pencerenin render ağacı root view'dan başlar. Pencere açılırken bu root genellikle `Entity<V>` olarak oluşturulur. | Pencerenin ana içeriği root view'dur; onun altında üretilen elementler her render'da yeniden kurulur. |
| `Render` | View'u ekrana çeviren metot sözleşmesi | Bir `Entity<V>`'nin ekrana nasıl görüneceğini üretir. `render(&mut self, window, cx)` her render döngüsünde element ağacı döndürür. | `Render` trait'ini uygulayan tip genellikle kendi verisini alanlarında tutar ve o veriye göre element üretir. |
| `RenderOnce` | Tek seferlik bileşen tarifi | Kendi kalıcı verisini tutmayan, eldeki veriden element üreten küçük bileşenler için kullanılır. Zed UI bileşenlerinde sık görülür. | `self` tüketilir; seçili satır, açık menü gibi kalıcı bilgileri tutmak için değil, tekrar kullanılabilir element tarifi yazmak içindir. |
| `IntoElement` | Element'e dönüşebilme | Bir değerin GPUI element ağacına katılabileceğini söyler. `String`, `div()`, Zed UI bileşenleri veya özel bileşenler bu yolla alt öğe olabilir. | Çoğu API `impl IntoElement` alır; bu yüzden farklı görünen birçok şey aynı `.child(...)` çağrısına girebilir. |
| `Element` | Yerleşim ve çizim yapan düğüm | `request_layout`, `prepaint` ve `paint` aşamalarını tanımlar. `div`, `canvas`, `list`, `img`, `svg` gibi hazır elementler bunun implementasyonlarıdır. | Element ağacı kalıcı değildir; ekran karesi sonunda düşer ve sonraki render'da yeniden kurulur. |
| `div()` | Temel kapsayıcı | Flex/grid yerleşimi, boşluk, kenarlık, arka plan, alt öğe ekleme ve interaktif davranışların çoğu burada başlar. | Web'deki `div` gibi düşünmek yardımcıdır, ama GPUI'nin Rust trait zinciriyle çalışır. |
| `Styled` | Stil zinciri | `.flex()`, `.p_2()`, `.bg(...)`, `.text_color(...)`, `.rounded_sm()` gibi stil metotlarını açar. | Stil metotları element tarifi oluşturur; seçili değer veya açık/kapalı durumu gibi kalıcı veri saklamaz. |
| `InteractiveElement` | Etkileşim zinciri | `.on_click(...)`, `.on_mouse_down(...)`, `.on_action(...)`, `.track_focus(...)`, `.key_context(...)` gibi kullanıcı girdisi metotlarını açar. | Bir elementin tıklama, klavye veya komut alabilmesi için ilgili hitbox, focus veya action bağlantısının kurulması gerekir. |
| `Animation` | Zaman tabanlı geçiş | Süre ve easing bilgisiyle değerleri ekran kareleri arasında yumuşak şekilde değiştirir. | Animasyonun devam etmesi için pencerenin yeni ekran karesi istemesi gerekir. |

### Görsel Veri, Ölçü ve Asset

| Kavram | Basit karşılık | Ne işe yarar? | İlk okurken dikkat |
|---|---|---|---|
| `Pixels` | Mantıksal piksel | Boyut, konum, padding ve bounds değerlerinde kullanılan ana ölçü birimidir. | Fiziksel ekran pikseliyle bire bir aynı olmak zorunda değildir; scale factor devrededir. |
| `Hsla` / `Rgba` | Renk tipleri | UI renklerini HSLA veya RGBA uzayında taşır. | Zed tarafında çoğu renk doğrudan sabit değil, tema üzerinden gelir. |
| `Background` | Dolgu tanımı | Düz renk, gradient veya pattern gibi arka plan dolgularını temsil eder. | Renk ile background aynı şey değildir; background daha geniş bir dolgu tarifidir. |
| `AssetSource` | Asset byte kaynağı | SVG, image, font veya paketlenmiş dosya gibi varlıkların nereden okunacağını uygulamaya söyler. | Başlangıçta `Application` üzerinde kurulur; elementler asset isterken bu kaynağa dayanır. |

### Hangi Kavramı Ne Zaman Aramalıyım?

- "Bu veri ekranda değişince görüntü de değişsin" diyorsanız `Entity<T>`, `Context<T>` ve `cx.notify()` üçlüsüne bakın.
- "Bu iş bir pencerenin klavye odağı, imleci, boyutu veya çizim aşaması ile ilgili" diyorsanız `Window` tarafındasınız.
- "Bu şey ekranda nasıl görünüyor?" sorusu `Render`, `RenderOnce`, `IntoElement`, `Element` ve `Styled` zincirine çıkar.
- "Kullanıcı bir komut verdi" diyorsanız `Action`, `Keymap`, klavye odağı ve `key_context` birlikte değerlendirilir.
- "Async iş bitince hâlâ aynı view var mı?" sorusu `Task<T>`, `WeakEntity<T>` ve `AsyncApp` ile ilgilidir.
- "Bu veri bütün uygulamanın ortak bilgisi mi, yoksa yalnızca tek bir ekran parçasının bilgisi mi?" ayrımı `Global` ile `Entity<T>` arasındaki ana seçimdir.

Zed'in `crates/ui` içindeki `Button`, `Icon`, `Label`, `Modal`, `Tooltip` gibi bileşenleri bu çekirdek kavramların üstüne kuruludur. GPUI size veri ve durum, pencere, element, kullanıcı girdisi ve çizim altyapısını verir; Zed UI ise bu altyapıyı kullanarak ürün içinde tekrar edilen hazır bileşenleri sağlar.

---
