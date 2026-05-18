# Pencere Yönetimi

---

## Pencere Oluşturma

GPUI'de pencereyi açan ana API `cx.open_window(options, root_builder)`'dır. İlk parametre, pencerenin başlangıç davranışını anlatan `WindowOptions`; ikincisi ise pencerenin root view'unu kuran closure'dır. Tipik kullanım şu kalıbı izler:

```rust
let handle = cx.open_window(
    WindowOptions {
        window_bounds: Some(WindowBounds::centered(size(px(900.), px(700.)), cx)),
        titlebar: Some(TitlebarOptions {
            title: Some("My Window".into()),
            appears_transparent: true,
            traffic_light_position: Some(point(px(9.), px(9.))),
        }),
        focus: true,
        show: true,
        kind: WindowKind::Normal,
        is_movable: true,
        is_resizable: true,
        is_minimizable: true,
        window_min_size: Some(size(px(360.), px(240.))),
        window_background: cx.theme().window_background_appearance(),
        window_decorations: Some(gpui::WindowDecorations::Client),
        app_id: Some(ReleaseChannel::global(cx).app_id().to_owned()),
        ..Default::default()
    },
    |window, cx| {
        window.activate_window();
        cx.new(|cx| MyRootView::new(window, cx))
    },
)?;
```

**`WindowOptions` alanları.** Aşağıdaki alanlar pencerenin oluşumunda sorumluluğu olan başlıca parametreleri tanımlar:

- `window_bounds`: `None` verilirse GPUI, ekran için varsayılan sınırları seçer. `Some` ile gelen değer `Windowed`, `Maximized` veya `Fullscreen` başlangıcını belirler. Varsayılan seçilirken baz alınan boyutlar `gpui::DEFAULT_WINDOW_SIZE: Size<Pixels>` (1536×1095, ana Zed penceresi için) ve `gpui::DEFAULT_ADDITIONAL_WINDOW_SIZE` (900×750, 6:5 oranında settings veya rules library benzeri ek pencereler için) sabitleridir (`window.rs:70,74`); kendi varsayılan boyutunu ayrıca ezmek gerekmiyorsa bu değerlere güvenilebilir.
- `titlebar`: `Some(TitlebarOptions)`, sistem başlık çubuğu ayarı için kullanılır. `None` verildiğinde özel başlık çubuğu yolu açılır.
- `focus`: pencere oluşturulduğu anda klavye odağını alıp almayacağını belirler.
- `show`: pencerenin hemen gösterilip gösterilmeyeceğini kontrol eder. Zed ana pencereleri başlangıçta `show: false`, `focus: false` ile açar ve hazır olduğunda gösterir.
- `kind`: `Normal`, `PopUp`, `Floating`, `Dialog`; Linux Wayland özelliğiyle birlikte `LayerShell` de mevcuttur.
- `is_movable`, `is_resizable`, `is_minimizable`: platform seviyesindeki pencere kabiliyetleridir.
- `display_id`: belirli bir monitörü hedefler.
- `window_background`: `Opaque`, `Transparent`, `Blurred` değerleri; Windows için ayrıca `MicaBackdrop` ve `MicaAltBackdrop` seçenekleri de vardır.
- `app_id`: Linux masaüstlerinde uygulama gruplandırması ve görev çubuğu davranışı için kullanılır.
- `window_min_size`: minimum içerik boyutu.
- `window_decorations`: `Server` veya `Client` seçimini taşır. Linux'ta kritik bir alandır; macOS ve Windows tarafında ise pratikte `TitlebarOptions` daha belirleyicidir.
- `icon`: X11 üzerinde pencere ikonu için kullanılır.
- `tabbing_identifier`: macOS yerel pencere sekmesi gruplaması için.

`Window::new` çağrısı, GPUI platform penceresini açtıktan sonra şu sırayı izler:

1. `platform_window.request_decorations(...)` çağrılır.
2. `platform_window.set_background_appearance(window_background)` çağrılır.
3. Pencere sınırları `Fullscreen` ise tam ekran, `Maximized` ise yakınlaştırma uygulanır.
4. Platform geri çağrıları bağlanır.
5. İlk render gerçekleştirilir.

## Zed'de Ana Pencere Nasıl Açılır?

Zed'in ana pencere açma akışı `crates/zed/src/zed.rs::build_window_options` fonksiyonunda toplanır. Yeni bir workspace penceresi açılacağında bu fonksiyon tercih edilir:

```rust
let options = zed::build_window_options(display_uuid, cx);
let window = cx.open_window(options, |window, cx| {
    cx.new(|cx| Workspace::new(/* ... */, window, cx))
})?;
```

Fonksiyon kendi içinde şu işleri sırayla yapar:

- `display_uuid` ile uygun ekranı bulur.
- `ZED_WINDOW_DECORATIONS=server|client` ortam değişkeniyle üzerine yazmayı okur.
- Aksi durumda `WorkspaceSettings::window_decorations` ayarını kullanır.
- `TitlebarOptions { appears_transparent: true, traffic_light_position: (9,9) }` kurar.
- `focus: false`, `show: false`, `kind: Normal` olarak ayarlar.
- `window_background` değerini aktif temadan alır.
- Linux/FreeBSD'de uygulama ikonunu ekler.
- macOS yerel sekmeleme istendiğinde `tabbing_identifier: Some("zed")` verir.

Modal veya About benzeri küçük pencerelerde bu fonksiyonu kullanmak yerine doğrudan `WindowOptions` kurmak daha yaygın bir desendir. Örneğin `crates/zed/src/zed.rs::about` şu seçenekleri kullanır:

- Merkezlenmiş sınırlar
- `appears_transparent: true`
- `is_resizable: false`
- `is_minimizable: false`
- `kind: Normal`

## Ekran ve Çoklu Monitör

Birden fazla ekran olduğunda hedef ekran, `cx.displays()` listesi üzerinden bulunur. Liste her ekran için kimlik, sınırlar ve UUID bilgisini sağlar:

```rust
for display in cx.displays() {
    let id = display.id();
    let bounds = display.bounds();
    let visible = display.visible_bounds();
    let uuid = display.uuid().ok();
}
```

Belirli bir ekrana pencere açmak için seçilen ekranın id'si seçeneklere verilir:

```rust
WindowOptions {
    display_id: Some(display.id()),
    window_bounds: Some(WindowBounds::Windowed(bounds)),
    ..Default::default()
}
```

`Bounds` her zaman ekran koordinatlarında ifade edilir. `WindowBounds::centered(size, cx)` çağrısı, ana ya da varsayılan ekran üzerinde merkezleme yapar. Elle konumlandırma gerektiğinde `Bounds::new(origin, size)` kullanılır.

## WindowKind Davranışı

Pencerenin rolü `WindowKind` ile belirlenir; bu seçim pencerenin odak politikasını, z-order davranışını ve süslemesini doğrudan etkiler:

- `Normal`: ana uygulama penceresi.
- `PopUp`: diğer pencerelerin üstünde duran, bildirim ve geçici popup'lar için. Zed bildirim pencerelerinde bu türü kullanır.
- `Floating`: üst pencere üzerinde sabit duran yüzen panel.
- `Dialog`: üst pencerenin etkileşimini engelleyen modal platform penceresi.
- `LayerShell`: Wayland `layer-shell` özelliği aktifken dock, üst katman veya duvar kağıdı benzeri yüzeyler için.

Pop-up ve bildirim pencerelerinde tipik konfigürasyon şuna benzer:

```rust
WindowOptions {
    titlebar: None,
    kind: WindowKind::PopUp,
    focus: false,
    show: true,
    is_movable: false,
    window_background: WindowBackgroundAppearance::Transparent,
    window_decorations: Some(WindowDecorations::Client),
    ..Default::default()
}
```

Zed içindeki örnekler `crates/agent_ui/src/ui/agent_notification.rs` ve `crates/collab_ui/src/collab_ui.rs` dosyalarındadır.

## Başlık Çubuğu ve Pencere Süslemesi

Başlık çubuğu ve pencere süslemesi iki ayrı kavramdır. Karışmaması için ikisinin sorumluluğu ayrı düşünülmelidir:

- `TitlebarOptions`: macOS ve Windows yerel başlık çubuğunun görünümü, başlık metni ve macOS traffic light konumu burada belirlenir.
- `WindowDecorations`: Linux, Wayland ve X11 tarafında süslemenin sunucu tarafında mı (`server-side`) yoksa istemci tarafında mı (`client-side`) olacağını söyler.

GPUI tarafındaki tipler şunlardır:

```rust
pub enum WindowDecorations {
    Server,
    Client,
}

pub enum Decorations {
    Server,
    Client { tiling: Tiling },
}
```

`WindowDecorations`, pencere açılırken istenen kiptir; `Decorations` ise platformun fiili durumudur ve `window.window_decorations()` ile okunur. Compositor sınırları nedeniyle istenen kip her zaman aynen karşılanmayabilir; bu yüzden bu ikisi ayrı tutulur.

Zed ayarı tek bir alan üzerinden ifade edilir:

```json
{
  "window_decorations": "client"
}
```

Ortam değişkeniyle de üzerine yazma mümkündür:

```sh
ZED_WINDOW_DECORATIONS=server
ZED_WINDOW_DECORATIONS=client
```

Zed settings tipi `settings_content::workspace::WindowDecorations` yalnızca `client` ve `server` değerlerini destekler; varsayılan değer `client`'tır.

## Özel Başlık Çubuğu Nasıl Tanımlanır?

Basit bir GPUI uygulamasında yerel başlık çubuğu kapatılır ve özel başlık çubuğu root view içine yerleştirilir:

```rust
cx.open_window(
    WindowOptions {
        titlebar: None,
        ..Default::default()
    },
    |_, cx| cx.new(|_| MyView),
)?;
```

Root view içinde özel başlık çubuğu şu kalıpta çizilir:

```rust
div()
    .flex()
    .flex_col()
    .size_full()
    .child(
        h_flex()
            .window_control_area(WindowControlArea::Drag)
            .h(px(34.))
            .child("Title")
    )
    .child(content)
```

Windows tarafında başlık butonu (`caption button`) bölgeleri için `window_control_area` çağrısı kritik öneme sahiptir; yerel çarpışma testi bu alanlar üzerinden çözülür:

- `WindowControlArea::Drag`: sürüklenebilir başlık alanı.
- `WindowControlArea::Close`: yerel kapatma çarpışma testi alanı.
- `WindowControlArea::Max`: ekranı kaplama veya geri yükleme çarpışma testi alanı.
- `WindowControlArea::Min`: küçültme çarpışma testi alanı.

Zed'de yeni bir workspace benzeri pencere yapıldığında özel başlık çubuğu sıfırdan yazılmaz; bunun yerine hazır `PlatformTitleBar` bileşeni kullanılır:

```rust
let platform_titlebar = cx.new(|cx| PlatformTitleBar::new("my-titlebar", cx));

platform_titlebar.update(cx, |titlebar, _| {
    titlebar.set_children([my_left_or_center_content.into_any_element()]);
});

platform_titlebar.into_any_element()
```

`PlatformTitleBar` hazır olarak şu işleri halleder:

- Linux istemci tarafı süslemesi için sol ve sağ pencere kontrol butonları.
- Windows pencere kontrol butonları.
- macOS traffic light iç boşluğu ve çift tıklama davranışı.
- Linux çift tıklama ile yakınlaştırma veya ekranı kaplama.
- Başlık çubuğu sürükleme alanı.
- Linux'ta sağ tık ile pencere menüsü.
- Yan çubuk açıkken kontrol butonları ile köşe yuvarlamalarının ayarlanması.

Zed'in başlık çubuğu davranışında dikkat çeken iki ayrıntı vardır:

- `TitleBar`, `SkillsFeatureFlag` açıkken `OnboardingBanner` ile "Introducing: Skills" afişini kurar; afiş tıklaması, agent veya skills bilgilendirme modalını açan action'ı yönlendirir. Başlık çubuğu etiketi bu modalın sonucuna göre değişmez.
- `UpdateButton::checking`, `downloading` ve `installing` durumları pasif buton olarak çizilir. Sürüm ipucu metni `"Update to Version: ..."` biçimindedir; SHA tabanlı sürümde kısa SHA yerine tam SHA gösterilir.

## Kontrol Butonları Nasıl Yönetilir?

Pencere kontrol butonları her platformda farklı çizilir; doğru bileşen seçildiği sürece çizim ayrıntıları kullanıcının önüne çıkmaz:

- macOS: yerel traffic light kullanılır; Zed yalnızca iç boşluk ve `traffic_light_position` değerini ayarlar.
- Windows: `platform_title_bar::platforms::platform_windows::WindowsWindowControls`, başlık butonlarını render eder; her buton `WindowControlArea` üzerinden yerel çarpışma testi alanına bağlanır.
- Linux: `platform_title_bar::platforms::platform_linux::LinuxWindowControls`, `WindowButtonLayout` ve `WindowControls` bilgilerine göre kapatma, küçültme ve ekranı kaplama butonlarını çizer.

Sol ve sağ kontrolleri hazır çizmek için iki yardımcı fonksiyon vardır:

```rust
platform_title_bar::render_left_window_controls(
    cx.button_layout(),
    Box::new(workspace::CloseWindow),
    window,
)

platform_title_bar::render_right_window_controls(
    cx.button_layout(),
    Box::new(workspace::CloseWindow),
    window,
)
```

Kapatma butonu doğrudan `window.remove_window()` çağırmaz; bunun yerine Zed kapatma action'ını yönlendirir:

```rust
window.dispatch_action(workspace::CloseWindow.boxed_clone(), cx);
```

Böylece kaydedilmemiş arabellek (`dirty buffer`) kontrolü, kullanıcıya sorma diyaloğu, workspace kapatma mantığı ve kısayol ile aynı akış kullanılır.

**Linux `WindowButtonLayout`.** Linux tarafında buton düzeni esnektir ve kullanıcı tarafında özelleştirilebilir:

- `WindowButton::{Minimize, Maximize, Close}`, sıralı kontrol tipleridir; düzen, sol ve sağ taraf için `Option<WindowButton>` yuva dizileri tutar. Yuva başına sayı `gpui::MAX_BUTTONS_PER_SIDE: usize = 3` (`platform.rs:457`) ile sabittir; `WindowButtonLayout::{left, right}` bu sayıda elemanlı dizilerdir.
- Düzen, platformdan `cx.button_layout()` ile gelir.
- GNOME tarzı `"close,minimize:maximize"` biçimi ayrıştırılabilir.
- Varsayılan Linux yedeği, sağda küçültme, ekranı kaplama, kapatma şeklindedir.
- `TitleBarSettings` içinde kullanıcı tarafından üzerine yazma da yer alır; `TitleBar`, bunu `PlatformTitleBar::set_button_layout` ile aktarır.

## İstemci Tarafı Süslemesi ve Yeniden Boyutlandırma

Zed'in istemci tarafı süsleme (`client-side decoration`) sarmalayıcısı tek bir yardımcı üzerinde toplanmıştır:

```rust
workspace::client_side_decorations(element, window, cx, border_radius_tiling)
```

Bu sarmalayıcının yaptığı işler şunlardır:

- `window.window_decorations()` ile fiili süsleme kipini okur.
- İstemci süslemesi durumunda `window.set_client_inset(theme::CLIENT_SIDE_DECORATION_SHADOW)` çağırır.
- Sunucu süslemesi durumunda inset değerini `0` yapar.
- `window.client_inset()`, platform penceresine son atanan inset değerini okumak için kullanılabilir; bu değer iç boşluk ve gölge hesabıyla uyumlu tutulmalıdır.
- Döşeme (`tiling`) durumuna göre köşe yuvarlamalarını kaldırır.
- Kenarlık ve gölge çizer.
- Kenar ve köşe bölgelerinde imleci yeniden boyutlandırma imlecine çevirir.
- Fare basmada `window.start_window_resize(edge)` çağırır.

Özel bir istemci tarafı süslemesi yapıldığında aynı prensiplerin tekrarlanması gerekir:

1. Fiili kip, `window.window_decorations()` ile okunur.
2. İstemci kipiyse gölge veya görünmez yeniden boyutlandırma alanı kadar `set_client_inset` verilir.
3. Döşeme varsa ilgili kenar ve köşeye yuvarlama, iç boşluk ve gölge verilmez.
4. Yeniden boyutlandırma bölgelerinde `ResizeEdge` hesaplanır.
5. Hareket için başlık çubuğuna `WindowControlArea::Drag` bağlanır ya da Linux ve macOS için `window.start_window_move()` çağrılır.

Linux'ta sunucu tarafı süslemesi her zaman mümkün olmayabilir:

- Wayland'de compositor süsleme protokolü sağlamazsa sunucu isteği istemciye düşürülür.
- X11'de compositor olmadığında istemci tarafı süslemesi sunucuya düşebilir.

Bu nedenle pencere açılırken istenen kip değil, her render'da alınan fiili `window.window_decorations()` sonucu esas alınır.

## Platforma Göre Süsleme Davranışı

#### macOS

- `TitlebarOptions::appears_transparent = true`, stil maskesine `NSFullSizeContentViewWindowMask` ekler.
- `traffic_light_position`, yerel kapatma, küçültme ve yakınlaştırma butonlarının konumunu taşır.
- `titlebar_double_click()`, yerel çift tıklama aksiyonunu uygular.
- `start_window_move()`, yerel `performWindowDragWithEvent` çağırır.
- `tabbing_identifier` verildiğinde yerel pencere sekmesi açılır.
- `WindowDecorations` pratikte platformda işlem yapmaz; macOS için başlık çubuğu davranışını `TitlebarOptions` belirler.

#### Windows

- `TitlebarOptions::appears_transparent`, özel veya tam içerikli başlık çubuğu için kullanılır.
- Başlık butonlarının yerel çarpışma testi davranışı, `WindowControlArea` üzerinden `HTCLOSE`, `HTMAXBUTTON`, `HTMINBUTTON`, `HTCAPTION` değerlerine platform olay katmanında eşlenir.
- `WindowBackgroundAppearance::MicaBackdrop` ve `MicaAltBackdrop` değerleri, DWM arka plan özniteliği ile uygulanır.
- `WindowControls` çizimi, Zed tarafında Windows bileşeni ile yapılır.

#### Linux/FreeBSD - Wayland

- `WindowDecorations::Server`, `xdg-decoration` protokolü ile istenir.
- Compositor sunucu tarafı süslemesi desteklemiyorsa istemci tarafına düşülür.
- `window_controls()`, Wayland yetenek bilgisinden gelir: tam ekran, ekranı kaplama, küçültme, pencere menüsü.
- `show_window_menu`, `start_window_move`, `start_window_resize`, `xdg_toplevel` üzerinden compositor'a devredilir.
- Bulanıklaştırma için compositor `blur_manager` destekliyorsa `Blurred` yüzeye bulanıklık uygulanır.

#### Linux/FreeBSD - X11

- `request_decorations`, `_MOTIF_WM_HINTS` yazar.
- İstemci tarafı süslemesi compositor gerektirir; yoksa sunucu tarafına düşer.
- `show_window_menu`, `_GTK_SHOW_WINDOW_MENU` istemci mesajını gönderir.
- Taşıma veya yeniden boyutlandırma işlemi `_NET_WM_MOVERESIZE` tarzı bir mesajla başlatılır.
- Döşeme, tam ekran ve ekranı kaplama durumları `Decorations::Client { tiling }` sonucunu etkiler.

#### Web/WASM

- Web platformunda yerel pencere süslemesi kavramı bulunmaz.
- `WindowBackgroundAppearance` şu anda web penceresinde opak veya işlem yapmaz kabul edilir.
- Giriş noktasında `gpui_platform::web_init()` çağrılır.

## Bulanıklık, Şeffaflık ve Mica Yönetimi

Pencere arka planının görünümü `WindowBackgroundAppearance` enum'u ile ifade edilir:

```rust
pub enum WindowBackgroundAppearance {
    Opaque,
    Transparent,
    Blurred,
    MicaBackdrop,
    MicaAltBackdrop,
}
```

Zed tema ayarı bu enum'un tamamını değil, yalnızca seçili bir alt kümesini kullanıcı içeriği üzerinden destekler:

```json
{
  "experimental.theme_overrides": {
    "window_background_appearance": "blurred"
  }
}
```

Desteklenen setting değerleri `opaque`, `transparent` ve `blurred`'dir. `MicaBackdrop` ve `MicaAltBackdrop` değerleri GPUI seviyesinde mevcuttur, ancak Zed tema şeması şu anda bunları kullanıcıya ifşa etmez.

**Zed akışı.** Tema değişimleri tüm açık pencerelere yansıtılır:

- Tema rafine edilirken `WindowBackgroundContent` -> `WindowBackgroundAppearance` dönüştürülür.
- Ana pencere açılırken `window_background: cx.theme().window_background_appearance()` verilir.
- Ayarlar veya tema değiştiğinde `crates/zed/src/main.rs`, tüm açık pencerelere `window.set_background_appearance(background_appearance)` çağrısı yapar.
- UI tarafında public yol `ui::theme_is_transparent(cx)`'tir; şeffaf veya bulanıksa `true` döner. Opak arka plan varsayan bileşenler buna göre davranmalıdır.

**Platform davranışı.** Aynı enum değeri her platformda farklı bir mekanizmayla ifade edilir:

- macOS:
  - `Opaque`, pencereyi opak yapar.
  - `Transparent` ve `Blurred` için render aracı şeffaflığı açılır.
  - `Blurred` için `NSVisualEffectView` tabanlı bulanıklık view'u eklenir ya da kaldırılır.
- Windows:
  - `Opaque`: kompozisyon özniteliği kapatılır.
  - `Transparent`: kompozisyon durumu şeffaf olarak işaretlenir.
  - `Blurred`: acrylic veya bulanıklık benzeri kompozisyon özniteliği uygulanır.
  - `MicaBackdrop`: DWM `DWMSBT_MAINWINDOW`.
  - `MicaAltBackdrop`: DWM `DWMSBT_TABBEDWINDOW`.
- Wayland:
  - Compositor bulanıklık protokolünü destekliyorsa `Blurred` yüzeye bulanıklık uygulanır.
  - Aksi durumda bulanıklık isteği gözle görülür bir değişiklik üretmeyebilir.
- X11:
  - Şeffaf veya bulanık render aracı şeffaflığını etkiler; gerçek arka plan bulanıklığı için pencere yöneticisi veya compositor desteği gereklidir.

**Pratik karar tablosu.** Hangi değerin nerede tercih edileceği şu şekilde özetlenebilir:

- Tema ve ana pencere için: `cx.theme().window_background_appearance()`.
- Geçici üst katman ve bildirim için: `Transparent`.
- Windows 11'e özel Mica isteniyorsa: doğrudan `WindowBackgroundAppearance::MicaBackdrop` veya `MicaAltBackdrop` verilir; ancak bu seçim Zed tema ayarına otomatik bağlanmaz.
- Bulanıklık tercih edildiğinde içerikte gerçekten alfa bırakılır; tamamen opak bir kök arka plan, bulanıklığın görünmez olmasına yol açar.

## Pencere Üzerinden Yapılan İşlemler

Pencerenin durumuna ve görünümüne dair sık kullanılan `Window` API'leri şunlardır; her biri ilgili platform çağrısının sade bir kapısıdır:

- `window.bounds()` — genel ekran koordinatlarındaki sınırlar.
- `window.window_bounds()` — tekrar açma ve geri yükleme için `WindowBounds`.
- `window.inner_window_bounds()` — Linux inset hariç sınırlar.
- `window.viewport_size()` — çizilebilir içerik boyutu.
- `window.resize(size)` — içerik boyutunu değiştirir.
- `window.is_fullscreen()`, `window.is_maximized()`
- `window.activate_window()`
- `window.minimize_window()`
- `window.zoom_window()`
- `window.toggle_fullscreen()`
- `window.remove_window()`
- `window.set_window_title(title)`
- `window.set_app_id(app_id)`
- `window.set_background_appearance(appearance)`
- `window.set_window_edited(true/false)` — macOS değiştirildi göstergesi.
- `window.set_document_path(path)` — macOS belge erişilebilirliği ve yolu.
- `window.show_window_menu(position)` — Linux başlık çubuğu bağlam menüsü.
- `window.start_window_move()`, `window.start_window_resize(edge)`
- `window.request_decorations(WindowDecorations::Client/Server)`
- `window.window_decorations()`
- `window.window_controls()`
- `window.prompt(...)`
- `window.play_system_bell()`

macOS yerel pencere sekmesi için ek bir API ailesi vardır:

- `window.tabbed_windows()`
- `window.tab_bar_visible()`
- `window.merge_all_windows()`
- `window.move_tab_to_new_window()`
- `window.toggle_window_tab_overview()`
- `window.set_tabbing_identifier(...)`

## Pencere Sınırlarının Saklanması ve Geri Yüklenmesi

`crates/gpui/src/platform.rs::WindowBounds`, Zed tarafında `crates/workspace/src/persistence/`, `crates/workspace/src/workspace.rs` ve `crates/zed/src/zed.rs`.

`WindowBounds` enum'u pencerenin üç ana durumunu kapsar:

```rust
pub enum WindowBounds {
    Windowed(Bounds<Pixels>),
    Maximized(Bounds<Pixels>),
    Fullscreen(Bounds<Pixels>),
}
```

`Bounds`, her üç durumda da geri yüklemeye hazır koordinatları taşır. `Maximized` ve `Fullscreen` içindeki sınır değeri, ilgili durum kapatıldığında geri dönülecek pencereli (`windowed`) sınırları temsil eder.

**Saklama akışı.** Sınırlar saklanırken tipik kullanım şudur:

```rust
let bounds = window.inner_window_bounds();
serialize(bounds, display_uuid);
```

Zed varsayılan pencere boyutunu saklarken `inner_window_bounds()` kullanır. Workspace serileştirilirken bazı akışlarda `window.window_bounds()` da tercih edilir. İkisi arasındaki fark, dahil edilen platform veya başlık çubuğu dikdörtgeninin farklı olmasından kaynaklanır. Tam ekran ya da ekranı kaplama durumlarında enum içindeki sınırlar, geri yüklenecek pencereli sınırları temsil eder. Ekran UUID'si ayrı saklanır; kullanıcı sonradan monitörü ayırabileceği için bu kimliğin pencere sınırlarından bağımsız tutulması gerekir.

**Geri yükleme akışı.** Workspace açılırken `zed::build_window_options` üstünde şu sıra izlenir:

1. Saklı `display_uuid`, `cx.displays()` içindeki `display.uuid()` değerleriyle eşleştirilir.
2. Ekran bulunmuşsa `options.display_id` ayarlanır ve kayıtlı `WindowBounds` değeri `options.window_bounds`'a yerleştirilir.
3. Workspace'e özel sınırlar bulunmuyorsa varsayılan pencere sınırları okunur.
4. Hiç kayıt yoksa `WindowOptions.window_bounds = None` bırakılır ve GPUI, platformun varsayılan veya kademeli sınırlarını seçer.

Sınır değişimini izlemek için abonelik kullanılır:

```rust
cx.observe_window_bounds(window, |this, window, cx| {
    let bounds = window.inner_window_bounds();
    this.persist_bounds(bounds);
}).detach();
```

Aynı şekilde `cx.observe_window_appearance(window, ...)` açık veya koyu görünüm değişimini, `cx.observe_window_activation(window, ...)` ise ön plan veya arka plan değişimini takip eder.

**Tuzaklar.** Sınırlar tarafında karşılaşılan tipik karışıklıklar şunlardır:

- `window.bounds()` (canlı ekran dikdörtgeni), `window.window_bounds()` ve `window.inner_window_bounds()` farklı olabilir; geri yükleme veya saklama akışında hangi dikdörtgenin beklendiği mevcut Zed çağrı noktasına göre seçilmelidir.
- Ekranı kaplama ve tam ekran enum'larının içindeki `Bounds<Pixels>`, geri yükleme boyutudur; canlı platform sınırları ekranı tamamen kaplasa bile geri yükleme sonrasında bu pencereli sınırlara geri dönülür.
- Ekran UUID'si Linux/Wayland tarafında boş olabilir; bu durumda `display.uuid().ok()` `None` döner ve uygun bir yedek davranış düşünülmelidir.

## Yerel Pencere Sekmeleri ve SystemWindowTabController

macOS yerel pencere sekmeleri, GPUI'de iki katmanlı bir yapı üzerinde durur:

- `WindowOptions::tabbing_identifier` — aynı tanımlayıcıya sahip pencerelerin yerel sekme grubuna girmesini sağlar.
- `SystemWindowTabController` — GPUI `Global`'i olarak yerel sekme gruplarını ve görünürlük durumunu izler.

İlgili `Window` API'leri şunlardır:

- `window.tabbed_windows() -> Option<Vec<SystemWindowTab>>`
- `window.tab_bar_visible() -> bool`
- `window.merge_all_windows()`
- `window.move_tab_to_new_window()`
- `window.toggle_window_tab_overview()`
- `window.set_tabbing_identifier(Some(identifier))`

**Kullanım kararı.** Yerel sekme ile uygulama içi sekme sistemleri farklı kavramlardır:

- Zed workspace sekmesi ve pane sistemi için yerel sekme yerine `workspace::Pane` ve `TabBar` kullanılır.
- İşletim sistemi seviyesinde birden çok üst düzey pencerenin aynı yerel sekme grubuna alınması gerektiğinde `tabbing_identifier` verilir.
- Yerel sekme verisi platformdan gelir; Linux ve Windows üzerinde bu API'lerin bir kısmı işlem yapmayabilir veya `None` dönebilir.

**Tuzaklar.** Yerel sekme kullanırken dikkat edilmesi gerekenler:

- Yerel pencere sekmesi ile Zed pane sekmesi aynı kavram değildir; kalıcılık ve komut yönlendirmesi farklıdır.
- Pencere başlığı değiştiğinde yerel sekme başlığı için `window.set_window_title(...)` ve denetleyici güncellemesi birlikte düşünülmelidir.

## Layer Shell ve Özel Platform Pencereleri

Normal Zed pencereleri `WindowKind::Normal` ile açılır. Linux Wayland özelliği aktifken `WindowKind::LayerShell(LayerShellOptions)`, üst katman, dock veya duvar kağıdı benzeri yüzeyler için kullanılabilir:

```rust
use gpui::layer_shell::*;

WindowOptions {
    titlebar: None,
    window_background: WindowBackgroundAppearance::Transparent,
    kind: WindowKind::LayerShell(LayerShellOptions {
        namespace: "gpui".to_string(),
        layer: Layer::Overlay,
        anchor: Anchor::LEFT | Anchor::RIGHT | Anchor::BOTTOM,
        margin: Some((px(0.), px(0.), px(40.), px(0.))),
        keyboard_interactivity: KeyboardInteractivity::None,
        ..Default::default()
    }),
    ..Default::default()
}
```

Layer shell ayarları, compositor'a yüzeyin nerede ve nasıl davranacağını anlatır:

- `Layer`: `Background`, `Bottom`, `Top`, `Overlay`.
- `layer_shell::Anchor`: bit bayrağı; `TOP`/`BOTTOM`/`LEFT`/`RIGHT` birleştirilir.
- `exclusive_zone`: compositor'ın başka yüzeylerin bu alanı kaplamamasını istemesi için.
- `exclusive_edge`: özel alanın (`exclusive zone`) hangi kenara ait olduğunu belirtir.
- `margin`: CSS sırasıyla üst, sağ, alt, sol.
- `KeyboardInteractivity`: `None`, `Exclusive`, `OnDemand`.

Bu API yalnızca `#[cfg(all(target_os = "linux", feature = "wayland"))]` altında mevcuttur. Compositor protokolü desteklemediğinde arka uç `LayerShellNotSupportedError` döndürür; bu durumda normal uygulama penceresine düşen bir yedek akış planlanmalıdır.

---
