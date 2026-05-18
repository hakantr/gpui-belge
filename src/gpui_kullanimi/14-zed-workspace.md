# Zed Workspace

---

## Zed Workspace Dock ve Panel Modeli

Bu bölüm GPUI çekirdeğine değil, Zed'in `workspace` crate'i üstünde duran dock ve panel katmanına aittir. İlgili dosyalar: `crates/workspace/src/workspace.rs`, `crates/workspace/src/dock.rs`, `crates/workspace/src/pane.rs`.

**Panel yardımcı yüzeyi.** Panel UI'ı yazarken aşağıdaki sınırlar bilinmelidir:

- `panel::PanelHeader` varsayılan `header_height` veya `panel_header_container` sağlayan bir yardımcı trait değildir; `workspace::Panel` üstünde işaretleyici bir trait'tir. Başlık yüksekliği gerekiyorsa doğrudan `Tab::container_height(cx)`, kapsayıcı gerekiyorsa `h_flex()`/`v_flex()` ve `ui::Button`/`ui::IconButton` bileşenleri kurulur.
- `panel_button`, `panel_filled_button`, `panel_icon_button` ve `panel_filled_icon_button` serbest fonksiyon yardımcıları yoktur. Panel UI'ında buton layer/size/style kararları doğrudan bileşen üzerinde açıkça belirtilir.
- Git paneli `GitPanelTab::{Changes, History}` durumuyla iki tab çizer. Changes sekmesi staged/unstaged liste ve commit footer akışını taşır; History sekmesi commit geçmişini `UniformListScrollHandle` ile sanallaştırır, ok tuşlarıyla `focused_history_entry` seçer ve confirm ile `CommitView::open` çağırır. Panel action dinleyicilerine `ActivateChangesTab` ve `ActivateHistoryTab` eklenmelidir.
- Branch diff görünümü toolbar'daki `Base: ...` popover'ı ile diff baz branch'ini değiştirir. Picker `branch_picker::select_popover(...)` üzerinden checkout yapmadan branch seçer, geri çağrı `DiffBase::Merge { base_ref }` ayarlar ve `BranchDiff::set_diff_base` `BranchDiffEvent::DiffBaseChanged` yayar. Ağaç tabanlı merge-base diff hesabı sürerken `is_tree_base_loading()` true döner; boş görünümler bunu yükleme göstergesiyle ayırmalı, eski statik baz varsayımına dönmemelidir.

**Workspace yapısı.** Workspace üç ana dock'u ve merkezdeki pane grubunu bir arada tutar:

- `Workspace` merkezde pane grubunu, solda `left_dock`'u, sağda `right_dock`'u ve altta `bottom_dock`'u taşır.
- Dock entity'si `DockPosition::{Left, Bottom, Right}` ile konumlanır.
- `Workspace::left_dock()`, `right_dock()`, `bottom_dock()`, `all_docks()`, `dock_at_position(position)` ile erişim sağlanır.
- Aksiyonlar: `ToggleLeftDock`, `ToggleRightDock`, `ToggleBottomDock`, `ToggleAllDocks`, `CloseActiveDock`, `CloseAllDocks`, `Increase/DecreaseActiveDockSize`, `ResetActiveDockSize` gibi.

**Panel yazma.** Yeni bir panel `Panel` trait'i uygulanarak tanımlanır:

- `persistent_name()` ve `panel_key()` persist, keymap ve telemetry kimliğidir.
- `position`, `position_is_valid`, `set_position` panelin hangi dock'ta oturduğunu yönetir.
- `default_size`, `min_size`, `initial_size_state`, `size_state_changed`, `supports_flexible_size`, `set_flexible_size` boyut ve kalıcılık davranışını belirler.
- `icon`, `icon_tooltip`, `icon_label`, `toggle_action`, `activation_priority` status bar butonunu ve sıralamayı tanımlar.
- `starts_open`, `set_active`, `is_zoomed`, `set_zoomed`, `pane`, `remote_id` dock durumu ve uzak workspace entegrasyonudur.
- Panel `Focusable + EventEmitter<PanelEvent> + Render` olmalıdır.

**Dock davranışı.** Dock entity'sinin panel ekleme ve görünürlük yönetimi şu şekildedir:

- `Dock::add_panel` paneli `activation_priority` sırasına göre ekler. Aynı priority'i kullanan iki panel hata ayıklama build'inde panic'e yol açar; her panel benzersiz bir priority seçmelidir.
- `Dock::set_open`, `activate_panel`, `active_panel`, `visible_panel`, `panel::<T>()`, `remove_panel`, `resize_active_panel`, `resize_all_panels` temel yönetim API'leridir.
- Panel `PanelEvent::Activate` yaydığında dock açılır, panel aktiflenir ve odak panele taşınır.
- `PanelEvent::Close` aktif görünür paneli kapatır.
- `PanelEvent::ZoomIn/ZoomOut` workspace zoom katmanı durumunu günceller.
- Boyut durumu `PanelSizeState { size, flex }` olarak kalıcılaştırılır.

**Workspace `toggle_dock` akışı.** Dock'u açıp kapatan tipik akış birkaç adımdan oluşur:

1. Dock görünürse açık pozisyonlar kaydedilir.
2. Dock açık durumu terslenir.
3. Aktif panel yoksa ilk etkin panel aktif edilir.
4. Açılıyorsa odak, panelin focus handle'ına taşınır; kapanıyorsa odaktaki panelden geliniyorsa orta pane'e geri verilir.
5. Workspace serileştirilir.

**Yeni panel eklerken kontrol.** Aşağıdaki noktalar yeni bir panel hazırlanırken gözden geçirilmelidir:

- `panel_key` kalıcılaştırma ve keymap kimliğidir; yeni panelde baştan stabil bir değer seçilmelidir.
- `position_is_valid` alt ve yan sınırlamalarını net tanımlamalıdır.
- `toggle_action()` action'ı önceden kaydedilmiş olmalıdır.
- `activation_priority()` benzersiz olmalıdır.
- `set_active` içinde UI durumu değiştiriliyorsa `cx.notify()` çağrısı unutulmamalıdır.
- Dock değiştiren ayar gözlemcilerinde panel taşınırken boyut durumu ekseni değişiyorsa sıfırlanabilir; bu mevcut `Dock::add_panel`/ayar gözlemci akışında zaten yapılır.

## Workspace Item, Pane, Modal, Toast ve Notification Sistemi

GPUI bir UI framework'üdür. Zed'in workspace katmanı bunun üstünde tab/pane, modal, toast ve bildirim akışlarını standartlaştırır. Yeni bir editör benzeri panel veya komut yazılırken bu sözleşmeler bilinmelidir.

#### Item ve ItemHandle

`crates/workspace/src/item.rs:167+`. Pane içindeki her tab içeriği `Item` trait'ini uygular:

```rust
pub trait Item: Focusable + EventEmitter<Self::Event> + Render + Sized {
    type Event;

    fn tab_content(&self, params: TabContentParams, window: &Window, cx: &App)
        -> AnyElement;
    fn tab_tooltip_text(&self, _: &App) -> Option<SharedString> { None }
    fn deactivated(&mut self, window: &mut Window, cx: &mut Context<Self>) {}
    fn workspace_deactivated(&mut self, window: &mut Window, cx: &mut Context<Self>) {}
    fn telemetry_event_text(&self, _: &App) -> Option<&'static str> { None }
    fn navigate(&mut self, _data: Box<dyn Any>, _window: &mut Window, _cx: &mut Context<Self>) -> bool { false }
    // ... save/save_as, project_path, can_split, breadcrumbs, dragged_selection, ...
}
```

`ItemHandle` boxed veya dyn karşılığıdır; pane API'leri çoğunlukla `Box<dyn ItemHandle>` ile çalışır. `FollowableItem` ise collab takibi (workspace follow) için ek bir sözleşmedir.

**Tipik akış.** Yeni bir tab türü oluşturmak şu adımlardan geçer:

- `impl Item for MyView` yazılır.
- `Workspace::open_path`, `open_paths` veya `open_abs_path` zaten `ProjectItem` üreterek doğru `Item` view'ini açar; özel bir akışta `Pane::add_item(Box::new(view), ...)` kullanılır.
- `Pane::activate_item`, `close_active_item`, `navigate_backward`, `navigate_forward`, `split` (split direction ile yeni pane) Pane API'leridir.
- `Workspace::split_pane(pane, direction, cx)` mevcut pane'i böler.
- `Workspace::register_action::<A>(|workspace, &A, window, cx| ...)` workspace seviyesinde global action'ları ekler (komut paleti veya keymap üzerinden tetiklenir).

#### ModalView ve Modal Layer

`crates/workspace/src/modal_layer.rs:13+`:

```rust
pub trait ModalView: ManagedView {
    fn on_before_dismiss(&mut self, window, cx) -> DismissDecision { ... }
    fn fade_out_background(&self) -> bool { false }
    fn render_bare(&self) -> bool { false }
}
```

`ManagedView = Focusable + EventEmitter<DismissEvent> + Render`. Modal yazarken bu bileşik trait'in sağlanması gerekir.

**Açma ve kapama.** Modal toggle akışı şu yardımcılara dayanır:

```rust
workspace.toggle_modal(window, cx, |window, cx| {
    MyModal::new(window, cx)
});

workspace.hide_modal(window, cx);
```

`toggle_modal` aynı tipte bir modal zaten açıksa onu kapatır; aksi halde yenisini açar. `on_before_dismiss` `DismissDecision::Dismiss(false)` veya `Pending` döndürürse yeni modal görünmez.

#### StatusBar ve StatusItemView

`crates/workspace/src/status_bar.rs`:

```rust
pub trait StatusItemView: Render {
    fn set_active_pane_item(
        &mut self,
        active_pane_item: Option<&dyn ItemHandle>,
        window: &mut Window,
        cx: &mut Context<Self>,
    );

    fn hide_setting(&self, cx: &App) -> Option<HideStatusItem>;
}
```

Workspace status bar'a item eklemek için:

```rust
workspace.status_bar().update(cx, |status_bar, cx| {
    status_bar.add_left_item(my_view, window, cx);
    status_bar.add_right_item(other_view, window, cx);
});
```

Status item'lar aktif pane item değiştikçe `set_active_pane_item` ile bilgilendirilir; bu sayede git branch indicator veya imleç konumu gibi item'lar odaktaki buffer'a göre güncellenir. `hide_setting()` `Some(HideStatusItem)` döndürürse status bar sağ tık menüsüne "Hide Button" kaydı eklenir ve kullanıcı ayar dosyası `update_settings_file` üzerinden güncellenir. Item zaten başka bir ayarla koşullu görünüyorsa `None` döndürülebilir.

#### Notification ve Toast Sistemi

`crates/workspace/src/notifications.rs`:

```rust
pub trait Notification:
    EventEmitter<DismissEvent> + EventEmitter<SuppressEvent> + Focusable + Render
{}

pub enum NotificationId {
    Unique(TypeId),               // tip başına tek
    Composite(TypeId, ElementId), // tip + sub-id
    Named(SharedString),          // serbest isim
}

// Yapıcı yardımcıları:
// NotificationId::unique::<MyNotification>()
// NotificationId::composite::<MyNotification>(element_id)
// NotificationId::named("save".into())
```

Mesaj göstermek için kullanılan başlıca akışlar şunlardır:

```rust
workspace.show_notification(
    NotificationId::unique::<MyNotification>(),
    cx,
    |cx| cx.new(|cx| MyNotification::new(cx)),
);

workspace.show_toast(
    Toast::new(NotificationId::named("save".into()), "Saved")
        .autohide(),
    cx,
);

workspace.show_error(&error, cx);
```

`Toast` hafif ve geçicidir (autohide); `Notification` ise kalıcı bir view'dir ve kullanıcı dismiss edene kadar görünür kalır. `SuppressEvent` aynı kaynaktan gelen tekrarlı bildirimleri bastırmak için kullanılır.

`Workspace::toggle_status_toast<V: ToastView>` modal layer mantığında `ToastView` üzerinden toast'ı toggle eder; tipik UI elemanları (örneğin async iş ilerleme göstergeleri) bu yolla bağlanır.

```rust
pub trait ToastView: ManagedView {
    fn action(&self) -> Option<ToastAction>;
    fn auto_dismiss(&self) -> bool { true }
}
```

`ToastAction::new(label, on_click)` toast içindeki action butonunu tanımlar. `ToastView` tabanlı toast'larda auto dismiss varsayılan olarak `true`'dur. `Workspace::show_toast` ile gösterilen hafif `Toast` struct'ında ise `.autohide()` çağrılmadıkça otomatik kapanma yoktur.

#### `Workspace::open_*` Akışı

```rust
let task = workspace.open_paths(
    vec![PathBuf::from("src/main.rs")],
    OpenOptions {
        visible: Some(OpenVisible::All),
        ..Default::default()
    },
    None,
    window,
    cx,
);
```

**Önemli giriş noktaları.** Workspace içinde içerik açmak için farklı ihtiyaçlara karşılık veren birkaç yardımcı vardır:

- `workspace::open_paths(paths, app_state, open_options, cx)` — bağımsız yardımcıdır; gerekirse pencere açar veya mevcut workspace'i yeniden kullanır.
- `Workspace::open_paths(abs_paths, OpenOptions, pane, window, cx)` — mevcut workspace içinde birden çok mutlak path açar.
- `Workspace::open_path(project_path, pane, focus, window, cx)` — belirli bir `ProjectPath`'i mevcut workspace içinde açar; `Task<Result<Box<dyn ItemHandle>>>` döner.
- `Workspace::open_abs_path(path, options, window, cx)` — `PathBuf` alır, dosyayı worktree'ye ekler ve item açar.
- `Workspace::open_path_preview(path, pane, focus_item, allow_preview, activate, window, cx)` — dosya bulucu gibi önizleme akışları için.
- `Workspace::split_abs_path(...)`, `split_path(...)`, `split_item(...)` — yeni pane oluşturarak path veya item'i split içinde açar.

#### Tuzaklar

Item ve workspace tarafında dikkat edilmesi gereken yaygın hatalar:

- `Item` uygulamasında `Self::Event` türünün doğru tanımlanması ve `EventEmitter<Self::Event>` uygulanması şarttır; aksi halde `Item` trait bound'u tutmaz.
- `Pane::add_item` `Box::new(view)` ile yapılır; pane item sahipliğini alır.
- `Workspace::register_action` geri çağrı imzası `Fn(&mut Self, &A, &mut Window, &mut Context<Self>)` biçimindedir; diğer GPUI `on_action` dinleyicilerinden farklı bir pozisyonel düzene sahiptir (`&A` ortada).
- `NotificationId::Unique(TypeId::of::<T>())` ile aynı tipte iki notification açıldığında ikincisi birincinin yerine geçer; farklı bir sub-id isteniyorsa `Composite(TypeId, ElementId)` kullanılır.
- `Toast` autohide süresi varsayılan değildir; uzun mesajlarda elle `dismiss_toast` çağrısı gerekebilir.
- `ModalView::on_before_dismiss` `Pending` döndürürse modal kapanma akışı beklemeye girer; testte `run_until_parked()` ile resolve sürecinin ilerletilmesi gerekir.

## Workspace Serialization, OpenOptions, ProjectItem ve SearchableItem

Workspace item açma yalnız `Pane::add_item` çağrısından ibaret değildir. Zed session restore, project item çözme, search bar ve collab follow gibi katmanlar da item trait'leri üzerinden bağlanır.

#### SerializableItem ve Restore

`SerializableItem`, workspace kapanırken veya item olayı geldiğinde item durumunu workspace DB'ye yazmak ve daha sonra geri yüklemek için kullanılır:

```rust
pub trait SerializableItem: Item {
    fn serialized_item_kind() -> &'static str;

    fn cleanup(
        workspace_id: WorkspaceId,
        alive_items: Vec<ItemId>,
        window: &mut Window,
        cx: &mut App,
    ) -> Task<Result<()>>;

    fn deserialize(
        project: Entity<Project>,
        workspace: WeakEntity<Workspace>,
        workspace_id: WorkspaceId,
        item_id: ItemId,
        window: &mut Window,
        cx: &mut App,
    ) -> Task<Result<Entity<Self>>>;

    fn serialize(
        &mut self,
        workspace: &mut Workspace,
        item_id: ItemId,
        closing: bool,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<Task<Result<()>>>;

    fn should_serialize(&self, event: &Self::Event) -> bool;
}
```

Kayıt için tek satırlık bir çağrı yeterlidir:

```rust
workspace::register_serializable_item::<MyItem>(cx);
```

- `serialized_item_kind()` session DB'deki discriminant'tır; restore akışı item tipini bu değer üzerinden bulur.
- `serialize(..., closing, ...)` `None` döndürürse o olay için yazma yapılmaz.
- `should_serialize(event)` item olayından sonra serileştirmenin gerekip gerekmediğini belirler.
- `cleanup(workspace_id, alive_items, ...)` DB'de canlı olmayan item kayıtlarını temizlemek için çağrılır.
- `SerializableItemHandle` `Entity<T: SerializableItem>` için blanket olarak uygulanır; pane ve workspace tip silme bu handle üzerinden çalışır.

#### OpenOptions ve open_paths

Üst seviye `workspace::open_paths` ve `Workspace::open_paths` aynı seçenek modelini kullanır:

- `visible: Option<OpenVisible>` — `All`, `None`, `OnlyFiles`, `OnlyDirectories`. `All` hem dosya hem dizinleri project panelde görünür yapar; `None` hiçbirini görünür yapmaz; `OnlyFiles` dizinleri, `OnlyDirectories` ise dosyaları dışarıda bırakır.
- `focus: Option<bool>` — açılan item odak alsın mı?
- `workspace_matching: WorkspaceMatching` — `None`, `MatchExact`, `MatchSubdirectory`.
- `add_dirs_to_sidebar` — eşleşmeyen dizin yeni pencere açmak yerine mevcut yerel `MultiWorkspace` penceresine workspace olarak eklensin mi? Varsayılan `true`. Ekleme yalnız hedef pencerede `multi_workspace_enabled(cx)` `true` olduğunda yapılır; bu durumda `requesting_window` aktif veya ilk yerel multi-workspace penceresine ayarlanır ve sidebar açılır.
- `wait` — CLI `--wait` benzeri akışlarda pencerenin kapanmasını bekleme davranışı.
- `requesting_window` — hedef `WindowHandle<MultiWorkspace>` varsa onun kullanılması.
- `open_mode: OpenMode` — `NewWindow`, `Add`, `Activate`.
- `env` — açılan workspace için ortam üzerine yazması.
- `open_in_dev_container` — dev container açma isteği.

`OpenMode::Activate` `NewWindow` gibi hedef pencereyi öne getirir. Varsayılan davranışta `-n`, `-a` veya `-r` verilmeden klasör açmak mevcut pencerenin Threads sidebar'ına yeni bir project olarak ekler. Yeni pencere isteniyorsa CLI'da `zed -n path`, Open Recent'te değiştiricili enter veya tıklama ya da `cli_default_open_behavior = "new_window"` ayarı kullanılır.

`OpenResult { window, workspace, opened_items }` üst seviye açma sonucudur. İç workspace açma fonksiyonları çoğunlukla `Task<Result<Box<dyn ItemHandle>>>` veya çoklu path için `Task<Vec<Option<Result<Box<dyn ItemHandle>>>>>` döndürür.

#### ProjectItem

`ProjectItem` Zed project entry'sinden workspace item view'i üretir:

```rust
pub trait ProjectItem: Item {
    type Item: project::ProjectItem;

    fn project_item_kind() -> Option<ProjectItemKind> { None }

    fn for_project_item(
        project: Entity<Project>,
        pane: Option<&Pane>,
        item: Entity<Self::Item>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Self;

    fn for_broken_project_item(...) -> Option<InvalidItemView> { None }
}
```

Normal dosya açma `project::ProjectItem::try_open` üzerinden item'i çözer; hata durumunda `for_broken_project_item` bozuk veya eksik kaynağı temsil eden bir view üretmek için kullanılabilir.

#### SearchableItem

Workspace search bar'ın bir item içinde çalışması için `SearchableItem` gereklidir:

- `type Match` — arama sonucunu temsil eden, klonlanabilir match tipi.
- `supported_options() -> SearchOptions` — case, word, regex, replacement, selection, select_all, find_in_results desteklerini bildirir.
- `find_matches(query, window, cx) -> Task<Vec<Match>>` veya token'lı `find_matches_with_token`.
- `update_matches`, `clear_matches`, `activate_match`, `select_matches`.
- `replace` ve `replace_all` replace destekleyen item'lar içindir.
- `SearchEvent::{MatchesInvalidated, ActiveMatchChanged}` search UI'ını yeniden sorgulamaya zorlar.
- `SearchableItemHandle` tipi silinmiş search item'ıdır; `Item::as_searchable` bunu döndürerek pane toolbar search bar'ına bağlanır.
- `query_suggestion(seed_query_override: Option<SeedQuerySetting>, window, cx)` search bar'ın başlangıç sorgusunu üretir. `None` normal `seed_search_query_from_cursor` ayarını kullanır; `Some(Always)` gibi üzerine yazmalar Cmd-E veya Vim search gibi "bu çağrıda imleç altındaki kelimeyi kesinlikle kullan" davranışını açıkça ifade eder.

#### FollowableItem

Collab ve follow akışı için `FollowableItem` kullanılır:

- `remote_id()`, `to_state_proto`, `from_state_proto` uzak view durumunu taşır.
- `to_follow_event(event)` item olayını follow olayına çevirir.
- `add_event_to_update_proto` ve `apply_update_proto` artımlı uzak güncelleme akışıdır.
- `set_leader_id` takip edilen kullanıcı bilgisini item durumuna işler.
- `dedup(existing, ...) -> Option<Dedup>` uzak item açılırken mevcut item ile birleştirme veya değiştirme kararıdır.

**Tuzaklar.** Serileştirme, seçenekler ve search tarafında karşılaşılabilen hatalar:

- Serializable item kaydedilmediğinde `deserialize` hiç çağrılmaz; session restore sessiz biçimde geçersiz item'e düşebilir.
- `serialized_item_kind` global bir namespace gibidir; başka item ile çakıştırılmamalıdır.
- Search match tipi byte offset, buffer snapshot ve token ile uyumlu tutulmalıdır; bayat match'in yeni buffer üzerinde kullanılması yanlış aralığa gider.
- `OpenOptions::visible = None` varsayılan olarak workspace'e görünür worktree ekleme anlamı taşımaz; path açma davranışı dizin/dosya ayrımı için açıkça seçilmelidir.

## PaneGroup, NavHistory, Toolbar ve Sidebar Entegrasyonu

Pane ve workspace yalnızca tab listesinden ibaret değildir; split ağacı, gezinme geçmişi, toolbar item'ları ve multi-workspace sidebar birlikte çalışır.

#### PaneGroup ve SplitDirection

`PaneGroup` merkez veya dock içindeki pane ağacını taşır. Kök `Member::Pane` veya `Member::Axis(PaneAxis)` olabilir.

- `PaneGroup::new(pane)` tek bir pane ile başlar.
- `split(old_pane, new_pane, SplitDirection, cx)` ağaca yeni pane ekler; `old_pane` bulunamazsa ilk pane yedek olarak kullanılır.
- `remove`, `resize`, `reset_pane_sizes`, `swap`, `move_to_border` split ağacını değiştirir.
- `pane_at_pixel_position(point)`, `bounding_box_for_pane(pane)`, `find_pane_in_direction` sürükle-bırak ve klavyeyle pane gezinme için kullanılır.
- `SplitDirection::{Up, Down, Left, Right}`; `vertical(cx)` ve `horizontal(cx)` kullanıcı ayarına göre varsayılan split yönünü üretir.
- `SplitDirection::axis()`, `opposite()`, `edge(bounds)`, `along_edge(bounds, length)` resize ve bırakma göstergesi hesaplarında kullanılır.

#### Pane Preview, Pin ve NavHistory

Pane item listesinde preview ve sabitlenmiş ayrımı vardır; her ikisi de benzer ama farklı durumlarda yönetilir:

- `preview_item_id`, `preview_item`, `is_active_preview_item`, `unpreview_item_if_preview`, `replace_preview_item_id` preview tab akışıdır.
- `pinned_count`, `set_pinned_count` sabitlenmiş tab sınırını yönetir.
- `activate_item`, `activate_previous_item`, `activate_next_item`, `activate_last_item`, `swap_item_left/right` tab seçim ve sırasının yönetimidir.
- `close_active_item`, `close_item_by_id`, `close_other_items`, `close_clean_items`, `close_all_items` save intent ve sabitlenmiş davranışı hesaba katar.
- Workspace bir pane'i kaldırırken aktif pane kaldırılıyorsa `Workspace::force_remove_pane` önce `active_pane`'i kalan bir pane'e günceller: `focus_on` verilmişse o pane aktif olur, verilmemişse son kalan pane yedek olarak seçilir. Aktif modal varsa odak yedek pane'e taşınmayabilir; ancak `active_pane` yine kaldırılmış pane olarak bırakılmaz.

**Gezinme.** Geçmiş yönetimi item ve pane düzeyinde çalışır:

- `Pane::nav_history_for_item(item)` item'e bağlı bir `ItemNavHistory` üretir.
- `ItemNavHistory::push(data, row, cx)` item geçmişine giriş ekler; item `include_in_nav_history()` `false` döndürdüğünde eklenmez.
- `NavHistory::pop(GoingBack/GoingForward, cx)`, `clear`, `disable`, `enable`, `set_mode`, `for_each_entry` geçmiş yönetimini yapar.
- `push_tag` ve `pop_tag` tanım veya referans gibi tag gezinme yığınını yönetir.

#### ToolbarItemView

Pane toolbar'a katkı veren view'lar `ToolbarItemView` uygular:

```rust
pub trait ToolbarItemView: Render + EventEmitter<ToolbarItemEvent> {
    fn set_active_pane_item(
        &mut self,
        active_pane_item: Option<&dyn ItemHandle>,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> ToolbarItemLocation;

    fn pane_focus_update(&mut self, pane_focused: bool, window: &mut Window, cx: &mut Context<Self>) {}
    fn contribute_context(&self, context: &mut KeyContext, cx: &App) {}
}
```

- `ToolbarItemLocation::{Hidden, PrimaryLeft, PrimaryRight, Secondary}` çizim yerini belirler.
- Item kendi yerini değiştirmek isterse `ToolbarItemEvent::ChangeLocation(...)` yayar.
- `Toolbar::add_item(entity, window, cx)` item'i kaydeder.
- `Toolbar::set_active_item` aktif pane item değiştiğinde tüm toolbar item'larını günceller.
- `contribute_context` görünür toolbar item'larının key context'e katkı vermesini sağlar.

#### Sidebar ve MultiWorkspace

AI ve multi-workspace sidebar ayrı bir trait üzerinden bağlanır:

```rust
pub trait Sidebar: Focusable + Render + EventEmitter<SidebarEvent> + Sized {
    fn width(&self, cx: &App) -> Pixels;
    fn set_width(&mut self, width: Option<Pixels>, cx: &mut Context<Self>);
    fn has_notifications(&self, cx: &App) -> bool;
    fn side(&self, cx: &App) -> SidebarSide;
    fn serialized_state(&self, cx: &App) -> Option<String> { None }
    fn restore_serialized_state(&mut self, state: &str, window: &mut Window, cx: &mut Context<Self>) {}
}
```

Sidebar yaşam döngüsü `MultiWorkspace` üzerinde yönetilir:

- `MultiWorkspace::register_sidebar(entity, cx)` sidebar'ı handle olarak saklar, gözlemler ve `SidebarEvent::SerializeNeeded` geldiğinde serileştirir.
- `toggle_sidebar`, `open_sidebar`, `close_sidebar`, `focus_sidebar` görünürlük ve odak akışıdır.
- `set_sidebar_overlay(Some(AnyView), cx)` sidebar üzerine kaplama yerleştirir.
- `multi_workspace_enabled(cx)`, `!DisableAiSettings::disable_ai` ve `AgentSettings::enabled` koşullarının birlikte true olmasına bağlıdır. Bu false olduğunda sidebar açma veya focus işlemleri erken döner ve açık sidebar çizilmez.
- `sidebar_render_state(cx)` çizim tarafında open ve side bilgisini taşır; `open` değeri hem sidebar'ın açık olmasına hem de `multi_workspace_enabled(cx)` sonucuna bağlıdır.
- `sidebar_has_notifications(cx)` başlık çubuğu veya durum göstergesi için kullanılır.
- Workspace aktivasyonunda tutma kararı sidebar'ın açık olmasına bağlı değildir. Multi-workspace etkinse aktif workspace ve son transient aktif workspace tutulur; multi-workspace devre dışıysa transient workspace ayrılır. Ayar değişiminde etkin durumdan devre dışına geçiş `collapse_to_single_workspace` ile tüm grupları atar.
- Threads sidebar thread'lerle birlikte terminal girişlerini de MRU switcher'a dahil eder; terminal aktivasyonu `AgentPanel::activate_terminal` üzerinden yapılır ve `ArchiveSelectedThread` aktif terminalde kapatma davranışına bağlanır.
- Sidebar ve panel boş durum akışları kök yolu olmayan workspace'te yeni thread veya terminal oluşturmaz; kullanıcı önce bir proje açmaya yönlendirilir.
- Taslak thread girişleri ayrı icon ve kapatma davranışıyla gösterilir. Taslak başlığı mesaj editör içeriğinden üretildiği için sidebar görünür taslak editör'leri gözlemleyip yazıldıkça girişleri yeniler.

**Tuzaklar.** Bu entegrasyonlarda dikkat edilmesi gerekenler:

- Toolbar item `set_active_pane_item` içinde konum döndürür; ancak daha sonra yer değiştireceği zaman olay yaymalıdır, sadece durum değiştirip `cx.notify()` yeterli değildir.
- Sidebar durumu serileştirilecekse `SidebarEvent::SerializeNeeded` yaymak unutulmamalıdır.
- Workspace tutma veya ayırma davranışı sidebar açık/kapalı durumuna bağlanmamalıdır; güncel karar kaynağı `multi_workspace_enabled(cx)` sonucudur.
- Nav history preview item'ı ayrı işaretler; preview tab gerçek tab'a sabitlendiğinde history girişleri buna göre güncellenmelidir.
- Split yönü sabit kodlanmış olarak verilmek yerine kullanıcı ayarlı varsayılan isteniyorsa `SplitDirection::vertical(cx)` veya `horizontal(cx)` kullanılır.

## Workspace Notification Yardımcıları ve Async Hata Gösterimi

Bildirim sistemi sadece `show_notification` çağrısından ibaret değildir. Workspace dışı app seviyesinde bildirim ve async hata yayılımı için ek yardımcı trait'ler bulunur.

**App-level notification.** Aktif workspace olup olmadığına bakmadan bildirim göstermek için:

- `show_app_notification(id, cx, build)` — aktif workspace varsa orada, yoksa tüm workspace'lerde notification gösterir.
- `dismiss_app_notification(id, cx)` — aynı id'li app notification'ları kapatır.
- `NotificationFrame` — başlık, içerik, suppress/close butonu ve suffix kompoze etmek için kullanılan standart çerçeve.
- `simple_message_notification::MessageNotification` — birincil/ikincil mesaj, ikon, tıklama işleyicisi, close/suppress ve "more info" URL gibi hazır alanlar sağlar.

**Hata yayılımı.** Async ve workspace bağlamlarındaki hatalar için yardımcılar şu şekildedir:

- `NotifyResultExt::notify_err(workspace, cx)` — `Result` hata olduğunda workspace notification gösterir ve `None` döndürür.
- `notify_workspace_async_err(weak_workspace, async_cx)` — async task içinde weak workspace'e hata notification gönderir.
- `notify_app_err(cx)` — aktif workspace yoksa app seviyesinde notification gösterir.
- `NotifyTaskExt::detach_and_notify_err(workspace, window, cx)` — `Task<Result<...>>` sonucunu pencere üzerinde spawn eder ve hatayı workspace notification'a çevirir.
- `DetachAndPromptErr::prompt_err` ve `detach_and_prompt_err` — `anyhow::Result` task'ını prompt tabanlı bir kullanıcı hatasına çevirir.

**Kullanım seçimi.** Hangi yardımcının seçileceği bağlama göre belirlenir:

- Kullanıcı aksiyonunun sonucu doğrudan workspace içinde görünmeliyse `notify_err` veya `detach_and_notify_err`.
- Kritik onay ya da seçim gerekiyorsa `detach_and_prompt_err`.
- Workspace yokken de görünmesi gereken startup veya global hata için `notify_app_err` veya `show_app_notification`.

**Tuzaklar.** Bildirim yardımcılarında atlanan noktalar:

- `detach_and_log_err` yalnızca loglar; kullanıcıya görünür bir hata isteniyorsa workspace notification veya prompt yardımcılarından biri tercih edilir.
- `show_app_notification` aynı id ile birden fazla workspace'te gösterim yapabilir; id `NotificationId::named` veya `composite` ile bilinçli seçilmelidir.
- `MessageNotification` tıklama işleyicileri `Window` ve `App` alır; workspace durumu gerekiyorsa weak workspace veya entity yakalanır ve düşmüş olma ihtimali ele alınır.

## AppState, WorkspaceStore, WorkspaceDb ve OpenListener Akışı

Zed uygulamasında workspace açmak yalnızca `open_window` çağrısı değildir. Startup, CLI veya open-url istekleri, workspace DB ve collab follow durumu birkaç global ve handle üzerinden birbirine bağlanır.

#### AppState

`AppState` Zed workspace açma ve restore işlemlerinde taşınan uygulama servis paketidir:

- `languages: Arc<LanguageRegistry>`
- `client: Arc<Client>`
- `user_store: Entity<UserStore>`
- `workspace_store: Entity<WorkspaceStore>`
- `fs: Arc<dyn fs::Fs>`
- `build_window_options: fn(Option<Uuid>, &mut App) -> WindowOptions`
- `node_runtime: NodeRuntime`
- `session: Entity<AppSession>`

`AppState::set_global(state, cx)` global olarak kurar; `AppState::global(cx)` ve `try_global(cx)` okuma yapar. Testlerde `AppState::test(cx)` sahte FS, test language registry ve test settings store kurar.

#### WorkspaceStore

`WorkspaceStore` açık workspace'leri `AnyWindowHandle + WeakEntity<Workspace>` çifti olarak izler. Collab tarafındaki follow ve update follower mesajları bu store üzerinden uygun workspace'e yönlendirilir.

- `WorkspaceStore::new(client, cx)` client request ve message işleyicilerini kaydeder.
- `workspaces()` weak workspace iterator'ı döndürür.
- `workspaces_with_windows()` window handle ile birlikte verir.
- `update_followers(project_id, update, cx)` aktif call üzerinden follower update mesajı yollar.
- Collab, titlebar ve follow akışlarında client tarafındaki `User`, oda/protokol kimliğini `legacy_id: LegacyUserId` alanında taşır. Proto room role lookup, participant index ve `join_in_room_project` çağrılarında bu alan kullanılır. Bu client modelinde ayrı bir `user.id` alanı yoktur.

#### WorkspaceDb ve HistoryManager

`WorkspaceDb::global(cx)` oturum ve workspace kalıcılığı için kullanılan SQLite bağlantı sarmalayıcısıdır. Workspace restore ve yakın zamanlı proje geçmişi şu katmanlara dağılır:

- `open_workspace_by_id(workspace_id, app_state, requesting_window, cx)` DB'deki serileştirilmiş workspace'i açar.
- `read_serialized_multi_workspaces`, `SerializedMultiWorkspace`, `SerializedWorkspaceLocation`, `SessionWorkspace`, `ItemId` kalıcılık modelidir.
- `HistoryManager::global(cx)` yakın zamanlı yerel workspace geçmişini verir.
- `HistoryManager::update_history(id, entry, cx)` yakın zamanlı listeyi günceller ve platform jump list'ini yeniler.
- `HistoryManager::delete_history(id, cx)` boşaltılan workspace'i geçmişten kaldırır.

#### OpenListener ve RawOpenRequest

`zed::open_listener` uygulama dışından gelen açma isteklerini kuyruğa alır:

```rust
let (listener, rx) = OpenListener::new();
listener.open(RawOpenRequest {
    urls,
    diff_paths,
    diff_all,
    dev_container,
    wsl,
    cwd,
});
```

- `OpenListener` bir `Global`'dir; `open(...)` isteği sınırsız bir kanala gönderir.
- `RawOpenRequest` ham CLI veya URL alanlarını taşır.
- `OpenRequest::parse(raw, cx)` bunları tipli `OpenRequest`'a çevirir.
- `OpenRequestKind` kaynak türünü belirtir: CLI connection, extension, agent panel, shared agent thread, dock menu action, builtin JSON schema, setting, git clone, git commit vb.
- Linux ve FreeBSD'de `listen_for_cli_connections` release-channel socket'i üzerinden CLI isteklerini alır.
- `RawOpenRequest::cwd` CLI işleminin çalışma dizinini taşır. Yalnızca `--diff` path'leri verildiğinde workspace bağlamı için bu cwd kullanılır; Zed app işleminin `std::env::current_dir()` değeri macOS bundle veya zaten çalışan örnek yüzünden güvenilir değildir.
- SSH URL ayrıştırma akışı normal URL'lere ek olarak SCP veya git tarzı `ssh://user@host:~/project` ve `ssh://user@host:/absolute/path` biçimlerini normalleştirir. Kullanıcı adı ve parola URL kodu çözülür; IPv6 SCP-style authority ve çift port benzeri belirsiz biçimler reddedilir.
- `open_paths_with_positions` diff path kanonikleştirme için `app_state.fs` kullanır; hataları `opened_items` listesine taşıyarak diğer path'leri açmaya devam eder.

**Global agent yönergesi.** Kullanıcının kişisel AGENTS.md dosyası şu akışla ele alınır:

- Startup sırasında `zed::watch_user_agents_md(app_state.fs.clone(), cx)` çağrılır. Bu, `paths::agents_file()` (`~/.config/zed/AGENTS.md`, platforma göre eşdeğer) dosyasını izler ve `agent::UserAgentsMd` global'ine yükler.
- Boş veya yalnızca boşluk içeren dosya `UserAgentsMdState::Empty`, başarılı okuma `Loaded`, okunamayan ama mevcut dosya `Error` olur. Hata durumunda ayar hatalarıyla aynı app seviyesindeki notification yolu kullanılır.
- Yerel agent system prompt'u kişisel `AGENTS.md` içeriğini "Personal `AGENTS.md`" olarak project rules'tan önce çizer; çakışma durumunda project rules daha sonra geldiği için daha spesifik kabul edilir.

**Tuzaklar.** Open akışı ve global durum ile çalışırken karşılaşılan hatalar:

- Workspace açma akışında `AppState::build_window_options` kullanılır; doğrudan `WindowOptions` kopyalamak Zed'in başlık çubuğu, app id, sınır geri yükleme ve platform ayarlarını atlar.
- `WorkspaceStore` weak workspace tutar; iterasyon sırasında upgrade başarısız olabilir.
- `OpenListener::open` dinleyici yokken hatayı loglar; talebin teslim edildiği varsayımıyla kullanıcı akışının başlatılmaması gerekir.
- DB restore yolunda serializable item kind eksikse item restore edilemez; yeni bir item türü eklenirken `register_serializable_item` startup init'inde çağrılmalıdır.

## Item Ayarları, Bağlam Menüsü, ApplicationMenu ve Focus-Follows-Mouse

Zed UI kodunda sık görülen ama GPUI çekirdeği olmayan birkaç yardımcı katman daha vardır; bunlar item davranışı, bağlam menüsü, uygulama menüsü ve focus-follows-mouse gibi konuları kapsar.

#### Item Ayarları ve SaveIntent

Item ve tab davranışını ayarlar tarafına bağlayan tipler şunlardır:

- `ItemSettings` — `git_status`, `close_position`, `activate_on_close`, `file_icons`, `show_diagnostics`, `show_close_button` alanlarını `tabs` ve `git` ayarlarından üretir.
- `PreviewTabsSettings` — preview tab kaynaklarını ayrı ayrı açıp kapatır: project panel, file finder, multibuffer, kod gezinme, keep-preview gibi.
- `TabContentParams { detail, selected, preview, deemphasized }` — tab çizimine seçim, preview ve odak dışı durumu taşır; `text_color()` anlamsal `Color` döndürür.
- `TabTooltipContent::{Text, Custom}` — tab tooltip'ini string veya özel view olarak tanımlar.
- `ItemBufferKind::{Multibuffer, Singleton, None}` — item'in buffer ilişkisini sınıflandırır.

**Kaydetme ve kapatma akışı.** İşlemler `SaveIntent` ile yönlendirilir:

- Varyantlar: `Save`, `FormatAndSave`, `SaveWithoutFormat`, `SaveAll`, `SaveAs`, `Close`, `Overwrite`, `Skip`.
- Pane close action'ları `CloseActiveItem`, `CloseOtherItems`, `CloseAllItems` gibi action struct'larında optional `SaveIntent` taşır. Dirty/format/conflict davranışı doğrudan bool ile çoğaltılmaz; mevcut save intent zincirine bağlanılır.
- `SaveOptions { format, force_format, autosave }` item save uygulamasının düşük seviyeli karar paketidir.

#### ContextMenu ve PopoverMenu

`ContextMenu` `ManagedView` olarak modal/popover zincirine takılır. İçerik modeli şu öğelerden oluşur:

- `ContextMenuItem::{Separator, Header, HeaderWithLink, Label, Entry, CustomEntry, Submenu}`.
- `ContextMenuEntry` label, icon, checked/toggle, action, disabled, secondary handler, documentation aside ve end-slot gibi alanları taşır.
- `ContextMenu::build(window, cx, |menu, window, cx| ...)` menü entity'sini üretir.
- `menu.context(focus_handle)` menü action kullanılabilirliği ve keybinding görüntüsü için belirli bir odak bağlamını kullanır.

`PopoverMenu<M: ManagedView>` anchor element ile yönetilen menü view'ini bağlar:

- `PopoverMenu::new(id)`, `.menu(...)`, `.with_handle(handle)`, `.anchor(...)`, `.attach(...)`, `.offset(...)`, `.full_width(...)`, `.on_open(...)`.
- `PopoverMenuHandle<M>` dışarıdan toggle, kapatma ve açık menü entity'sine erişmek için saklanır.
- Popover konumlandırması `window.layout_bounds` ve çift `on_next_frame` desenini kullanabilir; anchor sınırları (`bounds`) ilk karede, menü sınırları bir sonraki karede bilinir.

#### İstemci Tarafı ApplicationMenu

macOS dışındaki istemci tarafı application menüsü `title_bar::ApplicationMenu` ile çizilir:

- `ApplicationMenu::new(window, cx)` `cx.get_menus()` ile platform ve app menülerini okur; her üst seviye menü için bir `PopoverMenuHandle<ContextMenu>` saklar.
- `OpenApplicationMenu(String)` action'ı belirli menüyü açar.
- `ActivateMenuLeft` ve `ActivateMenuRight` client-side menü bar içinde yatay gezinmeyi sağlar.
- `ApplicationMenu` boş alt menüleri ve ardışık veya izleyen ayırıcıları temizler, sonra `OwnedMenuItem::{Action, Submenu, Separator, SystemMenu}` değerlerini işler. `Action`, `Submenu` ve `Separator` `ContextMenu` girişlerine dönüşür; `SystemMenu(_)` client-side context'te anlamlı olmadığı için yok sayılır.

#### FocusFollowsMouse

`FocusFollowsMouse` trait'i `StatefulInteractiveElement` üzerine eklenir:

```rust
element.focus_follows_mouse(WorkspaceSettings::get_global(cx).focus_follows_mouse, cx)
```

- Ayar açık olduğunda hover girişi sırasında hedef `AnyWindowHandle + FocusHandle` global duruma yazılır.
- Debounce için `cx.background_executor().timer(settings.debounce).await` kullanılır.
- Debounce sonunda `cx.update_window(window, |_, window, cx| window.focus(&focus, cx))` çağrılır.
- Daha spesifik bir alt focus hedefi varken üst hover'ın bunu ezmesi istenmediğinde `focus_handle.contains(existing, window)` kontrolü yapılır.

**Tuzaklar.** Bu yardımcı katmanlarda dikkat edilmesi gerekenler:

- Bağlam menüsü action'ları odaktaki element context'ine göre enable veya disable olur; menü odak bağlamı olmadan kurulduğunda bazı action'lar görünür ama çalışmayabilir.
- ApplicationMenu platform menü çubuğu değildir; macOS yerel menüsü ayrı platform menü akışından gelir.
- Focus-follows-mouse global debounce durumu kullanır; aynı anda birden çok hover hedefi yarışabilir, bu nedenle daha spesifik alt kontrol kaldırılmamalıdır.

---
