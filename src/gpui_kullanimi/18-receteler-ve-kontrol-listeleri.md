# Reçeteler ve Kontrol Listeleri

---

## Reçeteler

Bu bölüm, önceki başlıklardaki API'leri günlük senaryolara oturtarak özetler. Her reçete, ihtiyaç duyulan ayarları ve çağrı sırasını tek yerde toplar.

#### Yeni Workspace Penceresi

Bir Zed workspace penceresi açılırken adımlar şu sırayla işler:

1. `zed::build_window_options(display_uuid, cx)` çağrılır.
2. Kök view olarak workspace veya multi-workspace entity oluşturulur.
3. Başlık çubuğu için `TitleBar`/`PlatformTitleBar` yolu izlenir.
4. Kök içerik `workspace::client_side_decorations(...)` ile sarılır.
5. Kapatma işlemi için `workspace::CloseWindow` action'ı dispatch edilir.

#### Küçük Diyalog Penceresi

Küçük modal benzeri pencerelerde tipik konfigürasyon aşağıdaki gibidir; ana pencere değil bir diyalog hedeflendiği için `WindowKind::Dialog` ve resize/minimize kısıtları kullanılır:

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

#### Saydam/Bulanık Bildirim

Bildirim ve kaplama pencerelerinde tipik olarak başlık çubuğu kapatılır, pencere odak almaz ve arka plan saydam olarak ayarlanır:

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

Bulanıklık isteniyorsa `Transparent` yerine `Blurred` kullanılır; bunun için içerik kökünün tamamen opak bir arka plan çizmediğinden emin olmak gerekir, aksi halde bulanıklık görünmez kalır.

#### Platforma Göre UI Ayırma

Çalışma zamanında platforma göre dallanma gerektiğinde `PlatformStyle` kullanılır:

```rust
match PlatformStyle::platform() {
    PlatformStyle::Mac => { /* macOS */ }
    PlatformStyle::Linux => { /* Linux */ }
    PlatformStyle::Windows => { /* Windows */ }
}
```

Derleme zamanı farkı gerekiyorsa `cfg!(target_os = "...")` veya `#[cfg(...)]` tercih edilir. Çalışma zamanı stillendirmesi için `PlatformStyle` daha okunaklıdır.

#### Başlık Çubuğu Sürükleme ve Çift Tıklama

Sürükleme ve çift tıklama davranışı tek bir tıklama işleyicisi içinde toplanır. Çift tıklamada platforma göre yerel ya da fluent yardımcı kullanılır:

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

Linux veya macOS'ta elle sürükleme başlatılması gerektiğinde fare hareketi sırasında şu çağrı yapılır:

```rust
window.start_window_move();
```

Windows tarafında `WindowControlArea::Drag` yerel hit-test üzerinden daha doğru sonucu verir; bu nedenle Windows'ta sürükleme için ayrı bir `start_window_move` çağrısına gerek kalmaz.

#### İstemci Tarafı Yeniden Boyutlandırma Tutamacı

İstemci tarafı süslemesi ile birlikte sunulan yeniden boyutlandırma tutamaçlarında kenar hesabı yapılıp ilgili `ResizeEdge` ile platform çağrısı tetiklenir:

```rust
.on_mouse_down(MouseButton::Left, move |event, window, _| {
    if let Some(edge) = resize_edge(event.position, shadow, size, tiling) {
        window.start_window_resize(edge);
    }
})
```

İmleç stilinin de aynı kenara göre `ResizeUpDown`, `ResizeLeftRight`, `ResizeUpLeftDownRight` veya `ResizeUpRightDownLeft` olarak ayarlanması gerekir; aksi halde yeniden boyutlandırma bölgesinde görsel ipucu eksik kalır.

#### Tema Değişince Pencere Arka Planını Güncelleme

Tema akışı tüm pencerelere yansıtılırken ayar gözlemcisi içinden tek tek pencerelerin arka plan görünüşü güncellenir:

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

Git Graph commit bağlam menüsünden özel bir task çalıştırmak için global `tasks.json` içine `git-command` etiketli bir task eklenir. Worktree yerel task'lar bu akışta desteklenmez. Task seçili commit ve repository bağlamıyla çözülür; varsayılan çalışma dizini seçili repository köküdür.

Desteklenen Git değişkenleri şu şekildedir:

Bu bağlamda yalnız Git değişkenleri sağlanır. `ZED_FILE`, `ZED_SELECTED_TEXT`, `ZED_WORKTREE_ROOT`, `ZED_MAIN_GIT_WORKTREE` gibi editör/worktree değişkenleri varsayılan değer taşımadıkça çözümlenmez.

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

- **İstenen süslemeye güvenme** — `WindowOptions.window_decorations` yalnız bir istektir. Çizim sırasında fiili sonucu `window.window_decorations()` çağrısı verir; karar bu sonuca dayanmalıdır.
- **Bulanıklık görünmüyor** — Kök view veya tema tamamen opak bir renk çiziyor olabilir. Bulanıklık efektinin görünmesi için saydam bir surface ve içerikte alfa bırakılması şarttır.
- **Linux kontrol butonları yanlış tarafta** — Doğru kaynak `cx.button_layout()`'tur ve değişimi `observe_button_layout_changed` ile izlenmelidir.
- **Windows başlık butonları tıklanmıyor** — Butonlarda `window_control_area(Close/Max/Min)` çağrısının eksik kalması yerel hit-test'i bozar.
- **Kapatma davranışı atlanıyor** — Zed workspace penceresinde doğrudan `remove_window` yerine `workspace::CloseWindow` action'ı dispatch edilmelidir; aksi halde kirli buffer ve kullanıcı onayı akışları atlanır.
- **Async task çalışırken yok oluyor** — Dönen `Task` saklanmamış ya da detach edilmemiştir; drop edildiği anda iş iptal olur.
- **Entity sızıntı** — Uzun yaşayan task veya abonelik içinde güçlü `Entity` yakalamak döngü üretir; bunun yerine `WeakEntity` kullanılmalıdır.
- **Çizim güncellenmiyor** — Durum değişiminden sonra `cx.notify()` unutulmuştur; view aynı verilerle yeniden çizilir.
- **Odak geri çağrısı tetiklenmiyor** — Element `.track_focus(&focus_handle)` ile ağaca bağlanmamış olabilir.
- **Özel başlık çubuğu altında içerik tıklanamıyor** — Sürükleme veya window control hitbox'ı fazla geniş tutulmuş ya da `.occlude()` yanlış yere konmuş olabilir.
- **İstemci süslemesi gölge boşluğu** — `set_client_inset` ve dış sarmalayıcının padding/shadow değerleri birlikte yönetilmelidir; aralarındaki uyumsuzluk görünür bir boşluk üretir.

## Yeni Pencere Eklerken Kontrol Listesi

Yeni bir pencere eklenirken aşağıdaki kontrol listesi unutulan bir ayrıntı kalmaması için bir hatırlatma görevi görür:

1. Bu pencerenin workspace mi, modal mı, popup mu olduğuna karar verilir ve uygun `WindowKind` seçilir.
2. Ana Zed penceresi ise `build_window_options` kullanılır.
3. Bounds geri yüklenecekse `WindowBounds` kalıcılaştırılır.
4. Hangi display'de açılacağı belirlenir; `display_id` veya `display_uuid` seçilir.
5. Başlık çubuğu yerel mi, özel mi olacak? `TitlebarOptions` ile `PlatformTitleBar` arasındaki karar verilir.
6. Linux dekorasyon modu ayardan geliyorsa `window_decorations` bağlanır.
7. İstemci süslemesi varsa sarmalayıcı, inset, yeniden boyutlandırma tutamacı ve tiling durumu eklenir.
8. Kapatma action'ı doğrudan pencereyi mi kapatmalı, yoksa workspace kapatma akışını mı tetiklemeli, belirlenir.
9. Bulanıklık veya saydamlık gerekiyorsa `window_background` ile kök alfa uyumu kontrol edilir.
10. Odak başlangıcı doğru mu? `focus`, `show`, `activate_window` ve focus handle gözden geçirilir.
11. Minimum boyutu gerekli mi, sorulur.
12. App id ve Linux ikonu gerekiyor mu, kontrol edilir.
13. macOS yerel sekmeleme isteniyorsa `tabbing_identifier` ayarlanır.
14. Ayar veya tema değişiminde arka planın güncellenip güncellenmeyeceği planlanır.
15. Buton yerleşim değişiminde başlık çubuğunun yeniden çizilip çizilmeyeceği gözden geçirilir.
16. Testte timer gerekiyorsa GPUI executor timer'ı kullanılır.

## Kısa Cevaplar

Bu başlık altında rehber boyunca en çok sorulan dört konunun kısa özeti yer alır.

**İleride pencere oluşturmak için izlenecek yol.** Workspace penceresi için başlangıç noktası `zed::build_window_options`'tır. Özel ve küçük bir pencere için doğrudan `cx.open_window(WindowOptions { ... }, |window, cx| cx.new(...))` çağrısı kullanılır. Kök view, `Render` uygulayan bir `Entity` olmalıdır.

**Pencere dekorunun tanımlanması.** Linux için `WindowOptions.window_decorations = Some(WindowDecorations::Client/Server)` verilir. Çizim tarafında fiili sonuç `window.window_decorations()` ile okunur. Zed tarzı istemci süslemesi için `workspace::client_side_decorations` kullanılır. macOS ve Windows'ta özel başlık çubuğu için `TitlebarOptions { appears_transparent: true }` ya da `titlebar: None` ile `PlatformTitleBar` tercih edilir.

**Kontrol butonlarının yönetimi.** Zed içinde `platform_title_bar::render_left_window_controls` ve `render_right_window_controls` kullanılır. Linux'ta `cx.button_layout()` ve `window.window_controls()` sonucu belirleyicidir. Windows'ta butonlar `WindowControlArea::{Min, Max, Close}` ile yerel hit-test'e bağlanır. Kapatma için workspace akışında `CloseWindow` action'ı dispatch edilir.

**Bulanıklık yönetiminin işletim sistemine göre uygulanması.** Pencere açılırken veya tema değiştiğinde `window.set_background_appearance(...)` çağrılır. Zed tema akışı `opaque`, `transparent` ve `blurred` değerlerini destekler. macOS gerçek bulanıklığı `NSVisualEffectView` ile, Windows composition/DWM ile, Wayland ise compositor bulanıklık protokolü ile uygular. Destek olmadığında `Blurred` saydam gibi davranabilir. Kök UI opak çizdiğinde bulanıklık görünmez kalır.

**Platform farklarının soyutlanacağı yer.** Davranış pencere ile ilgiliyse GPUI `Platform` ve `PlatformWindow` katmanına bağlanır. Zed UI görünümüyle ilgiliyse `PlatformStyle::platform()` ve `platform_title_bar` bileşenleri kullanılır. Ayar farkı gerekiyorsa `settings_content` şeması ve `settings` dönüşümleri devreye girer.

## Dosya Yoluna Göre Ne Nerede?

Aşağıdaki liste, rehberde anlatılan kavramların kodda hangi dosyada bulunduğunu tek bakışta verir. Yeni bir bileşen yazılırken benzer örneğin nerede olduğunu hızla bulmaya yarar.

- Pencere açma API'si: `crates/gpui/src/app.rs::open_window`
- Pencere seçenekleri: `crates/gpui/src/platform.rs::WindowOptions`
- Platform penceresi sözleşmesi: `crates/gpui/src/platform.rs::PlatformWindow`
- Pencere sarmalayıcı metotları: `crates/gpui/src/window.rs`
- Element ve çizim trait'leri: `crates/gpui/src/element.rs`, `view.rs`
- Style fluent API: `crates/gpui/src/styled.rs`
- Interactivity fluent API: `crates/gpui/src/elements/div.rs`
- Platform seçimi: `crates/gpui_platform/src/gpui_platform.rs`
- macOS pencere davranışı: `crates/gpui_macos/src/window.rs`
- Windows pencere davranışı: `crates/gpui_windows/src/window.rs`, `events.rs`
- Linux Wayland davranışı: `crates/gpui_linux/src/linux/wayland/window.rs`
- Linux X11 davranışı: `crates/gpui_linux/src/linux/x11/window.rs`
- Zed ana window options: `crates/zed/src/zed.rs::build_window_options`
- Zed platform başlık çubuğu: `crates/platform_title_bar/src/platform_title_bar.rs`
- Linux kontroller: `crates/platform_title_bar/src/platforms/platform_linux.rs`
- Windows kontroller: `crates/platform_title_bar/src/platforms/platform_windows.rs`
- Workspace istemci süslemesi: `crates/workspace/src/workspace.rs::client_side_decorations`
- Zed başlık çubuğu kompozisyonu: `crates/title_bar/src/title_bar.rs`
- Tema arka plan görünüşü: `crates/theme/src/theme.rs`, `crates/theme_settings/src/theme_settings.rs`, `crates/settings/src/content_into_gpui.rs`
- UI bileşen dışa aktarım listesi: `crates/ui/src/components.rs`
- UI input: `crates/ui_input/src/ui_input.rs`, `input_field.rs`
