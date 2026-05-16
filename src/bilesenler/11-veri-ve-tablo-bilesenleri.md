# 11. Veri ve Tablo Bileşenleri

Zed UI tarafında tabloya ihtiyaç duyulduğunda ana giriş noktası `Table` bileşenidir. Küçük ve sabit satırlı tablolar doğrudan `.row(...)` çağrılarıyla kurulur. Satır sayısı büyüdüğünde ise tablo, GPUI'nin sanallaştırılmış liste altyapısına bağlanan `.uniform_list(...)` veya `.variable_row_height_list(...)` çağrılarıyla render edilir. Yani tablo tek bir kalıba sıkışmaz; satır modeline göre üç farklı kullanım biçimi sunar.

## GPUI uniform_list ile köprü

`Table::uniform_list(...)` ve Bölüm 9'daki büyük listeler GPUI'nin `uniform_list(...)` elementine bağlanır. Bu element yalnızca görünür satır aralığını render eder; böylece binlerce satırlık listeler gereksiz render maliyeti yaratmadan ekrana basılabilir. Kullanırken şu kurallara dikkat etmek gerekir:

- `uniform_list(id, item_count, |range, window, cx| Vec<AnyElement>)` imzası bir id ile bir satır sayısı alır; kalan kısımda yalnızca görünür `range` için satırlar üretilir. Range içindeki indeks dizisi `range.map(|ix| ...)` ifadesiyle dolaşılır.
- Satır yüksekliğinin homojen olması beklenir. İçerik her satırda farklı bir yükseklik gerektiriyorsa, GPUI tarafındaki `list(...)` elementi ve `ListState` ile çalışan `Table::variable_row_height_list(...)` daha uygun bir seçim olur.
- Scroll davranışı için `UniformListScrollHandle` view struct'ında saklanır ve `.track_scroll(&handle)` ile bağlanır. Tablo tarafında `Table::interactable(...)` çağrısı kullanıldığında bu, içerideki `TableInteractionState` üzerinden yönetilir.
- `with_sizing_behavior(ListSizingBehavior::Infer)`, listenin içeriğine göre yükseklik almasını sağlar; `Fill` ise parent yüksekliğini kullanır.
- `with_decoration(...)` slot'una `IndentGuides` ve `StickyItems` gibi decoration'lar bağlanır; bu decoration'ların `UniformListDecoration` trait'ini implement etmesi gerekir.

Karar matrisi:

| Satır modeli | Kullanım |
| :-- | :-- |
| Sabit, az satır | `List::new()` ile `ListItem::new(...)`; scroll doğrudan parent içinde yapılır. |
| Sabit yükseklik, çok satır | `uniform_list(id, count, ...)` veya `Table::uniform_list(...)`. |
| Değişken yükseklik, çok satır | `gpui::list(...) + ListState` veya `Table::variable_row_height_list(...)`. |
| Hiyerarşik ya da sticky parent | `uniform_list(...)` ile birlikte `IndentGuides` ve `StickyItems`. |

`gpui::ListAlignment` (`Top`, `Bottom`) ve `ListSizingBehavior` (`Fill`, `Infer`) için tip referansları `gpui` crate'inde tanımlıdır. UI tarafında bu değerleri kullanan örnekler `crates/keymap_editor`, `crates/csv_preview` ve `crates/project_panel` içinde yer alır.

Bu ailede tablo kurarken üç karar birlikte düşünülür; biri değiştiğinde diğerleri de genellikle etkilenir:

- **Satır modeli:** sabit bir satır listesi mi, sabit yükseklikli sanallaştırılmış bir liste mi, yoksa değişken yükseklikli sanallaştırılmış bir liste mi.
- **Kolon genişliği modeli:** otomatik, explicit, redistributable veya resizable.
- **Etkileşim modeli:** sadece görsel bir tablo mu, yoksa focus, scroll ve resize state'i tutan interactable bir tablo mu.

## Table

Kaynak:

- Tanım: `../zed/crates/ui/src/components/data_table.rs`
- Export: `ui::Table`, `ui::UncheckedTableRow`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for Table`.

Ne zaman kullanılır:

- Header, satır border'ları, striped görünüm ve kolon hizası gereken veri görünümleri için.
- Satır sayısı büyüdüğünde sanallaştırma üzerinden performanslı bir şekilde render almak gerektiğinde.
- Kolon genişliklerinin tek API üzerinden yönetilmesi istendiğinde.

Ne zaman kullanılmaz:

- Tek kolonlu seçim listelerinde `List` ile `ListItem` çok daha uygundur.
- Hiyerarşik veri için `TreeViewItem` doğru yüzeydir.
- Form satırları veya toolbar bilgileri için tablo yerine `h_flex()` veya `v_flex()` ile açık bir layout kurmak genellikle daha okunur olur.

Temel API:

- Constructor: `Table::new(cols)`.
- Header: `.header(headers)`.
- Sabit satır: `.row(items)`.
- Sanallaştırılmış sabit yükseklikli satırlar: `.uniform_list(id, row_count, render_item_fn)`.
- Sanallaştırılmış değişken yükseklikli satırlar: `.variable_row_height_list(row_count, list_state, render_row_fn)`.
- Görsel builder'lar: `.striped()`, `.hide_row_borders()`, `.hide_row_hover()`, `.no_ui_font()`, `.disable_base_style()`.
- Genişlik: `.width(width)`, `.width_config(config)`.
- Sabit ilk kolonlar: `.pin_cols(n)`.
- Etkileşim: `.interactable(&table_interaction_state)`.
- Satır özelleştirme: `.map_row(callback)`.
- Boş durum: `.empty_table_callback(callback)`.

Davranış:

- `cols`, tablo satırlarının ve header'ın beklenen kolon sayısıdır.
- `.header(...)` ve `.row(...)` içine verilen `Vec<T>` içeride bir `TableRow<T>` değerine çevrilir. Eleman sayısı `cols` ile eşleşmezse panic oluşur.
- Varsayılan hücre stili `px_1()`, `py_0p5()`, `whitespace_nowrap()`, `text_ellipsis()` ve `overflow_hidden()` özelliklerini uygular.
- `.disable_base_style()` çağrısı hücre baz stilini kapatır. CSV önizleme gibi her hücrenin kendi layout'unu taşıdığı durumlarda kullanılır.
- `.row(...)` yalnızca tablo sabit satır modundayken satır ekler. Tablo `.uniform_list(...)` veya `.variable_row_height_list(...)` ile kurulduysa satırlar bir closure üzerinden üretilir.
- `.map_row(...)`, tablonun ürettiği `Stateful<Div>` satır container'ını alır. Bu sayede seçili satır, hover state'i, sağ tık veya özel click davranışı gibi ek davranışlar eklemek mümkün hale gelir.
- `.pin_cols(n)`, ilk `n` kolonu yatay scroll sırasında görünür tutar. Kaynakta yalnızca `ColumnWidthConfig::Resizable` ile desteklenir; `n == 0` veya `n >= cols` durumunda tablo tek bölümlü normal bir layout'a döner.
- Pinned layout'ta header, satırlar ve resize overlay aynı yatay `ScrollHandle`'ı izler. Pinned kolonlar, scrollable kolonlarla aynı list item içinde render edildiği için değişken yükseklikli satırlarda iki tarafın yüksekliği ayrışmaz. Bu nedenle `.pin_cols(...)` kullanılırken tablonun `.interactable(...)` ile bağlanması pratikte zorunlu hâle gelir.

Minimum örnek:

```rust
use ui::{Table, prelude::*};

fn render_model_table() -> impl IntoElement {
    Table::new(3)
        .width(px(520.))
        .header(vec!["Model", "Provider", "Status"])
        .row(vec!["gpt-5.2", "OpenAI", "Ready"])
        .row(vec!["claude-sonnet", "Anthropic", "Needs key"])
        .row(vec!["local-llm", "Ollama", "Offline"])
        .striped()
}
```

Karışık hücre içeriği gerekiyorsa, her hücrenin `.into_any_element()` çağrısıyla aynı tipe çevrilmesi yeterlidir:

```rust
use ui::{Button, ButtonStyle, Indicator, Table, prelude::*};

fn render_package_row_table() -> impl IntoElement {
    Table::new(4)
        .width(px(720.))
        .header(vec![
            "State".into_any_element(),
            "Package".into_any_element(),
            "Version".into_any_element(),
            "Action".into_any_element(),
        ])
        .row(vec![
            Indicator::dot().color(Color::Success).into_any_element(),
            Label::new("rust-analyzer").truncate().into_any_element(),
            Label::new("1.0.0").color(Color::Muted).into_any_element(),
            Button::new("open-rust-analyzer", "Open")
                .style(ButtonStyle::Subtle)
                .into_any_element(),
        ])
}
```

Sabit yükseklikli büyük bir liste için tablo `.uniform_list(...)` ile kurulur. Burada satır sayısı view içinde tutulur ve sadece görünür range render edilir:

```rust
use gpui::Entity;
use ui::{Table, TableInteractionState, prelude::*};

#[derive(Clone)]
struct PackageRow {
    name: SharedString,
    version: SharedString,
    enabled: bool,
}

struct PackagesTable {
    table_state: Entity<TableInteractionState>,
    rows: Vec<PackageRow>,
}

impl PackagesTable {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            table_state: cx.new(|cx| TableInteractionState::new(cx)),
            rows: Vec::new(),
        }
    }
}

impl Render for PackagesTable {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let rows = self.rows.clone();

        Table::new(3)
            .interactable(&self.table_state)
            .striped()
            .header(vec!["Package", "Version", "State"])
            .uniform_list("packages-table", rows.len(), move |range, _window, _cx| {
                range
                    .map(|index| {
                        let row = &rows[index];
                        vec![
                            Label::new(row.name.clone()).truncate().into_any_element(),
                            Label::new(row.version.clone())
                                .color(Color::Muted)
                                .into_any_element(),
                            Label::new(if row.enabled { "Enabled" } else { "Disabled" })
                                .into_any_element(),
                        ]
                    })
                    .collect()
            })
    }
}
```

Değişken yükseklikli satırlarda ise `ListState` ile birlikte `.variable_row_height_list(...)` kullanılır. Veri her değiştiğinde state'in `reset(...)` veya uygun bir `splice(...)` ile güncellenmesi gerekir:

```rust
use gpui::{Entity, ListAlignment, ListState};
use ui::{Table, TableInteractionState, prelude::*};

#[derive(Clone)]
struct LogRow {
    level: SharedString,
    message: SharedString,
}

struct LogTable {
    table_state: Entity<TableInteractionState>,
    list_state: ListState,
    rows: Vec<LogRow>,
}

impl LogTable {
    fn new(rows: Vec<LogRow>, cx: &mut Context<Self>) -> Self {
        Self {
            table_state: cx.new(|cx| TableInteractionState::new(cx)),
            list_state: ListState::new(rows.len(), ListAlignment::Top, px(100.)),
            rows,
        }
    }

    fn replace_rows(&mut self, rows: Vec<LogRow>, cx: &mut Context<Self>) {
        self.list_state.reset(rows.len());
        self.rows = rows;
        cx.notify();
    }
}

impl Render for LogTable {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        let rows = self.rows.clone();

        Table::new(2)
            .interactable(&self.table_state)
            .header(vec!["Level", "Message"])
            .variable_row_height_list(rows.len(), self.list_state.clone(), move |index, _, _| {
                let row = &rows[index];
                vec![
                    Label::new(row.level.clone()).color(Color::Muted).into_any_element(),
                    div()
                        .whitespace_normal()
                        .child(Label::new(row.message.clone()))
                        .into_any_element(),
                ]
            })
    }
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/ui/src/components/data_table.rs`: component preview içindeki basit, striped ve karışık içerikli tablo örnekleri.
- `../zed/crates/keymap_editor/src/keymap_editor.rs`: keymap tablosu, `uniform_list`, `TableInteractionState` ve redistributable kolonlar.
- `../zed/crates/csv_preview/src/renderer/render_table.rs`: CSV için `ResizableColumnsState`, `disable_base_style()` ve iki farklı render mekanizması. İlk kolon `pin_cols(1)` ile yatay scroll sırasında sabitlenir.
- `../zed/crates/edit_prediction_ui/src/edit_prediction_context_view.rs`: metadata için küçük, UI fontu kapatılmış bir tablo.

Dikkat edilecek noktalar:

- Header ve tüm satırların aynı kolon sayısında olması gerekir.
- Bir `Vec` içinde farklı element tipleri kullanılıyorsa, her hücrenin `.into_any_element()` çağrısıyla aynı tipe çevrilmesi gerekir.
- Büyük veri setlerinde `.row(...)` ile binlerce satır eklemek beklenmez; bu durumda `.uniform_list(...)` veya `.variable_row_height_list(...)` doğru tercihtir.
- `variable_row_height_list` için kullanılan `ListState`, satır sayısıyla senkron tutulmalıdır. Veri sayısı değiştiğinde `reset(...)` veya uygun `splice(...)` çağrısı yapılır.

## TableInteractionState

Kaynak:

- Tanım: `../zed/crates/ui/src/components/data_table.rs`
- Export: `ui::TableInteractionState`.
- Prelude: Hayır; ayrıca import edilir.
- Render modeli: `Entity<TableInteractionState>` olarak view state'inde tutulur.

Ne zaman kullanılır:

- Tablo kendi içinde dikey kaydırma yapacaksa.
- Kolon resize handle'ları kullanılacaksa.
- Tablonun focus handle'ı, scroll offset'i veya özel scrollbar ayarı dışarıdan yönetilecekse.

Temel API:

- `TableInteractionState::new(cx)`.
- `.with_custom_scrollbar(scrollbars)`.
- `.scroll_offset() -> Point<Pixels>`.
- `.set_scroll_offset(offset)`.
- `TableInteractionState::listener(&entity, callback)`.

Davranış:

- `focus_handle`, `scroll_handle`, `horizontal_scroll_handle` ve isteğe bağlı bir `custom_scrollbar` taşır.
- `.interactable(&state)` verilmedikçe tablo scroll ve focus state'ini bu entity'ye bağlamaz.
- Yatay scroll, tablo genişliği modeline bağlıdır. Sabit bir toplam genişlik veya `ResizableColumnsState` yoksa, tablo genellikle container'a sığacak şekilde davranır.
- `Table::pin_cols(...)` kullanılan bir resizable tabloda aynı `horizontal_scroll_handle`, header'ın ve her satırın scrollable bölümünü kilitli tutmak için kullanılır.
- `with_custom_scrollbar(...)`, Zed ayarlarından gelen scrollbar davranışını tabloya taşımak için tercih edilir.

Örnek:

```rust
use gpui::Entity;
use ui::{ScrollAxes, Scrollbars, Table, TableInteractionState, prelude::*};

struct AuditTable {
    table_state: Entity<TableInteractionState>,
}

impl AuditTable {
    fn new(cx: &mut Context<Self>) -> Self {
        let table_state = cx.new(|cx| {
            TableInteractionState::new(cx)
                .with_custom_scrollbar(Scrollbars::new(ScrollAxes::Both))
        });

        Self { table_state }
    }

    fn render_table(&self) -> impl IntoElement {
        Table::new(2)
            .interactable(&self.table_state)
            .header(vec!["Time", "Event"])
            .row(vec!["09:42", "Project opened"])
    }
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/keymap_editor/src/keymap_editor.rs`: custom scrollbar ile interactable bir keymap tablosu.
- `../zed/crates/csv_preview/src/csv_preview.rs`: CSV önizleme scroll state'i.
- `../zed/crates/git_graph/src/git_graph.rs`: tablo focus handle'ını selection davranışıyla birleştiren örnek.

Dikkat edilecek noktalar:

- `TableInteractionState` doğrudan bir struct alanı olarak değil, bir `Entity` içinde tutulmalıdır.
- Scroll offset elle set ediliyorsa, aynı frame içinde veri sayısının veya liste state'inin değişiklikleriyle çakışmamasına dikkat etmek gerekir.
- Focus davranışı gerekiyorsa, `focus_handle` alanı public olduğu için Zed'deki örnekler gibi `tab_index(...)` veya `tab_stop(...)` ile yapılandırılabilir.

## ColumnWidthConfig

Kaynak:

- Tanım: `../zed/crates/ui/src/components/data_table.rs`
- Export: `ui::ColumnWidthConfig`, `ui::StaticColumnWidths`.
- Prelude: Hayır; ayrıca import edilir.

Ne zaman kullanılır:

- Kolonların otomatik, oranlı, explicit veya kullanıcı tarafından yeniden boyutlandırılabilir olmasını seçmek için.
- Sanallaştırılmış bir tabloda yatay sizing davranışını doğru kurmak için.

Temel API:

- `ColumnWidthConfig::auto()`: kolonlar ve tablo otomatik olarak genişler.
- `ColumnWidthConfig::auto_with_table_width(width)`: `.width(width)` ile aynı davranış; tablo genişliği sabit, kolonlar ise otomatik şekillenir.
- `ColumnWidthConfig::explicit(widths)`: her kolon için explicit bir `DefiniteLength` alır.
- `ColumnWidthConfig::redistributable(columns_state)`: toplam alan korunarak kolonlar yeniden paylaştırılır.
- `ColumnWidthConfig::Resizable(columns_state)`: kolonlar mutlak bir genişlik taşır ve tablo toplam genişliği kolon toplamına göre değişir.

Explicit genişlik örneği:

```rust
use ui::{ColumnWidthConfig, Table, prelude::*};

fn render_explicit_width_table() -> impl IntoElement {
    Table::new(3)
        .width_config(ColumnWidthConfig::explicit(vec![
            DefiniteLength::Absolute(AbsoluteLength::Pixels(px(96.))),
            DefiniteLength::Fraction(0.35),
            DefiniteLength::Fraction(0.65),
        ]))
        .header(vec!["Kind", "Name", "Path"])
        .row(vec!["File", "main.rs", "crates/app/src/main.rs"])
}
```

Dikkat edilecek noktalar:

- `.width(width)` aslında `ColumnWidthConfig::auto_with_table_width(width)` ifadesinin kısaltmasıdır. Resize gerekiyorsa `.width_config(...)` doğrudan kullanılır.
- `explicit(widths)` içinde `widths.len()` ile tablo kolon sayısının birbirine eşit olması gerekir.
- `Resizable` için associated bir constructor yoktur; enum variant'ı doğrudan `ColumnWidthConfig::Resizable(entity)` biçiminde kullanılır.
- `Table::pin_cols(n)` yalnızca `ColumnWidthConfig::Resizable(entity)` ile anlamlıdır. `Auto`, `Explicit` ve `Redistributable` modlarında pinned split layout devreye girmez.
- Pinned layout'ta pinned bölümün resize divider'ları yalnızca görsel çizgi olarak render edilir; drag etkileşimi scrollable bölümün divider'larında kalır. Header hücresine çift tıklama ile kolon reset davranışı ise `HeaderResizeInfo` üzerinden çalışmaya devam eder.

## RedistributableColumnsState

Kaynak:

- Tanım: `../zed/crates/ui/src/components/redistributable_columns.rs`
- Export: `ui::RedistributableColumnsState`.
- İlgili tipler: `ui::TableResizeBehavior`, `ui::HeaderResizeInfo`.
- Prelude: Hayır; ayrıca import edilir.

Ne zaman kullanılır:

- Tablo container genişliği korunacak ve kullanıcı yalnızca kolonların birbirine göre oranını değiştirebilecekse.
- Keymap editor ve git graph gibi tabloda toplam alanın sabit kalması gereken yerlerde.
- Aynı tabloda hem oranlı hem mutlak başlangıç genişliklerinin kullanılması gerekiyorsa.

Ne zaman kullanılmaz:

- CSV veya spreadsheet benzeri bir tabloda kullanıcı tek bir kolonu genişletince toplam tablo genişliği de büyümeliyse, `ResizableColumnsState` çok daha uygundur.
- Yalnızca sabit oranlı kolon gerekiyorsa `ColumnWidthConfig::explicit(...)` çok daha basit bir çözümdür.

Temel API:

- `RedistributableColumnsState::new(cols, initial_widths, resize_behavior)`.
- `.cols()`.
- `.initial_widths()`.
- `.preview_widths()`.
- `.resize_behavior()`.
- `.widths_to_render()`.
- `.preview_fractions(rem_size)`.
- `.preview_column_width(column_index, window)`.
- `.cached_container_width()`.
- `.set_cached_container_width(width)`.
- `.commit_preview()`.
- `.reset_column_to_initial_width(column_index, window)`.

Davranış:

- Başlangıç genişlikleri `DefiniteLength` alır. Aynı tabloda `DefiniteLength::Fraction(...)` ile `DefiniteLength::Absolute(...)` birlikte kullanılabilir.
- Drag sırasında `preview_widths` güncellenir; drop sonrasında `commit_preview()` çağrısıyla kalıcı genişliklere aktarılır.
- `Table` içinde `.interactable(...)` ve `.width_config(ColumnWidthConfig::redistributable(...))` birlikte kullanıldığında resize handle binding'i normal tablo için otomatik olarak yapılır.
- `TableResizeBehavior::None`, ilgili divider yönünde resize yayılımını engeller.
- `TableResizeBehavior::Resizable`, varsayılan minimum sınırla resize'a izin verir.
- `TableResizeBehavior::MinSize(value)`, redistributable algoritmasında minimum kolon oranı olarak değerlendirilir.

Örnek:

```rust
use gpui::Entity;
use ui::{
    ColumnWidthConfig, RedistributableColumnsState, Table, TableInteractionState,
    TableResizeBehavior, prelude::*,
};

struct KeyBindingTable {
    table_state: Entity<TableInteractionState>,
    columns: Entity<RedistributableColumnsState>,
}

impl KeyBindingTable {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            table_state: cx.new(|cx| TableInteractionState::new(cx)),
            columns: cx.new(|_cx| {
                RedistributableColumnsState::new(
                    4,
                    vec![
                        DefiniteLength::Absolute(AbsoluteLength::Pixels(px(36.))),
                        DefiniteLength::Fraction(0.42),
                        DefiniteLength::Fraction(0.28),
                        DefiniteLength::Fraction(0.30),
                    ],
                    vec![
                        TableResizeBehavior::None,
                        TableResizeBehavior::Resizable,
                        TableResizeBehavior::Resizable,
                        TableResizeBehavior::Resizable,
                    ],
                )
            }),
        }
    }

    fn render_table(&self) -> impl IntoElement {
        Table::new(4)
            .interactable(&self.table_state)
            .width_config(ColumnWidthConfig::redistributable(self.columns.clone()))
            .header(vec!["", "Action", "Keystrokes", "Context"])
            .empty_table_callback(|_, _| Label::new("No keybindings").into_any_element())
    }
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/keymap_editor/src/keymap_editor.rs`: oranlı kolonlar ile resize edilebilir bir keybinding tablosu.
- `../zed/crates/git_graph/src/git_graph.rs`: graph alanı ve commit tablosu aynı redistributable state ile hizalanır.

Dikkat edilecek noktalar:

- `cols`, `initial_widths.len()` ve `resize_behavior.len()` değerlerinin birbirine eşit olması gerekir.
- Normal bir `Table` kullanımında `bind_redistributable_columns(...)` ve `render_redistributable_columns_resize_handles(...)` çağırmaya gerek yoktur; `Table` bunu kendi wrapper'ında zaten yapar.
- Aynı kolon state'i farklı görsel bölgelerde paylaşılıyorsa, düşük seviyeli helper'ları kullanmak gerekir.

## ResizableColumnsState

Kaynak:

- Tanım: `../zed/crates/ui/src/components/data_table.rs`
- Export: `ui::ResizableColumnsState`.
- İlgili tipler: `ui::TableResizeBehavior`.
- Prelude: Hayır; ayrıca import edilir.

Ne zaman kullanılır:

- Her kolonun mutlak genişliği ayrı ayrı değişecekse.
- Kullanıcı bir kolonu büyüttüğünde tablonun toplam genişliği büyümeli ve yatay scroll devreye girmeliyse.
- CSV, spreadsheet veya geniş veri önizlemeleri gibi senaryolar için.

Temel API:

- `ResizableColumnsState::new(cols, initial_widths, resize_behavior)`.
- `.cols()`.
- `.resize_behavior()`.
- `.set_column_configuration(col_idx, width, resize_behavior)`.
- `.reset_column_to_initial_width(col_idx)`.

Davranış:

- Başlangıç genişlikleri `AbsoluteLength` alır.
- Resize edilen kolonun genişliği değişir; komşu kolonlardan oran çalınmaz.
- `ColumnWidthConfig::Resizable(entity)` ile tablo toplam genişliğini kolon genişliklerinin toplamı üzerinden hesaplar.
- `TableResizeBehavior::MinSize(value)` değeri, resizable algoritmasında rem tabanlı bir minimum eşik olarak uygulanır.

Örnek:

```rust
use gpui::Entity;
use ui::{
    ColumnWidthConfig, ResizableColumnsState, Table, TableInteractionState,
    TableResizeBehavior, prelude::*,
};

struct CsvLikeTable {
    table_state: Entity<TableInteractionState>,
    columns: Entity<ResizableColumnsState>,
}

impl CsvLikeTable {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            table_state: cx.new(|cx| TableInteractionState::new(cx)),
            columns: cx.new(|_cx| {
                ResizableColumnsState::new(
                    3,
                    vec![
                        AbsoluteLength::Pixels(px(56.)),
                        AbsoluteLength::Pixels(px(180.)),
                        AbsoluteLength::Pixels(px(320.)),
                    ],
                    vec![
                        TableResizeBehavior::None,
                        TableResizeBehavior::Resizable,
                        TableResizeBehavior::MinSize(8.),
                    ],
                )
            }),
        }
    }

    fn render_table(&self) -> impl IntoElement {
        Table::new(3)
            .interactable(&self.table_state)
            .width_config(ColumnWidthConfig::Resizable(self.columns.clone()))
            .pin_cols(1)
            .header(vec!["#", "Name", "Value"])
            .row(vec!["1", "language", "Rust"])
    }
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/csv_preview/src/csv_preview.rs`: CSV kolon state'i `ResizableColumnsState` ile tutulur.
- `../zed/crates/csv_preview/src/renderer/render_table.rs`: tablo `ColumnWidthConfig::Resizable(...)` ile render edilir.

Dikkat edilecek noktalar:

- Bu model yatay scroll üretebilir; bu yüzden tablonun `.interactable(...)` ile bağlanması gerekir.
- Kolon sayısı değişirse, eski state'i güncellemek yerine yeni bir `ResizableColumnsState` oluşturmak çok daha net bir tercih olur.
- `set_column_configuration(...)`, runtime'da tek bir kolonun başlangıç ve mevcut genişliğini birlikte günceller.
- İlk kolonun row number veya seçim sütunu gibi her zaman görünür kalması gerekiyorsa, `ColumnWidthConfig::Resizable(entity)` ile birlikte `Table::pin_cols(n)` kullanılır. Zed CSV preview ilk kolonu bu şekilde sabitler. Kullanıcı pinned bölümdeki divider'ı sürükleyemez; boyut değiştirme ihtiyacı scrollable kolonlarda beklenir veya kolon konfigürasyonu state üzerinden güncellenir.

## TableRow ve UncheckedTableRow

Kaynak:

- Tanım: `../zed/crates/ui/src/components/data_table/table_row.rs`
- Export: `ui::table_row::TableRow`.
- Alias: `ui::UncheckedTableRow<T> = Vec<T>`.
- Prelude: Hayır.

Ne zaman kullanılır:

- Düşük seviye tablo helper'larına doğrulanmış bir satır verilmesi gerektiğinde.
- Kolon sayısı invariant'ının tek bir noktada kontrol edilmesi istendiğinde.
- Tablo dışındaki veri motorlarında satırların rectangular biçimde tutulması gerektiğinde.

Temel API:

- `TableRow::from_vec(data, expected_length)`.
- `TableRow::try_from_vec(data, expected_length)`.
- `TableRow::from_element(element, length)`.
- `.cols()`.
- `.get(col)`, `.expect_get(col)`.
- `.as_slice()`, `.into_vec()`.
- `.map(...)`, `.map_ref(...)`, `.map_cloned(...)`.

Davranış:

- `from_vec(...)` uzunluk eşleşmezse panic üretir.
- `try_from_vec(...)`, uzunluk hatasını `Result::Err` olarak döndürür.
- `Table::header(...)`, `Table::row(...)`, `.uniform_list(...)` ve `.variable_row_height_list(...)` public API'sinde `Vec<T>` kabul eder; `TableRow` dönüşümü içeride yapılır.
- `IntoTableRow` trait'i, `Vec<T>` için tek bir `.into_table_row(expected_length)` yöntemi sağlar; uzunluk eşleşmezse panic üretir. Kaynakta `Table` bunu içeride kullandığı için normal kullanımda import etmeye gerek yoktur. Düşük seviyeli helper'lara inildiğinde ise `use ui::table_row::IntoTableRow as _;` ifadesiyle çağrılabilir. Doğrulanmış (`Result` döndüren) bir dönüşüm için ise doğrudan `TableRow::try_from_vec(data, expected_length)` kullanılır; `try_into_table_row` adlı bir trait yöntemi yoktur.

Örnek:

```rust
use ui::{AnyElement, table_row::TableRow};

fn checked_cells(cells: Vec<AnyElement>, cols: usize) -> Option<TableRow<AnyElement>> {
    TableRow::try_from_vec(cells, cols).ok()
}
```

Dikkat edilecek noktalar:

- Normal `Table` kullanımı sırasında `TableRow` üretmeye gerek yoktur.
- `expect_get(...)`, veri motoru invariant'ı bozulduğunda erken hata vermek için uygundur; kullanıcı girdisinden gelen satırlarda `get(...)` daha güvenli bir seçim olur.

## Düşük Seviye Resize ve Render Helper'ları

Kaynak:

- `render_table_row`: `../zed/crates/ui/src/components/data_table.rs`
- `render_table_header`: `../zed/crates/ui/src/components/data_table.rs`
- `TableRenderContext`: `../zed/crates/ui/src/components/data_table.rs`
- `HeaderResizeInfo`: `../zed/crates/ui/src/components/redistributable_columns.rs`
- `bind_redistributable_columns`: `../zed/crates/ui/src/components/redistributable_columns.rs`
- `render_redistributable_columns_resize_handles`: `../zed/crates/ui/src/components/redistributable_columns.rs`

Ne zaman kullanılır:

- Tek bir `Table` yeterli değilse; örneğin header, graph alanı ve tablo gövdesi farklı container'larda ama aynı kolon state'iyle hizalanacaksa.
- Resize handle'larının tablo dışındaki sibling elemanların üzerine bind edilmesi gerekiyorsa.
- Satır veya header render'ının `Table` dışındaki özel bir layout içinde yeniden kullanılması gerektiğinde.

Ne zaman kullanılmaz:

- Normal bir veri tablosu için bu helper'lara inmeye gerek yoktur. `Table` zaten header, row, scroll ve resize binding'ini tek bir yerde yönetir.
- Sadece genişlik ayarlamak için `bind_redistributable_columns(...)` çağrılmaz; `ColumnWidthConfig` bu amaç için yeterlidir.

Temel API:

- `TableRenderContext::for_column_widths(column_widths, use_ui_font)`:
  - `column_widths`: `Option<TableRow<Length>>`. `None` verildiğinde hücreler sabit bir genişlik almaz. Redistributable veya resizable bir state üzerinden geliyorsa `columns_state.read(cx).widths_to_render()` çağrısıyla beslenir.
  - `use_ui_font`: `true` olduğunda hücre içeriği `text_ui(cx)` ile çizilir; `false` olduğunda font ailesi parent'tan miras alınır. `Table::no_ui_font()` ile kapatılan davranışla aynıdır. CSV preview monospace bir görünüm için bu değeri `false` verir.
  - `striped`, `show_row_borders`, `show_row_hover`, `total_row_count`, `disable_base_cell_style`, `map_row`, `pinned_cols` ve `h_scroll_handle` alanları, `Default::default()` benzeri varsayılanlarla doldurulur. Özel bir görünüm gerekiyorsa `for_column_widths(...)` çıktısı alan alan değiştirilebilir.
- `render_table_header(headers, table_context, resize_info, entity_id, cx) -> AnyElement`.
- `render_table_row(row_index, items, table_context, window, cx)`.
- `HeaderResizeInfo::from_redistributable(&columns_state, cx)`.
- `HeaderResizeInfo::from_resizable(&columns_state, cx)`.
  - `resize_behavior: TableRow<TableResizeBehavior>` public alanı, header hücresinin resizable olup olmadığını okumak için kullanılır. İlgili kolon state'i public bir alan değildir; reset ve state update için `reset_column(...)` çağrılır.
- `bind_redistributable_columns(container, columns_state)`.
- `render_redistributable_columns_resize_handles(&columns_state, window, cx)`.

Örnek:

```rust
use gpui::Entity;
use ui::{
    HeaderResizeInfo, RedistributableColumnsState, TableRenderContext,
    bind_redistributable_columns, render_redistributable_columns_resize_handles,
    render_table_header, table_row::TableRow, prelude::*,
};

fn render_custom_table_header(
    columns: &Entity<RedistributableColumnsState>,
    window: &mut Window,
    cx: &mut App,
) -> impl IntoElement {
    let widths = columns.read(cx).widths_to_render();
    let context = TableRenderContext::for_column_widths(Some(widths), true);
    let resize_info = HeaderResizeInfo::from_redistributable(columns, cx);

    bind_redistributable_columns(
        div()
            .relative()
            .child(render_table_header(
                TableRow::from_vec(
                    vec![
                        Label::new("Graph").into_any_element(),
                        Label::new("Description").into_any_element(),
                        Label::new("Author").into_any_element(),
                    ],
                    3,
                ),
                context,
                Some(resize_info),
                Some(columns.entity_id()),
                cx,
            ))
            .child(render_redistributable_columns_resize_handles(columns, window, cx)),
        columns.clone(),
    )
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/git_graph/src/git_graph.rs`: graph canvas ve commit tablosu aynı redistributable kolon state'iyle hizalanır; header ve resize handle'ları düşük seviyeli helper'larla kurulur.

Dikkat edilecek noktalar:

- `bind_redistributable_columns(...)`, drag move sırasında preview width'i günceller ve drop sırasında commit eder.
- `render_redistributable_columns_resize_handles(...)`, kolon state'inden divider'ları üretir; container'ın `relative()` olması handle yerleşimini çok daha öngörülebilir hâle getirir.
- `render_table_header(...)` içinde çift tıklama ile kolon reset davranışı `HeaderResizeInfo` üzerinden bağlanır.
- Header ve row için aynı `TableRenderContext` genişlik modelinin kullanılması gerekir; aksi halde hücreler hizalanmaz.
- Pinned kolonlu özel bir render akışı kuruluyorsa, `TableRenderContext.pinned_cols` ile `TableRenderContext.h_scroll_handle` birlikte ayarlanmalıdır. Normal `Table::pin_cols(...)` kullanımı bu iki alanı kendi içinde zaten doldurur. Scrollable satır ve header bölümleri `overflow_x_scroll()` ile aynı handle'ı takip eder; ayrıca resize drag koordinatı bu yatay offset'e göre düzeltilir.

## Veri Tablosu Kompozisyon Örnekleri

Boş durumlu küçük bir tabloda `empty_table_callback(...)`, hiç satır yokken kullanıcıya açıklayıcı bir mesaj gösterir:

```rust
use ui::{Table, prelude::*};

fn render_empty_jobs_table() -> impl IntoElement {
    Table::new(3)
        .width(px(560.))
        .header(vec!["Job", "Status", "Duration"])
        .empty_table_callback(|_, _| {
            v_flex()
                .p_3()
                .gap_1()
                .child(Label::new("No jobs").color(Color::Muted))
                .child(Label::new("Queued jobs will appear here").size(LabelSize::Small))
                .into_any_element()
        })
}
```

Satır seçimi için `map_row(...)` ile satır container'ının üzerinde özel bir görsel state uygulanabilir. Aşağıdaki örnek seçili satıra arka plan rengi atar:

```rust
use ui::{Table, prelude::*};

fn render_selectable_rows(selected_index: Option<usize>) -> impl IntoElement {
    Table::new(2)
        .header(vec!["Name", "Role"])
        .row(vec!["Ada", "Admin"])
        .row(vec!["Linus", "Maintainer"])
        .map_row(move |(index, row), _window, cx| {
            row.when(selected_index == Some(index), |row| {
                row.bg(cx.theme().colors().element_selected)
            })
            .into_any_element()
        })
}
```

Karar rehberi olarak şu kısa özet işe yarar:

- Az satır ve basit bir görünüm gerekiyorsa `Table::new(...).header(...).row(...)` yeterlidir.
- Çok satır ama tek satır yüksekliği varsa `.uniform_list(...)` doğru tercihtir.
- Çok satır ve multiline veya değişken içerik varsa `.variable_row_height_list(...)` kullanılır.
- Container genişliği sabit kalmalı ama kolon oranları değişmeliyse `RedistributableColumnsState` devreye girer.
- Kolonlar mutlak genişlikte olacak ve yatay scroll oluşabilecekse `ResizableColumnsState` seçilir.
- Header, gövde ve ek görsel bölgeler aynı kolon state'ini paylaşacaksa düşük seviyeli render ve resize helper'larına inilir.
