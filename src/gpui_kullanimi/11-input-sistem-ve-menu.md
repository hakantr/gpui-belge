# Girdi, Sistem ve Menü

---

## Girdi, Pano, Prompt ve Platform Servisleri

Element seviyesinde GPUI birçok girdi olayını tek tipli bir fluent API üzerinden açar. Aşağıdaki listeler, farklı olay tiplerinin hangi metotlarla yakalandığını özetler:

- Klavye: `.on_key_down`, `.capture_key_down`, `.on_key_up`, `.capture_key_up`.
- Fare: `.on_mouse_down`, `.capture_any_mouse_down`, `.on_mouse_up`, `.capture_any_mouse_up`, `.on_mouse_move`, `.on_mouse_down_out`, `.on_mouse_up_out`, `.on_click`, `.on_hover`.
- Hareket ve scroll: `.on_scroll_wheel`, `.on_pinch`, `.capture_pinch`.
- Sürükle-bırak: `.on_drag`, `.on_drag_move`, `.on_drop`.
- Action: `.capture_action::<A>`, `.on_action::<A>`, `.on_boxed_action`.

Olay tipleri `interactive.rs` ve `platform.rs` içinde tanımlıdır: `KeyDownEvent`, `KeyUpEvent`, `MouseDownEvent`, `MouseUpEvent`, `MouseMoveEvent`, `MousePressureEvent`, `ScrollWheelEvent`, `PinchEvent`, `FileDropEvent`, `ExternalPaths`, `ClickEvent`. `ScrollDelta::pixel_delta(line_height)`, satır tabanlı scroll'u piksele çevirir; `coalesce` aynı yöndeki delta'ları birleştirir.

**Pano.** Pano okuma ve yazma sade çağrılarla yapılır:

```rust
baglam.write_to_clipboard(ClipboardItem::new_string("metin".to_string()));

if let Some(oge) = baglam.read_from_clipboard()
    && let Some(metin) = oge.text()
{
    // kullan
}
```

`ClipboardItem` birden çok `ClipboardEntry` taşıyabilir: `String`, `Image` veya `ExternalPaths`. `String` girdisine üst veri eklemek istendiğinde `new_string_with_metadata` veya `new_string_with_json_metadata` kullanılır. Linux/FreeBSD için primary selection `read_from_primary`/`write_to_primary`; macOS Find pasteboard için `read_from_find_pasteboard`/`write_to_find_pasteboard`, `cfg` koşullu API'lerdir.

**Prompt ve dosya seçici.** Kullanıcıyla iletişim kuran platform diyalogları da bağlam üzerinden çalışır:

- `window.prompt(level, message, detail, answers, cx) -> oneshot::Receiver<usize>`
- `cx.set_prompt_builder(...)`, özel GPUI prompt UI'ını kurar; `reset_prompt_builder`, yerel veya varsayılan akışa döner.
- `cx.prompt_for_paths(PathPromptOptions { files, directories, multiple, prompt })`, dosya veya dizin seçici açar.
- `cx.prompt_for_new_path(directory, suggested_name)`, kaydetme diyaloğunu açar.
- `cx.open_url(url)`, `cx.register_url_scheme(scheme)`, `cx.reveal_path(path)`, `cx.open_with_system(path)`, platform servislerine gider.
- Platformun kimlik bilgisi deposu için `cx.write_credentials(url, username, password)`, `cx.read_credentials(url)` ve `cx.delete_credentials(url)` async `Task<Result<_>>` döndürür.
- Uygulama yolu ve sistem bilgisi için `cx.app_path()`, `cx.path_for_auxiliary_executable(name)`, `cx.compositor_name()`, `cx.should_auto_hide_scrollbars()`.
- Yeniden başlatma ve HTTP istemcisi tarafında `cx.set_restart_path(path)`, `cx.restart()`, `cx.http_client()` ve `cx.set_http_client(client)` bulunur.

**Platform ve prompt davranışı.** Diyaloglarda platforma özel davranışlar sürpriz oluşturabilir; bilinmesi gereken birkaç nokta:

- macOS `Window::prompt` NSAlert akışında Return ilk butona, Escape Cancel'a gider; Space ile odak son "iptal olmayan ve varsayılan olmayan" butona taşınır. "Save / Don't Save / Cancel" gibi üçlü prompt'larda orta seçenek klavyeyle erişilebilir kalır.
- Wayland'da pano ve primary selection yazılırken, tuş veya fare basma türüne göre süzülmüş seri yerine alınan en güncel compositor serisi kullanılır; aksi halde bazı compositor'lar seçim isteğini sessizce reddedebilir.
- `open_path_prompt` sonuç sıralaması `ProjectPanelSettings.sort_mode` ile uyumlu çalışır. Proje panelindeki "önce dizinler / önce dosyalar / karışık" seçimi, dosya yolu prompt'unun aday listesinde de aynı şekilde uygulanır.

## Prompt Builder, PromptHandle ve Fallback Prompt

`Window::prompt` platform diyaloğunu açar. Platform prompt'u desteklemiyorsa veya özel bir prompt builder ayarlanmışsa GPUI içinde çizilen prompt kullanılır:

```rust
let yanit = pencere.prompt(
    PromptLevel::Warning,
    "Kaydedilmemiş değişiklikler",
    Some("Kaydetmeden kapatılsın mı?"),
    &[PromptButton::cancel("İptal"), PromptButton::ok("Kapat")],
    baglam,
);

let secilen_sira = yanit.await?;
```

**Prompt tipleri.** Prompt akışında kullanılan tipler şu rolleri üstlenir:

- `PromptLevel::{Info, Warning, Critical}` — görsel önem seviyesidir.
- `PromptButton::ok(label)`, `cancel(label)`, `new(label)` — sırasıyla ok, cancel ve genel action butonu üretir; `label()` ve `is_cancel()` okunabilir.
- `PromptResponse(pub usize)` — özel prompt view'unun seçilen buton indeksini yaydığı olaydır.
- `Prompt` — `EventEmitter<PromptResponse> + Focusable` trait birleşimidir.
- `PromptHandle::with_view(view, window, cx)` — özel prompt entity'sini pencereye bağlar, önceki odağı kaydeder ve prompt yanıtında odağı geri verir.
- `fallback_prompt_renderer(...)` — `set_prompt_builder` ile varsayılan GPUI prompt çizimini zorlamak için kullanılır.

**Zed entegrasyonu** (`crates/ui_prompt`):

- `ui_prompt::init(cx)`, `WorkspaceSettings::use_system_prompts` ayarını `SettingsStore` üzerinden gözlemler. Sistem prompt'ları açıksa `cx.reset_prompt_builder()` çağrılarak platform diyaloğuna düşülür; aksi halde `cx.set_prompt_builder(zed_prompt_renderer)` ile GPUI içindeki markdown destekli prompt akışına geçilir. Linux/FreeBSD'de sistem prompt'u yoksayılır, daima Zed çizimi kullanılır.
- `ZedPromptRenderer`, `pub` bir struct'tır: `Markdown` entity'siyle mesaj ve detay metnini çizer; cancel ve confirm action'larını içeride yönlendirir. Uygulama kodu doğrudan oluşturmaz; yalnızca prompt builder fonksiyonu tarafından kurulur.

**Özel builder.** Tamamen özel bir prompt görsel akışı tanımlamak için builder kayda alınır:

```rust
baglam.set_prompt_builder(|seviye, mesaj, ayrinti, eylemler, tutamac, pencere, baglam| {
    let mesaj = mesaj.to_string();
    let ayrinti = ayrinti.map(ToString::to_string);
    let eylemler = eylemler.to_vec();
    let gorunum = baglam.new(|baglam| IstemGorunumu::new(seviye, mesaj, ayrinti, eylemler, baglam));
    tutamac.with_view(gorunum, pencere, baglam)
});
```

**Tuzaklar.** Prompt'larla çalışırken dikkat edilmesi gereken noktalar:

- GPUI iç içe (`re-entrant`) prompt desteklemez; bir prompt açıkken aynı pencerede ikinci prompt'un nasıl açılacağı ayrıca tasarlanmalıdır.
- Özel prompt `Focusable` sağlamalıdır; aksi halde `PromptHandle::with_view`, odak geri yükleme zincirini tamamlayamaz.
- Prompt sonucu buton etiketi değil, `answers` dizisindeki indekstir.

## Uygulama Menüsü ve Dock

`crates/gpui/src/platform/app_menu.rs`.

Menü modeli birkaç ana tip etrafında kurulur:

- `Menu { name, items, disabled }`
- `MenuItem`:
  - `Separator`
  - `Submenu(Menu)`
  - `SystemMenu(OsMenu)` — macOS Services gibi sistem alt menüleri.
  - `Action { name, action, os_action, checked, disabled }`
- `OsAction`: `Cut`, `Copy`, `Paste`, `SelectAll`, `Undo`, `Redo`. Yerel düzenleme menüsü eşlemesi için kullanılır.

**Builder örneği.** Üst seviye menü ağacı kurulurken builder kalıbı şu şekildedir:

```rust
baglam.set_menus(vec![
    Menu::new("Zed").items([
        MenuItem::action("Zed Hakkında", zed::About),
        MenuItem::Separator,
        MenuItem::action("Çık", workspace::Quit),
    ]),
    Menu::new("Düzenle").items([
        MenuItem::os_action("Geri Al", editor::Undo, OsAction::Undo),
        MenuItem::os_action("Yinele", editor::Redo, OsAction::Redo),
        MenuItem::Separator,
        MenuItem::os_action("Kes", editor::Cut, OsAction::Cut),
        MenuItem::os_action("Kopyala", editor::Copy, OsAction::Copy),
        MenuItem::os_action("Yapıştır", editor::Paste, OsAction::Paste),
        MenuItem::os_action("Tümünü Seç", editor::SelectAll, OsAction::SelectAll),
    ]),
]);
```

`MenuItem::action(name, action)`, veri taşımayan `unit struct` action'lar için bir kısayoldur. Veri taşıyan action'larda da doğrudan action değeri geçirilebilir: `MenuItem::action("Satıra Git", SatiraGit { satir: 1 })`. Aynı menü modeli klonlanmak istenirse `Menu::owned()` ve `MenuItem::owned()` kullanılır.

**Diğer menü API'leri** (`App` üzerinde):

- `cx.set_dock_menu(Vec<MenuItem>)` — macOS dock sağ tıklama menüsü; Windows'ta dock menüsü veya jump list modelinin bir parçası olarak çalışır.
- `cx.add_recent_document(path)` — macOS'taki son kullanılan öğeler listesine ekler.
- `cx.update_jump_list(menus, entries) -> Task<Vec<SmallVec<[PathBuf; 2]>>>` — Windows jump list'ini günceller ve kullanıcının listeden kaldırdığı girişleri `Task` sonucu olarak döndürür. Zed `HistoryManager`, bu sonucu geçmişten siler.
- `cx.get_menus()` — şu an ayarlanmış menü modelini okur.

**Platform davranışı.** Aynı menü modeli her platformda farklı bir kanal üzerinden çizilir:

- macOS'ta yerel `NSMenu` ile çizilir; klavye kısayolları kısayol kayıtlarından okunur.
- Windows ve Linux, platform durumunu `OwnedMenu` olarak saklar; Zed bu modeli uygulama içi menü ve çizim katmanlarında kullanır.
- Linux dock menüsü arka uçta `todo` veya işlem yapmayan (`no-op`) durumdadır; dock veya jump-list davranışı için platforma özel bir yedek akış hazırlanmalıdır.

**Tuzak.** Aynı action birden çok menü öğesine bağlandığında keymap'te tek bir kısayol gösterilir. `os_action` yalnızca macOS yerel düzenleme menüsü eşlemesini etkiler; diğer platformlarda sıradan bir action gibi davranır.

---
