# Zed UI Bileşenleri

---

## Zed UI Prelude, Stil Uzantı Trait'leri ve Bileşen Sözleşmesi

Zed uygulama kodu çoğunlukla doğrudan `gpui::prelude::*` değil `ui::prelude::*` import eder. Bu prelude GPUI çekirdeğini yeniden dışa aktarır ve üstüne Zed'e özgü bileşen ile stil katmanını ekler. Böylece tek bir import deyimi UI yazımının büyük kısmını karşılar.

**`ui::prelude::*` içindeki başlıca dışa aktarımlar.** Aşağıdaki tipler ve trait'ler prelude ile gelir:

- `gpui::prelude::*` üzerinden trait ve tipler: `AppContext` (anonim), `BorrowAppContext`, `Context`, `Element`, `InteractiveElement`, `IntoElement`, `ParentElement`, `Refineable`, `Render`, `RenderOnce`, `StatefulInteractiveElement`, `Styled`, `StyledImage`, `TaskExt` (anonim), `VisualContext`, `FluentBuilder`.
- `gpui` doğrudan tipleri: `App`, `Window`, `Div`, `AnyElement`, `ElementId`, `SharedString`, `Pixels`, `Rems`, `AbsoluteLength`, `DefiniteLength`, `div`, `px`, `rems`, `relative`.
- Zed yerleşim yardımcıları: `h_flex()`, `v_flex()`, `h_group()`, `h_group_sm()`, `h_group_lg()`, `h_group_xl()`, `v_group()`, `v_group_sm()`, `v_group_lg()`, `v_group_xl()`.
- Tema erişimi: `theme::ActiveTheme`, `Color`, `PlatformStyle`, `Severity`.
- Stil ve aralık yardımcıları: `StyledTypography`, `TextSize`, `DynamicSpacing`, `rems_from_px`, `vh`, `vw`.
- Animasyon: `AnimationDirection`, `AnimationDuration`, `DefaultAnimations`.
- Bileşen sistemi: `Component`, `ComponentScope`, `RegisterComponent`, `example_group`, `example_group_with_title`, `single_example`.
- Bileşenler: `Button`, `ButtonCommon`, `ButtonSize`, `ButtonStyle`, `IconButton`, `SelectableButton`, `Headline`, `HeadlineSize`, `Icon`, `IconName`, `IconPosition`, `IconSize`, `Label`, `LabelCommon`, `LabelSize`, `LineHeightStyle`, `LoadingLabel`.
- Trait'ler (`crate::traits::*`): `StyledExt`, `Clickable`, `Disableable`, `Toggleable`, `VisibleOnHover`, `FixedWidth`.

`ButtonLike` prelude'da değildir; ihtiyaç duyulduğunda `ui::ButtonLike` şeklinde import edilir. Aynı şekilde prelude'da yer almayan `Tooltip`, `ContextMenu`, `Popover`, `PopoverMenu`, `Modal`, `RightClickMenu` gibi tipler doğrudan `ui::` namespace'i üzerinden çağrılır.

`CommonAnimationExt` de crate kökünden (`ui::CommonAnimationExt`) açıkça import edilir; `with_rotate_animation(duration_secs)` ve `with_keyed_rotate_animation(id, duration_secs)` yardımcılarını sağlar.

**Animasyon yüzeyi.** Hazır animasyon yardımcıları birkaç enum ve trait üzerinde toplanır:

- `AnimationDuration::{Instant, Fast, Slow}` sırasıyla 50, 150 ve 300 ms döner; `.duration()` ya da `Into<Duration>` ile GPUI animation API'sine verilir.
- `AnimationDirection::{FromBottom, FromLeft, FromRight, FromTop}` giriş animasyon yönünü ifade eder.
- `DefaultAnimations` trait'i `.animate_in(direction, fade)`, `.animate_in_from_bottom(fade)`, `.animate_in_from_left(fade)`, `.animate_in_from_right(fade)`, `.animate_in_from_top(fade)` metotlarını sağlar. Blanket impl `Styled + Element` olan elementler içindir.
- `CommonAnimationExt` döndürme yardımcısı yalnızca `Transformable` uygulayan elementlerde çalışır; Zed UI tarafında bu pratikte `Icon` ve `Vector` anlamına gelir.
- **Animasyon sarmalayıcı şeffaflığı:** `popover_menu.rs:16` ve `:32`, `impl<T: Clickable> Clickable for gpui::AnimationElement<T>` ve `impl<T: Toggleable> Toggleable for gpui::AnimationElement<T>` impl'leri taşır. Bu sayede bir element `.with_animation(...)` veya `.animate_in_from_bottom(...)` ile sarıldığında `Clickable`/`Toggleable` trait yüzeyini kaybetmez — `PopoverMenu::trigger(...)` animasyonlu trigger da kabul eder, çünkü `PopoverTrigger: IntoElement + Clickable + Toggleable + 'static` supertrait listesi blanket impl ile karşılanır (`impl<T: IntoElement + Clickable + Toggleable + 'static> PopoverTrigger for T {}`).

**Genel dışa aktarım modeli.** UI crate'inin hangi tipi nereden açtığı konusunda bilinmesi gereken birkaç asimetri vardır:

- `crates/ui/src/ui.rs` iç modülleri (`components`, `styles`, `traits`) `private` tutar ve `pub use components::*`, `pub use prelude::*`, `pub use styles::*`, `pub use traits::animation_ext::*` ile genel yüzeyi düzleştirir. Bu nedenle tüketici kodda `ui::Button`, `ui::Color`, `ui::Clickable`, `ui::theme_is_transparent` gibi yollar kullanılır; `ui::components::...` veya `ui::styles::...` genel API yolu değildir.
- Bilinçli olarak genel iç içe modüller (`pub mod`) kendi namespace'iyle kullanılır:
  - `ui::scrollbars::{ShowScrollbar, ScrollbarVisibility, ScrollbarAutoHide}` — scrollbar görünürlük ayarları yüzeyi.
  - `ui::animation::{AnimationDirection, AnimationDuration, DefaultAnimations}` — prelude'da da olan animasyon yardımcı modülü; `workspace::toast_layer` gibi kodlar açıkça `ui::animation::DefaultAnimations` import edebilir.
  - `ui::table_row::{TableRow, IntoTableRow}` — `data_table.rs` içinden genel açılan, kolon sayısı doğrulamalı tablo satırı modülü.
  - `ui::utils::*` — stil dışı yardımcılar (utils başlığında listelenir).
  - `ui::component_prelude::*` — bileşen yazılırken kullanılan derive ve registry yüzeyi: `Component`, `ComponentId`, `ComponentScope`, `ComponentStatus`, `example_group`, `example_group_with_title`, `single_example`, `RegisterComponent` (ui_macros), `Documented` (documented crate). Yeni bileşen yazılırken `use ui::component_prelude::*;` deyimi `ui::prelude::*`'a ek olarak import edilir.
- `ui::prelude::*` sık kullanılan küçük yüzeydir; tüm bileşen envanteri değildir. `ButtonLike`, `ContextMenu`, `Tooltip`, `Modal`, `Table`, `Vector`, `ThreadItem` gibi tipler crate kökünden açıkça import edilir.

**Stil uzantıları.** `StyledExt` üzerinden gelen ek metotlar günlük kullanımda fluent zinciri kısaltır:

- `StyledExt::h_flex()` = `flex().flex_row().items_center()`.
- `StyledExt::v_flex()` = `flex().flex_col()`.
- `elevation_1/2/3(cx)` ve `*_borderless(cx)` Zed elevation katmanlarını uygular: `Surface`, `ElevatedSurface`, `ModalSurface`. Popover ve modal gibi katmanlarda elle gölge veya border üretmek yerine bu yardımcılar tercih edilir. **Referans asimetrisi:** `elevation_1(cx)`, `elevation_2(cx)`, `elevation_3(cx)` `&App` alır; `elevation_1_borderless(cx)`, `elevation_2_borderless(cx)`, `elevation_3_borderless(cx)`, `border_primary(cx)` ve `border_muted(cx)` `&mut App` ister. Salt okunur çizim yolunda borderless varyantı çağrılamaz; ya mut erişim verilir ya da bordered varyant seçilir.
- `debug_bg_red/green/blue/yellow/cyan/magenta()` yalnızca geliştirme sırasında yerleşim teşhisi için kullanılır.

**Tipografi.** Yazı boyutları anlamsal yardımcılarla ifade edilir; bu hem tema değişimlerine dayanıklılığı hem de tutarlılığı artırır:

- `StyledTypography::font_ui(cx)` ve `font_buffer(cx)` tema ayarları içindeki UI ve buffer font ailesi değerlerini bağlar.
- `text_ui_size(TextSize, cx)` enum'dan anlamsal metin boyutu uygular.
- `text_ui_lg(cx)`, `text_ui(cx)`, `text_ui_sm(cx)`, `text_ui_xs(cx)` UI ölçeğini dikkate alan anlamsal metin boyutlarıdır.
- `text_buffer(cx)` buffer font boyutuna uyar; editör içeriğiyle aynı boyda görünmesi gereken metinde kullanılır.
- `TextSize::{Large, Default, Small, XSmall, Ui, Editor}` hem `.rems(cx)` hem `.pixels(cx)` verir. Sabit `px(14.)` yerine anlamsal boyut tercih edilir.

**Anlamsal renk ve elevation.** Tema değişikliklerine dayanıklı renk yüzeyi anlamsal enum'larla kurulur:

- `Color` temaya göre HSLA'ya çevrilen anlamsal enum'dur (`crates/ui/src/styles/color.rs`). Varyantlar: `Default`, `Accent`, `Conflict`, `Created`, `Custom(Hsla)`, `Debugger`, `Deleted`, `Disabled`, `Error`, `Hidden`, `Hint`, `Ignored`, `Info`, `Modified`, `Muted`, `Placeholder`, `Player(u32)`, `Selected`, `Success`, `VersionControlAdded`, `VersionControlConflict`, `VersionControlDeleted`, `VersionControlIgnored`, `VersionControlModified`, `Warning`. Üretim ve diff renkleri için git veya VCS varyantları, oyuncu vurgusu için `Player(u32)`, vurgu için `Selected` veya `Hint` tercih edilir; bunlar `cx.theme().status()` veya oyuncu paleti üzerinden HSLA'ya çözülür.
- `TintColor::{Accent, Error, Warning, Success}` buton tint stillerine kaynaklık eder ve `Color`'a dönüştürülebilir.
- `Severity::{Info, Success, Warning, Error}` `Banner` veya `Callout` gibi geri bildirim bileşenlerinin anlamsal durumudur; `TintColor` veya `Color` yerine bileşen severity sözleşmesi gerekiyorsa bu seçilir.
- `PlatformStyle::{Mac, Linux, Windows}` ve `PlatformStyle::platform()` platforma göre UI ayrımı gereken bileşenlerde küçük dallanma yüzeyidir.
- `ElevationIndex::{Background, Surface, EditorSurface, ElevatedSurface, ModalSurface}` gölge, arka plan ve "bu elevation üzerinde okunacak renk" kararlarını toplar. Genel yardımcılarında receiver ve mut asimetrisi vardır:
  - `.shadow(self, cx: &App) -> Vec<BoxShadow>` — `self`'i tüketir; tipik olarak `match` veya clone sonrası kullanılır.
  - `.bg(&self, cx: &mut App) -> Hsla` — `&mut App` ister (yalnızca bu metot).
  - `.on_elevation_bg(&self, cx: &App) -> Hsla`, `.darker_bg(&self, cx: &App) -> Hsla` — `&App` yeterlidir.

**Buton ve etiket ortak sözleşmeleri.** Etkileşimli bileşenlerin ortak trait'leri sade bir setten oluşur:

- `Clickable` — `.on_click(...)` ve `.cursor_style(...)`.
- `Disableable` — `.disabled(bool)`.
- `Toggleable` — `.toggle_state(bool)`. `ToggleState::{Unselected, Indeterminate, Selected}` üç durumlu checkbox veya ağaç seçimi için kullanılır. `ToggleState` ayrıca `.inverse()`, `.selected()` ve `from_any_and_all(any_checked, all_checked)` yardımcılarını sağlar.
- `SelectableButton` — seçili durumda farklı bir `ButtonStyle` tanımlar.
- `ButtonCommon` — `.id()` erişim metodu ve `.style(ButtonStyle)`, `.size(ButtonSize)`, `.tooltip(...)`, `.tab_index(...)`, `.layer(ElevationIndex)`, `.track_focus(...)` metotları.
- `ButtonStyle::{Filled, Tinted(TintColor), Outlined, OutlinedGhost, OutlinedCustom(Hsla), Subtle, Transparent}`.
- `ButtonSize::{Large, Medium, Default, Compact, None}`.
- `LabelCommon` — `.size(LabelSize)`, `.weight(FontWeight)`, `.line_height_style(LineHeightStyle::{TextLabel, UiLabel})` — `TextLabel` varsayılan (UI/buffer varsayılan satır yüksekliği), `UiLabel` satır yüksekliğini 1.0'a sabitler; `.color(Color)`, `.strikethrough()`, `.italic()`, `.underline()`, `.alpha(f32)`, `.truncate()`, `.single_line()`, `.buffer_font(cx)`, `.inline_code(cx)`.
- `LabelLike` (alttaki primitive struct) ayrıca doğrudan `.truncate_start()` taşır — `LabelCommon` trait'inde bu yoktur; başlangıçtan üç nokta ile kesim gerektiğinde `LabelLike` veya `Label` üzerinde doğrudan çağrılır.
- `FixedWidth` — `.width(DefiniteLength)` ve `.full_width()`. Mevcut UI katmanında `Button`, `IconButton` ve `ButtonLike` üzerinde uygulanır.

**Diğer UI yardımcıları.** Sık kullanılan ek tipler ve fonksiyonlar şunlardır:

- `VisibleOnHover::visible_on_hover(group)` elementi başlangıçta görünmez yapar, belirtilen grup hover olduğunda görünür hale getirir. `""` genel gruptur.
- `WithRemSize::new(px(...))` alt ağaçta `window.with_rem_size` uygular; özel önizleme veya küçük bileşen ölçeklendirmesi için kullanılır. `.occlude()` fare etkileşimini kapatır.
- `ui::utils` genel bir iç içe namespace olarak kalır. Sık kullanılan dışa aktarımlar: `is_light(cx)`, `reveal_in_file_manager_label(is_remote)`, `capitalize(str)`, `SearchInputWidth::calc_width(container_width)`, `platform_title_bar_height(&Window)`, `calculate_contrast_ratio(fg, bg)`, `apca_contrast(text, bg)`, `ensure_minimum_contrast(...)`, `CornerSolver::new(root_radius, root_border, root_padding)`, `inner_corner_radius(...)`, `FormatDistance::{new(date, base_date), from_now(date), include_seconds(bool), add_suffix(bool), hide_prefix(bool)}`, `format_distance(...)`, `format_distance_from_now(...)`, `DateTimeType::{Naive(NaiveDateTime), Local(DateTime<Local>)}`.
- `BASE_REM_SIZE_IN_PX` `ui::utils` altında değil, `styles/units.rs` üzerinden crate köküne gelen `ui::BASE_REM_SIZE_IN_PX` sabitidir; `rems_from_px(...)`, `vh(...)`, `vw(...)` ile aynı birim yüzeyi içindedir.
- `ui::ui_density(cx: &mut App) -> UiDensity` styles modülünden gelen ergonomik yardımcıdır; `theme::theme_settings(cx).ui_density(cx)`'nin kısayoludur. Aralık kararları için `DynamicSpacing` enum'u tercih edilir; density yalnızca stil dışı kararlar (örneğin ikon boyutu veya compact toggle) için kullanılır.
- Keybinding görünümü için crate kökünde genel serbest fonksiyonlar sağlanır:
  - `text_for_action(action: &dyn Action, window, cx) -> Option<String>`
  - `text_for_keystrokes(keystrokes: &[Keystroke], cx) -> String`
  - `text_for_keystroke(modifiers: &Modifiers, key: &str, cx) -> String`
  - `text_for_keybinding_keystrokes(keystrokes: &[KeybindingKeystroke], cx) -> String`
  - `render_keybinding_keystroke(...)`, `render_modifiers(...)` — element üretir.

  Bu yardımcılar tooltip veya menü metni üretirken `KeyBinding` ile `KeybindingHint` bileşenlerinden ayrı düşünülmelidir; bileşen çizer, serbest fn yalnızca metin üretir.
- `Transformable` trait'i crate dışına genel değildir (`traits` modülü `private`); yalnızca `Icon` ve `Vector` uygular. Bu nedenle `CommonAnimationExt` üzerindeki `with_rotate_animation`/`with_keyed_rotate_animation` zinciri yalnız bu iki tipte derlenir. Üçüncü taraf bir element için döndürme isteniyorsa ya `Icon`/`Vector` sarılır ya da kendi `with_animation` zinciri kurulur.
- Scrollbar katmanı **iki ayrı API yüzeyi** sunar; doğru olanın seçimi ihtiyaca bağlıdır:
  - Düşük seviye: `Scrollbars::new(ScrollAxes)` builder zinciri — `.tracked_scroll_handle(handle)`, `.id(...)`, `.notify_content()`, `.style(ScrollbarStyle)`, `.with_track_along(...)` vb. — sonra `div().custom_scrollbars(config, window, cx)` ile uygulanır. `custom_scrollbars` ve `vertical_scrollbar_for` yalnız `WithScrollbar` trait üzerindedir; `WithScrollbar` `Div` ve `Stateful<Div>` için uygulanır.
  - Kısayol: `div().vertical_scrollbar_for(&scroll_handle, window, cx)` — `WithScrollbar::vertical_scrollbar_for` varsayılan impl'i içeride `Scrollbars::new(ScrollAxes::Vertical).tracked_scroll_handle(handle)` kurar. **`Scrollbars` üzerinde doğrudan `.vertical_scrollbar_for(...)` yoktur**; zincir üst element üzerinde başlar.
  - Görünürlük ayarı crate kökünde değil `ui::scrollbars::{ShowScrollbar, ScrollbarVisibility, ScrollbarAutoHide}` namespace'inde bulunur; `ScrollbarVisibility` ayar trait'i, `ScrollbarAutoHide` ise otomatik gizleme durumunu taşıyan `Global` tipidir.
  - Enum yüzeyi: `ScrollAxes::{Horizontal, Vertical, Both}` (Scrollbars'a verilen üç değerli eksen seçimi) ve `ScrollbarStyle::{Regular, Editor}`. Özel scroll handle yazılırken `ScrollableHandle: 'static + Any + Sized + Clone` trait'i `max_offset(&self) -> Point<Pixels>`, `set_offset(&self, Point<Pixels>)`, `offset(&self) -> Point<Pixels>`, `viewport(&self) -> Bounds<Pixels>`, varsayılan impl'li `drag_started(&self)` / `drag_ended(&self)`, `scrollable_along(&self, ScrollbarAxis) -> bool` ve `content_size(&self) -> Size<Pixels>` sözleşmesini taşır. **`ScrollbarAxis` ≠ `ScrollAxes`.** `ScrollbarAxis` scrollbar.rs içinde `gpui::Axis as ScrollbarAxis` takma adıdır; iki varyantı (`Horizontal`, `Vertical`) vardır ve scrollable handle'ın tek eksenli sorgusunda kullanılır. `ScrollAxes` (Horizontal/Vertical/Both) ise `Scrollbars::new(...)` yapılandırma yüzeyidir.
  - `on_new_scrollbars<T: gpui::Global>(cx)` bir global ayarı `ScrollbarState::settings_changed` gözlemcisine bağlamak için düşük seviyeli yardımcıdır; normal bileşen çiziminde çağrılan bir builder değildir.

**Tuzaklar.** UI yüzeyiyle çalışırken sık karşılaşılan hatalar:

- Uygulama UI'ında doğrudan `cx.theme().colors().text_*` yazmak mümkündür, ancak tekrar kullanılabilir bileşen için `Color` veya `TintColor` anlamsal katmanı daha dayanıklı kalır.
- `ButtonLike` güçlü ama sınırları geniş bir primitive'dir; hazır `Button`, `IconButton`, `ToggleButtonGroup`, `ToggleButtonSimple` veya `ToggleButtonWithIcon` yeterliyse onlar tercih edilir. `ToggleButton` adında bağımsız bir genel tip yoktur.
- `VisibleOnHover` için üst öğede aynı grup adıyla hover grubu kurulmadıysa element hiçbir zaman görünmez.
- Genel görünen bazı durum struct'ları kullanıcıya dönük builder değildir: `PopoverMenuElementState`, `PopoverMenuFrameState`, `MenuHandleElementState`, `RequestLayoutState`, `PrepaintState`, `ScrollbarPrepaintState`. Bunlar `Element` ilişkili durum tipleri olarak genel kalmıştır; doğrudan oluşturulup bileşen API'si gibi kullanılmamalıdır.

**Zed uygulamasında yönetim.** Bileşen registry'si ve önizleme mekanizması kısaca şöyle akar:

- Bileşen registry'si çalışma zamanı çizim akışı değildir; uygulama kodu bileşenleri doğrudan `ui::{...}` veya `ui::prelude::*` ile import edip element ağacına yerleştirir.
- `workspace::init` içinde `component::init()` çağrılır. Bu çağrı `#[derive(RegisterComponent)]` tarafından envantere eklenen `ComponentFn` kayıtlarını çalıştırır ve `ComponentRegistry`'yi doldurur.
- `zed/src/main.rs` `component_preview::init(app_state, cx)` çağırır. Bileşen önizleme, `component::components().sorted_components()` ile registry'yi okur ve isim, scope ve açıklama üzerinden filtreler.
- `Component` trait'inin genel sözleşmesi şu metotları içerir: `id()`, `scope()`, `status()`, `name()`, `sort_name()`, `description()` ve `preview(&mut Window, &mut App) -> Option<AnyElement>`. Tüm metotların varsayılan uygulaması vardır; pratikte minimum uygulama yalnızca `scope()` ve `preview()` üzerine yazmaktır. Varsayılan davranış şu şekildedir:
  - `id() -> ComponentId(Self::name())`
  - `name() -> std::any::type_name::<Self>()` (modül yolu dahil)
  - `scope() -> ComponentScope::None`
  - `status() -> ComponentStatus::Live`
  - `sort_name() -> Self::name()`
  - `description()` ve `preview()` `None` döner.

  Bu sözleşme görsel test, hata ayıklama ve dokümantasyon içindir; normal UI kullanımında bileşen registry'si sorgulanmaz.
- `ComponentScope` 17 varyanta sahiptir (organizasyon kovaları; strum `Display`/`EnumString` ile string olarak serileştirilir): `Agent`, `Collaboration`, `DataDisplay` (`"Data Display"`), `Editor`, `Images` (`"Images & Icons"`), `Input` (`"Forms & Input"`), `Layout` (`"Layout & Structure"`), `Loading` (`"Loading & Progress"`), `Navigation`, `None` (`"Unsorted"`), `Notification`, `Overlays` (`"Overlays & Layering"`), `Onboarding`, `Status`, `Typography`, `Utilities`, `VersionControl` (`"Version Control"`).
- `ComponentStatus` önizleme ekranındaki durum metnini belirler. Yeni bileşenlerde pratik akış `WorkInProgress`, `EngineeringReady` ve varsayılan `Live` durumları üzerinden yürür.
- Diğer genel registry yüzeyleri: `ComponentId(pub &'static str)`, `ComponentMetadata { id, description, name, preview, scope, sort_name, status }`, `ComponentExample { variant_name, description, element, width }`, `ComponentExampleGroup`, `empty_example(variant_name)`, `register_component::<T: Component>()`. `COMPONENT_DATA: LazyLock<RwLock<ComponentRegistry>>` genel depodur; tüketici kod yerine `component::components()` erişim metodu kullanılır.
- **`ui` prelude vs `component_prelude` asimetrisi:** `ui::prelude::*` yalnızca `Component` ve `ComponentScope`'u yeniden dışa aktarır. `ComponentStatus`, `ComponentId` ve `Documented` ise `ui::component_prelude::*` üzerinden gelir; `ui::ComponentStatus` doğrudan çalışmaz, `ui::component_prelude::ComponentStatus` veya `component::ComponentStatus` biçiminde yazılır.

## Zed UI Bileşen Envanteri

Zed içinde yeni bir UI yazılırken önce `ui` bileşenleri taranır. Aşağıdaki liste hazır bileşen ailelerini kategorilere göre düzenler. Mevcut bir tip ihtiyacı karşılıyorsa sıfırdan üretmek yerine onu seçmek tutarlılığı korur.

- **Metin:** `Label`, `LabelLike` (alttaki primitive), `Headline`, `HighlightedLabel` ve `highlight_ranges(...)` serbest fn yardımcısı, `LoadingLabel`, `SpinnerLabel` (`SpinnerVariant::{Dots (varsayılan), DotsVariant, Sand}`). `SpinnerLabel` yapıcıları: `new()` varsayılan Dots, `with_variant(SpinnerVariant)` ve doğrudan kısayollar `dots()`, `dots_variant()`, `sand()`. `SpinnerLabel` ayrıca `LabelCommon` uyguladığı için `.size/.color/.weight/...` zinciri kullanılabilir.
- **Buton:** `Button` (`KeybindingPosition::{Start, End}` ile `.key_binding_position(...)`), `IconButton` (`IconButtonShape::{Square, Wide}`), `SelectableButton`, `ButtonLike` (`ButtonBuilder`/`ButtonConfiguration` sealed yardımcılar), `ButtonLink`, `CopyButton`, `SplitButton` (`SplitButtonStyle::{Filled, Outlined, Transparent}`, `SplitButtonKind::{ButtonLike(ButtonLike), IconButton(IconButton)}` — segment olarak hem button-like hem icon button kabul eder), `ToggleButtonGroup` (`ToggleButtonGroupStyle::{Transparent, Filled, Outlined}`, `ToggleButtonGroupSize::{Default, Medium, Large, Custom(Rems)}`, `ToggleButtonPosition` — bu **enum değil struct**'tır; `leftmost`, `rightmost`, `topmost`, `bottommost` private `bool` alanlarıyla grup içi konum bayrağıdır) ve giriş tipleri `ToggleButtonSimple`, `ToggleButtonWithIcon`. `ToggleButton` adında bağımsız bir struct yoktur; tekil toggle yerine her zaman grup içinde segment kullanılır.
- **İkon:** `Icon`, `DecoratedIcon`, `IconDecoration`, `IconDecorationKind::{X, Dot, Triangle}`, `IconName`, `IconPosition::{Start, End}` (buton içinde etiket-ikon sırası), `IconSize`, `IconWithIndicator`, `KnockoutIconName::{XFg, XBg, DotFg, DotBg, TriangleFg, TriangleBg}` (IconDecoration için eşli fg/bg svg adları), `AnyIcon::{Icon(Icon), AnimatedIcon(AnimationElement<Icon>)}` (tipi silinmiş ikon; hem statik hem animasyonlu varyantı tek tipte taşır). `IconName` `crates/icons` crate'inde tanımlıdır, `ui::prelude` üzerinden yeniden dışa aktarılır. `IconSize` varyantları: `Indicator`, `XSmall`, `Small`, `Medium`, `XLarge`, `Custom(Rems)`; `.rems()`, `.square_components(window, cx)`, `.square(window, cx)` metotları verir.
- **Form/toggle:** `Checkbox`, `Switch` (`SwitchColor::{Accent, Custom(Hsla)}`, `SwitchLabelPosition::{Start, End}`), `SwitchField`, `DropdownMenu` (`DropdownStyle::{Solid, Outlined, Subtle, Ghost}`), `ToggleStyle::{Ghost, ElevationBased(ElevationIndex), Custom(Hsla)}`. Küçük harfli yardımcı fonksiyonlar `checkbox(id, state)` ve `switch(id, state)` yapıcı kısayollarıdır.
- **Metin girişi:** `ui_input::InputField` ayrı bir crate'tedir; `ui` yeniden dışa aktarmaz. `InputField::new(window, cx, placeholder)` editör factory gerektirir ve `.label`, `.start_icon`, `.masked`, `.tab_index`, `.text`, `.set_text`, `.clear` gibi form alanı API'sini sağlar.
- **Menü/popup:** `ContextMenu`, `ContextMenuEntry`, `ContextMenuItem::{Separator, Header(SharedString), HeaderWithLink(title, link_label, link_url), Label(SharedString), Entry(ContextMenuEntry), CustomEntry { entry_render, handler, selectable, documentation_aside }, Submenu { label, icon, icon_color, builder }}` — `ContextMenu` builder'ına doğrudan eklenen yedi tip menü satırı, `DocumentationAside { side: DocumentationSide, render: Rc<...> }`, `DocumentationSide::{Left, Right}`, `RightClickMenu<M: ManagedView>` ve serbest fn `right_click_menu(id)`, `Popover`, `PopoverMenu<M: ManagedView>`, `PopoverMenuHandle<M>` (buyruk tarzı kontrol için `show(window, cx)`, `hide(cx)`, `toggle(window, cx)`, `is_deployed() -> bool`, `is_focused(window, cx) -> bool`, `refresh_menu(...)` metotları; `Default` uygular ve `PopoverMenu::with_handle(handle)` ile bağlanır), `PopoverTrigger` (`IntoElement + Clickable + Toggleable + 'static` supertraitlerini taşıyan her tipe blanket impl), `Tooltip`, `LinkPreview` ve serbest fn `tooltip_container(cx, f)`.
- **Liste/tree:** `List` (`EmptyMessage::{Text(SharedString), Element(AnyElement)}`), `ListItem` (`ListItemSpacing::{Dense, ExtraDense, Sparse}`), `ListHeader`, `ListSubHeader`, `ListSeparator` (`pub struct ListSeparator;` — unit struct, builder yok), `ListBulletItem`, `TreeViewItem`, `StickyCandidate` (`depth(&self) -> usize`), `StickyItems<T>` ve serbest fn yapıcı `sticky_items<V, T>(...)`, `StickyItemsDecoration` (`compute(indents, bounds, scroll_offset, item_height, window, cx)`), `IndentGuides` (`IndentGuideColors` — `panel(cx) -> Self` factory ile; `IndentGuideLayout { offset: Point<usize>, length: usize, continues_offscreen: bool }`; `RenderIndentGuideParams { indent_guides: SmallVec<[IndentGuideLayout;12]>, indent_size, item_height }`; `RenderedIndentGuide { bounds, layout, is_active, hitbox: Option<Bounds<Pixels>> }`) ve serbest fn yapıcı `indent_guides(indent_size: Pixels, colors: IndentGuideColors)`. `IndentGuides` builder yüzeyi: `.on_click(|&IndentGuideLayout, &mut Window, &mut App|)`, `.with_compute_indents_fn::<V>(entity, |&mut V, Range<usize>, &mut Window, &mut Context<V>| -> SmallVec<[usize;64]>)` ve `.with_render_fn::<V>(entity, |&mut V, RenderIndentGuideParams, &mut Window, &mut App| -> SmallVec<[RenderedIndentGuide;12]>)`. `IndentGuides` `UniformListDecoration` trait'ini uyguladığı için `uniform_list(...).with_decoration(guides)` ile bağlanır. GPUI tarafında `impl<T: UniformListDecoration + 'static> UniformListDecoration for Entity<T>` blanket impl'i mevcuttur — bir dekorasyon tipi `Entity<T>` içinde saklanıp aynı `.with_decoration(...)` yüzeyine geçirilebilir.
- **Tab:** `Tab`, `TabBar`, `TabPosition::{First, Middle(Ordering), Last}`, `TabCloseSide::{Start, End}`.
- **Yerleşim yardımcıları:** `h_flex()`, `v_flex()`, `h_group*()`, `v_group*()`, `Divider` (`DividerColor::{Border, BorderFaded, BorderVariant}`), `divider()` ve `vertical_divider()` serbest fn yapıcıları, `Scrollbars`. `Stack` ve `Group` adında genel bir struct yoktur; bunlar yardımcı fonksiyon aileleridir.
- **Veri:** `Table`, `TableInteractionState`, `ColumnWidthConfig::{Static { widths, table_width }, Redistributable { columns_state, table_width }, Resizable(Entity<ResizableColumnsState>)}` — bu üçlü tablo kolon modunu belirler: `Static` resize handle vermez, `Redistributable` toplam tablo genişliğini sabit tutarak komşuya devreder, `Resizable` her kolonu bağımsız büyütür ve toplam genişlik değişir. `StaticColumnWidths::{Auto, Explicit(TableRow<DefiniteLength>)}` Static modun alt seçimidir. `ResizableColumnsState`, `RedistributableColumnsState`, `HeaderResizeInfo`, `TableResizeBehavior::{None, Resizable, MinSize(f32)}`, `TableRow<T>` (`pub struct TableRow<T>(Vec<T>)` — kolon sayısı doğrulanmış satır), `UncheckedTableRow<T>` (`pub type UncheckedTableRow<T> = Vec<T>` — doğrulamasız satır; struct değil tip takma adıdır), `TableRenderContext`, `IntoTableRow` trait, `render_table_row`, `render_table_header`, `bind_redistributable_columns()` ve `render_redistributable_columns_resize_handles()`.
- **Geri bildirim:** `Banner`, `Callout` (`BorderPosition::{Top, Bottom}`), `Modal`, `ModalHeader`, `ModalFooter`, `ModalRow`, `Section`, `SectionHeader`, `AlertModal`, `AnnouncementToast`, `CountBadge`, `Indicator`, `ProgressBar`, `CircularProgress`. `ui::Notification` bağımsız bir bileşen değildir; workspace bildirim sistemi ayrı `workspace::notifications` trait'leriyle yönetilir.
- **Diğer:** `Avatar`, `AvatarAudioStatusIndicator` (`AudioStatus::{Muted, Deafened}` enum'u ile), `AvatarAvailabilityIndicator` (`CollaboratorAvailability::{Free, Busy}` enum'u ile), `Facepile`, `Chip`, `DiffStat`, `Disclosure`, `GradientFade`, `Vector`, `VectorName`, `KeyBinding`, `KeybindingHint`, `Key`, `KeyIcon` (keybinding sub-primitives — `text_for_keystroke` veya `render_modifiers` serbest fn'leriyle birlikte kullanılır), `Navigable` ve `NavigableEntry`. `Image` adında genel bir Zed UI bileşeni yoktur; raster görsel için GPUI `img(...)`/`ImageSource`, paketlenmiş SVG için `Vector` kullanılır.
- **AI/collab özel:** `AiSettingItem` (`AiSettingItemStatus`, `AiSettingItemSource`), `AgentSetupButton`, `ConfiguredApiCard`, `ParallelAgentsIllustration`, `ThreadItem` (`AgentThreadStatus`, `WorktreeKind`, `ThreadItemWorktreeInfo`), `CollabNotification`, `UpdateButton`.

**Bileşen API yüzeyi.** Bazı bileşenlerin daha az bilinen ama önemli ayrıntıları vardır:

- `Table::pin_cols(n)` ilk `n` kolonu yatay scroll sırasında sabit tutar. `ColumnWidthConfig::Resizable` ile birlikte kullanılır; `n == 0` veya `n >= cols` durumunda tek parçalı yerleşime düşülür. Sabitli yerleşimde başlık ve satırların scroll edilebilir bölümleri ortak `ScrollHandle` ile senkron tutulur.
- `ResizableColumnsState::drag_to(col_idx, drag_x, rem_size)` test edilebilir, düşük seviyeli yeniden boyutlandırma yüzeyidir. Sabitli yerleşimde drag koordinatı doğal, scroll edilmemiş kolon şeridi koordinatıdır; `on_drag_move` yatay scroll offset'ini bu hesabı yapmak için alır.
- `ProjectEmptyState::new(label, focus_handle, open_project_key_binding)` ortak boş-proje UI'ıdır. Project panel, Git panel ve Threads sidebar aynı "Open Project / Clone Repository" bileşenini kullanır.
- `Button::loading(true)` `start_icon` yerine dönen `LoadCircle` yükleme göstergesini çizer. Yükleme durumu açıkken ayrıca başlangıç ikonu beklenmez; bileşen bunu bilinçli olarak bastırır.

**Genel kural.** Yeni UI yazılırken aşağıdaki kararlar tutarlılığı korur:

- Zed içinde ham `div().on_click(...)` ile buton üretmeden önce `Button` veya `IconButton` tercih edilir.
- Yalnız görsel veya tek seferlik bir parça için `RenderOnce`, stateful view için `Render` kullanılır.
- Listeler çok büyükse `list` veya `uniform_list` tercih edilir.
- Tooltip, popover ve bağlam menüsü için hazır bileşenler kullanılır; odaklanma/odak kaybı kapanma davranışı orada zaten çözülmüştür.

## ManagedView, DismissEvent, Modal, Popover ve Tooltip Yaşam Döngüsü

`ManagedView` GPUI'da yaşam döngüsü başka bir view tarafından yönetilen UI parçaları için kullanılan bir blanket trait'tir:

```rust
pub trait ManagedView: Focusable + EventEmitter<DismissEvent> + Render {}
```

Bir modal, popover veya bağlam menüsü kapatılmak istediğinde kendi entity context'inden `DismissEvent` yayar:

```rust
impl EventEmitter<DismissEvent> for ModalGorunumu {}

fn kapat(&mut self, baglam: &mut Context<Self>) {
    baglam.emit(DismissEvent);
}
```

**Zed/UI bileşenleri.** ManagedView'i kullanan başlıca bileşenler:

- `ContextMenu` — `ManagedView` uygular; komut listesi ve ayırıcı yönetimi için kullanılır.
- `PopoverMenu<M: ManagedView>` — anchor elementten odaklanılabilen popover açar; `PopoverMenuHandle<M>` dışarıdan toggle veya kapatma için tutulabilir.
- `right_click_menu(id)` — bağlam menüsünü fare olay akışına bağlayan UI yardımcısıdır.
- Workspace modal katmanı `ModalView` veya `ToastView` gibi katmanlarda `DismissEvent` aboneliği ile kapanmayı yönetir; `on_before_dismiss` varsa kapanmadan önce çağrılır.

**Tooltip.** Tooltip iki ayrı fluent metot üzerinden tanımlanır:

- Element fluent API: `.tooltip(|window, cx| AnyView)` ve `.hoverable_tooltip(|window, cx| AnyView)`.
- Buyruk tarzı `Interactivity` API aynı geri çağrı imzasını kullanır.
- `hoverable_tooltip` fare tooltip içine girdiğinde kapanmaz; normal tooltip ise işaretçi sahip elementten ayrıldığında kaldırılır.

**Tuzaklar.** Yaşam döngüsü yönetiminde sık görülen hatalar:

- Modal veya popover view `Focusable` sağlamadığında klavye ve dismiss davranışı eksik kalır.
- `DismissEvent` yayan entity'nin aboneliği saklanmadığında katman kapanma geri çağrısı düşer.
- Aynı element üzerinde birden fazla tooltip tanımlamak hata ayıklama assert'ine yol açar.

---
