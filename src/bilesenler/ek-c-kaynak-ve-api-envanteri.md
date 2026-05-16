# Ek C. Kaynak ve API Envanteri

Bu ek, rehberde anlatılan bileşenlerin kaynak dosyalarını, export yollarını, prelude durumlarını ve preview desteklerini tek yerde toplar. Ayrıntılı kullanım notları ilgili bileşen başlıklarında anlatılır; burası ise hızlı bakış için başvurulacak referans sayfasıdır.

## Export modeli

`crates/ui/src/components.rs`, component modüllerini crate seviyesine açar:

```rust
mod button;
mod icon;
mod label;

pub use button::*;
pub use icon::*;
pub use label::*;
```

Bu düzen nedeniyle bileşenlerin büyük çoğunluğu `ui::Button`, `ui::Icon`, `ui::Label` gibi doğrudan crate kökünden çağrılır. Alt modüller de kendi içlerinde `pub use *` kullanır. Örneğin `crates/ui/src/components/button.rs`, `Button`, `IconButton`, `ButtonLike`, `CopyButton`, `SplitButton` ve toggle button tiplerini dışarı açar.

`crates/ui/src/ui.rs` ise Zed UI crate'inin gerçek export kapısıdır:

```rust
pub mod component_prelude;
mod components;
pub mod prelude;
mod styles;
mod traits;
pub mod utils;

pub use components::*;
pub use prelude::*;
pub use styles::*;
pub use traits::animation_ext::*;
```

Bunun günlük kullanım açısından sonuçları şunlardır:

- `components`, `styles` ve `traits` modülleri kaynakta `mod` yani crate-içi olarak tanımlandığı için doğrudan public path değildir. `ui::components::button::Button` gibi yollar yoktur. Tüketici kod `ui::Button`, `ui::ContextMenu`, `ui::Color`, `ui::TextSize` gibi crate kökü re-export'larını kullanır. `pub use components::*`, `pub use styles::*` ve `pub use traits::animation_ext::*` ifadeleri bu özel modüllerin içindeki public adları crate köküne taşır.
- Public bir alt modül olarak kalıcı şekilde görünen yollar `ui::prelude`, `ui::component_prelude`, `ui::utils` ile re-export zincirinden gelen `ui::animation`, `ui::scrollbars` ve `ui::table_row` yollarıdır.
- `traits::animation_ext` crate köküne açıldığı için `ui::CommonAnimationExt` doğrudan import edilebilir. Buna karşılık `traits::transformable` crate kökünden açılmaz; kaynakta `pub trait Transformable` görünmesi tek başına onun bir tüketici API'si olduğu anlamına gelmez.

`crates/ui/src/prelude.rs` ise daha seçici davranır. Her bileşeni değil, sık kullanılan temel UI primitive'lerini ve trait'lerini getirir:

- GPUI yeniden ihraçları (`pub use gpui::prelude::*` ile ek olarak): `AbsoluteLength`, `AnyElement`, `App`, `Context`, `DefiniteLength`, `Div`, `Element`, `ElementId`, `InteractiveElement`, `ParentElement`, `Pixels`, `Rems`, `RenderOnce`, `SharedString`, `Styled`, `Window`, `div`, `px`, `relative`, `rems`.
- Component preview tipleri: `Component`, `ComponentScope`, `example_group`, `example_group_with_title`, `single_example`, `RegisterComponent`.
- Ortak trait'ler: `Clickable`, `Disableable`, `FixedWidth`, `StyledExt`, `Toggleable`, `VisibleOnHover`.
- Tasarım sistemi token'ları ve yardımcıları: `DynamicSpacing`, `PlatformStyle`, `Severity`, `StyledTypography`, `TextSize`, `rems_from_px`, `vh`, `vw`, `ActiveTheme`.
- Animasyon yardımcıları: `AnimationDirection`, `AnimationDuration`, `DefaultAnimations`.
- Sık kullanılan bileşenler ve enum'lar: `Button`, `IconButton`, `SelectableButton`, `ButtonCommon`, `ButtonSize`, `ButtonStyle`, `Color`, `Headline`, `HeadlineSize`, `Icon`, `IconName`, `IconPosition`, `IconSize`, `Label`, `LabelCommon`, `LabelSize`, `LineHeightStyle`, `LoadingLabel`, `h_flex`, `v_flex`, `h_group*`, `v_group*`.

Rehberdeki örneklerde izlenen kural şudur: her örnek önce `use ui::prelude::*;` ile başlar. Prelude içinde bulunmayan bileşenler ek bir `use ui::{...};` satırıyla ayrıca belirtilir.

### Public Yüzey Özeti (mevcut `../zed`)

Aşağıdaki liste `crates/ui/src/components`, `crates/ui/src/styles`, `crates/ui/src/utils`, `crates/ui/src/traits`, `crates/component/src` ve `crates/ui_input/src` public yüzeyini özetler. Ayrıntılı builder imzaları ilgili başlıkta yer alır; lifecycle API'leri ise bu tablonun hemen altında ayrıca ele alınır.

| Alan | Public adlar |
| :-- | :-- |
| Crate kapıları | `ui::prelude`, `ui::component_prelude`, `ui::utils`, `ui::animation`, `ui::scrollbars`, `ui::table_row` |
| Button ailesi | `Button`, `IconButton`, `ButtonLike`, `ButtonLink`, `CopyButton`, `SplitButton`, `ToggleButtonGroup`, `ToggleButtonSimple`, `ToggleButtonWithIcon`, `ButtonBuilder`, `ButtonConfiguration`, `SelectableButton`, `ButtonCommon`, `ButtonSize`, `ButtonStyle`, `IconPosition`, `KeybindingPosition`, `TintColor`, `IconButtonShape`, `SplitButtonStyle`, `SplitButtonKind`, `ToggleButtonPosition`, `ToggleButtonGroupStyle`, `ToggleButtonGroupSize` |
| Label / ikon | `Label`, `LabelLike`, `HighlightedLabel`, `LoadingLabel`, `SpinnerLabel`, `SpinnerVariant`, `LabelCommon`, `LabelSize`, `LineHeightStyle`, `Icon`, `IconName`, `IconSize`, `AnyIcon`, `IconWithIndicator`, `DecoratedIcon`, `IconDecoration`, `IconDecorationKind`, `KnockoutIconName`, `highlight_ranges` |
| Form ve popup | `checkbox`, `switch`, `Checkbox`, `Switch`, `SwitchField`, `ToggleStyle`, `SwitchColor`, `SwitchLabelPosition`, `DropdownMenu`, `DropdownStyle`, `ContextMenu`, `ContextMenuEntry`, `ContextMenuItem`, `DocumentationAside`, `DocumentationSide`, `Popover`, `POPOVER_Y_PADDING`, `PopoverMenu`, `PopoverMenuHandle`, `PopoverTrigger`, `RightClickMenu`, `right_click_menu`, `Tooltip`, `LinkPreview`, `tooltip_container` |
| Liste / tree | `List`, `EmptyMessage`, `ListItem`, `ListItemSpacing`, `ListHeader`, `ListSubHeader`, `ListSeparator`, `ListBulletItem`, `TreeViewItem`, `StickyCandidate`, `StickyItems`, `StickyItemsDecoration`, `sticky_items`, `IndentGuideColors`, `IndentGuides`, `indent_guides`, `RenderIndentGuideParams`, `RenderedIndentGuide`, `IndentGuideLayout` |
| Tablo | `UncheckedTableRow`, `Table`, `TableInteractionState`, `TableRenderContext`, `ColumnWidthConfig`, `StaticColumnWidths`, `ResizableColumnsState`, `RedistributableColumnsState`, `TableResizeBehavior`, `HeaderResizeInfo`, `render_table_row`, `render_table_header`, `bind_redistributable_columns`, `render_redistributable_columns_resize_handles`, `table_row::TableRow`, `table_row::IntoTableRow` |
| Feedback / durum | `Banner`, `Callout`, `BorderPosition`, `Modal`, `ModalHeader`, `ModalRow`, `ModalFooter`, `Section`, `SectionHeader`, `AlertModal`, `AnnouncementToast`, `CountBadge`, `Indicator`, `ProgressBar`, `CircularProgress`, `Severity` |
| Tab | `Tab`, `TabBar`, `TabPosition`, `TabCloseSide` |
| Layout / divider | `h_flex`, `v_flex`, `h_group`, `h_group_sm`, `h_group_lg`, `h_group_xl`, `v_group`, `v_group_sm`, `v_group_lg`, `v_group_xl`, `Divider`, `DividerColor`, `divider`, `vertical_divider` |
| Scrollbar | `Scrollbars`, `ScrollAxes`, `ScrollbarStyle`, `ScrollableHandle`, `WithScrollbar`, `on_new_scrollbars`, `EDITOR_SCROLLBAR_WIDTH` (ek olarak `ui::scrollbars` modülü altında `ShowScrollbar`, `ScrollbarVisibility`, `ScrollbarAutoHide`) |
| Diğer component'ler | `Avatar`, `AudioStatus`, `AvatarAudioStatusIndicator`, `CollaboratorAvailability`, `AvatarAvailabilityIndicator`, `Facepile`, `EXAMPLE_FACES`, `Chip`, `DiffStat`, `Disclosure`, `GradientFade`, `Vector`, `VectorName`, `KeyBinding`, `Key`, `KeyIcon`, `KeybindingHint`, `Navigable`, `NavigableEntry`, `ProjectEmptyState`, `render_keybinding_keystroke`, `render_modifiers`, `text_for_action`, `text_for_keystrokes`, `text_for_keybinding_keystrokes`, `text_for_keystroke` |
| AI / collab | `AiSettingItem`, `AiSettingItemStatus`, `AiSettingItemSource`, `AgentSetupButton`, `ThreadItem`, `AgentThreadStatus`, `ThreadItemWorktreeInfo`, `WorktreeKind`, `ConfiguredApiCard`, `ParallelAgentsIllustration`, `CollabNotification`, `UpdateButton` |
| Style / trait / utils | `Color`, `ElevationIndex`, `DynamicSpacing`, `ui_density`, `PlatformStyle`, `StyledTypography`, `TextSize`, `Headline`, `HeadlineSize`, `AnimationDuration`, `AnimationDirection`, `DefaultAnimations`, `CommonAnimationExt`, `Clickable`, `Disableable`, `FixedWidth`, `StyledExt`, `Toggleable`, `ToggleState`, `VisibleOnHover`, `WithRemSize`, `SearchInputWidth`, `FormatDistance`, `DateTimeType`, `CornerSolver`, `inner_corner_radius`, `apca_contrast`, `ensure_minimum_contrast`, `calculate_contrast_ratio`, `format_distance`, `format_distance_from_now`, `is_light`, `capitalize`, `reveal_in_file_manager_label`, `platform_title_bar_height`, `TRAFFIC_LIGHT_PADDING`, `BASE_REM_SIZE_IN_PX`, `rems_from_px`, `vw`, `vh`, `theme_is_transparent` |
| Component preview | `components`, `init`, `register_component`, `Component`, `ComponentFn`, `ComponentRegistry`, `ComponentId`, `ComponentMetadata`, `ComponentStatus`, `ComponentScope`, `ComponentExample`, `ComponentExampleGroup`, `single_example`, `empty_example`, `example_group`, `example_group_with_title` |
| `ui_input` | `InputField`, `InputFieldStyle`, `ErasedEditor`, `ErasedEditorEvent`, `ERASED_EDITOR_FACTORY` |

**Public görünen ama kullanım yüzeyi olmayanlar.** Kaynakta `MenuHandleElementState`, `RequestLayoutState`, `PrepaintState`, `PopoverMenuElementState`, `PopoverMenuFrameState` ve `ScrollbarPrepaintState` element ve layout state taşıyıcılarıdır. Bu tiplerin kaynakta `pub struct` olarak görünmesi, tüketiciye önerilen builder API'leri oldukları anlamına gelmez. `Element` implementasyonu içinde `RequestLayoutState` ve `PrepaintState` layout, prepaint ve paint geçişleri arasında veri taşır. `MenuHandleElementState` ile `PopoverMenuElementState` ise hover veya açık menü durumlarını element id'sine bağlar.

**Callback yüzeyi olarak public, state taşıyıcı değil.** `RenderIndentGuideParams`, `RenderedIndentGuide` ve `IndentGuideLayout` `IndentGuides` callback'lerinin sözleşme tipleridir. `IndentGuides::with_render_fn(...)` callback'i `RenderIndentGuideParams`'ı girdi olarak alır ve bir `SmallVec<[RenderedIndentGuide; 12]>` döndürür; `IndentGuides::on_click(...)` ise ilk parametre olarak bir `&IndentGuideLayout` verir. Bu nedenle üç tip de "IndentGuides" başlığında, alanlarıyla birlikte listelenir; bir element state taşıyıcısı sayılmaz.

Benzer sözleşme tiplerine ait public alanlı tipler şunlardır:

- `TableRenderContext`, `render_table_row(...)` ve `render_table_header(...)` çağrıları için düşük seviyeli bir render bağlamıdır. `TableInteractionState` gibi saklanan bir view state'i değildir; `striped`, `show_row_borders`, `column_widths`, `map_row`, `disable_base_cell_style`, `pinned_cols` ve `h_scroll_handle` alanları render helper'larına aktarılır.
- `HeaderResizeInfo`, header resize ve reset sözleşmesidir. `resize_behavior` alanı public şekilde okunur, ancak kolon state'i içeride bir `WeakEntity` olarak tutulur. Reset için kullanılan public yol `reset_column(...)` metodudur.
- `DocumentationAside`, context menu custom entry'leri için bir `side` değeri ve bir `render` callback'ini taşır; tek başına render edilen bir component değildir.
- `ThreadItemWorktreeInfo`, `ThreadItem::worktrees(...)` için domain bir veri nesnesidir. `worktree_name`, `branch_name`, `full_path`, `highlight_positions` ve `kind` alanları thread metadata satırını besler.
- `ComponentExample` ve `ComponentExampleGroup`, component preview layout verileridir. Alanları public olsa da normal preview kodunda `single_example(...)`, `empty_example(...)`, `example_group(...)` ve builder metotları tercih edilir.
- `NavigableEntry`, `Navigable` wrapper'ına eklenen focus ve scroll entry sözleşmesidir. `focus_handle` ve `scroll_anchor` alanları publictir; ancak çoğu kullanım `NavigableEntry::new(...)` veya `focusable(...)` üzerinden kurulur.

### Ek Public API Notları

Kaynakta birkaç ayrım özellikle dikkat ister:

- `tab.rs` ve `tab_bar.rs`: `Tab`, `TabBar`, `TabPosition` ve `TabCloseSide` ayrı bir Tab yüzeyidir. Zed içinde pane tab bar akışı `workspace/src/pane.rs` dosyasında `TabPosition::{First, Middle(Ordering), Last}`, `TabCloseSide::{Start, End}` ve `TabBar::new(...)` ile kurulur.
- `stack.rs`, `group.rs` ve `divider.rs`: `h_flex`, `v_flex`, `h_group*`, `v_group*`, `Divider`, `DividerColor`, `divider()` ve `vertical_divider()` layout ve divider yüzeyidir; `Stack` veya `Group` adlı bir public struct yoktur.
- `scrollbar.rs`: `Scrollbars`, `ScrollAxes`, `ScrollbarStyle`, `ScrollableHandle`, `WithScrollbar`, `on_new_scrollbars` ve `EDITOR_SCROLLBAR_WIDTH` root export'tur; `ShowScrollbar`, `ScrollbarVisibility` ve `ScrollbarAutoHide` ise `ui::scrollbars` public alt modülü altındadır. Zed `main.rs` dosyası `on_new_scrollbars::<SettingsStore>(cx)` çağırır; editor ve panel kodları ise `Scrollbars::for_settings::<...>()` kullanır.
- `keybinding.rs`: `render_keybinding_keystroke`, `render_modifiers`, `text_for_action`, `text_for_keystrokes`, `text_for_keybinding_keystrokes` ve `text_for_keystroke` public free helper'lardır. Bunlar `KeyBinding` component'inin constructor'ı değildir; arama, keymap editor, which-key ve quick action preview gibi yerlerde doğrudan kullanılırlar.

Public tuple struct alanları ile payload taşıyan enum variant'ları da construction yüzeyinin bir parçasıdır:

- Public tuple alanları: `ComponentId(pub &'static str)` ve `ScrollbarAutoHide(pub bool)`. İlki registry id değerini, ikincisi ise global auto-hide bayrağını taşır.
- Payload variant'ları: `SplitButtonKind::{ButtonLike(ButtonLike), IconButton(IconButton)}`, `ToggleButtonGroupSize::Custom(Rems)`, `StaticColumnWidths::Explicit(TableRow<DefiniteLength>)`, `LabelSize::Custom(Rems)`, `EmptyMessage::{Text(SharedString), Element(AnyElement)}`, `ToggleStyle::{ElevationBased(ElevationIndex), Custom(Hsla)}`, `SwitchColor::Custom(Hsla)`, `Color::Player(u32)` ve `DateTimeType::{Naive(NaiveDateTime), Local(DateTime<Local>)}` gibi variant'lar yalnızca bir isim değil, veri taşıyan public bir construction yüzeyidir.

Public trait implementasyonları da dış crate için ergonomik bir construction yüzeyi oluşturur. Kaynakta kullanılan dönüşümler:

- `ToggleState`: `From<bool>` ve `From<Option<bool>>`; `None`, `Indeterminate` anlamına gelir.
- `Color`: `From<Hsla>`, `From<TintColor>`, `From<ButtonStyle>` ve `From<SwitchColor>`. `ButtonStyle::Tinted(tint)` tint rengini taşır; diğer button stilleri `Color::Default` olarak çözümlenir. `SwitchColor::Custom(_)` de `Color` dönüşümünde custom rengi taşımaz, `Default` döner.
- `Hsla`: `proptest` feature'ı açıkken `Hsla::opaque_strategy()` ve `Arbitrary for Hsla` property test yüzeyi sağlar. Bu üretim component API'si değil, renk ve tema algoritmalarını test etmek için kullanılan GPUI yardımcı yüzeyidir.
- `AnyIcon`: `From<Icon>` ve `From<AnimationElement<Icon>>`; `Icon` ise `From<IconName>` sağlar.
- `SplitButtonKind`: `From<IconButton>` ve `From<ButtonLike>`. Bu yüzden `SplitButton::new(left, right)` sol parçada iki component türünü de kabul eder.
- `EmptyMessage`: `From<String>`, `From<&str>` ve `From<AnyElement>`.
- `SectionHeader`: `From<SharedString>` ve `From<&'static str>`.
- `ContextMenuItem`: `From<ContextMenuEntry>`.
- `AnimationDuration`: `impl Into<std::time::Duration>`; iç gövdesi `self.duration()` çağırır, dolayısıyla `Duration::from(duration)` ya da `gpui::Animation::new(AnimationDuration::Fast.into())` gibi kullanımlarda tipi otomatik çözer.

Aynı kategoride, isim olarak görünmeyen ama trigger ergonomisi açısından kritik olan blanket impl'ler `popover_menu.rs` içinde tanımlanır:

- `impl<T: Clickable> Clickable for gpui::AnimationElement<T>` ve `impl<T: Toggleable> Toggleable for gpui::AnimationElement<T>` blanket impl'leri `.map_element(...)` ile delege eder. Bu sayede `IconButton::new(...).with_rotate_animation(2)` gibi bir `AnimationElement<IconButton>` döndüren zincir, `PopoverTrigger` (`IntoElement + Clickable + Toggleable + 'static` alias'ı) için kabul edilir. Bu trait'ler olmasa `PopoverMenu::trigger(...)`, animasyonlu icon button'ları reddederdi.

Private tiplerdeki dönüşümler ise bir tüketici yüzeyi sayılmaz. Örneğin `tooltip.rs` içindeki private `Title` enum'u için `From<SharedString>` vardır; ancak dış API `Tooltip::text(...)`, `Tooltip::simple(...)` ve `Tooltip::for_action*` constructor'ları üzerinden görünür kalır.

### Uygulama Crate'lerinde Component Benzeri Yüzeyler

Bu rehberin ana envanteri `crates/ui` public bileşenlerini kapsar. Yeni Zed düzenlemelerinde bazı önemli yüzeyler uygulama crate'lerinde tanımlı olduğu için public `ui::*` API tablosuna girmez ama bileşen kullanımı açısından bilinmelidir:

- `activity_indicator::ActivityIndicator`, workspace status item olarak LSP, debug, git, filesystem, extension ve formatlama aktivitelerini tek trigger'da gösterir. Loading durumları `Button::loading(true)` üzerinden, statik durumlar `Button::start_icon(...)` üzerinden çizilir.
- `agent_ui` içindeki `RulesToSkillsModal`, `AlertModal` tabanlı bir modal view'dır. Title bar'daki Skills onboarding banner'ından açılır ve Skills tanıtımı veya migration özetini gösterir.
- `agent_ui::completion_provider`, `PromptContextType::Skill`, `AvailableSkill` ve `MentionUri::Skill` üzerinden slash completion ve mention akışını sağlar. Skill completion'ları Agent Commands grubundan önce listelenir ve seçildiğinde ilgili `SKILL.md` dosyasına açılabilen bir mention link'i üretir.
- `settings_ui` sayfaları `SettingItem`, renderer registry ve tool permission satırlarıyla component'leri birleştirir. `CompletionMenuItemKind` dropdown renderer'ı, `git.show_stage_restore_buttons` switch/boolean satırı ve `skill` tool permission satırı bu katmandadır.

### Lifecycle API İmzaları

Bu grup, callback imzaları veya generic bound'ları yüzünden en kolay yanlış aktarılabilecek yüzeydir:

```rust
pub fn right_click_menu<M: ManagedView>(
    id: impl Into<ElementId>,
) -> RightClickMenu<M>;

impl<M: ManagedView> RightClickMenu<M> {
    pub fn menu(
        self,
        f: impl Fn(&mut Window, &mut App) -> Entity<M> + 'static,
    ) -> Self;
    pub fn trigger<F, E>(self, e: F) -> Self
    where
        F: FnOnce(bool, &mut Window, &mut App) -> E + 'static,
        E: IntoElement + 'static;
}

impl<M: ManagedView> PopoverMenuHandle<M> {
    pub fn show(&self, window: &mut Window, cx: &mut App);
    pub fn hide(&self, cx: &mut App);
    pub fn toggle(&self, window: &mut Window, cx: &mut App);
    pub fn is_deployed(&self) -> bool;
    pub fn is_focused(&self, window: &Window, cx: &App) -> bool;
    pub fn refresh_menu(
        &self,
        window: &mut Window,
        cx: &mut App,
        new_menu_builder: Rc<dyn Fn(&mut Window, &mut App) -> Option<Entity<M>>>,
    );
}

impl<M: ManagedView> PopoverMenu<M> {
    pub fn new(id: impl Into<ElementId>) -> Self;
    pub fn menu(
        self,
        f: impl Fn(&mut Window, &mut App) -> Option<Entity<M>> + 'static,
    ) -> Self;
    pub fn trigger<T: PopoverTrigger>(self, t: T) -> Self;
    pub fn trigger_with_tooltip<T: PopoverTrigger + ButtonCommon>(
        self,
        t: T,
        tooltip_builder: impl Fn(&mut Window, &mut App) -> AnyView + 'static,
    ) -> Self;
}

impl ContextMenu {
    pub fn build(
        window: &mut Window,
        cx: &mut App,
        f: impl FnOnce(Self, &mut Window, &mut Context<Self>) -> Self,
    ) -> Entity<Self>;
    pub fn build_persistent(
        window: &mut Window,
        cx: &mut App,
        builder: impl Fn(Self, &mut Window, &mut Context<Self>) -> Self + 'static,
    ) -> Entity<Self>;
}
```

**Zed kullanım paritesi.** Activity indicator, file finder, status bar, pane tab bar, git branch picker ve settings UI aynı modeli izler: trigger `Button` veya `IconButton` ile kurulur, menü `ContextMenu::build(...)` içinde üretilir ve popover açıkken focus `ManagedView`/`DismissEvent` zinciriyle yönetilir. Bir context menu'yu elde tutmak gerektiğinde `PopoverMenuHandle<ContextMenu>` view state'inde saklanır. Sağ tık menülerinde ise `right_click_menu(id).trigger(...).menu(...)` akışı kullanılır.

## Ortak trait ve sistem tipleri

| Tip | Kaynak | Not |
| :-- | :-- | :-- |
| `Component` | `crates/component/src/component.rs` | Component gallery kaydı ve preview sözleşmesi. |
| `ComponentScope` | `crates/component/src/component.rs` | Preview'ların kategori/scope ayrımı. |
| `RegisterComponent` | `crates/ui/src/prelude.rs` üzerinden `ui_macros` | Component registry'ye otomatik kayıt için derive makrosu. |
| `RenderOnce` | `gpui`, `ui::prelude::*` içinde | Zed UI bileşenlerinde yaygın render modeli. |
| `ParentElement` | `gpui`, `ui::prelude::*` içinde | Slot ve child kabul eden bileşenlerde kullanılır. |
| `Clickable` | `crates/ui/src/traits/clickable.rs` | `.on_click(...)` yüzeyi taşıyan bileşenler. |
| `Toggleable` / `ToggleState` | `crates/ui/src/traits/toggleable.rs` | Selected, unselected ve indeterminate state modeli. |
| `Disableable` | `crates/ui/src/traits/disableable.rs` | Disabled builder yüzeyi. |
| `FixedWidth` | `crates/ui/src/traits/fixed.rs` | Sabit genişlik davranışı. |
| `VisibleOnHover` | `crates/ui/src/traits/visible_on_hover.rs` | Hover grubuna bağlı görünürlük davranışı. |
| `StyledExt` | `crates/ui/src/traits/styled_ext.rs` | Flex, elevation, border ve debug background yardımcıları. |
| `CommonAnimationExt` | `crates/ui/src/traits/animation_ext.rs` | Döndürme animasyonu gibi ortak animation extension yüzeyi. |
| `Transformable` | `crates/ui/src/traits/transformable.rs` | Kaynakta `pub trait`, ancak `ui.rs` tarafından re-export edilmez; tüketici API'si olarak değil, `CommonAnimationExt` bound'u olarak değerlendirilir. |
| `LabelCommon` | `crates/ui/src/components/label/label_like.rs` | Label ailesinin ortak size/color/weight/truncation yüzeyi. |
| `ButtonCommon` | `crates/ui/src/components/button/button_like.rs` | Button ailesinin ortak accessor/builder yüzeyi: `id`, `style`, `size`, `tooltip`, `tab_index`, `layer`, `track_focus`. `Clickable + Disableable` supertrait. |
| `SelectableButton` | `crates/ui/src/components/button/button_like.rs` | `Button`, `IconButton`, `ButtonLike` için seçilebilirlik sözleşmesi. |
| `WithScrollbar` / `ScrollableHandle` | `crates/ui/src/components/scrollbar.rs` | Elementlere özel scrollbar bağlama ve scroll handle soyutlaması. |
| `IntoTableRow` | `crates/ui/src/components/data_table/table_row.rs` | `Vec<T>` değerlerini kolon sayısı doğrulanmış `TableRow<T>` tipine dönüştürme trait'i. |

## Kaynak indeksi

| Kategori | Bileşen / API | Tanım kaynağı | Export | Prelude | Preview |
| :-- | :-- | :-- | :-- | :-- | :-- |
| Metin | `Label` | `crates/ui/src/components/label/label.rs` | `ui::Label` | Evet | Evet |
| Metin | `LabelLike` | `crates/ui/src/components/label/label_like.rs` | `ui::LabelLike` | Hayır | Evet |
| Metin | `Headline` | `crates/ui/src/styles/typography.rs` | `ui::Headline` | Evet | Evet |
| Metin | `HighlightedLabel` | `crates/ui/src/components/label/highlighted_label.rs` | `ui::HighlightedLabel` | Hayır | Evet |
| Metin | `LoadingLabel` | `crates/ui/src/components/label/loading_label.rs` | `ui::LoadingLabel` | Evet | Hayır |
| Metin | `SpinnerLabel` | `crates/ui/src/components/label/spinner_label.rs` | `ui::SpinnerLabel` | Hayır | Evet |
| Tasarım | `Color` | `crates/ui/src/styles/color.rs` | `ui::Color` | Evet | Evet |
| Buton | `Button` | `crates/ui/src/components/button/button.rs` | `ui::Button` | Evet | Evet |
| Buton | `IconButton` | `crates/ui/src/components/button/icon_button.rs` | `ui::IconButton` | Evet | Evet |
| Buton | `ButtonLike` | `crates/ui/src/components/button/button_like.rs` | `ui::ButtonLike` | Hayır | Evet |
| Buton | `SelectableButton` | `crates/ui/src/components/button/button_like.rs` | `ui::SelectableButton` | Evet | Trait |
| Buton | `ButtonLink` | `crates/ui/src/components/button/button_link.rs` | `ui::ButtonLink` | Hayır | Evet |
| Buton | `CopyButton` | `crates/ui/src/components/button/copy_button.rs` | `ui::CopyButton` | Hayır | Evet |
| Buton | `SplitButton` | `crates/ui/src/components/button/split_button.rs` | `ui::SplitButton` | Hayır | Hayır |
| Buton | `ToggleButtonGroup` | `crates/ui/src/components/button/toggle_button.rs` | `ui::ToggleButtonGroup` | Hayır | Evet |
| Buton | `ToggleButtonSimple` / `ToggleButtonWithIcon` | `crates/ui/src/components/button/toggle_button.rs` | `ui::ToggleButtonSimple`, `ui::ToggleButtonWithIcon` | Hayır | Yardımcı |
| Buton | `ButtonBuilder` / `ButtonConfiguration` | `crates/ui/src/components/button/toggle_button.rs` | `ui::ButtonBuilder`, `ui::ButtonConfiguration` | Hayır | Sealed helper |
| Buton | `KeybindingPosition` | `crates/ui/src/components/button/button_like.rs` | `ui::KeybindingPosition` | Hayır | Enum |
| İkon | `Icon` | `crates/ui/src/components/icon.rs` | `ui::Icon` | Evet | Evet |
| İkon | `DecoratedIcon` | `crates/ui/src/components/icon/decorated_icon.rs` | `ui::DecoratedIcon` | Hayır | Evet |
| İkon | `IconDecoration` | `crates/ui/src/components/icon/icon_decoration.rs` | `ui::IconDecoration` | Hayır | Hayır |
| İkon | `IconWithIndicator` | `crates/ui/src/components/icon.rs` | `ui::IconWithIndicator` | Hayır | Hayır |
| İkon | `AnyIcon` | `crates/ui/src/components/icon.rs` | `ui::AnyIcon` | Hayır | Enum |
| İkon | `IconName` | `crates/icons/src/icons.rs` | `ui::IconName` | Evet | Enum |
| İkon | `IconSize` | `crates/ui/src/components/icon.rs` | `ui::IconSize` | Evet | Enum |
| Form / Toggle | `Checkbox` | `crates/ui/src/components/toggle.rs` | `ui::Checkbox` | Hayır | Evet |
| Form / Toggle | `Switch` | `crates/ui/src/components/toggle.rs` | `ui::Switch` | Hayır | Evet |
| Form / Toggle | `SwitchField` | `crates/ui/src/components/toggle.rs` | `ui::SwitchField` | Hayır | Evet |
| Form / Input | `InputField` | `crates/ui_input/src/input_field.rs` | `ui_input::InputField` | Hayır | Evet |
| Form / Toggle | `DropdownMenu` | `crates/ui/src/components/dropdown_menu.rs` | `ui::DropdownMenu` | Hayır | Evet |
| Menü / Popup | `ContextMenu` | `crates/ui/src/components/context_menu.rs` | `ui::ContextMenu` | Hayır | Hayır |
| Menü / Popup | `RightClickMenu` | `crates/ui/src/components/right_click_menu.rs` | `ui::RightClickMenu` | Hayır | Hayır |
| Menü / Popup | `Popover` | `crates/ui/src/components/popover.rs` | `ui::Popover` | Hayır | Hayır |
| Menü / Popup | `PopoverMenu` | `crates/ui/src/components/popover_menu.rs` | `ui::PopoverMenu` | Hayır | Hayır |
| Menü / Popup | `Tooltip` | `crates/ui/src/components/tooltip.rs` | `ui::Tooltip` | Hayır | Evet |
| Menü / Popup | `LinkPreview` / `tooltip_container` | `crates/ui/src/components/tooltip.rs` | `ui::LinkPreview`, `ui::tooltip_container` | Hayır | Yardımcı |
| Liste / Tree | `List` | `crates/ui/src/components/list/list.rs` | `ui::List` | Hayır | Evet |
| Liste / Tree | `ListItem` | `crates/ui/src/components/list/list_item.rs` | `ui::ListItem` | Hayır | Evet |
| Liste / Tree | `ListHeader` | `crates/ui/src/components/list/list_header.rs` | `ui::ListHeader` | Hayır | Evet |
| Liste / Tree | `ListSubHeader` | `crates/ui/src/components/list/list_sub_header.rs` | `ui::ListSubHeader` | Hayır | Evet |
| Liste / Tree | `ListSeparator` | `crates/ui/src/components/list/list_separator.rs` | `ui::ListSeparator` | Hayır | Hayır |
| Liste / Tree | `ListBulletItem` | `crates/ui/src/components/list/list_bullet_item.rs` | `ui::ListBulletItem` | Hayır | Evet |
| Liste / Tree | `TreeViewItem` | `crates/ui/src/components/tree_view_item.rs` | `ui::TreeViewItem` | Hayır | Evet |
| Liste / Tree | `StickyItems` | `crates/ui/src/components/sticky_items.rs` | `ui::StickyItems` | Hayır | Hayır |
| Liste / Tree | `IndentGuides` | `crates/ui/src/components/indent_guides.rs` | `ui::IndentGuides` | Hayır | Hayır |
| Tab | `Tab` | `crates/ui/src/components/tab.rs` | `ui::Tab` | Hayır | Evet |
| Tab | `TabBar` | `crates/ui/src/components/tab_bar.rs` | `ui::TabBar` | Hayır | Evet |
| Layout | `h_flex` / `v_flex` | `crates/ui/src/components/stack.rs` | `ui::h_flex`, `ui::v_flex` | Evet | Fonksiyon |
| Layout | `h_group*` / `v_group*` | `crates/ui/src/components/group.rs` | `ui::h_group*`, `ui::v_group*` | Evet | Fonksiyon |
| Layout | `Divider` | `crates/ui/src/components/divider.rs` | `ui::Divider` | Hayır | Evet |
| Layout | `Scrollbars` | `crates/ui/src/components/scrollbar.rs` | `ui::Scrollbars`, `ui::ScrollAxes`, `ui::ScrollbarStyle`, `ui::scrollbars::{ShowScrollbar, ScrollbarVisibility, ScrollbarAutoHide}` | Hayır | Yardımcı |
| Layout | `WithScrollbar` / `on_new_scrollbars` | `crates/ui/src/components/scrollbar.rs` | `ui::WithScrollbar`, `ui::on_new_scrollbars` | Hayır | Trait / setup |
| GPUI primitive | `Div` / `div` | `crates/gpui/src/elements/div.rs` | `gpui::Div`, `gpui::div` | Evet | Primitive |
| GPUI primitive | `Styled` / `ParentElement` / `InteractiveElement` | `crates/gpui/src/styled.rs`, `crates/gpui/src/element.rs`, `crates/gpui/src/elements/div.rs` | `gpui::*` | Evet | Trait |
| GPUI primitive | `canvas` / `Canvas` | `crates/gpui/src/elements/canvas.rs` | `gpui::canvas`, `gpui::Canvas` | Evet | Primitive |
| GPUI primitive | `img` / `Img` / `ImageSource` / `StyledImage` | `crates/gpui/src/elements/img.rs` | `gpui::img`, `gpui::Img`, `gpui::ImageSource`, `gpui::StyledImage` | Evet | Primitive |
| GPUI primitive | `svg` / `Svg` / `Transformation` | `crates/gpui/src/elements/svg.rs` | `gpui::svg`, `gpui::Svg`, `gpui::Transformation` | Evet | Primitive |
| GPUI primitive | `anchored` / `Anchored` | `crates/gpui/src/elements/anchored.rs` | `gpui::anchored`, `gpui::Anchored` | Evet | Primitive |
| GPUI primitive | `deferred` / `Deferred` | `crates/gpui/src/elements/deferred.rs` | `gpui::deferred`, `gpui::Deferred` | Evet | Primitive |
| GPUI primitive | `surface` / `Surface` | `crates/gpui/src/elements/surface.rs` | `gpui::surface`, `gpui::Surface` | Evet | macOS primitive |
| GPUI primitive | `list` / `ListState` | `crates/gpui/src/elements/list.rs` | `gpui::list`, `gpui::ListState` | Evet | Variable-height virtualization |
| GPUI primitive | `uniform_list` / `UniformListScrollHandle` | `crates/gpui/src/elements/uniform_list.rs` | `gpui::uniform_list`, `gpui::UniformListScrollHandle` | Evet | Uniform virtualization |
| GPUI primitive | `StyledText` / `TextLayout` / `InteractiveText` | `crates/gpui/src/elements/text.rs` | `gpui::StyledText`, `gpui::TextLayout`, `gpui::InteractiveText` | Evet | Rich text primitive |
| Veri | `Table` | `crates/ui/src/components/data_table.rs` | `ui::Table` | Hayır | Evet |
| Veri | `TableInteractionState` | `crates/ui/src/components/data_table.rs` | `ui::TableInteractionState` | Hayır | State |
| Veri | `ColumnWidthConfig` | `crates/ui/src/components/data_table.rs` | `ui::ColumnWidthConfig` | Hayır | Enum |
| Veri | `StaticColumnWidths` | `crates/ui/src/components/data_table.rs` | `ui::StaticColumnWidths` | Hayır | Enum |
| Veri | `ResizableColumnsState` | `crates/ui/src/components/data_table.rs` | `ui::ResizableColumnsState` | Hayır | State |
| Veri | `RedistributableColumnsState` | `crates/ui/src/components/redistributable_columns.rs` | `ui::RedistributableColumnsState` | Hayır | State |
| Veri | `TableResizeBehavior` | `crates/ui/src/components/redistributable_columns.rs` | `ui::TableResizeBehavior` | Hayır | Enum |
| Veri | `TableRow` | `crates/ui/src/components/data_table/table_row.rs` | `ui::table_row::TableRow` | Hayır | Yardımcı tip |
| Veri | `IntoTableRow` | `crates/ui/src/components/data_table/table_row.rs` | `ui::table_row::IntoTableRow` | Hayır | Trait |
| Veri | `render_table_row` / `render_table_header` | `crates/ui/src/components/data_table.rs` | `ui::render_table_row`, `ui::render_table_header` | Hayır | Fonksiyon |
| Veri | `bind_redistributable_columns` | `crates/ui/src/components/redistributable_columns.rs` | `ui::bind_redistributable_columns` | Hayır | Fonksiyon |
| Veri | `render_redistributable_columns_resize_handles` | `crates/ui/src/components/redistributable_columns.rs` | `ui::render_redistributable_columns_resize_handles` | Hayır | Fonksiyon |
| Feedback | `Banner` | `crates/ui/src/components/banner.rs` | `ui::Banner` | Hayır | Evet |
| Feedback | `Callout` | `crates/ui/src/components/callout.rs` | `ui::Callout` | Hayır | Evet |
| Feedback | `Modal` | `crates/ui/src/components/modal.rs` | `ui::Modal` | Hayır | Hayır |
| Feedback | `ModalHeader` / `ModalFooter` / `ModalRow` | `crates/ui/src/components/modal.rs` | `ui::ModalHeader`, `ui::ModalFooter`, `ui::ModalRow` | Hayır | Hayır |
| Feedback | `Section` / `SectionHeader` | `crates/ui/src/components/modal.rs` | `ui::Section`, `ui::SectionHeader` | Hayır | Hayır |
| Feedback | `AlertModal` | `crates/ui/src/components/notification/alert_modal.rs` | `ui::AlertModal` | Hayır | Evet |
| Feedback | `AnnouncementToast` | `crates/ui/src/components/notification/announcement_toast.rs` | `ui::AnnouncementToast` | Hayır | Evet |
| Feedback | `CountBadge` | `crates/ui/src/components/count_badge.rs` | `ui::CountBadge` | Hayır | Evet |
| Feedback | `Indicator` | `crates/ui/src/components/indicator.rs` | `ui::Indicator` | Hayır | Evet |
| Feedback | `ProgressBar` | `crates/ui/src/components/progress/progress_bar.rs` | `ui::ProgressBar` | Hayır | Evet |
| Feedback | `CircularProgress` | `crates/ui/src/components/progress/circular_progress.rs` | `ui::CircularProgress` | Hayır | Evet |
| Diğer | `Avatar` | `crates/ui/src/components/avatar.rs` | `ui::Avatar` | Hayır | Evet |
| Diğer | `AvatarAudioStatusIndicator` | `crates/ui/src/components/avatar.rs` | `ui::AvatarAudioStatusIndicator`, `ui::AudioStatus` | Hayır | Yardımcı |
| Diğer | `AvatarAvailabilityIndicator` | `crates/ui/src/components/avatar.rs` | `ui::AvatarAvailabilityIndicator`, `ui::CollaboratorAvailability` | Hayır | Yardımcı |
| Diğer | `Facepile` | `crates/ui/src/components/facepile.rs` | `ui::Facepile` | Hayır | Evet |
| Diğer | `Chip` | `crates/ui/src/components/chip.rs` | `ui::Chip` | Hayır | Evet |
| Diğer | `DiffStat` | `crates/ui/src/components/diff_stat.rs` | `ui::DiffStat` | Hayır | Evet |
| Diğer | `Disclosure` | `crates/ui/src/components/disclosure.rs` | `ui::Disclosure` | Hayır | Evet |
| Diğer | `GradientFade` | `crates/ui/src/components/gradient_fade.rs` | `ui::GradientFade` | Hayır | Hayır |
| Diğer | `Vector` / `VectorName` | `crates/ui/src/components/image.rs` | `ui::Vector`, `ui::VectorName` | Hayır | Evet |
| Diğer | `KeyBinding` | `crates/ui/src/components/keybinding.rs` | `ui::KeyBinding` | Hayır | Evet |
| Diğer | `KeybindingHint` | `crates/ui/src/components/keybinding_hint.rs` | `ui::KeybindingHint` | Hayır | Evet |
| Diğer | `Navigable` | `crates/ui/src/components/navigable.rs` | `ui::Navigable` | Hayır | Hayır |
| Diğer | `ProjectEmptyState` | `crates/ui/src/components/project_empty_state.rs` | `ui::ProjectEmptyState` | Hayır | Hayır |
| AI / Collab | `AiSettingItem` | `crates/ui/src/components/ai/ai_setting_item.rs` | `ui::AiSettingItem` | Hayır | Evet |
| AI / Collab | `AiSettingItemStatus` / `AiSettingItemSource` | `crates/ui/src/components/ai/ai_setting_item.rs` | `ui::AiSettingItemStatus`, `ui::AiSettingItemSource` | Hayır | Enum |
| AI / Collab | `AgentSetupButton` | `crates/ui/src/components/ai/agent_setup_button.rs` | `ui::AgentSetupButton` | Hayır | Impl var / `None` |
| AI / Collab | `ThreadItem` | `crates/ui/src/components/ai/thread_item.rs` | `ui::ThreadItem` | Hayır | Evet |
| AI / Collab | `AgentThreadStatus` / `ThreadItemWorktreeInfo` | `crates/ui/src/components/ai/thread_item.rs` | `ui::AgentThreadStatus`, `ui::ThreadItemWorktreeInfo`, `ui::WorktreeKind` | Hayır | Yardımcı |
| AI / Collab | `ConfiguredApiCard` | `crates/ui/src/components/ai/configured_api_card.rs` | `ui::ConfiguredApiCard` | Hayır | Evet |
| AI / Collab | `ParallelAgentsIllustration` | `crates/ui/src/components/ai/parallel_agents_illustration.rs` | `ui::ParallelAgentsIllustration` | Hayır | Hayır |
| AI / Collab | `CollabNotification` | `crates/ui/src/components/collab/collab_notification.rs` | `ui::CollabNotification` | Hayır | Evet |
| AI / Collab | `UpdateButton` | `crates/ui/src/components/collab/update_button.rs` | `ui::UpdateButton` | Hayır | Evet |
| Utils | `WithRemSize` | `crates/ui/src/utils/with_rem_size.rs` | `ui::utils::WithRemSize` | Hayır | Hayır |
| Utils | `SearchInputWidth` | `crates/ui/src/utils/search_input.rs` | `ui::utils::SearchInputWidth` | Hayır | Hayır |
| Utils | `CornerSolver` / `inner_corner_radius` | `crates/ui/src/utils/corner_solver.rs` | `ui::utils::CornerSolver`, `ui::utils::inner_corner_radius` | Hayır | Fonksiyon |
| Utils | `FormatDistance` / `DateTimeType` | `crates/ui/src/utils/format_distance.rs` | `ui::utils::FormatDistance`, `ui::utils::DateTimeType`, `ui::utils::format_distance`, `ui::utils::format_distance_from_now` | Hayır | Yardımcı |
| Utils | `apca_contrast` / `calculate_contrast_ratio` / `ensure_minimum_contrast` | `crates/ui/src/utils/apca_contrast.rs`, `crates/ui/src/utils/color_contrast.rs` | `ui::utils::apca_contrast`, `ui::utils::calculate_contrast_ratio`, `ui::utils::ensure_minimum_contrast` | Hayır | Fonksiyon |
| Utils | `is_light` / `capitalize` / `reveal_in_file_manager_label` | `crates/ui/src/utils.rs` | `ui::utils::is_light`, `ui::utils::capitalize`, `ui::utils::reveal_in_file_manager_label` | Hayır | Fonksiyon |
| Utils | `platform_title_bar_height` / `TRAFFIC_LIGHT_PADDING` | `crates/ui/src/utils/constants.rs` | `ui::utils::platform_title_bar_height`, `ui::utils::TRAFFIC_LIGHT_PADDING` | Hayır | Sabit |
| Utils | `BASE_REM_SIZE_IN_PX` / `EDITOR_SCROLLBAR_WIDTH` | `crates/ui/src/styles/units.rs`, `crates/ui/src/components/scrollbar.rs` | `ui::BASE_REM_SIZE_IN_PX`, `ui::EDITOR_SCROLLBAR_WIDTH` | Hayır | Sabit |

## Constructor envanteri

Aşağıdaki tablo bileşenlerin başlangıç constructor'larını özetler. Ayrıntılı builder listeleri, ilgili bileşenin kendi başlığında yer alır.

| Bileşen / API | Constructor veya giriş noktası |
| :-- | :-- |
| `Label` | `Label::new(label)` |
| `LabelLike` | `LabelLike::new()` |
| `Color` | `Color::Default`, `Color::Muted`, `Color::Accent`, `Color::Custom(hsla)` ve `.color(cx)` |
| `Headline` | `Headline::new(text)` |
| `HighlightedLabel` | `HighlightedLabel::new(label, highlight_indices)` |
| `LoadingLabel` | `LoadingLabel::new(text)` |
| `SpinnerLabel` | `SpinnerLabel::new()` |
| `Button` | `Button::new(id, label)` |
| `IconButton` | `IconButton::new(id, icon)` |
| `ButtonLike` | `ButtonLike::new(id)` |
| `ButtonLink` | `ButtonLink::new(label, link)` |
| `CopyButton` | `CopyButton::new(id, message)` |
| `SplitButton` | `SplitButton::new(left, right)` |
| `ToggleButtonSimple` | `ToggleButtonSimple::new(label, on_click)` ve `.selected(selected)` |
| `ToggleButtonWithIcon` | `ToggleButtonWithIcon::new(label, icon, on_click)` ve `.selected(selected)` |
| `Icon` | `Icon::new(icon)`, `Icon::from_path(path)`, `Icon::from_external_svg(svg)` |
| `DecoratedIcon` | `DecoratedIcon::new(icon, decoration)` |
| `IconDecoration` | `IconDecoration::new(kind, knockout_color, cx)` |
| `IconWithIndicator` | `IconWithIndicator::new(icon, Option<Indicator>)` |
| `AnyIcon` | `AnyIcon::Icon(icon)`, `AnyIcon::AnimatedIcon(animation_element)` ve `From<Icon>`, `From<AnimationElement<Icon>>` dönüşümleri; `.map(|icon| ...)` ile içeriği dönüştürür |
| `Checkbox` | `Checkbox::new(id, checked)` veya `checkbox(id, state)` |
| `Switch` | `Switch::new(id, state)` veya `switch(id, state)` |
| `SwitchField` | `SwitchField::new(id, label: Option<impl Into<SharedString>>, description: Option<SharedString>, state: impl Into<ToggleState>, on_click)` |
| `InputField` | `ui_input::InputField::new(window, cx, placeholder_text)` |
| `DropdownMenu` | `DropdownMenu::new(id, label, menu)` veya `DropdownMenu::new_with_element(id, label, menu)` |
| `ContextMenu` | `ContextMenu::new(window, cx, builder)` veya `ContextMenu::build(window, cx, builder)` |
| `RightClickMenu` | `right_click_menu(id)` |
| `Popover` | `Popover::new()` |
| `PopoverMenu` | `PopoverMenu::new(id)` |
| `Tooltip` | `Tooltip::new(title)` |
| `LinkPreview` | `LinkPreview::new(url, cx) -> AnyView` |
| `DocumentationAside` | `DocumentationAside::new(side: DocumentationSide, render: Rc<dyn Fn(&mut App) -> AnyElement>)`; `DocumentationSide::Left` veya `Right` |
| `List` | `List::new()` |
| `ListItem` | `ListItem::new(id)` |
| `ListHeader` | `ListHeader::new(label)` |
| `ListSubHeader` | `ListSubHeader::new(label)` |
| `ListSeparator` | `ListSeparator` |
| `ListBulletItem` | `ListBulletItem::new(label)` |
| `TreeViewItem` | `TreeViewItem::new(id, label)` |
| `sticky_items` | `sticky_items(entity, compute_fn, render_fn)` |
| `indent_guides` | `indent_guides(indent_size, colors)` |
| `Tab` | `Tab::new(id)` |
| `TabBar` | `TabBar::new(id)` |
| `h_flex` / `v_flex` | `h_flex()`, `v_flex()` |
| `h_group*` / `v_group*` | `h_group_sm()`, `h_group()`, `h_group_lg()`, `h_group_xl()` ve dikey karşılıkları |
| `Divider` | `divider()`, `vertical_divider()` |
| `div` / `Div` | `div()`; ham GPUI container primitive'i |
| `canvas` / `Canvas` | `canvas(prepaint, paint)` |
| `img` / `Img` | `img(source)`, `Img::extensions()` |
| `image_cache` | `image_cache(provider)`, `retain_all(id)`, `RetainAllImageCache::new(cx)` |
| `svg` / `Svg` | `svg()` |
| `anchored` / `Anchored` | `anchored()` |
| `deferred` / `Deferred` | `deferred(child)` |
| `surface` / `Surface` | `surface(source)` |
| `list` / `ListState` | `list(state, render_item)`, `ListState::new(...)` |
| `uniform_list` | `uniform_list(id, item_count, render_item)`, `UniformListScrollHandle::new()` |
| `StyledText` / `InteractiveText` | `StyledText::new(text)`, `InteractiveText::new(styled_text, window, cx)` |
| `Scrollbars` | `Scrollbars::new(show_along)`, `Scrollbars::always_visible(show_along)`, `Scrollbars::for_settings::<S>()` |
| `Table` | `Table::new(cols)` |
| `TableInteractionState` | `TableInteractionState::new(cx)` |
| `ColumnWidthConfig` | `ColumnWidthConfig::auto()`, `ColumnWidthConfig::auto_with_table_width(width)`, `ColumnWidthConfig::explicit(widths)`, `ColumnWidthConfig::redistributable(columns_state)` |
| `ResizableColumnsState` | `ResizableColumnsState::new(cols, initial_widths, resize_behavior)` |
| `RedistributableColumnsState` | `RedistributableColumnsState::new(...)` |
| `TableRow` | `TableRow::from_vec(data, expected_length)`, `TableRow::try_from_vec(data, expected_length)`, `TableRow::from_element(element, length)` |
| `Banner` | `Banner::new()` |
| `Callout` | `Callout::new()` |
| `Modal` | `Modal::new(id, scroll_handle)` |
| `ModalHeader` / `ModalFooter` / `ModalRow` | `ModalHeader::new()`, `ModalFooter::new()`, `ModalRow::new()` |
| `Section` / `SectionHeader` | `Section::new()`, `Section::new_contained()`, `SectionHeader::new(label)` |
| `AlertModal` | `AlertModal::new(id)` |
| `AnnouncementToast` | `AnnouncementToast::new()` |
| `CountBadge` | `CountBadge::new(count)` |
| `Indicator` | `Indicator::dot()`, `Indicator::bar()`, `Indicator::icon(icon)` |
| `ProgressBar` | `ProgressBar::new(id, value, max_value, cx)` |
| `CircularProgress` | `CircularProgress::new(value, max_value, size, cx)` |
| `Avatar` | `Avatar::new(src)` |
| `AvatarAudioStatusIndicator` | `AvatarAudioStatusIndicator::new(audio_status)` |
| `AvatarAvailabilityIndicator` | `AvatarAvailabilityIndicator::new(availability)` |
| `Facepile` | `Facepile::new(faces)` |
| `Chip` | `Chip::new(label)` |
| `DiffStat` | `DiffStat::new(id, added, removed)` |
| `Disclosure` | `Disclosure::new(id, is_open)` |
| `GradientFade` | `GradientFade::new(base_bg, hover_bg, active_bg)` |
| `Vector` | `Vector::new(vector, width, height)` |
| `KeyBinding` | `KeyBinding::new(action, focus_handle, cx)` |
| `Key` / `KeyIcon` | `Key::new(key, color)`, `KeyIcon::new(icon, color)` |
| `KeybindingHint` | `KeybindingHint::new(keybinding, background_color)` |
| `NavigableEntry` / `Navigable` | `NavigableEntry::new(scroll_handle, cx)`, `Navigable::new(child)` |
| `ProjectEmptyState` | `ProjectEmptyState::new(label, focus_handle, open_project_key_binding)` |
| `AiSettingItem` | `AiSettingItem::new(id, label, status, source)` |
| `AgentSetupButton` | `AgentSetupButton::new(id)` |
| `ThreadItem` | `ThreadItem::new(id, title)` |
| `ConfiguredApiCard` | `ConfiguredApiCard::new(label)` |
| `ParallelAgentsIllustration` | `ParallelAgentsIllustration::new()` |
| `CollabNotification` | `CollabNotification::new(avatar_uri, accept_button, dismiss_button)` |
| `UpdateButton` | `UpdateButton::new(icon, message)` |
| `WithRemSize` | `WithRemSize::new(rem_size: impl Into<Pixels>)` |
| `CornerSolver` | `CornerSolver::new(root_radius, root_border, root_padding)` ve `.add_child(border, padding).corner_radius(level)` |
| `FormatDistance` | `FormatDistance::new(date, base_date)`, `FormatDistance::from_now(date)` |

## Public Yardımcı Metotlar

Kaynakta public olarak görünen ama genellikle builder bölümünde değil de state veya helper bölümünde kullanılan metotlar aşağıdaki tabloda özetlenir.

| Kaynak | Metotlar | Rol |
| :-- | :-- | :-- |
| `styles/typography.rs` | `TextSize::pixels(cx)` | `TextSize` token'ını bir `Pixels` değerine çevirir |
| `styles/elevation.rs` | `ElevationIndex::on_elevation_bg(cx)`, `.darker_bg(cx)` | Elevation arka plan varyantlarını tema üzerinden çözer |
| `icon.rs` | `IconSize::square_components(window, cx)`, `IconSize::square(window, cx)`, `IconWithIndicator::indicator_color(color)` | İkon kare ölçüsü ve indicator rengi |
| `toggle.rs` | `Checkbox::container_size()` | Checkbox kutusu için sabit bir `Pixels` ölçüsü; satır hizalama için kullanılır |
| `divider.rs` | `Divider::render_solid(base, cx)`, `Divider::render_dashed(base)` | `Divider` render stratejisi; normal kullanımda `divider()` veya `vertical_divider()` yeterlidir |
| `modal.rs` | `Modal::show_back(bool)`, `ModalHeader::show_back_button(bool)`, `Section::contained(bool)` | Modal seviyesinde geri butonu açma, header'a back button ikonu ekleme ve `Section` border'lı yüzey toggle'ı |
| `tab_bar.rs` | `TabBar::start_children_mut()`, `TabBar::end_children_mut()` | Builder zinciri dışında TabBar başlangıç ve bitiş child listelerini mutably düzenler (`SmallVec<[AnyElement; 2]>`) |
| `scrollbar.rs` | `ScrollbarAutoHide::should_hide()` | Scrollbar otomatik gizleme kararını okuyan global token; `Global` olarak set ve get edilir |
| `gpui/src/elements/list.rs` | `ListState::scrollbar_drag_started()`, `ListState::scrollbar_drag_ended()`, `ListState::is_scrollbar_dragging()`, `ListState::set_offset_from_scrollbar(point)`, `ListState::scroll_px_offset_for_scrollbar()` | Variable-height liste scrollbar drag state'i ve scrollbar offset eşlemesi; scroll yönü negatif `y` offset'iyle temsil edilir |
| `data_table.rs` | `ColumnWidthConfig::table_width(window, cx)`, `ColumnWidthConfig::list_horizontal_sizing(window, cx)`, `ResizableColumnsState::reset_column_to_initial_width(col_idx)`, `Table::pin_cols(n)`, `Table::map_row(callback)`, `Table::empty_table_callback(callback)` | Table genişliği, horizontal sizing, sabit ilk kolonlar ve kolon reset helper'ları; `Table` üzerinde satır ve empty callback'leri |
| `redistributable_columns.rs` | `TableResizeBehavior::is_resizable()`, `HeaderResizeInfo::reset_column(col_idx, window, cx)`, `RedistributableColumnsState::reset_column_to_initial_width(column_index, window)` | Resize davranışı sorgusu, header bilgi paketi üzerinden reset ve kolon initial width'e dönüş |
| `context_menu.rs` | `ContextMenu::selected_index()`, `.confirm(...)`, `.secondary_confirm(...)`, `.cancel(...)`, `.end_slot(...)`, `.clear_selected()`, `.select_first(...)`, `.select_last(...)`, `.select_next(...)`, `.select_previous(...)`, `.select_submenu_child(...)`, `.select_submenu_parent(...)`, `.on_action_dispatch(...)`, `.on_blur_subscription(subscription)` | Menü action, seçim, submenu traversal, end slot ve blur subscription state yönetimi |
