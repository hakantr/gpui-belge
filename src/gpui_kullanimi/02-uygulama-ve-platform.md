# Uygulama ve Platform

---

## Platform Başlatma

Bir GPUI uygulamasının ilk adımı platform seçimidir. Platform, işletim sistemiyle konuşan tarafı temsil eder. Pencere açmak, klavye okumak ve ekrana çizmek gibi işler bu seçilen platform üzerinden yürür. Standart başlangıç şu kalıbı izler:

```rust
use gpui::{App, AppContext as _, WindowOptions, div, prelude::*};
use gpui_platform::application;

struct KokGorunum;

impl Render for KokGorunum {
    fn render(&mut self, _pencere: &mut Window, _baglam: &mut Context<Self>) -> impl IntoElement {
        div().size_full().child("Merhaba")
    }
}

fn main() {
    application().run(|baglam: &mut App| {
        if let Err(hata) = baglam.open_window(WindowOptions::default(), |_, baglam| {
            baglam.new(|_| KokGorunum)
        }) {
            eprintln!("pencere açılamadı: {hata:?}");
        }
    });
}
```

`application()` çağrısı çalıştığı işletim sistemine göre doğru platform gerçekleştirmesini otomatik döndürür. Zed'deki seçim kabaca şöyledir:

- macOS: `gpui_macos::MacPlatform::new(headless)`
- Windows: `gpui_windows::WindowsPlatform::new(headless)`
- Linux/FreeBSD: `gpui_linux::current_platform(headless)`; Wayland veya X11 arka ucu platform crate'i içinde seçilir.
- Web/WASM: `gpui_web::WebPlatform`

Görsel olmayan senaryolar için ayrı bir başlatıcı vardır: `gpui_platform::headless()`, test ve başsız (`headless`) çalıştırma için pencere açmayan bir platform üretir. Test desteği gerektiğinde `gpui_platform::current_headless_renderer()` çağrılır; şu anda yalnızca macOS'ta Metal başsız çizim aracı döner, diğer hedeflerde `None` gelir.

## Application Yaşam Döngüsü ve Platform Olayları

`Application`, GPUI çalışmaya başlamadan önce kullanılan builder katmanıdır. Asset kaynağı, HTTP istemcisi ve çıkış politikası gibi süreç ömrü boyunca geçerli ayarlar burada kurulur. Tipik kurulum şu şekildedir:

```rust
let uygulama = gpui_platform::application()
    .with_assets(Varliklar)
    .with_http_client(http_istemcisi)
    .with_quit_mode(QuitMode::Default);

uygulama.on_open_urls(|urller| {
    // Platform URL açma olayı.
});

uygulama.on_reopen(|baglam| {
    baglam.activate(true);
});

uygulama.run(|baglam| {
    // Genel kurulum, keymap, pencereler.
});
```

`run` çağrısı, kontrolü platformun olay döngüsüne devreder. Bu noktadan sonra uygulama olay tabanlı çalışır.

**Çıkış ve etkinleştirme.** Uygulamanın hangi durumda çıkacağı `QuitMode` ile ayarlanır:

- `QuitMode::Default`: macOS'ta yalnızca açık bir çıkış isteğiyle sonlanır, diğer platformlarda ise son pencere kapandığında otomatik çıkış yapılır.
- `QuitMode::LastWindowClosed`: son pencere kapanır kapanmaz uygulama biter.
- `QuitMode::Explicit`: çıkış yalnızca `App::quit()` çağrılınca olur.
- `cx.on_app_quit(|cx| async { ... })` ile kaydedilen tüm geri çağrılar, uygulama tamamen sonlanmadan önce çalıştırılır. Bu geri çağrılar için ayrılan süre `gpui::SHUTDOWN_TIMEOUT: Duration = 100ms` (`app.rs:71`) sabitiyle belirlenir; bu eşik aşılırsa hâlâ bekleyen future'lar iptal edilir ve platform çıkışı sürdürülür. Bu nedenle uzun kapanış işleri, sonucunu beklemeden bırakılan bir `Task` yerine uygun bir yaşam döngüsü gözlemcisine bağlanmalıdır.
- `cx.activate(ignoring_other_apps)`, `cx.hide()`, `cx.hide_other_apps()`, `cx.unhide_other_apps()` platform genelindeki uygulama durumunu değiştirir.
- `window.activate_window()`, `window.minimize_window()`, `window.toggle_fullscreen()` ise pencere seviyesindeki kontrolleri verir.

**Platform sinyalleri.** Uygulama, işletim sisteminden gelen olayları çeşitli kanallarla dinleyebilir:

- `cx.on_keyboard_layout_change(...)` — klavye düzeni değiştiğinde tetiklenir.
- `cx.keyboard_layout()` ve `cx.keyboard_mapper()` — keystroke'ları action'lara eşlemek için gerekli verileri sağlar.
- `cx.thermal_state()` ve `cx.on_thermal_state_change(...)` — yoğun çizim, dizinleme veya arka plan işlerinin kısıtlama (`throttling`) kararlarında kullanılır.
- `cx.set_cursor_hide_mode(CursorHideMode::...)` — yazım veya action sonrasında imleci gizleme politikasını ayarlar.
- `cx.refresh_windows()` — tüm pencereleri tek bir etki döngüsü (`effect cycle`) içinde yeniden çizmeye zorlar.
- `cx.set_quit_mode(mode)` — çıkış politikasını çalışma zamanında değiştirir; builder tarafındaki `.with_quit_mode(...)` ile aynı alanı besler.
- `cx.on_window_closed(|cx, window_id| ...)` — pencere kapandıktan *sonra* çalışır; bu noktada pencere artık erişilebilir değildir ve geri çağrı yalnızca `WindowId` alır.

**Tuzaklar.** Bu API'lerde sık görülen birkaç yanlış anlama vardır:

- `on_open_urls` geri çağrısı `&mut App` almaz; uygulama verisi gerekiyorsa URL'leri kendi kuyruğunuza veya bir `Global`'e taşıyacak bir köprü kurulmalıdır.
- `on_reopen` özellikle macOS'ta Dock veya uygulama ikonuna tıklamayla yeniden açılma senaryosunda devreye girer; açık pencere yoksa yeni bir workspace açma mantığı burada tetiklenir.
- `refresh_windows()` veriyi değiştirmez, yalnızca yeniden çizim etkisini planlar; yani veri güncellemesi için kullanılmaz, sadece bir yeniden çizim talebidir.

## Platform Servisleri

`App` üzerinden ulaşılan platform servisleri uygulamanın dış dünyaya açılan kapılarıdır. Pencere yönetimi, panoya yazma, kimlik bilgileri, URL açma ve ekran yakalama buradan ilerler. Sarmalayıcılar (wrapper'lar) `crates/gpui/src/app.rs` içinde tanımlıdır; asıl davranış platforma özgü `Platform` trait gerçekleştirmesinde yaşar. Aşağıdaki gruplama, hangi işin nereden çağrılacağını hızlıca bulmak içindir.

- **Uygulama yaşam döngüsü:** `quit`, `restart`, `set_restart_path`, `on_app_quit(|cx| async ...)`, `on_app_restart(|cx| ...)`, `activate`, `hide`, `hide_other_apps`, `unhide_other_apps`.
- **Pencereler:** `windows`, `active_window`, `window_stack`, `refresh_windows`.
- **Ekran:** `displays`, `primary_display`, `find_display`.
- **Görünüm:** `window_appearance`, `button_layout`, `should_auto_hide_scrollbars`, `set_cursor_hide_mode`.
- **Pano:** `read_from_clipboard`, `write_to_clipboard`.
- **Linux primary selection:** `read_from_primary`, `write_to_primary` — X11 ve Wayland'da orta tıklamayla yapıştırma için ayrı bir pano vardır; bu çift o panoyu hedefler.
- **macOS find pasteboard:** `read_from_find_pasteboard`, `write_to_find_pasteboard` — macOS'taki uygulama geneli "son aranan metin" panosunu yönetir.
- **Keychain ve kimlik bilgisi deposu:** `write_credentials(url, username, password)`, `read_credentials(url) -> Task<Result<Option<(String, Vec<u8>)>>>`, `delete_credentials(url)`. Geri dönen `Task`, arka plan çalıştırıcısında çalışır; `await` edilebilir veya `detach_and_log_err(cx)` ile bırakılabilir.
- **URL:** `open_url`, `register_url_scheme`.
- **Dosya ve prompt:** `prompt_for_paths`, `prompt_for_new_path`, `reveal_path`, `open_with_system`, `can_select_mixed_files_and_dirs`.
- **Menü:** `set_menus`, `get_menus`, `set_dock_menu`, `add_recent_document`, `update_jump_list`.
- **Termal durum:** `thermal_state`, `on_thermal_state_change`.
- **İmleç görünürlüğü:** `cursor_hide_mode`, `set_cursor_hide_mode`, `is_cursor_visible`. İşaretçinin görsel stili, pencere veya hitbox bağlamında `window.set_cursor_style(style, &hitbox)` ile, sürükleme sırasında ise `cx.set_active_drag_cursor_style(...)` ile belirlenir.
- **Ekran yakalama:** `is_screen_capture_supported`, `screen_capture_sources`.
- **Klavye:** `keyboard_layout()`, `keyboard_mapper()`, `on_keyboard_layout_change(|cx| ...)`.
- **HTTP istemcisi:** `http_client() -> Arc<dyn HttpClient>`, `set_http_client(Arc<dyn HttpClient>)`. `Application::with_http_client(...)` ile başlangıçta da ayarlanabilir; tipik olarak `crates/http_client` içindeki Zed varsayılanı tercih edilir.
- **Uygulama yolu ve compositor:** `app_path() -> Result<PathBuf>` (macOS bundle yolu ya da Linux'ta çalıştırılabilir dosya), `path_for_auxiliary_executable(name)` (yardımcı çalıştırılabilirler için bundle araması), `compositor_name() -> &'static str` (Linux'ta `wayland`, `x11`, `xwayland` gibi adlar; diğer platformlarda boş metin).

`Window` üzerinden gelen pencereye özgü kontroller ise şunlardır:

- `window.on_window_should_close(cx, |window, cx| -> bool)` — kullanıcı kapatma butonuna bastığında çalışır; `false` döndürmek kapanışı iptal eder.
- `window.appearance()`, `window.observe_window_appearance(...)` — pencere görünüm modunu (light/dark vb.) okur ve değişimini dinler.
- `window.tabbed_windows()`, `window.set_tabbing_identifier(...)` ve diğer yerel pencere sekmesi API'leri ("Yerel Pencere Sekmeleri ve SystemWindowTabController" bölümüne bakın).

Yeni bir platform portu veya test arka ucu yazıldığında bu sözleşmelerin tamamı karşılanmalıdır. Normal uygulama geliştirirken ise trait'lerin kendisine değil, `App` ve `Window` sarmalayıcılarına dokunmak tercih edilir. Böylece platform farklarını sarmalayıcı katmanı üstlenir.

## Platform Trait Uygulaması ve Sarmalayıcı Sınırları

Uygulama kodu `Platform` veya `PlatformWindow` trait'lerini doğrudan çağırmaz; akış `App` ve `Window` sarmalayıcıları üzerinden ilerler. Trait sözleşmesini bilmek en çok yeni bir platform portu, test platformu veya başsız arka uç yazarken gerekir. Aşağıdaki listeler trait'lerin hangi büyük yetenek gruplarına ayrıldığını gösterir.

`Platform` ana grupları:

- **Çalıştırıcı ve metin:** `background_executor`, `foreground_executor`, `text_system` — görev çalıştırıcılar ve metin sistemi platforma bağlı kaynaklardır.
- **Uygulama yaşam döngüsü:** `run`, `quit`, `restart`, `activate`, `hide`, `hide_other_apps`, `unhide_other_apps`, `on_quit`, `on_reopen`.
- **Ekran ve pencere:** `displays`, `primary_display`, `active_window`, `window_stack`, `open_window`.
- **Görünüm ve UI politikası:** `window_appearance`, `button_layout`, `should_auto_hide_scrollbars`, imleç görünürlüğü ve stili.
- **URL, yol ve prompt:** `open_url`, `on_open_urls`, `register_url_scheme`, `prompt_for_paths`, `prompt_for_new_path`, `reveal_path`, `open_with_system`.
- **Menüler:** `set_menus`, `get_menus`, `set_dock_menu`, `on_app_menu_action`, `on_will_open_app_menu`, `on_validate_app_menu_command`.
- **Pano ve kimlik bilgisi:** normal pano, Linux primary selection, macOS find pasteboard ve kimlik bilgisi deposu görevleri.
- **Ekran yakalama ve klavye:** `is_screen_capture_supported`, `screen_capture_sources`, `keyboard_layout`, `keyboard_mapper`, `on_keyboard_layout_change`.

`PlatformWindow` ana grupları:

- **Sınırlar ve durum:** `bounds`, `window_bounds`, `content_size`, `resize`, `scale_factor`, `display`, `appearance`, `modifiers`, `capslock`.
- **Girdi:** `set_input_handler`, `take_input_handler`, `on_input`, `update_ime_position` — IME desteği ve klavye girişi bu metotlardan geçer.
- **Pencere yaşam döngüsü:** `activate`, `is_active`, `is_hovered`, `minimize`, `zoom`, `toggle_fullscreen`, `on_should_close`, `on_close`.
- **Çizim:** `on_request_frame`, `draw(scene)`, `completed_frame`, `sprite_atlas`, `is_subpixel_rendering_supported`.
- **Süsleme ve çarpışma testi:** `set_title`, `set_background_appearance`, `on_hit_test_window_control`, `request_decorations`, `window_decorations`, `window_controls`.
- **Platforma özel:** macOS sekme ve belge API'leri, Linux taşıma, yeniden boyutlandırma, menü, `app-id` ve `inset` desteği, Windows ham handle, yalnızca test için `render_to_image`.

**Sarmalayıcı sınırı.** Trait ve sarmalayıcı arasındaki en kritik ayrımlar şunlardır:

- `Platform::set_cursor_style`, genel platform imlecini ayarlar; uygulama UI'ında hitbox'a bağlı stil gerekiyorsa `Window::set_cursor_style` tercih edilir.
- `PlatformWindow::prompt`, yerel bir prompt döndürebilir; `Window::prompt` ise gerektiğinde özel prompt yedeğini de yönetir.
- `PlatformWindow::map_window`, Linux'ta `map` ve `show` ayrımı için vardır; uygulama kodunda doğrudan çağrılmaz, `WindowOptions.show` ve pencere sarmalayıcısının davranışı bu işi karşılar.
- Trait üzerinde varsayılan metot "desteklenmiyor" anlamına gelir; sarmalayıcı üzerinden dönen `None` veya işlem yapmayan (`no-op`) sonuçlar, o platformun yeteneksizliği olarak değerlendirilmelidir.

## Başsız Çalışma, Ekran Yakalama ve Test Çizim Aracı

Görsel arayüz olmadan da GPUI uygulaması başlatılabilir. Bu yol özellikle CLI alt komutları, toplu işler, sunucu süreçleri ve başarım ölçüm (`benchmark`) senaryoları için kullanılır. İlgili modüller `crates/gpui/src/platform.rs::screen_capture_sources` ve `crates/gpui_platform/src/gpui_platform.rs::headless()` içinde yer alır.

Başsız bir uygulama şu biçimde başlatılır:

```rust
gpui_platform::headless().run(|baglam: &mut App| {
    // Arka plan görevleri, asset yükleme, ağ IO; çizim yok.
});
```

Bu yapı pencere açmaz, dolayısıyla görsel doğrulama veya ekran görüntüsü (`screenshot`) üretimi için uygun değildir. UI testi gerektiğinde `gpui_platform::headless` yerine `HeadlessAppContext` veya `VisualTestContext` kullanılır. `gpui_platform::current_headless_renderer()` ise yalnızca `test-support` özelliği (`feature`) altında derlenir; şu anda macOS'ta Metal başsız çizim aracı döndürebilir, diğer platformlarda `None` gelebilir.

**Ekran yakalama API'si.** Ekran yakalama akışı `oneshot` kanallar üzerinde kurulur ve ekran kareleri bir geri çağrıya iletilir:

```rust
let destekleniyor_mu = baglam.update(|baglam| baglam.is_screen_capture_supported());

let kaynak_alici = baglam.update(|baglam| baglam.screen_capture_sources());
let kaynaklar = kaynak_alici.await??;
if let Some(kaynak) = kaynaklar.first() {
    let akis_alici = kaynak.stream(
        baglam.foreground_executor(),
        Box::new(|kare| {
            // kare: ScreenCaptureFrame
        }),
    );
    let akis = akis_alici.await??;
    let ust_veri = akis.metadata()?;
}
```

`ScreenCaptureSource`, her platformda farklı bir kaynak listesi sunar (ekran, pencere, alan gibi). Yakalama `ScreenCaptureSource::stream(&ForegroundExecutor, frame_callback)` ile başlar. Geri dönen `oneshot::Receiver<Result<Box<dyn ScreenCaptureStream>>>`, akış handle'ını taşır; ekran kareleri ise geri çağrıya `ScreenCaptureFrame` olarak iletilir.

**Tuzaklar.** Ekran yakalama ve başsız çalışma tarafında dikkat edilmesi gereken birkaç nokta vardır:

- macOS'ta `Screen Recording` izni kullanıcı onayı gerektirir; ilk çağrıda sistem bir izin penceresi açar, onaydan sonra ileriki çalıştırmalarda da izin geçerli kalır.
- Bazı platformlarda ekran yakalama desteklenmez; `is_screen_capture_supported()` `false` dönebilir veya kaynak listesi boş gelebilir. Bu durum uygulama tarafında kullanıcıya açıklayıcı bir mesajla ele alınmalıdır.
- UI testlerinde gerçek bir platform penceresi açmak yerine `TestAppContext`, `VisualTestContext` veya çizim aracı fabrikası verilen `HeadlessAppContext` tercih edilir; bu sayede testler CI ortamlarında ekran olmadan da çalışır.

---
