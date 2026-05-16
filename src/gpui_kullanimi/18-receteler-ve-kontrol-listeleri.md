# Reçeteler ve Kontrol Listeleri

---

## Reçeteler

Bu bölüm, önceki başlıklardaki API'leri günlük senaryolara oturtarak özetler. Her reçete, ihtiyaç duyulan ayarları ve çağrı sırasını tek yerde toplar.

#### Yeni Workspace Penceresi

Bir Zed workspace penceresi açılırken adımlar şu sırayla işler:

1. `zed::build_window_options(display_uuid, cx)` çağrılır.
2. Root view olarak workspace veya multi-workspace entity oluşturulur.
3. Titlebar için `TitleBar`/`PlatformTitleBar` yolu izlenir.
4. Root içerik `workspace::client_side_decorations(...)` ile sarılır.
5. Close işlemi için `workspace::CloseWindow` action'ı dispatch edilir.

#### Küçük Dialog Penceresi

Küçük modal benzeri pencerelerde tipik konfigürasyon aşağıdaki gibidir; ana pencere değil bir dialog hedeflendiği için `WindowKind::Dialog` ve resize/minimize kısıtları kullanılır:

```rust
cx.open_window(
    WindowOptions {
        titlebar: Some(TitlebarOptions {
            title: Some("Dialog".into()),
            appears_transparent: true,
            traffic_light_position: Some(point(px(12.), px(12.))),
        }),
        window_bounds: Some(WindowBounds::centered(size(px(440.), px(300.)), cx)),
        is_resizable: false,
        is_minimizable: false,
        kind: WindowKind::Dialog,
        app_id: Some(ReleaseChannel::global(cx).app_id().to_owned()),
        ..Default::default()
    },
    |window, cx| {
        window.activate_window();
        cx.new(|cx| DialogView::new(window, cx))
    },
)?;
```

#### Transparent/Blurred Notification

Bildirim ve overlay pencerelerinde tipik olarak titlebar kapatılır, pencere focus almaz ve arka plan saydam olarak ayarlanır:

```rust
WindowOptions {
    titlebar: None,
    focus: false,
    show: true,
    kind: WindowKind::PopUp,
    is_movable: false,
    is_resizable: false,
    window_background: WindowBackgroundAppearance::Transparent,
    window_decorations: Some(WindowDecorations::Client),
    ..Default::default()
}
```

Blur isteniyorsa `Transparent` yerine `Blurred` kullanılır; bunun için içerik root'unun tamamen opak bir arka plan çizmediğinden emin olmak gerekir, aksi halde blur görünmez kalır.

#### Platforma Göre UI Ayırma

Çalışma zamanında platforma göre dallanma gerektiğinde `PlatformStyle` kullanılır:

```rust
match PlatformStyle::platform() {
    PlatformStyle::Mac => { /* macOS */ }
    PlatformStyle::Linux => { /* Linux */ }
    PlatformStyle::Windows => { /* Windows */ }
}
```

Derleme zamanı farkı gerekiyorsa `cfg!(target_os = "...")` veya `#[cfg(...)]` tercih edilir. Çalışma zamanı styling için `PlatformStyle` daha okunaklıdır.

#### Titlebar Drag ve Double Click

Drag ve double-click davranışı tek bir click handler içinde toplanır. Double-click'te platforma göre native ya da fluent helper kullanılır:

```rust
h_flex()
    .window_control_area(WindowControlArea::Drag)
    .on_click(|event, window, _| {
        if event.click_count() == 2 {
            if cfg!(target_os = "macos") {
                window.titlebar_double_click();
            } else {
                window.zoom_window();
            }
        }
    })
```

Linux veya macOS'ta elle drag başlatılması gerektiğinde mouse move sırasında şu çağrı yapılır:

```rust
window.start_window_move();
```

Windows tarafında `WindowControlArea::Drag` native hit-test üzerinden daha doğru sonucu verir; bu nedenle Windows'ta drag için ayrı bir `start_window_move` çağrısına gerek kalmaz.

#### Client-Side Resize Handle

Client-side decoration ile birlikte sunulan resize handle'larında kenar hesabı yapılıp ilgili `ResizeEdge` ile platform çağrısı tetiklenir:

```rust
.on_mouse_down(MouseButton::Left, move |event, window, _| {
    if let Some(edge) = resize_edge(event.position, shadow, size, tiling) {
        window.start_window_resize(edge);
    }
})
```

Cursor stilinin de aynı kenara göre `ResizeUpDown`, `ResizeLeftRight`, `ResizeUpLeftDownRight` veya `ResizeUpRightDownLeft` olarak ayarlanması gerekir; aksi halde resize bölgesinde görsel ipucu eksik kalır.

#### Tema Değişince Pencere Arka Planını Güncelleme

Tema akışı tüm pencerelere yansıtılırken settings observer içinden tek tek pencerelerin background appearance'ı güncellenir:

```rust
cx.observe_global::<SettingsStore>(move |cx| {
    for window in cx.windows() {
        let appearance = cx.theme().window_background_appearance();
        window.update(cx, |_, window, _| {
            window.set_background_appearance(appearance);
        }).ok();
    }
}).detach();
```

Zed ana uygulaması bu deseni zaten kullanır.

#### Git Graph Özel Komut Task'ı

Git Graph commit context menu'sünden özel bir task çalıştırmak için global `tasks.json` içine `git-command` tag'li bir task eklenir. Worktree-local task'lar bu akışta desteklenmez. Task seçili commit ve repository context'iyle çözülür; varsayılan çalışma dizini seçili repository root'udur.

Desteklenen Git değişkenleri şu şekildedir:

Bu bağlamda yalnız Git değişkenleri sağlanır. `ZED_FILE`, `ZED_SELECTED_TEXT`, `ZED_WORKTREE_ROOT`, `ZED_MAIN_GIT_WORKTREE` gibi editor/worktree değişkenleri varsayılan değer taşımadıkça çözümlenmez.

- `ZED_GIT_SHA`
- `ZED_GIT_SHA_SHORT`
- `ZED_GIT_REPOSITORY_NAME`
- `ZED_GIT_REPOSITORY_PATH`

Tipik bir tanım şöyledir:

```json
[
  {
    "label": "Branches containing commit: $ZED_GIT_SHA_SHORT",
    "command": "git",
    "args": ["branch", "-a", "--contains", "$ZED_GIT_SHA"],
    "tags": ["git-command"]
  }
]
```

## Sık Hatalar ve Doğru Desenler

Aşağıdaki liste rehber boyunca anlatılan tuzakları tek bir noktada toparlar; her madde belirtisi ile birlikte altta yatan nedeni de işaret eder.

- **İstenen decoration'a güvenme** — `WindowOptions.window_decorations` yalnız bir istektir. Render sırasında fiili sonucu `window.window_decorations()` çağrısı verir; karar bu sonuca dayanmalıdır.
- **Blur görünmüyor** — Root view veya tema tamamen opak bir renk çiziyor olabilir. Blur efektinin görünmesi için transparan bir surface ve içerikte alfa bırakılması şarttır.
- **Linux kontrol butonları yanlış tarafta** — Doğru kaynak `cx.button_layout()`'tur ve değişimi `observe_button_layout_changed` ile izlenmelidir.
- **Windows caption butonları tıklanmıyor** — Butonlarda `window_control_area(Close/Max/Min)` çağrısının eksik kalması native hit-test'i bozar.
- **Close davranışı atlanıyor** — Zed workspace penceresinde doğrudan `remove_window` yerine `workspace::CloseWindow` action'ı dispatch edilmelidir; aksi halde dirty buffer ve kullanıcı onayı akışları atlanır.
- **Async task çalışırken yok oluyor** — Dönen `Task` saklanmamış ya da detach edilmemiştir; drop edildiği anda iş iptal olur.
- **Entity leak** — Uzun yaşayan task veya subscription içinde güçlü `Entity` yakalamak döngü üretir; bunun yerine `WeakEntity` kullanılmalıdır.
- **Render güncellenmiyor** — State değişiminden sonra `cx.notify()` unutulmuştur; view aynı verilerle yeniden çizilir.
- **Focus callback'i fire etmiyor** — Element `.track_focus(&focus_handle)` ile ağaca bağlanmamış olabilir.
- **Custom titlebar altında içerik tıklanamıyor** — Drag veya window control hitbox'ı fazla geniş tutulmuş ya da `.occlude()` yanlış yere konmuş olabilir.
- **Client decoration shadow boşluğu** — `set_client_inset` ve dış sarmalayıcının padding/shadow değerleri birlikte yönetilmelidir; aralarındaki uyumsuzluk görünür bir boşluk üretir.

## Yeni Pencere Eklerken Kontrol Listesi

Yeni bir pencere eklenirken aşağıdaki kontrol listesi unutulan bir ayrıntı kalmaması için bir hatırlatma görevi görür:

1. Bu pencerenin workspace mi, modal mı, popup mu olduğuna karar verilir ve uygun `WindowKind` seçilir.
2. Ana Zed penceresi ise `build_window_options` kullanılır.
3. Bounds geri yüklenecekse `WindowBounds` persist edilir.
4. Hangi display'de açılacağı belirlenir; `display_id` veya `display_uuid` seçilir.
5. Titlebar native mi, custom mı olacak? `TitlebarOptions` ile `PlatformTitleBar` arasındaki karar verilir.
6. Linux dekorasyon modu ayardan geliyorsa `window_decorations` bağlanır.
7. Client decoration varsa wrapper, inset, resize handle ve tiling durumu eklenir.
8. Close action'ı doğrudan pencereyi mi kapatmalı, yoksa workspace close akışını mı tetiklemeli, belirlenir.
9. Blur veya transparent gerekiyorsa `window_background` ile root alpha uyumu kontrol edilir.
10. Focus başlangıcı doğru mu? `focus`, `show`, `activate_window` ve focus handle gözden geçirilir.
11. Minimum size gerekli mi, sorulur.
12. App id ve Linux ikonu gerekiyor mu, kontrol edilir.
13. macOS native tabbing isteniyorsa `tabbing_identifier` ayarlanır.
14. Settings veya tema değişiminde arka planın güncellenip güncellenmeyeceği planlanır.
15. Button layout değişiminde titlebar'ın yeniden render edilip edilmeyeceği gözden geçirilir.
16. Testte timer gerekiyorsa GPUI executor timer'ı kullanılır.

## Kısa Cevaplar

Bu başlık altında rehber boyunca en çok sorulan dört konunun kısa özeti yer alır.

**İleride pencere oluşturmak için izlenecek yol.** Workspace penceresi için başlangıç noktası `zed::build_window_options`'tır. Özel ve küçük bir pencere için doğrudan `cx.open_window(WindowOptions { ... }, |window, cx| cx.new(...))` çağrısı kullanılır. Root view, `Render` implement eden bir `Entity` olmalıdır.

**Pencere dekorunun tanımlanması.** Linux için `WindowOptions.window_decorations = Some(WindowDecorations::Client/Server)` verilir. Render tarafında fiili sonuç `window.window_decorations()` ile okunur. Zed tarzı client decoration için `workspace::client_side_decorations` kullanılır. macOS ve Windows'ta özel titlebar için `TitlebarOptions { appears_transparent: true }` ya da `titlebar: None` ile `PlatformTitleBar` tercih edilir.

**Kontrol butonlarının yönetimi.** Zed içinde `platform_title_bar::render_left_window_controls` ve `render_right_window_controls` kullanılır. Linux'ta `cx.button_layout()` ve `window.window_controls()` sonucu belirleyicidir. Windows'ta butonlar `WindowControlArea::{Min, Max, Close}` ile native hit-test'e bağlanır. Close için workspace akışında `CloseWindow` action'ı dispatch edilir.

**Blur yönetiminin işletim sistemine göre uygulanması.** Pencere açılırken veya tema değiştiğinde `window.set_background_appearance(...)` çağrılır. Zed tema akışı `opaque`, `transparent` ve `blurred` değerlerini destekler. macOS gerçek blur'u `NSVisualEffectView` ile, Windows composition/DWM ile, Wayland ise compositor blur protokolü ile uygular. Destek olmadığında `Blurred` transparan gibi davranabilir. Root UI opak çizdiğinde blur görünmez kalır.

**Platform farklarının soyutlanacağı yer.** Davranış pencere ile ilgiliyse GPUI `Platform` ve `PlatformWindow` katmanına bağlanır. Zed UI görünümüyle ilgiliyse `PlatformStyle::platform()` ve `platform_title_bar` bileşenleri kullanılır. Ayar farkı gerekiyorsa `settings_content` şeması ve `settings` dönüşümleri devreye girer.

## Dosya Yoluna Göre Ne Nerede?

Aşağıdaki liste, rehberde anlatılan kavramların kodda hangi dosyada bulunduğunu tek bakışta verir. Yeni bir bileşen yazılırken benzer örneğin nerede olduğunu hızla bulmaya yarar.

- Pencere açma API'si: `crates/gpui/src/app.rs::open_window`
- Pencere seçenekleri: `crates/gpui/src/platform.rs::WindowOptions`
- Platform penceresi sözleşmesi: `crates/gpui/src/platform.rs::PlatformWindow`
- Pencere wrapper metotları: `crates/gpui/src/window.rs`
- Element ve render trait'leri: `crates/gpui/src/element.rs`, `view.rs`
- Style fluent API: `crates/gpui/src/styled.rs`
- Interactivity fluent API: `crates/gpui/src/elements/div.rs`
- Platform seçimi: `crates/gpui_platform/src/gpui_platform.rs`
- macOS pencere davranışı: `crates/gpui_macos/src/window.rs`
- Windows pencere davranışı: `crates/gpui_windows/src/window.rs`, `events.rs`
- Linux Wayland davranışı: `crates/gpui_linux/src/linux/wayland/window.rs`
- Linux X11 davranışı: `crates/gpui_linux/src/linux/x11/window.rs`
- Zed ana window options: `crates/zed/src/zed.rs::build_window_options`
- Zed platform titlebar: `crates/platform_title_bar/src/platform_title_bar.rs`
- Linux controls: `crates/platform_title_bar/src/platforms/platform_linux.rs`
- Windows controls: `crates/platform_title_bar/src/platforms/platform_windows.rs`
- Workspace client decoration: `crates/workspace/src/workspace.rs::client_side_decorations`
- Zed titlebar composition: `crates/title_bar/src/title_bar.rs`
- Theme background appearance: `crates/theme/src/theme.rs`, `crates/theme_settings/src/theme_settings.rs`, `crates/settings/src/content_into_gpui.rs`
- UI component export listesi: `crates/ui/src/components.rs`
- UI input: `crates/ui_input/src/ui_input.rs`, `input_field.rs`
