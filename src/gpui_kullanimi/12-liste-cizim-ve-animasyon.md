# Liste, Çizim ve Animasyon

---

## ScrollHandle ve Scroll Davranışı

`crates/gpui/src/elements/div.rs:3387+`.

`ScrollHandle`, scroll offset'ini paylaşılabilir bir handle olarak tutar. `Rc<RefCell<ScrollHandleState>>` üzerinden çalışır ve view'lar arasında ucuz biçimde klonlanabilir. Aynı handle birden fazla yerden okunup değiştirilebilir.

**Genel API.** Handle üzerinden ulaşılabilen başlıca metotlar şunlardır:

- `ScrollHandle::new()` — yeni handle.
- `offset() -> Point<Pixels>` — anlık scroll konumu.
- `max_offset() -> Point<Pixels>` — alınabilecek azami offset.
- `top_item()`, `bottom_item()` — görünür ilk ve son alt öğe dizini.
- `bounds()` — scroll kapsayıcısının sınırları.
- `bounds_for_item(ix)` — verilen alt öğenin sınırları.
- `scroll_to_item(ix)`, `scroll_to_top_of_item(ix)` — prepaint zamanında istenen öğeye scroll eder.
- `scroll_to_bottom()`
- `set_offset(point)` — offset'i doğrudan ayarlar. Offset, içerik orijininin üst öğe orijinine uzaklığıdır; aşağı kaydıkça Y genelde negatife gider.
- `logical_scroll_top()`, `logical_scroll_bottom()` — görünür alt öğe index'ini ve alt öğe içi piksel offset'ini döndürür.
- `children_count()` — scroll edilen alt öğe sayısı.

**Element üzerine bağlama.** Bir handle'ı div'e iliştirmek için stateful scroll API'leri kullanılır:

```rust
let handle = ScrollHandle::new();

div()
    .id("list")
    .overflow_y_scroll()
    .track_scroll(&handle)
    .child(/* ... */)
```

`overflow_scroll`, `overflow_x_scroll` ve `overflow_y_scroll` `StatefulInteractiveElement` metotlarıdır; pratikte önce `.id(...)` çağrısı yapılarak `Stateful<Div>` üretilir. Overflow `Scroll` olduğunda girdi `wheel` veya touch olayı bu kapsayıcı içinde tüketilir. `track_scroll` aynı handle'ı çizim geçişleri arasında bağlar; bu sayede handle başka yerden de okunup değiştirilebilir.

`ScrollAnchor` (`div.rs:3332+`) bir handle ile çalışan yardımcıdır. Ağaçta doğrudan alt öğe olmayan bir elementin görünür kalmasını ister:

```rust
let anchor = ScrollAnchor::for_handle(handle.clone());
anchor.scroll_to(window, cx);
```

**Tuzaklar.** Scroll handle ile çalışırken atlanan noktalar:

- `id(...)` çağırılmadan `overflow_*_scroll` çalışmaz; element etkileşimli olmaz.
- `track_scroll` çağırılmadığında handle değerleri güncel kalmaz; offset doğru okunmaz.
- Klavye ile scroll göndermek için `.on_key_down(...)` veya action kullanılarak `scroll_to_item` çağrılır; otomatik klavye scroll'u yoktur.

**Liste elementlerine özel durum/handle tipleri.** Büyük listelerde `ScrollHandle` yerine bu tiplere geçilir:

- `ListState` — `scroll_by`, `scroll_to`, `scroll_to_reveal_item`, `scroll_to_end`, `set_follow_mode(FollowMode::Tail)`, `logical_scroll_top`, `is_scrolled_to_end`, `is_following_tail`.
- `UniformListScrollHandle` — `scroll_to_item(..., ScrollStrategy)`, `scroll_to_item_strict`, `scroll_to_item_with_offset`, `scroll_to_item_strict_with_offset`, `logical_scroll_top_index`, `is_scrolled_to_end`, `scroll_to_bottom`.

## List ve UniformList Sanallaştırma

GPUI'de büyük listeler için iki çekirdek element vardır. İkisi farklı liste ihtiyaçları için tasarlanmıştır:

- `list(state, render_item)` — item yükseklikleri değişebilir. Ölçüm önbelleği `ListState` içinde tutulur.
- `uniform_list(id, item_count, render_range)` — tüm item'lar aynı yükseklikte olduğunda daha hızlıdır; ilk veya örnek item ölçülür ve yalnız görünür aralık çizilir.

**Değişken yükseklikli liste.** Durum view içinde tutulur ve veri seti değiştiğinde uygun yardımcılar çağrılır:

```rust
struct LogView {
    rows: Vec<Row>,
    list_state: ListState,
}

impl LogView {
    fn new() -> Self {
        Self {
            rows: Vec::new(),
            list_state: ListState::new(0, ListAlignment::Top, px(300.)),
        }
    }

    fn replace_rows(&mut self, rows: Vec<Row>, cx: &mut Context<Self>) {
        self.rows = rows;
        self.list_state.reset(self.rows.len());
        cx.notify();
    }
}
```

Çizim aşamasında item builder closure'u verilir:

```rust
list(self.list_state.clone(), |ix, window, cx| {
    render_row(ix, window, cx).into_any_element()
})
.with_sizing_behavior(ListSizingBehavior::Auto)
```

**`ListState` yönetimi.** Liste durumunu tutan tipin yüzeyi geniştir. Her metot ayrı bir senaryoya karşılık gelir:

- `new(item_count, alignment, overdraw)` — yapıcı.
- `measure_all()` (consuming) — `ListMeasuringBehavior::Measure(false)` ayarlayarak scrollbar boyutunun yalnız çizilmiş elementlere değil **tüm liste** ölçümüne dayanmasını sağlar.
- `item_count() -> usize` — o anki item sayısı.
- `reset(count)` — tüm item kümesi değişti.
- `splice(old_range, count)` — aralık değişti; scroll offset korunur.
- `splice_focusable(old_range, focus_handles)` — odaklanabilir item'lar sanallaştırılırken focus handle dizisi geçilir; aksi halde görünür olmayan odaktaki item çizim dışı kalabilir.
- `remeasure()` — font veya tema gibi tüm yükseklikleri etkileyen değişim.
- `remeasure_items(range)` — akışlı metin veya tembel içerik gibi belirli item'lar.
- `set_follow_mode(FollowMode::Tail)` — sohbet veya günlük gibi tail-follow davranışı.
- `is_following_tail() -> bool` — aktif takip durumu.
- `is_scrolled_to_end() -> Option<bool>` — en alta scroll mu? Henüz yerleşim yapılmamışsa `None`.
- `scroll_by(distance)`, `scroll_to_end()`, `scroll_to(ListOffset)`, `scroll_to_reveal_item(ix)`.
- `logical_scroll_top() -> ListOffset` — aktif scroll konumu.
- `bounds_for_item(ix) -> Option<Bounds<Pixels>>` — çizilmişse item rect'i.
- `set_scroll_handler(...)` — görünür aralık ve takip durumu izleme.

**Özel scrollbar API'si.** Kendi scrollbar widget'ı yazıldığında bu metotlar üzerinden çalışılır (`ui::Scrollbars` da aslında bu yüzeyin üstünde kuruludur):

- `viewport_bounds() -> Bounds<Pixels>` — en son yerleşim yapılmış viewport rect'i.
- `scroll_px_offset_for_scrollbar() -> Point<Pixels>` — scrollbar için uyarlanmış güncel scroll konumu.
- `max_offset_for_scrollbar() -> Point<Pixels>` — ölçülmüş item'lara göre maksimum scroll. Sürükleme sırasında bu değer sabit kalır; böylece scrollbar sıçramaz.
- `set_offset_from_scrollbar(point)` — scrollbar sürüklemesinden veya tıklamasından gelen offset'i uygular.
- `scrollbar_drag_started()` / `scrollbar_drag_ended()` — sürükleme sırasında overdraw ölçümünden kaynaklı yükseklik dalgalanmasını dondurmak veya serbest bırakmak için. Sürüklemeye girerken `started`, bırakırken `ended` çağrılmazsa scrollbar sürükleme boyunca beklenmedik biçimde kayabilir.
- `is_scrollbar_dragging() -> bool` — `scrollbar_drag_started` ile `scrollbar_drag_ended` arasındaki elle sürükleme durumunu okur. Wheel veya trackpad scroll ile scrollbar thumb sürüklemesini ayırmak ve sürükleme sırasında otomatik tail-follow davranışını bastırmak için kullanılır.
- `set_offset_from_scrollbar(point)` tarafında scroll offset'inin Y bileşeni, içerik yukarı kaydıkça negatiftir; özel scrollbar yazılırken pozitif "başlangıçtan uzaklık" yerine `point(px(0.), -distance)` üretmek doğru sonucu verir. Sürükleme sırasında içerik büyürse başlangıçtaki içerik yüksekliği dondurulur; thumb donmuş alta sürüklenirse `FollowMode::Tail` tekrar `is_following = true` olur.

**Uniform liste.** Sabit yükseklikli item'larda virtualizasyon daha agresiftir:

```rust
let scroll_handle = self.scroll_handle.clone();

uniform_list("search-results", self.items.len(), move |range, window, cx| {
    range
        .map(|ix| render_result(ix, window, cx))
        .collect()
})
.track_scroll(&scroll_handle)
.with_width_from_item(Some(0))
```

`UniformListScrollHandle` üzerindeki metotlar listenin tipik scroll ihtiyaçlarını karşılar:

- `scroll_to_item(ix, ScrollStrategy::Nearest)`
- `scroll_to_item_strict(ix, ScrollStrategy::Center)`
- `scroll_to_item_with_offset(ix, strategy, offset)`
- `scroll_to_bottom()`
- `is_scrollable()`, `is_scrolled_to_end()`
- `y_flipped(true)` — item 0 altta olacak şekilde ters akış.

**Karar.** Hangi listenin seçileceği şu kurallara göre belirlenir:

- Item yükseklikleri gerçekten aynıysa `uniform_list` tercih edilir.
- Yükseklik değişebiliyorsa `list` ve doğru `splice`/`remeasure` çağrıları kullanılır.
- Odaklanabilir item'lar sanallaştırılıyorsa `splice_focusable` ile focus handle verilir; aksi halde görünür olmayan odaktaki item çizim dışı kalabilir.

---

## Asset, Image ve SVG Yükleme

`crates/gpui/src/asset_cache.rs`, `assets.rs`, `elements/img.rs`, `svg.rs`.

`Asset` trait'i asenkron yükleyici sözleşmesidir; her özel kaynak türü bu trait'i uygular:

```rust
trait Asset {
    type Source: ...;
    type Output: ...;
    fn load(source: Self::Source, cx: &mut App) -> impl Future<Output = Self::Output>;
}
```

**Kaynak gösterimi.** Asset adresi farklı yerlerden gelebilir. `Resource` enum'u bu seçenekleri toplar:

- `Resource::Path(Arc<Path>)`
- `Resource::Uri(SharedUri)` — `http://`, `https://`, `file://` vb.
- `Resource::Embedded(SharedString)` — `AssetSource` üzerinden gömülü asset.

`AssetSource` trait'i `App::with_assets` ile kurulan genel asset sağlayıcısıdır. `crates/assets` Zed binary'sinde `RustEmbed` ile SVG ve ikonları gömer. Kayıtlı kaynak gerektiğinde `cx.asset_source()` ile okunabilir; çoğu UI kodu bunu doğrudan değil `Resource::Embedded`, `svg().path(...)` veya `window.use_asset` üzerinden kullanır.

**Görsel elementi.** `img` çağrısı farklı kaynak tiplerini kabul eder ve yükleme/yedek slotları sunar:

```rust
img(PathBuf::from("path/to/icon.png"))
    .w(px(24.))
    .h(px(24.))
    .object_fit(ObjectFit::Contain)
    .with_loading(|| div().bg(rgb(0xeeeeee)).into_any_element())
    .with_fallback(|| div().bg(rgb(0xffeeee)).into_any_element())
```

`img(impl Into<ImageSource>)` argümanları şu şekillerde gelebilir:

- `Resource(Resource)`
- `Render(Arc<RenderImage>)` — önceden rasterize edilmiş kareler.
- `Image(Arc<Image>)` — kodlanmış byte'lar (PNG/JPEG/WebP).
- `Custom(Arc<dyn Fn(&mut Window, &mut App) -> Option<Result<Arc<RenderImage>, ImageCacheError>>>)`

URL biçimindeki string otomatik olarak `Uri` olarak ayrıştırılır. URL olmayan `&str` veya `String` `Resource::Embedded` sayılır ve `AssetSource` içinden aranır. Dosya sistemi path'i için `Path`, `PathBuf` veya `Arc<Path>` geçirilir.

**SVG.** Vektör çizimi için ayrı bir element vardır:

```rust
svg().path("icons/check.svg").size(px(16.)).text_color(rgb(0x000000))
```

SVG path'i `AssetSource`'tan okunur. `text_color` SVG içindeki `currentColor` referanslarının renklendirilmesinde kullanılır. Özel path string'i yerine türetilmiş `IconName::path()` de geçirilebilir (Zed'de `Icon::new(IconName::Check)` doğrudan kullanılır).

**Önbellek davranışı.** İki katmanlıdır: `window.use_asset::<A>(source, cx)` aynı kaynak için tek asenkron yükleme görevini paylaşır ve tamamlandığında o anki view'i yeniden çizdirir. `ImageCache` ise kodu çözülmüş `RenderImage`'ı tutar. Element bazında `.image_cache(&entity)` veya ağacın üstünde `image_cache(retain_all("id"))` kullanılabilir. Hata kaydı `ImgResourceLoader = AssetLogger<...>` ile otomatiktir.

**Tuzaklar.** Asset yüklerken karşılaşılabilecek durumlar:

- URL ayrıştırma başarısız olduğunda string embedded asset sayılır; gerçek dosya yolu için `PathBuf` kullanılmadığında yanlış kaynaktan arama yapılır.
- Özel closure `'static` olmalı; `Window` ve `App` yalnızca closure çağrısında parametre olarak kullanılmalıdır.
- `with_fallback` yalnız yükleme tamamlandığında ve hatalıysa yedeği çizer.
- `with_loading` yükleme 200 ms'den uzun sürerse yükleme yedeğini çizer. Bu eşik `gpui::LOADING_DELAY: Duration` sabitiyle tanımlıdır (`elements/img.rs:31`); benzer bir gecikmeli yedek akışında aynı değer ödünç alınabilir.
- `RenderImage` GIF veya animasyonlu WebP için `frame_count()` ve `delay(frame_index)` sağlar; `img` elementi aktif pencerede kare ilerletir ve animasyon karesi ister.
- Yol sınıflandırıcı veya ayarlar güvenlik kontrolü yazılırken macOS ve Windows'un varsayılan büyük/küçük harfe duyarsız dosya sistemlerinin atlanmaması gerekir. `util::paths::component_matches_ignore_ascii_case(component, ".zed")` gibi ASCII duyarsız yardımcılar tercih edilir; `.ZED/settings.json` gibi varyantlar düz `== ".zed"` karşılaştırmasıyla kaçırılmamalıdır.

## Asset, ImageCache ve Surface Boru Hattı

GPUI asset katmanı üç seviyelidir ve her seviye ayrı bir sorumluluk taşır:

- `AssetSource` — embedded veya statik asset byte'larını sağlar.
- `Asset` — asenkron yükleyici trait'i; `Source -> Output`.
- `Resource` — görsel veya SVG kaynak adresi: `Uri(SharedUri)`, `Path(Arc<Path>)`, `Embedded(SharedString)`.

**Görsel önbellek elementleri.** Bir alt ağaç için görsel önbelleğini çevrelemenin iki yolu vardır:

```rust
div()
    .image_cache(retain_all("avatars"))
    .child(img(avatar_uri.clone()).object_fit(ObjectFit::Cover))
```

Alternatif olarak sarmalayıcı element kullanılabilir:

```rust
image_cache(retain_all("preview-cache"))
    .child(img(preview_path.clone()))
```

**`RetainAllImageCache`.** Ana önbellek uygulaması birkaç metot sağlar:

- `RetainAllImageCache::new(cx)` entity önbelleği oluşturur.
- `retain_all(id)` element-yerel önbellek sağlayıcısı üretir.
- `load(resource, window, cx)` sonuç hazır değilse `None`, hazırsa `Some(Result<Arc<RenderImage>, ImageCacheError>)` döndürür.
- Bırakma sırasında `cx.drop_image(...)` ile GPU görsel kaynaklarını serbest bırakır.

**Özel önbellek.** Özel bir önbellek mantığı gerektiğinde `ImageCache` trait'i uygulanır:

```rust
impl ImageCache for MyImageCache {
    fn load(
        &mut self,
        resource: &Resource,
        window: &mut Window,
        cx: &mut App,
    ) -> Option<Result<Arc<RenderImage>, ImageCacheError>> {
        self.load_or_poll(resource, window, cx)
    }
}
```

**`Surface`.** Ayrı bir yol izler: macOS'ta `CVPixelBuffer` gibi platform surface kaynaklarını `surface(buffer).object_fit(...)` ile çizmek için kullanılır. Genel görsel asset önbelleği yerine `window.paint_surface(...)` boru hattını kullanır ve şu anda platforma bağlıdır.

**Tuzaklar.** Önbellek ve surface tarafında atlanan noktalar:

- Önbellek ID'si değişirse kodu çözülmüş görsel durumu düşer.
- `img("literal")` URL değilse embedded resource olarak yorumlanır; dosya sistemi için `PathBuf` veya `Arc<Path>` verilmelidir.
- Çok büyük veya sürekli değişen görsel kümelerinde `RetainAllImageCache` sınırsız büyüyebilir; özel tahliye stratejisi gerekiyorsa kendi `ImageCache` uygulaması yazılır.

## Path Çizimi ve Özel Çizim

`crates/gpui/src/path_builder.rs`, `scene.rs`, `elements/canvas.rs`.

GPUI doğrudan path API'si yerine `canvas` elementi ve `PathBuilder` ile vektör çizimi sunar. `PathBuilder`, lyon tessellator'ının ince bir sarmalayıcısıdır.

```rust
canvas(
    |bounds, window, _cx| {
        // prepaint: hitbox, yerleşim zamanlı durum
        window.insert_hitbox(bounds, HitboxBehavior::Normal)
    },
    |bounds, _hitbox, window, _cx| {
        // paint: window.paint_path(...) çağrıları
        let mut path = PathBuilder::fill();
        path.move_to(bounds.origin);
        path.line_to(bounds.bottom_left());
        path.line_to(bounds.bottom_right());
        path.close();
        if let Ok(built) = path.build() {
            window.paint_path(built, rgb(0x4f46e5));
        }
    },
)
.size_full()
```

**`PathBuilder`.** Path inşası adım adım fluent metotlarla yapılır:

- `PathBuilder::fill()` veya `PathBuilder::stroke(width)` ile başlatılır.
- `move_to(point)`, `line_to(point)`, `curve_to(to, ctrl)`, `cubic_bezier_to(to, control_a, control_b)`, `arc_to(radii, x_rotation, large_arc, sweep, to)`, `relative_arc_to(...)`, `add_polygon(...)`, `close()`.
- `dash_array(&[Pixels])` yalnızca stroke path'lerde anlamlıdır; tek sayıda değer verilirse SVG/CSS davranışındaki gibi liste iki kez tekrarlanır.
- `transform(...)`, `translate(point)`, `scale(f32)`, `rotate(degrees)` build öncesi path'i dönüştürür.
- `build()` → tessellate edilmiş `Path<Pixels>` döner; hata `?` ile yayılır.

**Tessellator parametreleri** (`PathStyle`, `FillOptions`, `StrokeOptions`, `FillRule`).

GPUI bu tipleri lyon'dan yeniden dışa aktarır (`pub use lyon::tessellation::{FillOptions, FillRule, StrokeOptions}`, `path_builder.rs:11-12`). `PathBuilder.style: PathStyle` alanı `pub` görünürlüktedir; iki varyantı vardır:

```rust
pub enum PathStyle {
    Stroke(StrokeOptions),
    Fill(FillOptions),
}
```

Varsayılan yapıcılar, lyon'un varsayılan parametrelerini ayarlar (`PathBuilder::fill()` → `FillOptions::default()`; `PathBuilder::stroke(width)` → `StrokeOptions::default().with_line_width(width.0)`). Bu seçeneklerin özelleştirilmesi gerektiğinde `path.style` alanı doğrudan değiştirilir veya path inşa edildikten sonra yeni `PathStyle` atanır.

- `FillOptions` — `tolerance` (düzleştirme hassasiyeti, varsayılan 0.1), `fill_rule` (`FillRule::{EvenOdd, NonZero}`, SVG `fill-rule` semantiği; varsayılan **`EvenOdd`** — `lyon_tessellation::FillOptions::DEFAULT_FILL_RULE`), `sweep_orientation` (varsayılan `Orientation::Vertical`), `handle_intersections` (varsayılan `true`). Hızlı yardımcılar: `FillOptions::even_odd()`, `FillOptions::non_zero()`, `FillOptions::tolerance(t)`.
- `StrokeOptions` — `line_width` (varsayılan 1.0), `start_cap` ve `end_cap` (her sub-path için başlangıç ve bitiş ucu, varsayılan `LineCap::Butt`), `line_join` (varsayılan `LineJoin::Miter`), `miter_limit` (varsayılan 4.0), `tolerance` (varsayılan 0.1). Tüm sabitler `lyon_tessellation::StrokeOptions::{DEFAULT_LINE_CAP, DEFAULT_LINE_JOIN, DEFAULT_MITER_LIMIT, DEFAULT_LINE_WIDTH, DEFAULT_TOLERANCE}` const'larında bulunur.
- `FillRule::EvenOdd` (lyon ve gpui varsayılanı): SVG even-odd kuralıdır; iç içe path'lerde delik üretir. İki üst üste binen kapalı path'in çakışan bölgesi şeffaf olur. `FillRule::NonZero`: SVG non-zero winding kuralıdır; yön birleşimine göre kapsama hesaplar, çakışan path'ler genelde dolu kalır. Karmaşık kompozit şekiller için bilinçli olarak `non_zero()` seçilir.

Lyon API'sine inilmek isteniyorsa `lyon::tessellation::FillOptions::tolerance(0.5)` gibi builder zincirleri kullanılabilir; GPUI bu builder'ları olduğu gibi yeniden dışa aktardığı için ayrı bir sarmalayıcıya ihtiyaç yoktur.

**Window paint API'leri.** Path veya quad GPU'ya iletilirken pencere üzerindeki paint metotları kullanılır:

- `window.paint_path(path, color)` — tessellate edilmiş path.
- `window.paint_quad(quad)` — `fill(bounds, ...).border(...)` kısaltması.
- `window.paint_strikethrough(...)`, `paint_underline(...)`
- `window.paint_image(...)` — raster görsel çizimi.
- `window.paint_layer(bounds, |window| ...)` — aynı çizim sırasında toplanan geometri için yeni bir katman açar; çoğunlukla performans ve overdraw kontrolü için kullanılır.

**Tuzaklar.** Path çizimde dikkat edilecek noktalar:

- Tessellation pahalıdır; her karede yeni path inşa etmek FPS'i düşürür. Mümkün olduğunda prepaint'te inşa edilip paint'te yalnızca çizilir.
- Path bounds dışına taşan kısım kırpılmaz; `paint_layer` ile elle kırpma uygulanır.
- Stroke genişliği mantıksal Pixels'dir; DPI'si yüksek ekranlarda çok ince kalmaması için `px(1.0).max(...)` ile zemin tutulur.

## Anchored ve Popover Konumlandırma

`crates/gpui/src/elements/anchored.rs`.

`anchored()` fonksiyonu bir `Anchored` builder döndürür. Popover, menü ve tooltip benzeri konumlandırmalar bu element üzerinde kurulur:

```rust
anchored()
    .anchor(Anchor::TopLeft)
    .position(point(px(120.), px(80.)))
    .offset(point(px(0.), px(4.)))
    .snap_to_window_with_margin(Edges::all(px(8.)))
    .child(menu_view.into_any_element())
```

**API.** Konumlandırmayı belirleyen başlıca metotlar şunlardır:

- `anchor(Anchor)` — alt öğenin hangi referans noktasının `position`'a hizalanacağı. `Anchor` varyantları `TopLeft`, `TopRight`, `BottomLeft`, `BottomRight`, `TopCenter`, `BottomCenter`, `LeftCenter`, `RightCenter`.
- `position(point)` — anchor noktası (window veya yerel koordinatlarda).
- `offset(point)` — hizalama sonrası ek kayma.
- `position_mode(AnchoredPositionMode::Window)` veya `position_mode(AnchoredPositionMode::Local)` — koordinat referansı.
- `snap_to_window()` ve `snap_to_window_with_margin(Edges)` — pencere dışına taşıyorsa aynı anchor'ı koruyarak pencere içine kaydırır.

**`AnchoredFitMode`.** Pencere kenarına yaklaşıldığında nasıl davranılacağı:

- `SwitchAnchor` (varsayılan) — yetersiz alanda ters anchor'a geçilir.
- `SnapToWindow` — aynı köşede kalır, pencere kenarına oturur.
- `SnapToWindowWithMargin(Edges)` — marjin bırakarak oturur.

Anchored element ağaca normal bir alt öğe gibi eklenir; ancak yerleşim fazında üst öğe sınırlarını yok sayar ve mutlak konumlandırma gibi davranır. Tooltip, popover ve ContextMenu altta bu element üzerinde çalışır.

**Tuzaklar.** Anchored kullanırken yapılan hatalar:

- `Local` modda position üst öğenin içerik orijinine görelidir; window modda ise ekranda mutlak değil, **pencere içi** koordinatlardır.
- Snap fonksiyonları arasında en son çağrılan kazanır.
- Anchored alt öğesi kendi içinde overflow `Visible` davranır; içerik pencereyi taşırsa scroll için ekstra bir sarmalayıcı gerekir.

## PaintQuad, Window Paint Primitives ve BorderStyle

`canvas` ve özel `Element::paint` içinde GPU'ya gönderilen primitive'ler şu çağrılarla üretilir:

```rust
window.paint_quad(fill(bounds, rgb(0xeeeeee)));

window.paint_quad(
    quad(
        bounds,
        Corners::all(px(8.)),                  // corner_radii
        rgb(0xffffff),                         // background
        Edges::all(px(1.)),                    // border_widths
        rgb(0xdddddd),                         // border_color
        BorderStyle::Solid,                    // veya Dashed
    ),
);

window.paint_quad(outline(bounds, rgb(0xff0000), BorderStyle::Solid));
```

**`PaintQuad` builder yardımcıları** (`window.rs:5848+`):

- `.corner_radii(impl Into<Corners<Pixels>>)`
- `.border_widths(impl Into<Edges<Pixels>>)`
- `.border_color(impl Into<Hsla>)`
- `.background(impl Into<Background>)`

**Diğer paint API'leri.** Window üzerinde çağrılabilecek paint yardımcıları geniş bir yelpazeyi kapsar:

- `window.paint_path(Path<Pixels>, impl Into<Background>)` — tessellate edilmiş path.
- `window.paint_underline(Point, width, &UnderlineStyle)` — metin alt çizgisi.
- `window.paint_strikethrough(Point, width, &StrikethroughStyle)`.
- `window.paint_glyph(...)` — tek glif rasterize ve çizim. Genellikle TextLayout zaten kullanır; nadiren elle çağrılır.
- `window.paint_emoji(...)` — emoji renk glifi.
- `window.paint_image(bounds, corner_radii, RenderImage, ...)` — raster görsel.
- `window.paint_svg(bounds, path, data, transformation, color, cx)` — `SvgRenderer` atlas önbelleği üzerinden monokrom SVG maskesi.
- `window.paint_surface(bounds, CVPixelBuffer)` — yalnız macOS yerel surface'i.
- `window.paint_shadows(bounds, corner_radii, &[BoxShadow])` — drop shadow seti.
- `window.paint_layer(bounds, |window| ...)` — aynı bounds üzerinde kırpma ile yeni çizim katmanı; taşma gizleme ve dönüşüm için.

`BorderStyle` (`crates/gpui/src/scene.rs:544`) iki değer alır: `Solid` ve `Dashed`. `Corners<P>`, `Edges<P>`, `Bounds<P>`, `Hsla` ve `Background` zaten önceden bilinen geometri ve renk tipleridir; her builder bunları `Into` üzerinden kabul eder.

**Tuzaklar.** Paint çağrılarında dikkat edilmesi gerekenler:

- `paint_*` çağrıları yalnız `Element::paint` fazında geçerlidir; prepaint veya yerleşim fazında panic verir.
- `paint_path` her karede yeniden tessellate edilirse FPS düşer; mümkünse path prepaint'te oluşturulur ve element durumunda saklanır.
- `paint_layer` kırpma uyguladığı için içerik bounds dışına taşan kısımlar gizlenir; gölge gibi taşan efektler katman dışında çizilmelidir.
- `border_widths` dört kenara ayrı değer verebilir (`Edges { top, right, bottom, left }`); tek bir değer verildiğinde `Edges::all(px(1.))` kullanılır.

## Çizim Bağlamı Yığını, Asset Çekme ve SVG Dönüşümü

Özel element yazılırken `Window` yalnızca paint primitive çağrılarının yeri değildir; çizim fazlarında aktif stil, offset, kırpma ve asset yükleme bağlamını da taşır.

**Bağlam yığını yardımcıları.** Geçici olarak farklı stil, rem veya kırpma gerektiğinde yığın üzerine eklenir:

- `window.with_text_style(Some(TextStyleRefinement), |window| ...)` — aktif metin stili yığınına refinement ekler. İçeride `window.text_style()` birleşmiş sonucu verir.
- `window.with_rem_size(Some(px(...)), |window| ...)` — rem üzerine yazma yığını; içeride `window.rem_size()` üzerine yazılan değeri döndürür.
- `window.set_rem_size(px(...))` — pencerenin temel rem değerini kalıcı değiştirir.
- `window.with_content_mask(Some(ContentMask { bounds }), |window| ...)` — mevcut maske ile kesişim alır; paint veya prepaint içindeki `content_mask()` bu aktif kırpmayı verir.
- `window.with_image_cache(Some(cache), |window| ...)` — alt ağaç için aktif görsel önbellek yığınını değiştirir. `ImageCacheElement` ve `Div` arka plan görseli yolları bunu kullanır; normal bileşen kodu çoğunlukla `image_cache(retain_all(...))` fluent API'sini tercih eder.
- `window.with_element_offset(offset, |window| ...)` ve `with_absolute_element_offset(offset, |window| ...)` — prepaint sırasında alt öğe offset'ini değiştirir. Scroll ve list uygulamalarının hitbox ve yerleşim koordinatlarını doğru üretmesi bu API'lere dayanır.
- `window.element_offset()` — prepaint sırasında aktif offset'i okur.
- `window.transact(|window| -> Result<_, _> { ... })` — prepaint yan etkilerini deneme amaçlı yapar; closure `Err` döndüğünde hitbox/tooltip/dispatch/yerleşim kayıtları eski index'e kısaltılır.

**Kare ve paint yardımcıları.** Pencere üzerindeki bazı işler doğrudan çizim bağlamını etkiler:

- `window.set_window_cursor_style(style)` — hitbox'a bağlı olmayan, tüm pencere için imleç isteği. Paint fazında çağrılır ve hitbox imleçlerine göre önceliklidir.
- `window.set_tooltip(AnyTooltip) -> TooltipId` — tooltip isteği prepaint fazında kaydedilir.
- `window.paint_svg(...)` — `SvgRenderer` ve sprite atlas üzerinden monokrom SVG maskesi çizer. SVG her zaman hedef boyutun `gpui::SMOOTH_SVG_SCALE_FACTOR: f32 = 2.0` (`svg_renderer.rs:81`) katı çözünürlükte rasterize edilip sonra küçültülür; bu nedenle `paint_svg` çağrısı küçük ikon boyutlarında bile yumuşak kenar üretir. `paint_image` kodu çözülmüş raster kare, `paint_surface` ise macOS yerel surface'i içindir.

**Jenerik asset yükleme.** Asset bekleme ve önbellek paylaşımı için üç yardımcı bulunur:

```rust
if let Some(result) = window.use_asset::<MyAsset>(&source, cx) {
    render_loaded(result, window, cx);
}
```

- `window.use_asset::<A>(&source, cx) -> Option<A::Output>` — yükleme bitmediğinde `None` döner ve ilk yükleme tamamlandığında o anki view'i sonraki karede bildirir.
- `window.get_asset::<A>(&source, cx) -> Option<A::Output>` — önbelleği yoklar ancak tamamlandığında view yeniden çizimi planlamaz.
- `cx.fetch_asset::<A>(&source) -> (Shared<Task<A::Output>>, bool)` — daha düşük seviye ortak görev önbelleği; aynı asset türü ve kaynağı için tek `Asset::load` future'ı paylaşılır.
- `AssetLogger<T>` — `Asset<Output = Result<R, E>>` yükleyicisini sarar ve hata sonucunu loglar.

**SVG dönüşümü.** SVG elementi kendi içinde dönüşüm desteği sağlar:

```rust
svg()
    .path("icons/check.svg")
    .with_transformation(
        Transformation::rotate(radians(0.2))
            .with_scaling(size(1.2, 1.2))
            .with_translation(point(px(2.), px(0.))),
    )
```

- `svg().path(...)` — embedded `AssetSource` içinden SVG okur.
- `svg().external_path(...)` — dosya sistemi path'inden okur.
- `Transformation::{scale, translate, rotate}` ve `with_scaling`/`with_translation`/`with_rotation` yalnızca çizimi etkiler; hitbox ve yerleşim boyutu değişmez.
- Düşük seviyeli `TransformationMatrix::{unit, translate, rotate, scale}` sahne primitive'lerinde kullanılır.
- `SvgSize` `SvgRenderer` çizim isteğinin raster boyutunu tanımlar: `Size(Size<DevicePixels>)` mutlak boyut, `ScaleFactor(f32)` SVG'nin bildirdiği boyuta çarpan uygular.

**Tuzaklar.** Bu çizim bağlamı API'lerinde dikkat edilecek noktalar:

- `with_content_mask` yalnızca kırpma maskesidir; hitbox veya yerleşimi otomatik küçültmez.
- `use_asset` yeniden çizimi o anki view entity'sine bağlar; view dışı bir yardımcıda çağrılıyorsa o anki view beklentisi bozulmamalıdır.
- SVG dönüşümü görsel olarak döndürür ya da ölçeklendirir, ancak işaretçi hitbox'ı eski yerleşim rect'inde kalır.

---

## Animasyon Sistemi

Animasyon API'si `crates/gpui/src/elements/animation.rs` içinde yer alır.

`Animation` üç alandan oluşur: `duration: Duration`, `oneshot: bool` (false ise tekrar eder), `easing: Rc<dyn Fn(f32) -> f32>`. İnşa için `Animation::new(duration)` doğrusal easing ile tek seferlik animasyon oluşturur. `.repeat()` döngüye alır, `.with_easing(fn)` easing'i değiştirir.

`AnimationExt` trait'i herhangi bir `IntoElement` için iki metot ekler:

```rust
use gpui::{Animation, AnimationExt};
use std::time::Duration;

div()
    .size(px(100.))
    .with_animation(
        "grow",
        Animation::new(Duration::from_millis(500))
            .with_easing(gpui::ease_in_out),
        |el, delta| el.size(px(100. + 100. * delta)),
    )
```

Çoklu animasyon zinciri için `with_animations(id, vec![anim_a, anim_b], |el, ix, delta| ...)` kullanılır. Closure'a (element, animation_index, delta_in_animation) parametreleri verilir; `ix` aktif animasyonun vec içindeki sırası, `delta` ise o animasyona göre 0..1 ilerlemesidir. Sıralı veya çoklu fazlı geçiş yazılırken `ix` ile `match` yapılır.

**Yerleşik easing fonksiyonları** (`crates/gpui/src/elements/animation.rs:211+`): `linear`, `quadratic`, `ease_in_out`, `ease_out_quint()`, `bounce(inner)`, `pulsating_between(min, max)`. `pulsating_between` yön değiştirerek değer döndürür (yükleme göstergesi için ideal; `Animation::repeat()` ile birleştirilir).

**Tuzaklar.** Animasyon kullanımında atlanan noktalar:

- Element ID çizim boyunca stabil olmalıdır; değiştiği anda animasyon durumu sıfırlanır.
- Animator closure `'static` olduğundan dış durum `Rc`, `Arc` veya `clone` ile yakalanır.
- Tekrarlanan animasyon `window.request_animation_frame()` ile sonraki kareyi talep eder; bu da mevcut view'i sonraki karede bildirir. Gerekmiyorsa oneshot bırakılır.
- Kareler arası ilerleme değeri executor saatinden hesaplanır; normal `TestAppContext`/`VisualTestContext` testlerinde `cx.background_executor.advance_clock(...)` veya `TestApp::advance_clock(...)` kullanılır. macOS'a özel `VisualTestAppContext` üzerinde ayrıca doğrudan `advance_clock(...)` yardımcısı vardır.

---
