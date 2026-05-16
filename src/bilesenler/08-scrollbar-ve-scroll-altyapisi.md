# 8. Scrollbar ve Scroll Altyapısı

Scrollbar bileşeni, panel, liste, modal ve özel scroll container'larında Zed temasıyla uyumlu bir scroll geri bildirimi sağlar. Layout primitive'lerinden ayrı anlatılmasının nedeni şudur: scrollbar yalnızca çizilen bir görsel parça değildir. Tema ayarına, scrollbar görünürlüğü tercihine ve `ScrollableHandle` ile kurduğu bağlantıya göre davranır. Bu yüzden basit bir layout helper'ından daha fazla sorumluluk taşır.

Kaynak:

- Tanım: `../zed/crates/ui/src/components/scrollbar.rs`
- Export: `ui::Scrollbars`, `ui::ScrollAxes`, `ui::ScrollbarStyle`, `ui::scrollbars::{ShowScrollbar, ScrollbarAutoHide, ScrollbarVisibility}`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: Doğrudan bir component preview yok; gerçek kullanım örnekleri panel, modal ve tablo kompozisyonlarının içinde görünür.

Ne zaman kullanılır:

- Bir scroll container'a, Zed tema renkleriyle uyumlu bir scrollbar bağlamak için.
- Tablo, panel veya picker gibi içeriklerde, tema scrollbar ayarına saygı duyan otomatik gösterim ve gizleme davranışı gerektiğinde.
- Yatay, dikey veya iki yönlü scroll track'inin tek bir API üzerinden yönetilmesi gerektiğinde.

Ne zaman kullanılmaz:

- Doğal scroll davranışı yeterliyse `overflow_y_scroll()` veya `overflow_x_scroll()` ile basit bir container kullanılır. `Scrollbars`, tema ile hizalanmış özel bir scrollbar yüzeyi gerektiğinde devreye girer; her scroll alanına eklenmesi gerekmez.

Temel API:

- `Scrollbars::new(show_along: ScrollAxes)`.
- `Scrollbars::always_visible(show_along)`.
- `Scrollbars::for_settings::<S: ScrollbarVisibility>()`.
- `.id(ElementId)`, `.notify_content()`, `.tracked_entity(EntityId)`, `.tracked_scroll_handle(handle)`, `.show_along(axes)`, `.style(style)`, `.with_track_along(axes, bg)`, `.with_stable_track_along(axes, bg)`.
- `WithScrollbar`: elementlere `.vertical_scrollbar_for(...)` ve `.custom_scrollbars(...)` metodlarını ekleyen bir extension trait'tir. Kaynakta yatay ve çift yönlü helper taslakları yorum satırı olarak durur; bu helper'lar şu anda public API değildir.
- `ScrollableHandle`: kendi handle'ınızı yazıyorsanız, `max_offset`, `set_offset`, `offset`, `viewport` ve opsiyonel olarak drag hook'larını sağlamanız gerekir. Bu sözleşme, `Scrollbars`'ın handle ile nasıl konuşacağını belirler.
- `on_new_scrollbars::<S>(cx)`: scrollbar global setting tipinin değiştiği durumlarda, yeni `ScrollbarState` entity'lerinin bu ayar değişikliklerine abone olmasını sağlayan setup helper'ıdır.
- `ScrollAxes`: `Horizontal`, `Vertical`, `Both`.
- `ScrollbarStyle`: `Regular` (6px), `Editor` (15px).
- `ShowScrollbar`: `Auto`, `System`, `Always`, `Never`.

Davranış:

- `Scrollbars::new(ScrollAxes::Vertical)` varsayılan olarak tema scrollbar ayarına bağlı şekilde görünür. `always_visible(...)` ayarı ise bu tercihi yok sayar ve scrollbar'ı her zaman çizer.
- `tracked_scroll_handle(...)` ile harici bir `ScrollableHandle` (örneğin bir `ScrollHandle` veya `UniformListScrollHandle`) bağlanır; scrollbar bu handle üzerinden okuma ve yazma yapar.
- `Table::interactable(...)` ile `TableInteractionState::with_custom_scrollbar(...)` birlikte kullanıldığında `Scrollbars`, tablonun yatay ve dikey scroll handle'larına bağlanır; yani tablo kendi scroll davranışını dış bir scrollbar üzerinden sürer.
- `ScrollbarStyle::Editor`, editor görseliyle gelen scrollbar genişliğinde çizim yapar; panel ve listelerde ise `Regular` daha uygundur.
- `ListState` tabanlı variable-height listelerde scrollbar drag state'i `scrollbar_drag_started()`, `scrollbar_drag_ended()` ve `is_scrollbar_dragging()` ile izlenir. `set_offset_from_scrollbar(point)` çağrısında aşağı yöndeki scroll negatif `y` offset'iyle temsil edilir; `point(px(0.), px(-150.))` içeriğin 150px aşağı kaydırıldığı durumu ifade eder.
- Drag sırasında liste içeriği büyürse scrollbar konumu drag başlangıcındaki içerik yüksekliğine göre korunur. Kullanıcı drag'i frozen track'in sonuna getirirse tail-follow yeniden etkinleşir; bu özellikle log, terminal ve agent conversation gibi sona akması gereken listelerde elle scroll ile otomatik takip arasındaki ayrımı korur.

Örnek:

```rust
use gpui::ScrollHandle;
use ui::prelude::*;
use ui::{ScrollAxes, Scrollbars};

struct LogPanel {
    scroll: ScrollHandle,
}

impl Render for LogPanel {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        v_flex()
            .size_full()
            .child(
                div()
                    .id("log-body")
                    .flex_1()
                    .overflow_y_scroll()
                    .track_scroll(&self.scroll)
                    .child(Label::new("…")),
            )
            .child(
                Scrollbars::new(ScrollAxes::Vertical)
                    .tracked_scroll_handle(self.scroll.clone()),
            )
    }
}
```

Dikkat edilecek noktalar:

- `Scrollbars` kendi başına içerik kaydırmaz. İçeriğin gerçekten scroll olabilmesi için bir `ScrollableHandle` ile bağlanması veya bir `ScrollHandle::new()` üzerinden takip edilmesi gerekir.
- Tek bir scroll container'a doğrudan bağlanılıyorsa `WithScrollbar` helper'larını kullanmak, ayrı bir `Scrollbars` child'ı yazmaya göre hem daha kısa hem de daha az hataya açıktır.
- `with_stable_track_along(...)`, scroll alanı henüz yokken bile track için yer ayırır. Bu sayede scrollbar görünür ya da gizli olarak değiştiğinde layout aniden zıplamaz; sahne stabil kalır.
- Birden fazla scroll alanı bulunuyorsa, her birine `.id(...)` üzerinden benzersiz bir id verilmesi gerekir; aksi halde GPUI scroll state'leri karıştırabilir.
- `set_offset_from_scrollbar(...)` için pozitif offset kullanımı güncel sözleşmeye uymaz. Scrollbar handle yazarken `offset()` ve `set_offset(...)` değerlerinin aynı işaret yönünü kullandığından emin olunmalıdır.
