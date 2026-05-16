# Referans ve doğrulama

Detaylı public API envanteri ve kaynak doğrulama komutları ana okuma akışını bölmemesi için bu başvuru bölümünde toplandı. Önceki bölümler bir kez okunduktan sonra, günlük geliştirme sırasında bu dosya hızlı referans olarak kullanılır. "Şu fonksiyonun imzası neydi?" veya "bu davranış kaynakta hangi satırda?" gibi soruların cevabı burada bulunur.

## 21. Public API envanteri

Bu bölümde `pub` anahtar sözcüğünün iki farklı anlamı ayrılır. Bu ayrım gözden kaçarsa "neden bu tipi import edemiyorum?" sorusu sıkça gündeme gelir:

- **Dış API:** Başka crate'lerden path üzerinden doğrudan erişilebilen yüzeydir. Bu tipler dışarıdan kullanılmak üzere kasıtlı olarak açılmıştır.
- **Lexical `pub`:** Kaynakta `pub` yazsa da, tipi içeren modül crate kökünde private olabilir. Bu durumda öğeye yalnızca crate içinden ulaşılır. Yani `pub` işareti tek başına dış erişim sağlamaz.

Bu ayrımın iyi örneklerinden biri `system_window_tabs.rs` içindeki `SystemWindowTabs` tipidir. Kaynakta `pub struct` olarak yazılmıştır. Fakat modül crate kökünde `mod system_window_tabs;` biçiminde, yani private olarak bağlanır. Bu yüzden dışarıdan `platform_title_bar::system_window_tabs::SystemWindowTabs` yoluyla erişilemez. Crate dışına açılan parçalar yalnızca root dosyadaki `pub use system_window_tabs::{...}` satırlarıdır (`platform_title_bar.rs:24-26`).

### Crate kökü (`platform_title_bar`)

| Dış API | İmza / tanım | Not |
| :-- | :-- | :-- |
| `pub mod platforms` | `platform_title_bar.rs:1` | `platform_title_bar::platforms::{platform_linux, platform_windows}` path'ini açar. **Cfg-gate yoktur**: `platforms.rs` her iki alt modülü de koşulsuz `pub mod` ile expose eder (`platforms.rs:1-2`). Yani Windows derlemesinde dahi `platform_title_bar::platforms::platform_linux::LinuxWindowControls` derlenir; runtime seçimi `PlatformStyle::platform()` ile yapılır. |
| `pub use system_window_tabs::{...}` | `DraggedWindowTab`, `MergeAllWindows`, `MoveTabToNewWindow`, `ShowNextWindowTab`, `ShowPreviousWindowTab` (`platform_title_bar.rs:24-26`) | `SystemWindowTabs` re-export edilmez. |
| `pub struct PlatformTitleBar` | Private alanlar: `id: ElementId`, `platform_style: PlatformStyle`, `children: SmallVec<[AnyElement; 2]>`, `should_move: bool`, `system_window_tabs: Entity<SystemWindowTabs>` (**strong**), `button_layout: Option<WindowButtonLayout>`, `multi_workspace: Option<WeakEntity<MultiWorkspace>>` (**weak**) (`platform_title_bar.rs:28-36`) | Alanlara dışarıdan erişim yok. **Ownership farkı**: `system_window_tabs` strong `Entity` — titlebar tabs alt-entity'yi sahiplenir ve drop edildiğinde onu da sürükler. `multi_workspace` weak — workspace bağımsız yaşar; titlebar sadece gözlemler. Aksini yapmak (workspace'i strong tutmak) **ownership cycle** üretir. |
| `PlatformTitleBar::new` | `pub fn new(id: impl Into<ElementId>, cx: &mut Context<Self>) -> Self` (`platform_title_bar.rs:39`) | `SystemWindowTabs::new()` ile internal tab entity oluşturur. |
| `with_multi_workspace` | `pub fn with_multi_workspace(mut self, multi_workspace: WeakEntity<MultiWorkspace>) -> Self` (`platform_title_bar.rs:54`) | Builder tarzı ilk bağlantı. |
| `set_multi_workspace` | `pub fn set_multi_workspace(&mut self, multi_workspace: WeakEntity<MultiWorkspace>)` (`platform_title_bar.rs:59`) | Sonradan sidebar state kaynağı bağlar. |
| `title_bar_color` | `pub fn title_bar_color(&self, window: &mut Window, cx: &mut Context<Self>) -> Hsla` (`platform_title_bar.rs:63`) | Linux/FreeBSD'de aktif/pasif ve move state'ine bakar; diğer platformlarda active/inactive ayrımı yapmaz. |
| `set_children` | `pub fn set_children<T>(&mut self, children: T) where T: IntoIterator<Item = AnyElement>` (`platform_title_bar.rs:75-77`) | Render'da `mem::take` ile tüketildiği için her render'da tekrar çağrılır. |
| `set_button_layout` | `pub fn set_button_layout(&mut self, button_layout: Option<WindowButtonLayout>)` (`platform_title_bar.rs:82`) | Sadece Linux + `Decorations::Client` olduğunda `effective_button_layout` tarafından kullanılır. |
| `PlatformTitleBar::init` | `pub fn init(cx: &mut App)` (`platform_title_bar.rs:100`) | Internal `SystemWindowTabs::init(cx)` çağrısıdır. |
| `is_multi_workspace_enabled` | `pub fn is_multi_workspace_enabled(cx: &App) -> bool` (`platform_title_bar.rs:112`) | Zed'de `DisableAiSettings` tersine bağlı feature flag. |
| `render_left_window_controls` | `pub fn render_left_window_controls(button_layout: Option<WindowButtonLayout>, close_action: Box<dyn Action>, window: &Window) -> Option<AnyElement>` (`platform_title_bar.rs:121-125`) | Yalnız Linux/FreeBSD + CSD; `button_layout.left[0]` boşsa `None`. |
| `render_right_window_controls` | `pub fn render_right_window_controls(button_layout: Option<WindowButtonLayout>, close_action: Box<dyn Action>, window: &Window) -> Option<AnyElement>` (`platform_title_bar.rs:150-154`) | Linux/FreeBSD + CSD'de layout kullanır, Windows'ta `WindowsWindowControls::new(height)`, macOS'ta `None`. |

`PlatformTitleBar` tipinin render davranışı çoğu zaman public API imzalarından daha kritiktir. Gerçek port hatalarının büyük kısmı imza farklarından değil, render içindeki ince akışlardan doğar. Aşağıdaki maddeler bu davranışın kaçırılmaması gereken noktalarını sıralar:

- `close_action`, kaynakta sabit olarak `Box::new(workspace::CloseWindow)` ifadesiyle oluşturulur (`platform_title_bar.rs:189`). Buna karşılık serbest render fonksiyonları (`render_left_window_controls`, `render_right_window_controls`) close action'ı dışarıdan `Box<dyn Action>` olarak alır; bu, davranışın yapılandırılabilir olduğu tek noktadır.
- `button_layout`, private bir helper olan `effective_button_layout(...)` üzerinden çözülür. Bu çözüm yalnızca Linux + CSD durumunda yapılır ve mantığı `self.button_layout.or_else(|| cx.button_layout())` şeklindedir (`platform_title_bar.rs:86-98`). Yani önce uygulamadan gelen değere bakılır, yoksa platforma sorulur.
- Ana titlebar yüzeyi `WindowControlArea::Drag` ile etiketlenir (`platform_title_bar.rs:195-197`). Buna ek olarak macOS'ta çift tıklama `titlebar_double_click` çağrısına, Linux/FreeBSD'de ise `zoom_window` çağrısına bağlanır (`platform_title_bar.rs:225-237`).
- Sol veya sağ sidebar açıkken, sidebar tarafındaki pencere kontrolleri gizlenir (`platform_title_bar.rs:241-257`, `294-307`). Bu, görsel çakışmayı önlemek içindir.
- Linux CSD aktifken ve `supported_controls.window_menu` `true` ise, başlık çubuğunda sağ tıklama `window.show_window_menu(ev.position)` çağrısını tetikler (`platform_title_bar.rs:309-315`).
- CSD render'ında titlebar, kendi üst köşelerini de düzeltir: tiled olmayan ve sidebar tarafından kapatılmayan üst köşelere `theme::CLIENT_SIDE_DECORATION_ROUNDING` uygulanır; ardından şeffaf köşe boşluğunu kapatmak için `.mt(px(-1.)).mb(px(-1.)).border(px(1.))` ölçüleri ve `border_color(titlebar_color)` rengi eklenir (`platform_title_bar.rs:262-279`).
- Render zincirinin sonunda, internal `SystemWindowTabs` entity'si child olarak eklenir (`platform_title_bar.rs:322-325`). Yani sekme çubuğu titlebar'ın görsel olarak hemen altında çizilir.

**Render pipeline sıralaması**, port hedefinde hangi adımların hangi sırayla uygulanacağını gösterir. `.map(\|this\|` deseniyle yapılan awk taraması, `render` fonksiyonu gövdesinde birbirini takip eden dört ayrı dönüşüm aşamasını ortaya koyar:

| Aşama | Yer | İş |
| :-- | :-- | :-- |
| 1 | `platform_title_bar.rs:199-221` | **Drag tespiti**: `on_mouse_down/up/down_out/move` zincirinin `should_move` bayrağıyla `window.start_window_move()` tetiklemesi. Bu aşama her zaman ilk uygulanır; sonraki stage'ler bu state'in üstüne kurulur. |
| 2 | `platform_title_bar.rs:222-239` | **ID + çift tıklama**: `this.id(self.id.clone())` + Mac/Linux platform branch'leri `on_click`'i `event.click_count() == 2` kontrolüyle `titlebar_double_click` / `zoom_window`'a yönlendirir. Windows branch'i yoktur. |
| 3 | `platform_title_bar.rs:240-261` | **Sol kenar padding/kontrolleri**: 4-yollu seçim — fullscreen ise `pl_2`, Mac + show_left_controls ise `pl(TRAFFIC_LIGHT_PADDING)`, Linux + CSD + dolu sol layout ise `render_left_window_controls(...)` child, aksi halde `pl_2` fallback. |
| 4 | `platform_title_bar.rs:262-280` | **Decorations branch**: `match decorations` — `Server` ise `el` olduğu gibi; `Client { tiling, .. }` ise tiled olmayan üst köşelere `rounded_tr/tl(CLIENT_SIDE_DECORATION_ROUNDING)` + `mt(-1)/mb(-1)/border(1)` transparent gap düzeltmesi. |

Bu dört aşamalı zincirden sonra `.bg(titlebar_color).content_stretch().child(div().children(children))` ifadesi gelir. Bu ifade başlık çubuğunun ana içeriğini oluşturur. Ardından `.when(!is_fullscreen, |title_bar| ...)` zinciri sağ kontrolleri ve window_menu sağ-tık handler'ını ekler.

Burada önemli bir kural vardır: aşamalar **birbirleriyle commutative değildir**. Yani 3. aşamadaki sol padding seçimi, 4. aşamadaki corner rounding'in yatay hizalamasını doğrudan etkiler. Sıralamayı değiştirmek gözle hemen fark edilmeyen hizalama hataları üretebilir.

`PlatformTitleBar.children` alanı `SmallVec<[AnyElement; 2]>` tipinde tanımlıdır (`platform_title_bar.rs:31`). Yapı iki element için stack-inline kapasite ayırır. Bunun nedeni Zed'in tipik kullanım kalıbının **sol grup + sağ grup** olmasıdır (rehberin Konu 16'sındaki örneğe bakılabilir). Bu yapıya ikiden fazla element verildiğinde heap allocation yapılır. Bu yüzden içeriği iki gruba toplamak hem ergonomik açıdan tutarlı kalır hem de allocation sayısını azaltır.

### `platforms::platform_linux`

| Dış API | İmza / tanım | Not |
| :-- | :-- | :-- |
| `LinuxWindowControls` | `pub struct LinuxWindowControls` private alanlı (`platform_linux.rs:7-11`) | `#[derive(IntoElement)]`; dışarıdan alan set edilemez. |
| `LinuxWindowControls::new` | `pub fn new(id: &'static str, buttons: [Option<WindowButton>; MAX_BUTTONS_PER_SIDE], close_action: Box<dyn Action>) -> Self` (`platform_linux.rs:14-18`) | Layout slotlarını ve close action'ı saklar. Render'da `WindowControls` capability'sine göre minimize/maximize filtrelenir, **`WindowButton::Close => true` arm'ı koşulsuzdur** (`platform_linux.rs:35-39`); `supported_controls.close` false olsa bile close butonu render edilir. |
| `WindowControlType` | `pub enum WindowControlType { Minimize, Restore, Maximize, Close }` (`platform_linux.rs:84-90`) | Variant sırası kaynakta `Minimize, Restore, Maximize, Close`; `WindowButton::Maximize` runtime'da pencere maximized ise `Restore` ikonuna çevrilir. |
| `WindowControlType::icon` | `pub fn icon(&self) -> IconName` (`platform_linux.rs:97`) | `GenericMinimize`, `GenericRestore`, `GenericMaximize`, `GenericClose` döner. |
| `WindowControlStyle` | `pub struct WindowControlStyle` private alanlı (`platform_linux.rs:107-113`) | Alanlar public değildir; sadece builder yüzeyi var. |
| `WindowControlStyle::default` | `pub fn default(cx: &mut App) -> Self` (`platform_linux.rs:116`) | `Default` trait impl'i değildir; argümansız `WindowControlStyle::default()` derlenmez. |
| `background` | `pub fn background(mut self, color: impl Into<Hsla>) -> Self` (`platform_linux.rs:129`) | Builder. |
| `background_hover` | `pub fn background_hover(mut self, color: impl Into<Hsla>) -> Self` (`platform_linux.rs:136`) | Builder. |
| `icon` | `pub fn icon(mut self, color: impl Into<Hsla>) -> Self` (`platform_linux.rs:143`) | Builder. |
| `icon_hover` | `pub fn icon_hover(mut self, color: impl Into<Hsla>) -> Self` (`platform_linux.rs:150`) | Builder. |
| `WindowControl` | `pub struct WindowControl` private alanlı (`platform_linux.rs:156-162`) | `#[derive(IntoElement)]`; public setter yok. |
| `WindowControl::new` | `pub fn new(id: impl Into<ElementId>, icon: WindowControlType, cx: &mut App) -> Self` (`platform_linux.rs:165`) | `close_action: None`; close için kullanılırsa click handler panik atar. |
| `WindowControl::new_close` | `pub fn new_close(id: impl Into<ElementId>, icon: WindowControlType, close_action: Box<dyn Action>, cx: &mut App) -> Self` (`platform_linux.rs:176-181`) | `close_action.boxed_clone()` saklar (`platform_linux.rs:188`). |
| `WindowControl::custom_style` | `pub fn custom_style(id: impl Into<ElementId>, icon: WindowControlType, style: WindowControlStyle) -> Self` (`platform_linux.rs:193-197`) | `#[allow(unused)]`; crate içinde çağrılmıyor, close action `None`. |

Linux davranışının kritik private helper'ı `fn create_window_button(...)`, `platform_linux.rs:56-62` aralığında yer alır. Bu fonksiyon dış API değildir; buna rağmen davranış paritesi için zorunlu bir karar noktasıdır. İçinde `WindowButton::Close` branch'i yalnızca `WindowControl::new_close(...)` çağrısını kullanır (`platform_linux.rs:77-79`). Close butonunu `WindowControl::new(...)` ile üretmeye çalışmak da sessiz bir no-op değildir. Runtime'da `expect("Use WindowControl::new_close() for close control.")` ile panik fırlatılır (`platform_linux.rs:235-239`). Bu, "yanlış constructor ile close üretme" hatasının sessizce geçmemesi için konmuş bir güvenlik önlemidir.

**Derive ve clonability haritası** awk `#[derive(...)]` taramasıyla çıkarılır. `rg` satır-bazlı eşleşmede struct ile derive arasındaki bağı tek başına göstermez:

| Tip | Derive set | Önemi |
| :-- | :-- | :-- |
| `WindowControlType` | `Debug, PartialEq, Eq, PartialOrd, Ord, Hash, Clone, Copy` (`platform_linux.rs:84`) | `Hash` + `Copy` — `HashMap` key olabilir, value semantiği. `PartialOrd/Ord` deklarasyon sırasını kullanır: `Minimize < Restore < Maximize < Close`. |
| `WindowControlStyle` | **Hiçbiri yok** (`platform_linux.rs:107-108`) | `Clone`, `Copy`, `Default` impl'i **yoktur**; her yeni instance için `WindowControlStyle::default(cx)` çağrısı gerekir. `Default` trait olmadığı için generic koddan `D: Default` ile alınamaz. |
| `LinuxWindowControls` | `IntoElement` (`platform_linux.rs:6`) | `RenderOnce` yüzeyi; build edilip child olarak verilir, alanları tekrar erişilemez. |
| `WindowControl` | `IntoElement` (`platform_linux.rs:156`) | Aynı RenderOnce yüzeyi. |
| `WindowsWindowControls` | `IntoElement` (`platform_windows.rs:5`) | Aynı. |
| `DraggedWindowTab` | `Clone` (`system_window_tabs.rs:28`) | Drag payload tipi; clone'lanabilir ama `Copy` değil. |
| `PlatformTitleBar`, `SystemWindowTabs` | Yok | `Entity` ile yönetilir; trait impl'leri (`Render`, `ParentElement`) yüzeyi sağlar. |

**`WindowControlStyle::default(cx)` çağrısı hangi tema token'larını okur?** Bu soru port hedefinin tema sistemini doğrudan şekillendirir. Cevap `platform_linux.rs:117-124` aralığındadır ve aşağıdaki dört token'a karşılık gelir:

| Style alanı | Tema token |
| :-- | :-- |
| `background` | `colors.ghost_element_background` |
| `background_hover` | `colors.ghost_element_hover` |
| `icon` | `colors.icon` |
| `icon_hover` | `colors.icon_muted` |

Builder zincirinde override edilmeyen alanlar bu default değerlerinde kalır. Pratik sonuç nettir: port hedefinin tema sistemi **yukarıdaki dört token'ı mutlaka sağlamalıdır**. Aynı liste rehberin diğer bölümlerinde de yer alır. Bunlardan biri eksik kalırsa Linux pencere butonlarının görünümü beklenmeyen renklere düşer.

**Sabit ölçüler.** Linux render closure'larında piksel paritesini korumak için aşağıdaki sabit değerlerin port hedefinde de aynı biçimde kullanılması gerekir:

| Yer | Değer | Kaynak |
| :-- | :-- | :-- |
| `LinuxWindowControls` buton container gap'i | `.gap_3()` (12px @ default rem) | `platform_linux.rs:48` |
| `LinuxWindowControls` buton container yatay padding | `.px_3()` | `platform_linux.rs:49` |
| `WindowControl` buton boyutu | `.w_5().h_5()` (≈20px) | `platform_linux.rs:222-223` |
| `WindowControl` köşe yuvarlama | `.rounded_2xl()` | `platform_linux.rs:221` |

**`Box<dyn Action>` klonlama zinciri.** Close action'ının render sırasında kaç defa klonlandığı kolayca gözden kaçar. Performansı düşünen bir portta bu sayı önemlidir. Klonlama yerlerini görmek için awk üzerinde `boxed_clone()` deseniyle tarama yapılır. `rg` her satırı ayrı ayrı bulur, ama bu çağrıların oluşturduğu zinciri bütün halinde toplamaz. Zincirin adımları şöyledir:

| Adım | Yer | Tetikleyici | Çağrı |
| :-- | :-- | :-- | :-- |
| 1 | `platform_title_bar.rs:189` | `render()` başı | `let close_action = Box::new(workspace::CloseWindow);` (ilk Box üretimi, klon değil) |
| 2 | `platform_title_bar.rs:251` | Fullscreen değil, macOS trafik ışığı padding branch'i seçilmedi ve `show_left_controls` true | `close_action.as_ref().boxed_clone()` — `render_left_window_controls` argümanı |
| 3 | `platform_title_bar.rs:302` | `show_right_controls` true ve `!is_fullscreen` | `close_action.as_ref().boxed_clone()` — `render_right_window_controls` argümanı |
| 4 | `platform_linux.rs:78` | İlgili tarafta `WindowButton::Close` slot'u var | `create_window_button` → `WindowControl::new_close(..., close_action.boxed_clone(), cx)` |
| 5 | `platform_linux.rs:188` | `new_close` gövdesi | `close_action: Some(close_action.boxed_clone())` (parametre move'lanmak yerine yeniden klonlanır) |
| 6 | `platform_linux.rs:239` | Close butonuna **click anı** | `.expect(...).boxed_clone()` — `window.dispatch_action(...)` argümanı |

Adım 2 ve 3'teki klonlamalar render fonksiyonları çağrılmadan önce yapılır. Bunun anlamı şudur: ilgili tarafta close butonu hiç üretilmese bile klonlama maliyeti doğar. Fullscreen durumunda adım 2 ve 3 atlanır; çünkü fullscreen'de yan kontroller render'a girmez. macOS tarafında sol kenar genellikle trafik ışığı padding branch'i ile çözüldüğü için adım 2 burada çalışmaz. Ancak fullscreen değilse adım 3 hâlâ `render_right_window_controls(...)` çağrısı öncesinde bir klon üretir ve fonksiyon Mac'te zaten `None` döner. Yani Mac'te boşa bir klonlama maliyeti vardır. Windows tarafında `WindowsWindowControls` close action'ı kullanmaz; buna rağmen fullscreen olmadığı sürece adım 2 ve 3 çalışabilir.

Adım 4 ve 5 yalnızca Linux CSD ortamında ve close butonunun bulunduğu tarafta tetiklenir. Tipik bir Linux GNOME render'ında, yani close sağda ve sidebar kapalıyken, adımlar 2 + 3 + 4 + 5 birlikte çalışır. Bu da **render başına 4 adet `boxed_clone`** anlamına gelir. Adım 6 ise yalnızca close butonuna tıklandığında ek olarak **+1** klon üretir.

`Box<dyn Action>::boxed_clone()` çağrısı trait üzerinden v-table dispatch yapan bir klon işlemidir (`Action::boxed_clone(&self) -> Box<dyn Action>`). Bu klonun maliyeti concrete tipin `Clone` implementasyonuna bağlıdır. `workspace::CloseWindow` gibi unit struct'lar için maliyet neredeyse sıfırdır. Alan taşıyan action'larda ise bu maliyet her render'da çarpan etkisiyle artar. Port hedefinde action tipini alan taşımayacak şekilde tasarlamak bu yolu önemli ölçüde hızlandırır. Ayrıca adım 5 optimize edilebilir: parametre hareket ettirilebilir, yani `Some(close_action)` ile move'lanabilir biçimde alınırsa, bir klon adımı tamamen ortadan kalkar.

### `platforms::platform_windows`

| Dış API | İmza / tanım | Not |
| :-- | :-- | :-- |
| `WindowsWindowControls` | `pub struct WindowsWindowControls { button_height: Pixels }` private alanlı (`platform_windows.rs:6-8`) | `#[derive(IntoElement)]`; dışarıdan sadece constructor var. |
| `WindowsWindowControls::new` | `pub fn new(button_height: Pixels) -> Self` (`platform_windows.rs:11`) | Render'da minimize, maximize/restore ve close caption butonlarını üretir. |

`WindowsCaptionButton` tipi crate dışına açılmamıştır (`platform_windows.rs:58-64`); yalnızca crate içinde kullanılır. Tipin private metotları `id()`, `icon()` ve `control_area()` sırasıyla şu üç şeyi döner: stable element id, Segoe glyph karakteri ve `WindowControlArea::{Min, Max, Close}` varyantlarından biri (`platform_windows.rs:66-94`). Windows butonları, Linux'taki gibi click handler çağırmaz; bunun yerine `.window_control_area(...)` ile hit-test alanı üretirler (`platform_windows.rs:124-135`) ve davranış platform caption katmanı tarafından sürdürülür.

**Segoe Fluent Icons glyph kodları.** Windows native paritesini korumak için aşağıdaki dört Unicode codepoint birebir aynen kullanılır:

| Variant | Kodepoint | Kaynak |
| :-- | :-- | :-- |
| `Minimize` | `\u{e921}` | `platform_windows.rs:80` |
| `Restore` | `\u{e923}` | `platform_windows.rs:81` |
| `Maximize` | `\u{e922}` | `platform_windows.rs:82` |
| `Close` | `\u{e8bb}` | `platform_windows.rs:83` |

Bu glyph'lerin gösterilmesi için kullanılan font, çalışılan Windows build'ine göre değişir. Font seçimi `WindowsWindowControls::get_font()` fonksiyonunda yapılır (`platform_windows.rs:16/21`). Build numarası 22000 veya üzerindeyse `"Segoe Fluent Icons"` font'u tercih edilir; build numarası bunun altındaysa `"Segoe MDL2 Assets"` kullanılır. Port hedefinin çalıştığı sistemde bu font'lar yoksa glyph'ler boş kareler olarak render olur. Bu durumda SVG ikon fallback'i gerekebilir.

**Renk sabitleri.** Windows pencere butonlarının hover ve active durumlarındaki renkleri `platform_windows.rs:99-122` aralığında sabit olarak tanımlanır. Aşağıdaki tablo bu sabitleri özetler:

| Buton | Hover bg | Hover fg | Active bg | Active fg |
| :-- | :-- | :-- | :-- | :-- |
| `Close` | `Rgba { r: 232/255, g: 17/255, b: 32/255, a: 1.0 }` = `#E81120` | `gpui::white()` | `color.opacity(0.8)` | `white().opacity(0.8)` |
| Diğerleri | `theme.ghost_element_hover` | `theme.text` | `theme.ghost_element_active` | `theme.text` |

Burada özellikle dikkat çeken nokta close butonunun kırmızısıdır: `#E81120` **temadan değil, doğrudan koddan gelir**. Bu renk Microsoft'un Windows title bar kapatma kırmızısıdır ve native hissi korumak için sabit tutulur. Port hedefinin tema sistemi farklı bir close vurgu rengi istiyorsa bu sabit override edilmelidir. Aksi halde tema değişse bile close hover'ı Microsoft kırmızısında kalır.

**Sabit ölçüler** (Windows caption butonu):

| Yer | Değer | Kaynak |
| :-- | :-- | :-- |
| Caption buton genişliği | `.w(px(36.))` (36px) | `platform_windows.rs:129` |
| Glyph metin boyutu | `.text_size(px(10.0))` (10px) | `platform_windows.rs:131` |
| Buton yüksekliği | `WindowsWindowControls::new(button_height)`'ten `.h_full()` ile yayılır | `platform_windows.rs:11, 130` |
| Mouse propagation | `.occlude()` (alt katmanlara mouse event sızdırmaz) | `platform_windows.rs:128` |

### Root'tan re-export edilen system tab action'ları

Native pencere sekmeleri için gerekli action'lar `actions!(window, [...])` makrosu ile üretilir (`system_window_tabs.rs:18-26`). Bu makro dört unit struct'ı tek seferde tanımlar. Makro çıktısı şu derive setini otomatik ekler: `Clone`, `PartialEq`, `Default`, `Debug` ve `gpui::Action` (`gpui/src/action.rs:24-40`). Üretilen tipler şunlardır:

- `pub struct ShowNextWindowTab;`
- `pub struct ShowPreviousWindowTab;`
- `pub struct MergeAllWindows;`
- `pub struct MoveTabToNewWindow;`

Bu dört action `platform_title_bar.rs:24-26` aralığında crate kökünden re-export edilir. Ardından Zed'in `title_bar` crate'i aynı adları kendi seviyesinde bir kez daha re-export eder (`title_bar/src/title_bar.rs:13-16`). Bu çift re-export, tüketicilerin tipleri kendileri için en uygun yoldan import etmesine imkan verir.

`DraggedWindowTab` tipi de aynı şekilde crate kökünden re-export edilir. İmzası şöyledir:

```rust
#[derive(Clone)]
pub struct DraggedWindowTab {
    pub id: WindowId,
    pub ix: usize,
    pub handle: AnyWindowHandle,
    pub title: String,
    pub width: Pixels,
    pub is_active: bool,
    pub active_background_color: Hsla,
    pub inactive_background_color: Hsla,
}
```

Kaynak konumu `system_window_tabs.rs:28-38` aralığıdır. Bu tip aynı zamanda drag preview olarak kendini çizmek için `Render` trait'ini implement eder (`system_window_tabs.rs:498-528`). Yani hem veri taşır hem de görsel sunum sağlar.

### Lexical `pub` ama dış API olmayan parçalar

Aşağıdaki tablo, kaynakta `pub` görünmesine rağmen crate sınırının dışından erişilemeyen parçaları listeler. Bunları görmek port hedefinde davranış paritesi için önemlidir; çünkü Zed bu yardımcı parçaları kendi içinde aktif olarak kullanır.

| Öğe | Neden dış API değil? | Kullanım |
| :-- | :-- | :-- |
| `SystemWindowTabs` | `system_window_tabs` modülü private (`platform_title_bar.rs:2`) | `PlatformTitleBar::new` içinde entity olarak oluşturulur (`platform_title_bar.rs:39-42`) ve render sonunda child yapılır (`platform_title_bar.rs:322-325`). |
| `SystemWindowTabs::new` | Private modül içinde lexical `pub` (`system_window_tabs.rs:47`) | Internal scroll handle, ölçülen tab genişliği ve `last_dragged_tab` başlangıcı. |
| `SystemWindowTabs::init` | Private modül içinde lexical `pub` (`system_window_tabs.rs:55`) | `PlatformTitleBar::init(cx)` üzerinden çağrılır (`platform_title_bar.rs:100-101`). |
| `SystemWindowTabs::render_tab` | Private method (`system_window_tabs.rs:142`) | Tab elementlerini, drag/drop'u, middle-click close'u, close button'u ve context menu'yü kurar. |
| `handle_tab_drop` | Private method (`system_window_tabs.rs:358`) | Sadece same-bar drop reorder: `SystemWindowTabController::update_tab_position(...)`. |
| `handle_right_click_action` | Private method (`system_window_tabs.rs:362`) | Context menu action'larını hedef tab penceresinde çalıştırır. |

Bu ayrımın port hedefi için anlamı şudur: `SystemWindowTabs` tipi dış API olarak taşınmak zorunda değildir. Tüketicilere doğrudan göstermemek serbestlik sağlar. Ancak davranış mirror edilecekse private event router'ları da ayrı ayrı incelenmelidir; özellikle `handle_tab_drop` ve `handle_right_click_action` önemlidir. Yalnızca public yüzey taşınırsa bu router'ların yaptığı iş atlanır ve sekme davranışı eksik kalır.

### GPUI native tab destek yüzeyi

Aşağıdaki tipler `platform_title_bar` crate'inden değil, `gpui` crate'inden gelir. Buna rağmen native tab davranışının state kaynağını oluşturdukları için bu envanterde yer almaları gerekir. Port hedefi, sekme controller'ını fiilen bu yüzey üzerinden işletir:

| API | İmza / tanım | Not |
| :-- | :-- | :-- |
| `SystemWindowTab` | `#[doc(hidden)] pub struct SystemWindowTab { pub id, pub title, pub handle, pub last_active_at }` (`gpui/src/app.rs:276-283`) | Platform native tab metadata'sı. |
| `SystemWindowTab::new` | `pub fn new(title: SharedString, handle: AnyWindowHandle) -> Self` (`gpui/src/app.rs:287`) | `id` handle'dan, `last_active_at` `Instant::now()` ile gelir. |
| `SystemWindowTabController::new` | `pub fn new() -> Self` (`gpui/src/app.rs:308`) | Empty global controller. |
| `SystemWindowTabController::init` | `pub fn init(cx: &mut App)` (`gpui/src/app.rs:316`) | Global controller'ı resetler. |
| `tab_groups` | `pub fn tab_groups(&self) -> &FxHashMap<usize, Vec<SystemWindowTab>>` (`gpui/src/app.rs:321`) | Grupları doğrudan ref olarak verir. |
| `tabs` | `pub fn tabs(&self, id: WindowId) -> Option<&Vec<SystemWindowTab>>` (`gpui/src/app.rs:380`) | Verilen pencereyle aynı gruptaki tab listesi. |
| `init_visible` | `pub fn init_visible(cx: &mut App, visible: bool)` (`gpui/src/app.rs:387`) | Sadece `visible` `None` ise set eder. |
| `is_visible` | `pub fn is_visible(&self) -> bool` (`gpui/src/app.rs:395`) | `None` ise `false`. |
| `set_visible` | `pub fn set_visible(cx: &mut App, visible: bool)` (`gpui/src/app.rs:400`) | Platform toggle callback'i kullanır. |
| `update_last_active` | `pub fn update_last_active(cx: &mut App, id: WindowId)` (`gpui/src/app.rs:406`) | Aktif pencere değişiminde çağrılır. |
| `update_tab_position` | `pub fn update_tab_position(cx: &mut App, id: WindowId, ix: usize)` (`gpui/src/app.rs:418`) | Same-bar drag/drop reorder. |
| `update_tab_title` | `pub fn update_tab_title(cx: &mut App, id: WindowId, title: SharedString)` (`gpui/src/app.rs:432`) | Workspace title güncellemesinde kullanılır. |
| `add_tab` | `pub fn add_tab(cx: &mut App, id: WindowId, tabs: Vec<SystemWindowTab>)` (`gpui/src/app.rs:456`) | Platform tab listesinden controller grubu kurar. |
| `remove_tab` | `pub fn remove_tab(cx: &mut App, id: WindowId) -> Option<SystemWindowTab>` (`gpui/src/app.rs:489`) | Boş kalan grupları temizler. |
| `move_tab_to_new_window` | `pub fn move_tab_to_new_window(cx: &mut App, id: WindowId)` (`gpui/src/app.rs:504`) | Controller state'inde yeni grup açar; platform move ayrıca çağrılır. |
| `merge_all_windows` | `pub fn merge_all_windows(cx: &mut App, id: WindowId)` (`gpui/src/app.rs:515`) | Controller gruplarını tek grupta birleştirir; platform merge ayrıca çağrılır. |
| `select_next_tab` | `pub fn select_next_tab(cx: &mut App, id: WindowId)` (`gpui/src/app.rs:533`) | Sonraki handle'ı `activate_window()` ile aktive eder. |
| `select_previous_tab` | `pub fn select_previous_tab(cx: &mut App, id: WindowId)` (`gpui/src/app.rs:548`) | Önceki handle'ı aktive eder. |
| `get_next_tab_group_window` | `pub fn get_next_tab_group_window(cx: &mut App, id: WindowId) -> Option<&AnyWindowHandle>` (`gpui/src/app.rs:326`) | Grup id sırası `HashMap` key sırasından gelir; kaynakta TODO var. |
| `get_prev_tab_group_window` | `pub fn get_prev_tab_group_window(cx: &mut App, id: WindowId) -> Option<&AnyWindowHandle>` (`gpui/src/app.rs:351`) | Aynı key sırası belirsizliği geçerlidir. |

### Zed `title_bar` crate'inin public tüketim yüzeyi

Zed uygulaması platform crate'ini iki yoldan tüketir. Bazı parçalar doğrudan kök API olarak kullanılır; bazıları ise `title_bar` crate'i üzerinden re-export edilir. Aşağıdaki tablo bu yüzeyin ana parçalarını gösterir:

| API | İmza / tanım | Not |
| :-- | :-- | :-- |
| `pub mod collab` | `title_bar/src/title_bar.rs:2` | Platform titlebar değil, Zed collab UI helper modülü. |
| Platform re-export'ları | `pub use platform_title_bar::{ self, DraggedWindowTab, MergeAllWindows, MoveTabToNewWindow, PlatformTitleBar, ShowNextWindowTab, ShowPreviousWindowTab }` (`title_bar/src/title_bar.rs:13-16`) | Zed içi tüketiciler aynı action/tipleri `title_bar` üzerinden de alabilir. |
| `restore_banner` | `pub use onboarding_banner::restore_banner` (`title_bar/src/title_bar.rs:59`) | Product titlebar banner helper'ı. |
| `init` | `pub fn init(cx: &mut App)` (`title_bar/src/title_bar.rs:79`) | Platform titlebar init + per-workspace `TitleBar` entity kurulumu. |
| `TitleBar` | `pub struct TitleBar` private alanlı (`title_bar/src/title_bar.rs:150-163`) | Zed ürün başlığıdır; generic platform shell değildir. |
| `TitleBar::new` | `pub fn new(id: impl Into<ElementId>, workspace: &Workspace, multi_workspace: Option<WeakEntity<MultiWorkspace>>, window: &mut Window, cx: &mut Context<Self>) -> Self` (`title_bar/src/title_bar.rs:386-391`) | `PlatformTitleBar::new(...)` entity'sini oluşturur ve `observe_button_layout_changed` subscription'ı kurar (`title_bar.rs:442-455`). |
| Product helper'ları | `effective_active_worktree`, `render_restricted_mode`, `render_project_host`, `render_sign_in_button`, `render_user_menu_button` (`title_bar.rs:508`, `643`, `690`, `1160`, `1179`) | Zed'e özgü proje/kullanıcı UI yüzeyi; platform titlebar port API'si olarak kopyalanmamalıdır. |
| Ürün banner'ı | `OnboardingBanner::new(...)` | Feature flag'e bağlı duyuru/ürün mesajı katmanıdır; platform titlebar API'sine taşınmamalıdır. |
| Update bildirimi | `UpdateVersion::version_tooltip_message(...)` (`update_version.rs:66-75`) | Tooltip metnini `Update to Version: ...` biçiminde üretir. SHA için `short()` değil `full()` kullanılır. |

Zed'in ürün titlebar'ı, duyuru banner'larını da `TitleBar` katmanında yönetir. Pratik örnek şudur: Skills duyurusu `SkillsFeatureFlag` bayrağına bağlı olarak görünür hale gelir ve ilgili migration bilgi action'ını dispatch eder. `onboarding_banner.rs` dosyası artık "şimdilik kullanılmıyor" kabul edilmez; güncel kullanım Skills duyurusudur. Eski Claude Agent ve ACP banner kullanımları bu rehber için referans alınmaz. Port hedefinde benzer bir duyuru bileşeni gerekiyorsa, bu bileşenin yeri `AppTitleBar` child grubu olmalıdır. Bu sorumluluk platform kabuğuna eklenmez; çünkü duyuru içeriği ürünün diline aittir ve platform kabuğunu ürüne bağımlı hale getirir.

`OnboardingBanner` örneği, görünürlük koşulunu builder zincirinin sonundaki `.visible_when(|cx| cx.has_flag::<SkillsFeatureFlag>())` çağrısıyla alır (`title_bar/src/title_bar.rs:455-472`). Bu kalıp port hedefinde de kullanılabilir. Banner kurucusuna bir predicate kapanışı verilir. Bu kapanış her render geçişinde tekrar çağrılır ve `App`/`Context` üzerinden gelen feature flag ya da ayar durumuna göre banner tamamen gizlenebilir.

`title_bar.rs` içindeki bu çağrı, `feature_flags` crate'inden gelen `FeatureFlagAppExt` trait'i ile `cx.has_flag::<...>()` çağrısına dayanır. Port hedefinde aynı yardımcı yoksa benzer bir `AppSettings`/`AppFlags` API'si yeterlidir. `TitleBar::new` içinde banner `Some(...)` olarak kurulur; görünürlük kararı `visible_when` predicate'iyle alınır. Bu yüzden port hedefinde banner state'i ayrıca geçmiş bir başlangıç durumuna göre değil, doğrudan güncel feature flag veya ayar değerine göre yönetilmelidir.

`OnboardingBanner::new(...)` çağrısı, bu rehber yazıldığı sıradaki imzasıyla parametreleri şu sırayla alır: telemetri/dismiss kimliği olarak kullanılacak bir string (`"Skills Migration Announcement"`), `IconName::Sparkle`, banner üzerindeki sabit metin (`"Skills"`), opsiyonel ön ek (`Some("Introducing:".into())`) ve tıklama anında dispatch edilecek boxed action (`zed_actions::agent::OpenRulesToSkillsMigrationInfo.boxed_clone()`). Etiket migration sonucuna veya kullanıcının taşıyacak Rules içeriği olup olmamasına göre değişmez; migration'a özel özet modal içinde gösterilir. Port hedefinde eski metin varyantları için geriye uyumluluk katmanı tutulmaz. Somut içerik, ikon ve action ürünün ihtiyacına göre belirlenir.

`UpdateVersion` tarafında da eski tooltip formatı taşınmaz. `version_tooltip_message(...)` semantic version için `SemanticVersion::to_string()`, commit için `AppCommitSha::full()` çıktısını kullanır ve her iki durumda da sonucu `"Update to Version: {version}"` kalıbına sarar. `Downloading`, `Installing` ve `Updated` render kolları bu string'i `UpdateButton` kurucularına artık `tooltip` adıyla değil, doğrudan `version` değişkeni olarak geçirir. Port hedefinde kullanıcıya hâlâ kısa SHA göstermek isteniyorsa bu bilinçli bir ürün farkı olarak kaydedilir; Zed paritesi değildir.

`title_bar_settings.rs` içindeki `pub struct TitleBarSettings` tanımı `title_bar_settings.rs:5-15` aralığındadır. Bu tip private bir modülde kaldığı için crate dışı API değildir; yalnızca Zed'in kendi ayar sistemi içinde kullanılır. Kullanıcı ayarı tarafındaki dış veri tipi ise `settings_content::title_bar::WindowButtonLayoutContent` tipidir. Linux/FreeBSD'de bu tip, `pub fn into_layout(self) -> Option<WindowButtonLayout>` çağrısıyla doğrudan `WindowButtonLayout` değerine dönüştürülür (`settings_content/src/title_bar.rs:24-49`).

Aynı dosyada `settings_content::title_bar::TitleBarSettingsContent` de public bir ayar payload'ıdır. Şu alanları taşır ve hepsi `Option<...>` sarmalındadır: `show_branch_status_icon`, `show_onboarding_banner`, `show_user_picture`, `show_branch_name`, `show_project_items`, `show_sign_in`, `show_user_menu`, `show_menus` ve `button_layout` (`settings_content/src/title_bar.rs:83-126`). Runtime tarafındaki `TitleBarSettings` tipi bu payload'dan üretilir (`title_bar_settings.rs:17-32`).

Zed uygulamasında `TitleBar`, platform bileşenini iki farklı render modunda besler (`title_bar/src/title_bar.rs:348-379`). Karar doğrudan ayar alanından değil, `application_menu::show_menus(cx)` helper'ından gelir. Bu helper `TitleBarSettings::show_menus` değerini okur; ayrıca macOS'ta cross-platform menü ancak `ZED_USE_CROSS_PLATFORM_MENU` env'i varsa açılır (`application_menu.rs:274-276`). `show_menus(cx)` sonucu `true` ise `PlatformTitleBar::set_children(...)` yalnız uygulama menüsünü alır ve ürün başlığı ikinci bir satır olarak aynı `title_bar_color` ile render edilir. Sonuç `false` ise bu ikinci satır kurulmaz; tüm ürün child'ları doğrudan `PlatformTitleBar` içine verilir. İki render modunda da ortak olan nokta şudur: `set_button_layout(button_layout)` çağrısı render sırasında yapılır. Desktop layout değişimleri de `observe_button_layout_changed(...)` subscription'ı üzerinden `cx.notify()` ile yeni render tetikler (`title_bar/src/title_bar.rs:442`).

Bu bilgilerin port hedefi için anlamı şudur: `PlatformTitleBar` tek başına "Zed titlebar UI'si" değildir. Zed ürünü bu platform bileşenini menü moduna ve mevcut settings durumuna göre farklı child setleriyle besler. Port hedefinin ürün titlebar'ı da benzer bir mod farkındalığıyla yazılırsa ayar değişikliklerine doğru tepki veren esnek bir yapıya kavuşur.

## 22. Kaynak doğrulama komutları

Kaynak doğrulamasını yalnız public adlar ve payload alanlarıyla sınırlamak yeterli değildir. Owner/metot ayrımı ve olay hedefi farklılıkları da doğrulanmalıdır. Aksi halde sahnenin arkasındaki ince farklar gözden kaçar. Pratikte kontroller üç seviyede çalıştırılır:

1. Public API envanteri — kaynakta hangi tipler ve fonksiyonlar dış API olarak açık?
2. Owner/metot yüzeyi — bir `pub fn` hangi `impl` bloğunun içinde, yani hangi tipin metodu olarak yaşıyor?
3. Event akışı ve payload alan paritesi — hangi event hangi alanlarla tetikleniyor ve hangi route'a düşüyor?

Aşağıdaki komutlar bu üç seviyenin her biri için kullanılabilecek somut örneklerdir.

```sh
rg -n '^pub (struct|enum|fn)|^\s*pub fn|actions!\(' \
  ../zed/crates/platform_title_bar/src \
  -g '*.rs'
```

`PlatformTitleBar` owner/metot yüzeyi:

```sh
rg -n '^impl PlatformTitleBar|^\s*pub fn (new|with_multi_workspace|set_multi_workspace|title_bar_color|set_children|set_button_layout|init|is_multi_workspace_enabled)|^pub fn render_(left|right)_window_controls' \
  ../zed/crates/platform_title_bar/src/platform_title_bar.rs
```

System tab controller yüzeyi:

```sh
sed -n '270,560p' ../zed/crates/gpui/src/app.rs \
  | rg '^pub struct SystemWindowTab|^impl SystemWindowTab|^pub struct SystemWindowTabController|^impl SystemWindowTabController|^\s*pub fn'
```

`DraggedWindowTab` alan paritesi ve event akışı:

```sh
sed -n '28,39p' ../zed/crates/platform_title_bar/src/system_window_tabs.rs
rg -n 'on_drag|last_dragged_tab|drag_over::<DraggedWindowTab>|on_drop|on_mouse_up_out|handle_tab_drop|move_tab_to_new_window|merge_all_windows|update_tab_position' \
  ../zed/crates/platform_title_bar/src/system_window_tabs.rs
```

Fullscreen ve tab render ayrıntıları:

```sh
rg -n 'is_fullscreen|render_left_window_controls|render_right_window_controls|show_window_menu|CLIENT_SIDE_DECORATION_ROUNDING|mt\(px\(-1|SystemWindowTabs|Tab::content_height|Tab::container_height|measured_tab_width|max\(rem_size|ShowCloseButton|ClosePosition|visible_on_hover|drop_target|IconButton::new\("plus"' \
  ../zed/crates/platform_title_bar/src \
  ../zed/crates/ui/src/components/tab.rs \
  ../zed/crates/settings_content/src/workspace.rs
```

Native tab init ve sağ tık/action ayrımını görmek için:

```sh
rg -n 'observe_global::<SettingsStore>|was_use_system_window_tabs|set_tabbing_identifier|tabbed_windows|register_action_renderer|toggle_window_tab_overview|MergeAllWindows|build_window_options|tabbing_identifier' \
  ../zed/crates/platform_title_bar/src/system_window_tabs.rs \
  ../zed/crates/gpui/src/window.rs \
  ../zed/crates/zed/src/zed.rs
```

Controller grup mutasyonlarını görmek için:

```sh
rg -n 'pub fn (add_tab|remove_tab|move_tab_to_new_window|merge_all_windows|update_tab_position|update_tab_title|select_next_tab|select_previous_tab|get_next_tab_group_window|get_prev_tab_group_window)' \
  ../zed/crates/gpui/src/app.rs
```

Pencere seçenekleri ve CSD bağlantılarını kontrol etmek için:

```sh
find ../zed/crates -name '*.rs' -print0 |
  xargs -0 awk '/WindowOptions|WindowDecorations|Decorations::Client|WindowControlArea|set_tabbing_identifier|button_layout\\(|tab_bar_visible\\(|start_window_move|show_window_menu/ { print FILENAME ":" FNR ":" $0 }'
```

`WindowButtonLayout` ayar zincirini görmek için:

```sh
find ../zed/crates/gpui/src ../zed/crates/settings_content/src ../zed/crates/title_bar/src -name '*.rs' -print0 |
  xargs -0 awk '/MAX_BUTTONS_PER_SIDE|WindowButtonLayout|WindowButton|button_layout|linux_default|parse\\(|into_layout|observe_button_layout_changed/ { print FILENAME ":" FNR ":" $0 }'
```

Owner ayrımını yapmak için, yani bir `pub fn`'in struct mı, trait mi, inherent mi yoksa free fonksiyon mu olduğunu anlamak için **state-machine awk** kullanılır. `rg` aracının satır tabanlı eşleşmesi tek başına yeterli değildir; bir `pub fn`'in hangi `impl` bloğunun içinde durduğunu raporlamaz. Bu bilgiyi çıkarmak için awk içinde kalıcı state tutan aşağıdaki betik kullanılır:

```sh
find ../zed/crates/platform_title_bar/src -name '*.rs' -print0 |
  xargs -0 gawk '
    BEGIN { owner = "(free)" }
    /^pub struct [A-Za-z0-9_]+/  { print FILENAME ":" FNR ":STRUCT: " $0; next }
    /^pub enum [A-Za-z0-9_]+/    { print FILENAME ":" FNR ":ENUM: "   $0; next }
    /^pub trait [A-Za-z0-9_]+/   { print FILENAME ":" FNR ":TRAIT: "  $0; next }
    /^pub fn [A-Za-z0-9_]+/      { print FILENAME ":" FNR ":FREE_FN: " $0; next }
    /^impl[^!]/ {
      if (match($0, /for[[:space:]]+([A-Za-z0-9_]+)/, m))                  { owner = m[1] " (trait impl)" }
      else if (match($0, /impl(<[^>]+>)?[[:space:]]+([A-Za-z0-9_]+)/, m))   { owner = m[2] " (inherent)" }
      else                                                                  { owner = "?" }
      print FILENAME ":" FNR ":IMPL[" owner "]: " $0; next
    }
    /^[[:space:]]+pub fn [A-Za-z0-9_]+/ {
      print FILENAME ":" FNR ":  METHOD[" owner "]: " $0
    }
  '
```

Bu komutun çıktısında `WindowControl::new`, `WindowControl::new_close` ve `WindowControl::custom_style` üçlüsü ile `WindowControlStyle::default(cx)` gibi **inherent** ad çakışmaları, hangi tipe ait olduklarıyla birlikte görünür. Bunun yerine `rg '^impl '` çalıştırılırsa yalnızca `impl` header'ları listelenir. Metotların hangi owner'a ait olduğunu eşleştirmek için ya `-A N` ile blok büyüklüğünü tahmin etmek ya da awk state'i kullanmak gerekir. Tahmin yöntemi kırılgan olduğu için state-machine yolu daha güvenlidir.

**Modül görünürlüğü kontrolü.** Daha önce de vurgulandığı gibi, `pub struct` tek başına bir tipin dış API olduğu anlamına gelmez. Bunu doğru anlamak için önce crate kökündeki `pub mod`, `mod` ve `pub use` kapılarına bakılır:

```sh
rg -n '^(pub mod|mod |pub use)|^pub (struct|enum|fn)|^[[:space:]]+pub fn' \
  ../zed/crates/platform_title_bar/src/platform_title_bar.rs \
  ../zed/crates/platform_title_bar/src/platforms.rs \
  ../zed/crates/platform_title_bar/src/platforms/*.rs \
  ../zed/crates/platform_title_bar/src/system_window_tabs.rs
```

Bu çıktı dört basit kurala göre okunur:

- `platform_title_bar.rs` içindeki `pub mod platforms;` ifadesi dış path'i açar; yani `platforms` modülü crate dışından erişilebilir.
- `platform_title_bar.rs` içindeki `mod system_window_tabs;` ifadesi ise private bir kapıdır; yalnız crate içinden ulaşılabilir.
- Private modül içinde `pub struct SystemWindowTabs` yazılı olsa bile, dış API değildir; çünkü kapı kapalıdır.
- Aynı private modülden crate kökünde `pub use` ile dışarı verilen `DraggedWindowTab` ve tab action'ları ise dış API olur.

**Bir sınır vardır:** Yukarıdaki kalıplar `^pub fn` ve `^[[:space:]]+pub fn` desenleriyle yalnızca **public** öğeleri yakalar. Crate-içi `fn create_window_button(...)` gibi file-private free helper'lar, örneğin Linux render yolunun gerçek dispatch noktası, bu taramada görünmez. Davranış paritesi için bu helper'ların da görülmesi gerekiyorsa desen şu şekilde gevşetilir:

```sh
find ../zed/crates/platform_title_bar/src -name '*.rs' -print0 |
  xargs -0 gawk '
    /^fn [A-Za-z0-9_]+/         { print FILENAME ":" FNR ":FREE_FN(priv): " $0 }
    /^[[:space:]]+fn [A-Za-z0-9_]+/ { print FILENAME ":" FNR ":  METHOD(priv): " $0 }
  '
```

Bu betiğin çıktısında karşılaşılan file-private parçalar şunlardır:

- `fn create_window_button(...)` (`platform_linux.rs:56`) — Linux render yolunun gerçek dispatch noktasıdır.
- `fn id(&self)`, `fn icon(&self)`, `fn control_area(&self)` (`platform_windows.rs:68, 78, 88`) — `WindowsCaptionButton` tipi üzerindeki inherent yardımcı metotlardır.
- `fn get_font()` (`platform_windows.rs:16, 21`) — Windows 11 ve Windows 10 arasındaki font ayrımını yapan seçicidir (`Segoe Fluent Icons` ve `Segoe MDL2 Assets` arasında karar verir).
- `fn handle_tab_drop`, `fn handle_right_click_action` (`system_window_tabs.rs:358, 362`) — tab bar event router'larıdır.
- `PlatformTitleBar::effective_button_layout`, `::sidebar_render_state` (`platform_title_bar.rs:86, 104`) — public yüzeyin arkasındaki karar helper'larıdır.
- `fn render(...)` satırları — `Render` ve `RenderOnce` trait implementasyonlarının gövdeleridir; ayrı satırlarda göründüklerinde `IMPL[Owner (trait impl)]` header'ı ile eşleştirilmeleri gerekir.

Bu liste, **dış API olmayan ama davranış mirror'ı için kritik olan** parçaları açığa çıkarır. Port hedefinde aynı isimleri korumak şart değildir. Ancak bu fonksiyonların yaptığı işlerin her birine karşılık gelen bir karar noktası bulunmalıdır. Aksi halde "isim mirror edildi ama davranış eksik" durumuyla karşılaşılır.

Crate sınırını aşan keşif de bazen gerekir. Örneğin `SystemWindowTabController` adı `platform_title_bar` içinde geçer, ama gerçek tanım `gpui` crate'indedir. Bu tür sıçramaları otomatik yakalamak için şu komut zinciri kullanılır:

```sh
# Önce platform_title_bar'da geçen tüm tip adlarını çıkar
gawk '
  match($0, /\<([A-Z][A-Za-z0-9_]+)\>/, m) { print m[1] }
' ../zed/crates/platform_title_bar/src/*.rs \
  ../zed/crates/platform_title_bar/src/platforms/*.rs \
  | sort -u > /tmp/ptb_referenced.txt

# Sonra her birinin tanım crate'ini bul
while read name; do
  defs=$(rg -l "^pub (struct|enum|trait|fn|type) ${name}\b" ../zed/crates 2>/dev/null)
  [ -n "${defs}" ] && echo "${name}: ${defs}"
done < /tmp/ptb_referenced.txt
```

Bu betik `DraggedWindowTab` tipinin doğrudan `platform_title_bar` crate'inde tanımlı olduğunu, buna karşılık `SystemWindowTabController` tipinin aslında `gpui/src/app.rs` dosyasında durduğunu açıkça gösterir. Cross-crate sıçramaları yakalamanın güvenli yolu budur. Yalnız rehberin merkez crate'inde tarama yapılırsa bu tür sıçramalar gözden kaçar ve "bu tipin tanımı nerede?" sorusu cevapsız kalır.
