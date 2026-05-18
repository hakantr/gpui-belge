# Komut Paleti ve Picker

---

## CommandPalette: Filter, Aliases ve Interceptor

`crates/command_palette_hooks/` global'leri komut paletinin UX'ini şekillendirir. UI yazılmadan önce bu global'leri tanımak gerekir; çünkü odaktaki elementten toplanan action listesi, görünürlük filtresi ve alternatif komut sonuçları bu katmandan geçer. Zed başlangıcında `command_palette::init(cx)` `command_palette_hooks::init(cx)` çağırır; filtre global'i bu çağrı sırasında kurulur.

#### CommandPaletteFilter

```rust
pub struct CommandPaletteFilter {
    hidden_namespaces: HashSet<&'static str>,
    hidden_action_types: HashSet<TypeId>,
    shown_action_types: HashSet<TypeId>,
}
```

Erişim noktaları şunlardır:

- `CommandPaletteFilter::try_global(cx) -> Option<&Self>`
- `CommandPaletteFilter::global_mut(cx) -> &mut Self`
- `CommandPaletteFilter::update_global(cx, |filter, cx| ...)`

`update_global` global yoksa yenisini oluşturmaz; `cx.has_global` kontrolünden sonra çalışır ve global mevcut değilse no-op kalır. Bu nedenle command palette crate'i kullanılmadan yalnızca hook crate'ine erişiliyorsa önce `command_palette_hooks::init(cx)` çağrılmalıdır.

**Filtre yönetimi.** Komut paletinde hangi action'ların görünür kalacağı filtre üzerinden ayarlanır:

- `is_hidden(&action) -> bool` — belirli action namespace veya tip olarak gizliyse `true` döner. Komut paleti UI'ı bu sonuca göre gösterimi atlar. `shown_action_types` içine alınmış tipler, namespace gizli olsa bile görünür kalır.
- `hide_namespace(&'static str)` / `show_namespace(&'static str)` — bir namespace'in tüm action'larını gizler veya gösterir (örneğin başsız paneller için).
- `hide_action_types(impl IntoIterator<Item = &TypeId>)` ve `show_action_types(impl IntoIterator<Item = &TypeId>)` — belirli action tiplerini topluca yönetir. `hide_action_types` tipi `hidden_action_types` setine ekler ve `shown_action_types` setinden çıkarır; `show_action_types` ise tersini yapar.

Tipik bir kullanım, `CommandPaletteFilter::update_global(cx, |filter, _| { ... })` bloğu içinde aynı anda hem `hide_namespace` hem `hide_action_types` çağırarak özellik bayrağı tabanlı komut görünürlüğünü tek seferde değiştirmektir. Örneğin Vim entegrasyonu, vim modu açıldığında `vim` namespace'ini gösterir ve modu kapandığında tekrar gizler.

#### CommandAliases (WorkspaceSettings)

`WorkspaceSettings::command_aliases: HashMap<String, ActionName>`. Kullanıcı JSON'una `"command_aliases": { "ag": "search::ToggleSearch" }` yazıldığında komut paleti, sorgu tam olarak `ag` olduğunda bunu `search::ToggleSearch` string'ine çevirir. Bu çeviri fuzzy eşleşme ve interceptor çağrısından önce yapılır; alias bir action nesnesi üretmez, yalnızca palet sorgusunu canonical action adına yaklaştırır. Yeni uygulamada alias, kullanıcının kısa sorgu yazmasını kolaylaştıran bir palet kısayolu olarak düşünülmelidir. Keymap tarafında canonical action adı kullanılmalıdır.

#### CommandInterceptor

Komut paletindeki "tam string'i komuta dönüştür" davranışı (örneğin Vim ex komutları, line jump `:42`) `GlobalCommandPaletteInterceptor` üzerinden çalışır:

```rust
GlobalCommandPaletteInterceptor::set(cx, |query, workspace, cx| {
    parse_query_to_actions(query, workspace, cx)
});
```

İmzalar şu şekildedir:

- `set(cx, Fn(&str, WeakEntity<Workspace>, &mut App) -> Task<CommandInterceptResult>)`
- `clear(cx)` — interceptor'ı kaldırır.
- `intercept(query, workspace, cx) -> Option<Task<CommandInterceptResult>>` — komut paleti UI'ı her tuş vuruşunda çağırır.

`CommandInterceptResult` yapısı şu şekildedir:

```rust
pub struct CommandInterceptResult {
    pub results: Vec<CommandInterceptItem>,
    pub exclusive: bool, // true ise normal action eşleşmeleri gizlenir
}

pub struct CommandInterceptItem {
    pub action: Box<dyn Action>,
    pub string: String,             // palette'te gösterilecek metin
    pub positions: Vec<usize>,      // vurgu konumları
}
```

Tipik akış şudur: Vim modu açıkken `:w<CR>` gibi komutlar intercept edilip `SaveActiveItem` action'ına çevrilir. Benzer şekilde başka extension veya agent türleri de aynı mekanizmayı kullanır. Komut paleti interceptor sonuçlarını normal fuzzy action eşleşmeleriyle birleştirir. Aynı action zaten normal eşleşmelerde yer alıyorsa interceptor sonucu eklenmeden önce normal sonuçtan çıkarılır. `exclusive = true` olduğunda yalnız interceptor sonuçları gösterilir; `exclusive = false` ise interceptor sonuçları listenin başına eklenir ve normal eşleşmeler arkadan gelir.

#### Action Documentation

Action trait'inin `documentation()` yüzeyi komut paleti ile karıştırılmamalıdır:

- Komut paleti satırında şu anda insancıllaştırılmış action adı ve mevcut keybinding görünür; action documentation palet satırında ayrı bir açıklama olarak çizilmez.
- Komut paleti `window.available_actions(cx)` ile odaktaki dispatch path'ten action tiplerini toplar; `build_action_type(type_id)` ile canonical action adını üretir.
- Doc yorumu yazımı yine önemlidir: derive makrosu bunu `documentation()` üzerinden ifşa eder ve keymap editörü ile JSON schema gibi action keşif yüzeyleri bu bilgiyi kullanır.

#### Tuzaklar

Filter ve interceptor kullanımında karşılaşılan yaygın sorunlar:

- `CommandPaletteFilter` global durumdur; testlerde bir özellik açıldığında sonraki test başlamadan sıfırlanması gerekebilir.
- `hide_action_types` ile gizlenen tipin kaydedilmiş olması gerekir; aksi halde filtreye eklenmesine rağmen komut paleti listesinde zaten görünmez.
- `Interceptor::set` mevcut interceptor'ı ezer; çoklu kaynak gerektiğinde zincirin kendi kodunda kurulması gerekir (örneğin önce Vim, başarısızsa AI agent gibi).
- `CommandInterceptResult::exclusive = true` yoğun şekilde kullanıldığında kullanıcı normal action listesinden komutlara ulaşamaz; gerçekten "tek doğru sonuç var" durumunda ayarlanır.

## CommandPalette Çalışma Zamanı Akışı, Fuzzy Arama ve Geçmiş

`crates/command_palette/src/command_palette.rs` Zed'in gerçek komut paleti akışıdır. Başlatma ve açma sırası şu adımlarla işler:

1. `command_palette::init(cx)` hook global'lerini kurar ve `cx.observe_new(CommandPalette::register).detach()` ile her yeni `Workspace` için `zed_actions::command_palette::Toggle` action'ını kaydeder.
2. `CommandPalette::toggle(workspace, query, window, cx)` mevcut focus handle'ını alır. Focus yoksa palet açılmaz. Ardından `workspace.toggle_modal(...)` ile `CommandPalette` modal view olarak oluşturulur.
3. `CommandPalette::new(previous_focus_handle, query, workspace, window, cx)` `window.available_actions(cx)` çağırır. Bu liste bütün registry değildir; odaktaki dispatch path üzerinde `.on_action(...)` ile bağlı action'lar ve global action dinleyicileridir.
4. Her action `CommandPaletteFilter::is_hidden` ile elenir ve görünen action'lar `humanize_action_name(action.name())` sonucuyla `Command { name, action }` haline getirilir.
5. UI `Picker::uniform_list(delegate, window, cx)` ile kurulur; ardından başlangıç query'si `picker.set_query(query, window, cx)` ile editöre yazılır.

**Arama akışı.** Sorgu birkaç aşamadan geçer:

- `humanize_action_name("editor::GoToDefinition")` sonucu `"editor: go to definition"` olur; `go_to_line::Deploy` gibi snake case adlar boşluklu hale gelir.
- `normalize_action_query(input)` kırpar, ardışık boşluğu tek boşluğa indirir, `_` karakterlerini boşluğa çevirir ve ardışık `::` yazımlarını arama için sadeleştirir.
- `WorkspaceSettings::command_aliases` tam sorgu eşleşmesini canonical action adı string'ine çevirir.
- Sorgu bir Zed bağlantısı ise (`parse_zed_link`) palette `OpenZedUrl { url }` action'ını interceptor sonucu gibi listeye ekler.
- Normal komut listesi arka plan executor'ı üzerinde `fuzzy_nucleo::match_strings_async(..., Case::Smart, LengthPenalty::On, 10000, ...)` ile eşleştirilir. Eşleşmeler hit sayısına göre, sonra alfabetik ada göre sıralanan komut havuzundan gelir.
- Interceptor sonucu varsa `matches_updated` içinde normal eşleşmelerle birleştirilir; tekrarlanan action'lar `Action::partial_eq` ile ayıklanır.

**Geçmiş ve sıralama.** `CommandPaletteDB` SQLite tablosu komut geçmişini tutar:

- SQLite domain adı `CommandPaletteDB`'dir; tablo `command_invocations`.
- `write_command_invocation(command_name, user_query)` çalıştırılan komutu ve kullanıcının yazdığı sorguyu kaydeder. Tablo 1000 kaydı geçtiğinde en eski kayıt silinir.
- `list_commands_used()` komut başına invocation sayısı ile son kullanım zamanını döndürür; palet açılırken hit sayısı yüksek komutlar önce sıralanır.
- `list_recent_queries()` boş olmayan kullanıcı sorgularını son kullanım zamanına göre getirir; komut paletinde yukarı/aşağı gezinilirken aynı önek ile sorgu geçmişine dönülebilir.

**Onay davranışı.** Onay akışları iki şekilde işler:

- Normal onay seçili komutu alır, telemetry'ye `source = "command palette"` ile yazar, `CommandPaletteDB` kaydını bir arka plan görevi olarak başlatır, önceden odakta olan handle'a geri odaklanır, modalı kapatır ve `window.dispatch_action(action, cx)` çağırır.
- İkincil onay seçili action'ın canonical adını `String` olarak alır ve `zed_actions::ChangeKeybinding { action: action_name.to_string() }` action'ını dispatch eder. Buradaki `action` alanı bir action nesnesi değil, registry name string'idir (örneğin `"editor::GoToDefinition"`); keymap editörü bu string'i alır ve bağlama ekleme akışını başlatır. Footer'daki "Add/Change Keybinding" butonu da aynı yolu kullanır.
- `finalize_update_matches` bekleyen arka plan sonucu en fazla kısa bir süre ön planda bekleyebilir; bu, palet açılırken boş liste titremesini ve otomasyon sırasında erken enter basma durumunu azaltır.

## Picker, PickerDelegate ve PickerPopoverMenu

`crates/picker/` komut paleti dışında da kullanılan genel bir seçim ve arama bileşenidir. Yeni bir picker yazılırken esas iş, `PickerDelegate` uygulamasını yazmaktır:

```rust
pub trait PickerDelegate: Sized + 'static {
    type ListItem: IntoElement;

    fn match_count(&self) -> usize;
    fn selected_index(&self) -> usize;
    fn set_selected_index(&mut self, ix: usize, window: &mut Window, cx: &mut Context<Picker<Self>>);
    fn placeholder_text(&self, window: &mut Window, cx: &mut App) -> Arc<str>;
    fn update_matches(&mut self, query: String, window: &mut Window, cx: &mut Context<Picker<Self>>) -> Task<()>;
    fn confirm(&mut self, secondary: bool, window: &mut Window, cx: &mut Context<Picker<Self>>);
    fn dismissed(&mut self, window: &mut Window, cx: &mut Context<Picker<Self>>);
    fn render_match(&self, ix: usize, selected: bool, window: &mut Window, cx: &mut Context<Picker<Self>>) -> Option<Self::ListItem>;
}
```

**Sık üzerine yazılan davranışlar.** Picker farklı senaryolara uydurmak için bir dizi opsiyonel hook sağlar:

- `select_history(Direction, query, ...) -> Option<String>` — yukarı veya aşağı oklarını varsayılan seçim yerine sorgu geçmişinde gezdirmek için.
- `can_select(ix, ...)`, `select_on_hover()`, `selected_index_changed(...)` — seçilebilir satırları ve hover/seçim yan etkilerini yönetir.
- `no_matches_text(...)`, `render_header(...)`, `render_footer(...)` — boş durum ile sabit üst ve alt alanlar.
- `documentation_aside(...)` ve `documentation_aside_index()` — seçili veya hover edilen öğe için sağda dokümantasyon paneli göstermek.
- `confirm_update_query(...)`, `confirm_input(...)`, `confirm_completion(...)` — enter'ın seçimi onaylamak yerine sorguyu dönüştürdüğü veya birebir girdiyi action'a çevirdiği picker türleri.
- `editor_position() -> PickerEditorPosition::{Start, End}` — arama editörünün listenin üstünde mi altında mı duracağını belirler.
- `finalize_update_matches(query, duration, ...) -> bool` — arka plan eşleştirmesini kısa süreliğine bloklayarak ilk çizim ve onay yarışını azaltır.

**Yapıcı seçimi.** Picker üretmek için dört yapıcı vardır:

- `Picker::uniform_list(delegate, window, cx)` — aramalı picker; tüm satırlar aynı yükseklikteyse tercih edilir ve `gpui::uniform_list` kullanır.
- `Picker::list(delegate, window, cx)` — aramalı picker; satır yükseklikleri değişkense kullanılır.
- `Picker::nonsearchable_uniform_list(...)` ve `Picker::nonsearchable_list(...)` — arama editörü olmayan seçim listeleri.

**Kullanılabilir ayarlar.** Picker davranışı zincir üzerinden ince ayarlanır:

- `width(...)`, `max_height(...)`, `widest_item(...)` — ölçü ve liste genişliği.
- `show_scrollbar(bool)` — dış scrollbar gösterimi.
- `modal(bool)` — picker kendi başına modal gibi çiziliyorsa elevation verir; daha büyük bir modalın parçasıysa `false` yapılabilir.
- `list_measure_all()` — `ListState` tabanlı listede tüm öğeleri ölçmek için.
- `refresh(&mut self, window, cx)`, `update_matches_with_options(..., ScrollBehavior)` — eşleşme akışını dışarıdan tetikleyen değişebilen yardımcılar.
- `query(&self, cx: &App) -> String` — editördeki anlık sorguyu okur.
- `set_query(&self, query: &str, window: &mut Window, cx: &mut App)` — editör metnini değiştirir; `&self` aldığına dikkat — picker entity'sini bir `update` bloğunun içine sokmak şart değildir, doğrudan picker referansından çağrılabilir. `cx` burada `Context<...>` değil `&mut App` olduğu için entity context gerekiyorsa update bloğundan dışarı çıkmak gerekebilir.

**Action ve key context.** Picker kökü kendi key context'ini ve action dinleyicilerini kurar:

- Çizim kökü `"Picker"` key context'ini ekler.
- `menu::SelectNext`, `menu::SelectPrevious`, `menu::SelectFirst`, `menu::SelectLast`, `menu::Cancel`, `menu::Confirm`, `menu::SecondaryConfirm`, `picker::ConfirmCompletion` ve `picker::ConfirmInput` action'larını dinler.
- Tıklama onayı sırasında `cx.stop_propagation()` ve `window.prevent_default()` çağrılır; bu sayede picker satırına tıklama dış elementlere sızmaz.

**`PickerPopoverMenu`.** Bu sarmalayıcı, bir picker'ı `ui::PopoverMenu` içine yerleştiren ince bir yapıdır. `new(picker, trigger, tooltip, anchor, cx)` picker'ın `DismissEvent`'ini popover dismiss event'ine bağlar; `with_handle(...)` ve `offset(...)` ile dış popover handle ve konum ayarı yapılır. Picker bir toolbar butonu veya popover tetikleyicisi arkasında açılacaksa doğrudan modal yerine bu sarmalayıcı tercih edilir.

**Zed picker örneklerinde güncel kullanım notları.** `file_finder` artık `path:line-column` sorgularına ek olarak `path:start-end` satır aralıklarını da anlar. Örneğin `src/app.rs:12-20` dosyayı açıp ilgili satır aralığını seçer; aralık dosya sonunu aşarsa EOF'a kırpılır. Geçersiz veya ters aralıklar `PathWithPosition` davranışına düşer ve tek konuma gider. Sonda kalan tek satır iki noktası `path:12:` biçiminde temizlenir, fakat aralık biçimleri korunur.

Git branch picker tarafında `git_ui::branch_picker::select_popover(...)` checkout yapmayan seçim popover'ı üretir. Bu mod `BranchSelectionBehavior::Select` kullanır, placeholder olarak `Select branch...` gösterir, footer ve silme aksiyonlarını sunmaz, seçimden sonra `DismissEvent` yayar ve verilen `SelectBranchCallback` ile seçilen `Branch` değerini dışarı taşır. Branch sıralama seçili branch'i, aktif remote üzerindeki branch'leri, aktif/upstream bağlamını ve kalanları önceliklendirir; aynı öncelikte yerel branch'ler uzak branch'lerden önce gelir.

---
