# Render ve Element Modeli

---

## Render Modeli

Bir pencerenin kök view'u her zaman bir `Entity<V>` olur ve bu `V` tipi `Render` trait'ini uygulamak zorundadır. `Render::render` her ekran karesinde yeniden çağrılır; view'un kendi alanlarında tuttuğu veriyi geçici bir element ağacına çeviren ana metot budur:

```rust
struct MyView {
    focus_handle: FocusHandle,
}

impl Render for MyView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .id("my-view")
            .track_focus(&self.focus_handle)
            .key_context("my-view")
            .on_action(cx.listener(|this, _: &CloseWindow, window, cx| {
                window.remove_window();
            }))
            .size_full()
            .child("Content")
    }
}
```

Kendi kalıcı verisini taşımayan, yeniden kullanılan küçük bileşenler için ise `RenderOnce` trait'i kullanılır. Bu trait `self`'i tüketir ve genellikle Zed UI bileşenleri gibi statik veriden inşa edilen parçalar için tercih edilir:

```rust
#[derive(IntoElement)]
struct Badge {
    label: SharedString,
}

impl RenderOnce for Badge {
    fn render(self, _window: &mut Window, _cx: &mut App) -> impl IntoElement {
        div().rounded_sm().px_2().child(self.label)
    }
}
```

Aradaki ayrım sahiplik üzerinden anlaşılır: `Render::render` her ekran karesinde yeniden çağrıldığı için `&mut self` alır ve view'un alanlarında duran veriyi yerinde okur; `RenderOnce::render` ise yalnızca bir kez element üretip biteceği için `self`'i tamamen tüketir ve genellikle Zed UI bileşenlerinde tercih edilir. Seçim, bileşenin verisinin nerede saklandığına bağlıdır. Veri view içinde kalıcı olarak tutulacaksa `Render`, çağıran kodun geçirdiği yapıdan tek seferlik inşa ediliyorsa `RenderOnce` kullanılır.

## Element Yaşam Döngüsü ve Çizim Aşamaları

`Element` sözleşmesi üç ana aşamadan oluşur. Bu aşamalar aynı ekran karesi içinde sırayla çalışır:

1. `request_layout(...) -> (LayoutId, RequestLayoutState)` — stil ve alt öğelerin yerleşim istekleri Taffy yerleşim ağacına verilir. Bu aşamada çizim yapılmaz; yalnızca "ne kadar yer istiyorum, alt öğelerimin `LayoutId`'leri nedir" bilgisi hazırlanır.
2. `prepaint(...) -> PrepaintState` — yerleşim sonucu artık bilinir; dolayısıyla element'in son konum ve boyutuna göre yapılması gereken işler burada gerçekleşir: hitbox kaydı, scroll konumunun hazırlanması, element başına saklanan verinin okunması ve gerekli ölçümler.
3. `paint(...)` — sahneye çizilecek temel şekiller (`primitive`) üretilir. `paint_quad`, `paint_path`, `paint_image`, `paint_svg`, `set_cursor_style` gibi çağrılar bu aşamaya aittir.

`Window` üzerindeki hata ayıklama kontrolleri (`debug assertion`) aşama ihlallerini yakalar: `insert_hitbox` yalnızca prepaint'te; `paint_*` çağrıları paint'te; `with_text_style` ve bazı ölçüm yardımcıları ise prepaint veya paint aşamalarında geçerlidir. Yanlış aşamada yapılan bir çağrı, hata ayıklama derlemesinde `panic` ile sonuçlanır; böylece sorunlar erken yakalanır.

**Veri saklama yolları.** Element seviyesinde kalıcı verinin nerede tutulduğu, o verinin ne kadar süre yaşaması gerektiğine göre belirlenir:

- View verisi: `Entity<T>` alanları olarak tutulur; uygulama boyunca veya view kapanana kadar yaşar.
- Element başına saklanan veri: stabil bir `id(...)` ile birlikte `window.with_element_state` veya `with_optional_element_state` üzerinden tutulur. Aynı ID ardışık çizimlerde korunursa veri devam eder; ID değişirse sıfırlanır.
- Sonraki ekran karesine kayıt: `window.on_next_frame(...)` çağrısıyla yapılır.
- Etki (effect) sonuna erteleme: `cx.defer(...)`, `window.defer(cx, ...)`, `cx.defer_in(window, ...)`.
- Sürekli yeniden çizim: `window.request_animation_frame()` ile yeni bir ekran karesi talep edilir.

**Çizim katmanı.** GPUI'da çizim zinciri birkaç trait'in birlikte çalışmasıyla ortaya çıkar; her trait belirli bir yetenek setini temsil eder:

- `Render`: entity/view verisini her çizimde element ağacına çevirir.
- `RenderOnce`: yalnızca element'e dönüştürülecek hafif bileşenler için uygundur.
- `ParentElement`: alt öğe kabul eden elementlerin trait'idir.
- `Styled`: stil refinement zincirine dahil olan elementleri belirler.
- `InteractiveElement`: klavye odağı, action, tuş, fare, hover ve sürükle-bırak dinleyicilerini açar.
- `StatefulInteractiveElement`: `id(...)` çağrısından sonra scroll ve klavye odağı gibi ekran kareleri arasında veri korumayı gerektiren interaktif davranışları açar.

**Kritik kural.** `cx.notify()`, view'un çizim çıktısını etkileyen bir veri değiştiğinde çağrılır. Bu olmadan view yeniden çizilmez. `window.refresh()` ise tüm pencerenin tekrar çizimini ister. Yerel view verisindeki değişimler için önce `cx.notify()` tercih edilir; çünkü daha hedefli bir yenileme yapar.

## Element Haritası

GPUI'nın yerleşik elementleri farklı görevler için ayrı ayrı tasarlanmıştır. Aşağıdaki liste, hangi element'in hangi sorumluluk için seçileceğini hızlıca gösterir:

- `div()` — neredeyse tüm yerleşim ve kapsayıcı işlerinin temel taşıdır. Flex/grid, stil, alt öğe, olay, klavye odağı ve pencere kontrol alanı (`window-control area`) destekler.
- Metin — `&'static str`, `String`, `SharedString` doğrudan element olur. Daha karmaşık metin durumlarında `StyledText` ve `InteractiveText` devreye girer.
- `svg()` — satır içi (inline) path veya harici path ile SVG çizimi sağlar.
- `img(...)` — asset, dosya yolu, URL veya byte kaynağı gibi görsel kaynaklarını çizer; yükleme ve yedek görsel (fallback) bölümlerini de destekler.
- `canvas(prepaint, paint)` — düşük seviyeli çizim ya da hitbox/imleç gibi çizim hazırlığı gerektiren işler için kullanılır.
- `anchored()` — pencereye veya belirli bir noktaya sabitlenen popover ve menü benzeri UI parçaları içindir.
- `deferred(child)` — öncelikli veya ertelenmiş çizim gerektiren durumlar için.
- `list(...)` — değişken yükseklikli büyük listelerde tercih edilir.
- `uniform_list(...)` — sabit veya kolay ölçülen satır yüksekliği olan, yüksek performans gerektiren listeler için.
- `surface(...)` — platform/yerel bir yüzey kaynağını (`surface`) element olarak gösterir.

**Sık kullanılan stil grupları.** Fluent API'de tekrar tekrar karşılaşılan zincir parçaları genelde şu gruplar altında toplanır:

- Yerleşim: `.flex()`, `.flex_col()`, `.flex_row()`, `.grid()`, `.items_center()`, `.justify_between()`, `.content_stretch()`, `.size_full()`, `.w(...)`, `.h(...)`.
- Boşluk: `.p_*`, `.px_*`, `.gap_*`, `.m_*`.
- Metin: `.text_color(...)`, `.text_sm()`, `.text_xl()`, `.font_family(...)`, `.truncate()`, `.line_clamp(...)`.
- Kenarlık ve şekil: `.border_1()`, `.border_color(...)`, `.rounded_sm()`.
- Konum: `.absolute()`, `.relative()`, `.top(...)`, `.left(...)`.
- Durum: `.hover(...)`, `.active(...)`, `.focus(...)`, `.focus_visible(...)`, `.group(...)`, `.group_hover(...)`.
- Etkileşim: `.on_click(...)`, `.on_mouse_down(...)`, `.on_scroll_wheel(...)`, `.on_key_down(...)`, `.on_action(...)`, `.track_focus(...)`, `.key_context(...)`.

Zed kod tabanında `ui::prelude::*` genellikle `gpui::prelude::*` yerine tercih edilir; bu prelude tasarım sistemi tiplerini de birlikte getirir, böylece `use` listesi sade kalır.

## Element ID, Element Verisi ve Tip Soyutlaması

GPUI'da her çizimde element ağacı sıfırdan kurulur. Buna rağmen hover, scroll ve önbellek (cache) gibi durumların ekran kareleri arasında korunması gerekir. Bu kalıcılığı stabil ID'ler sağlar. İlgili ana tipler şunlardır:

- `ElementId` — `Name`, `Integer`, `NamedInteger`, `Path`, `Uuid`, `FocusHandle`, `CodeLocation` gibi varyantlar taşır.
- `GlobalElementId` — üst öğelerin isim alanı (`namespace`) zinciriyle birleşerek tam yol oluşturur.
- `AnyElement` — element için tip soyutlaması (`type erasure`); alt öğe listelerinde farklı türden element tutmak için kullanılır.
- `AnyView` / `AnyEntity` — view veya entity için tip soyutlaması.

Element başına saklanan veri için API'ler `Window` üzerindedir ve yalnızca element çizimi sırasında çağrılabilir. Yüksek seviyeli API bu veriyi otomatik yönetir:

```rust
let row_state = window.use_keyed_state(
    ElementId::named_usize("row", row_ix),
    cx,
    |_, cx| RowState::new(cx),
);
```

Daha düşük seviyeli ihtiyaçlar için global id ve element verisi API'leri doğrudan açıktır:

```rust
window.with_global_id("image-cache".into(), |global_id, window| {
    window.with_element_state::<MyState, _>(global_id, |state, window| {
        let mut state = state.unwrap_or_else(MyState::default);
        state.prepare(window);
        (state.snapshot(), state)
    })
});
```

**Kurallar.** Element id'siyle çalışırken gözetilmesi gereken disiplinler şunlardır:

- `window.with_id(element_id, |window| ...)` yerel element id yığınına bir id ekler; `with_global_id` bu yığından tam bir `GlobalElementId` üretir.
- Liste satırlarında `use_state` yerine `use_keyed_state` tercih edilir; `use_state` çağrı konumuna göre id üretir ve aynı çizim noktasındaki birden fazla satırı birbirinden ayıramaz.
- `with_element_namespace(id, ...)` özel bir element içinde alt öğe id çakışmalarını önlemek için kullanılır.
- Aynı `GlobalElementId` ve aynı veri tipi için iç içe `with_element_state` çağrısı `panic` üretir.
- ID değiştiğinde önceki ekran karesinin verisi devam etmez; animasyon, hover, scroll ve görsel önbellek verisi sıfırlanır.

**Tip soyutlaması (type erasure) kararları.** Tipli ve tipsiz element/view arasında seçim yapılırken şu yönlendirmeler işe yarar:

- Genel bir bileşen API'si alt öğe kabul ediyorsa `impl IntoElement` almak uygundur.
- Bir struct içinde saklanacaksa `AnyElement` kullanılır.
- View veya entity saklanıyorsa mümkün olduğu kadar tipli `Entity<T>` tutmak tercih edilir; yalnızca plugin, dock öğesi veya heterojen koleksiyon gerektiren durumlarda `AnyEntity`/`AnyView` seçilir.

## FluentBuilder ve Koşullu Element Üretimi

`crates/gpui/src/util.rs::FluentBuilder` trait'i tüm element tiplerine üç yardımcı ekler ve fluent zincirin if/match bloklarıyla kırılmasını engeller:

```rust
pub trait FluentBuilder {
    fn map<U>(self, f: impl FnOnce(Self) -> U) -> U;
    fn when(self, condition: bool, then: impl FnOnce(Self) -> Self) -> Self;
    fn when_else(
        self,
        condition: bool,
        then: impl FnOnce(Self) -> Self,
        else_fn: impl FnOnce(Self) -> Self,
    ) -> Self;
    fn when_some<T>(self, option: Option<T>, then: impl FnOnce(Self, T) -> Self) -> Self;
    fn when_none<T>(self, option: &Option<T>, then: impl FnOnce(Self) -> Self) -> Self;
}
```

Tipik bir kullanım birden fazla koşullu davranışı tek bir akıcı zincirde toparlar:

```rust
div()
    .flex()
    .when(self.is_active, |this| this.bg(rgb(0xFF0000)))
    .when_some(self.icon.as_ref(), |this, icon| this.child(icon.clone()))
    .when_else(self.is_loading,
        |this| this.opacity(0.5),
        |this| this.opacity(1.0),
    )
    .map(|this| match self.density {
        UiDensity::Compact => this.gap_1(),
        UiDensity::Default => this.gap_2(),
        UiDensity::Comfortable => this.gap_4(),
    })
```

**Avantajlar.** Bu yardımcıların getirdiği başlıca kolaylıklar şunlardır:

- Akıcı zincir (`method chain`) bozulmaz; if/match yapılarına başvurmadan koşullu UI yazılabilir.
- Closure içine geçen element'in tipi korunur; alt öğe eklemeye devam etmek serbesttir.
- `map` zincirin dışına kontrollü çıkmak için kullanılır; keyfi bir dönüşüm gerektiğinde işe yarar.

**Tuzaklar.** Aynı kolaylıkların yanlış kullanımı küçük sorunlar üretebilir:

- `when` closure her çizimde çalışır; içinde ağır hesap yapılması performans sorunu doğurur.
- Aynı element üzerinde defalarca `when_some` zincirlemek okunabilirliği bozuyorsa, veri önce normal bir `if let` ile önceden hesaplanır ve tek bir `.child(...)` çağrısı yapılır.
- `map` element tipini değiştirebilir; `when` ise tipi değiştirmez (refinement zincirinde kalır). Bu nedenle `map` kullanımı dikkatli yapılır.

## Refineable, StyleRefinement ve MergeFrom

GPUI ve Zed'de iki kompozisyon deseni paralel çalışır: çizim zincirinde `Refineable`, ayarlar ve tema yüklemesinde `MergeFrom`. İkisi de "varsayılan değerin üstüne adım adım üzerine yazma uygula" mantığını kullanır, ancak farklı yerlerde devreye girer.

#### Refineable

`crates/refineable/src/refineable.rs`:

```rust
pub trait Refineable: Clone {
    type Refinement: Refineable<Refinement = Self::Refinement> + IsEmpty + Default;

    fn refine(&mut self, refinement: &Self::Refinement);
    fn refined(self, refinement: Self::Refinement) -> Self;

    fn from_cascade(cascade: &Cascade<Self>) -> Self
        where Self: Default + Sized;

    fn is_superset_of(&self, refinement: &Self::Refinement) -> bool;
    fn subtract(&self, refinement: &Self::Refinement) -> Self::Refinement;
}

pub trait IsEmpty {
    fn is_empty(&self) -> bool;
}
```

Trait sözleşmesi göründüğünden zengindir ve birkaç ince detay içerir:

- `type Refinement` de `Refineable` olmalıdır; yani refinement'ın kendisi tekrar refine edilebilir. Bu sayede `refine_a.refine(&refine_b)` zincirleme birleştirme mümkün olur.
- Aynı `Refinement` ayrıca `IsEmpty + Default` zorunluluğunu taşır. `IsEmpty`, "bu refinement uygulansa hiçbir alan değişir mi?" sorusunu cevaplar; birleştirme, yerleşim önbelleğinin geçersiz sayılması ve `subtract` çıktısı bu kontrole dayanır.
- `is_superset_of(refinement)`, üzerinde çağrıldığı değerin bu refinement'ı zaten kapsayıp kapsamadığını söyler; gereksiz `refine` çağrıları bu sayede atlanabilir.
- `subtract(refinement)`, iki refinement arasındaki farkı yeni bir refinement olarak verir.
- `from_cascade(cascade)`, aşağıda anlatılan `Cascade` yapısını varsayılan değer üzerine uygular; tema ve stil katmanlamasının sondaki "düzleştirme" adımıdır.

`#[derive(Refineable)]` (gpui re-export'lu): orijinal struct ile aynı alanlara sahip, ama her alanı `Option`'lı hale getirilmiş bir `XRefinement` türü üretir. `refine` çağrısı yalnızca `Some` alanları yazar. Aşağıdaki somut türler her zaman derive ile üretilir, ayrıca elle yazmaya gerek kalmaz:

| Refinement türü | Üreten struct | Kaynak |
|---|---|---|
| `StyleRefinement` | `Style` | `style.rs:178` |
| `TextStyleRefinement` | `TextStyle` | `style.rs` |
| `UnderlineStyleRefinement` | `UnderlineStyle` | `style.rs` |
| `StrikethroughStyleRefinement` | `StrikethroughStyle` | `style.rs` |
| `BoundsRefinement` | `Bounds` | `geometry.rs` |
| `PointRefinement` | `Point` | `geometry.rs` |
| `SizeRefinement` | `Size` | `geometry.rs` |
| `EdgesRefinement` | `Edges` | `geometry.rs` |
| `CornersRefinement` | `Corners` | `geometry.rs` |
| `GridTemplateRefinement` | `GridTemplate` | `geometry.rs` |

Bu `*Refinement` tipleri çoğunlukla doğrudan adlandırılarak kullanılmaz; fluent API zinciri onları arka planda toplar. Doğrudan elle inşa etmek gerektiği tek tip genellikle `StyleRefinement`'tır — örneğin `.hover(|style| style.bg(...))` geri çağrısının (`callback`) imzasında bu tip görünür.

Tipik kullanım `Style`/`StyleRefinement` (`crates/gpui/src/style.rs:178`) üzerinden ilerler:

```rust
let mut style = Style::default();
style.refine(&StyleRefinement::default()
    .text_size(px(20.))
    .font_weight(FontWeight::SEMIBOLD));
```

Element fluent zinciri (örneğin `div().text_size(px(14.)).bg(rgb(0xff))`) arka planda bir `StyleRefinement` biriktirir; çizim sırasında temel stil üzerine refine eder. `TextStyle`/`TextStyleRefinement`, `HighlightStyle`, `PlayerColors`, `ThemeColors` gibi tüm tema yapıları aynı kalıbı kullanır.

`refined(self, refinement)` ise değiştirilemez bir kopya üretir; "ek stil ile yeni temel değer elde et" senaryolarında uygundur.

#### Cascade ve CascadeSlot

`Refineable` tek başına iki katmanı (temel değer + refinement) birleştirir. Daha derin hover/focus/active akışları için `crates/refineable/src/refineable.rs:80,93` katman yığını sunar:

```rust
pub struct Cascade<S: Refineable>(Vec<Option<S::Refinement>>);
pub struct CascadeSlot(usize);
```

API yüzeyi şu şekildedir:

- `Cascade::default()`, slot 0'ı `Some(default)` ile kurar; ek slotlar başta `None`'dur. Slot 0 her zaman dolu kalır ve temel refinement'tır.
- `cascade.reserve() -> CascadeSlot`, yeni bir `None` slot ekler ve onu sonradan bulmaya yarayan referansı (`handle`) döner. Hover, focus, active gibi her dinamik katman için ayrı bir slot ayrılır.
- `cascade.base() -> &mut S::Refinement`, slot 0'a değiştirilebilir erişim verir; her yerleşim hesabında asıl stil buraya yazılır.
- `cascade.set(slot, Option<S::Refinement>)`, belirli bir slot'a refinement koyar veya `None` ile o katmanı devre dışı bırakır.
- `cascade.merged() -> S::Refinement`, slot 0 üzerine diğer dolu slotları sırayla `refine` eder; sonraki slot önceki slotu ezer.
- `Refineable::from_cascade(&cascade) -> Self`, `default().refined(merged())` kısayoludur; çizim sırasında nihai stili üretmek için kullanılır.

**Önemli not.** GPUI'nın kendi `Interactivity` katmanı (`.hover(...)`, `.active(...)`, `.focus(...)`, `.focus_visible(...)`, `.in_focus(...)`, `.group_hover(...)`, `.group_active(...)` zinciri) **`Cascade`/`CascadeSlot` kullanmaz**. `Interactivity` struct'ı her durum için ayrı bir `Option<Box<StyleRefinement>>` alanı tutar (`elements/div.rs:1681+`'deki `hover_style`, `active_style`, `focus_style`, `in_focus_style`, `focus_visible_style`, `group_hover_style`, `group_active_style`) ve çizim aşamasında bu refinement'ları sırayla `refine` eder. Yani hover stilinde verilen `StyleRefinement::bg(...)` temel arka planı ezer; ama `font_size`'a dokunmayan bir refinement temel `font_size` değerini korur. `None` alan "etki yok" anlamına gelir.

`Cascade<S>` ve `CascadeSlot` arayüzü `refineable` crate'inde genel olarak durur; ancak GPUI çekirdeği veya Zed bu sürümde içeriden kullanmaz. Çoklu katmanlı (3+) refinement yığınını dışarıdan kurmak isteyen kütüphane yazarları için bir uzantı noktasıdır.

#### MergeFrom

`crates/settings_content/src/merge_from.rs`:

```rust
pub trait MergeFrom {
    fn merge_from(&mut self, other: &Self);
    fn merge_from_option(&mut self, other: Option<&Self>) {
        if let Some(other) = other { self.merge_from(other); }
    }
}
```

Varsayılan kurallar şu şekildedir:

- `HashMap`, `BTreeMap`, struct: derin birleştirme — yalnızca `other`'da var olan alanlar yazılır.
- `Option<T>`: `None` üzerine yazmaz; `Some` özyinelemeli olarak birleşir.
- Diğer tipler (`Vec`, ilkel tipler): tam üzerine yazma.

`#[derive(MergeFrom)]` derive'ı, struct alanları için özyinelemeli birleştirme üretir. Bu varsayılan davranışı değiştirmek için `ExtendingVec<T>` (her birleştirmede sona ekleme yapar) ve `SaturatingBool` (bir kez `true` olunca öyle kalır) gibi sarıcılar hazırdır.

**Ayar (`Settings`) yükleme zinciri.** Ayarlar okunurken katmanlar belli bir sırayla birleştirilir:

1. `assets/settings/default.json` → `SettingsContent::default()` baz alınır.
2. Kullanıcının `~/.config/zed/settings.json` dosyası ayrıştırılır → `merge_from_option`.
3. Aktif profil → `merge_from_option`.
4. Worktree `.zed/settings.json` → `merge_from_option`.
5. Sonuç, `Settings::from_settings(content)` ile somut bir struct'a çevrilir.

**Tuzaklar.** Refineable ve MergeFrom kullanımlarında karşılaşılabilecek hatalı kalıplar şunlardır:

- `Refineable` zincirinde `default()` temel değeri her seferinde yeniden hesaplanır; ağır temel stiller bir önbelleğe alınmalıdır.
- `MergeFrom` sıralaması alt-üst değildir: en spesifik kaynak en sona konulmalıdır (`local > profile > user > default`).
- `Vec`'lere sona eklemek gerekiyorsa `ExtendingVec` kullanılır; üzerine yazmak yeterliyse düz `Vec` tercih edilir.
- `Option<Option<T>>` gibi iç içe seçenek yapıları gerektiğinde `MergeFrom`'un varsayılan davranışı doğru sonucu vermeyebilir; bu durumda özel bir `impl` yazılması gerekir.

## Ertelenmiş Çizim, Çizim Hazırlığı Sırası ve Üst Katman

`deferred(child)` çağrısı, alt öğenin yerleşimini bulunduğu yerde hesaplar; ama çizimini, üst öğelerin çizimleri tamamlandıktan sonraya erteler. Bu davranış popover, sağ tıklama menüsü, yeniden boyutlandırma tutamacı (`resize handle`) ve dock üzerine bırakma alanı (`dock drop overlay`) gibi "üstte çizilmesi ama yerleşimde yer kaplamaması gereken" parçalar için tasarlanmıştır:

```rust
deferred(
    anchored()
        .anchor(Anchor::TopRight)
        .position(menu_position)
        .child(menu),
)
.with_priority(1)
```

**Davranış.** Üç aşama sırasıyla şu işleri yapar:

- `request_layout`: alt öğe normal şekilde yerleşim alır.
- `prepaint`: alt öğe, `window.defer_draw(...)` ile ertelenmiş kuyruğa taşınır.
- `paint`: ertelenmiş element kendi çizim aşamasında bir şey üretmez; gerçek çizim, ertelenmiş kuyrukta sıra geldiğinde yapılır.
- `with_priority(n)`: aynı ekran karesindeki ertelenmiş elementler arasında üstte/altta sırasını (`z-order`) belirler; yüksek öncelikli element üstte çizilir.

**`Div` çizim hazırlığı yardımcıları.** Yerleşim sonuçlarına göre çizim hazırlığı aşamasında aksiyon almak gerektiğinde iki yardımcı vardır:

- `on_children_prepainted(|bounds, window, cx| ...)` — alt öğelerin son konum ve boyutunu ölçer, sonraki çizim için veri üretir.
- `with_dynamic_prepaint_order(...)` — alt öğelerin çizim hazırlığı sırasını çalışma zamanında belirler. Özellikle bir alt öğenin otomatik kaydırması veya ölçüm sonucu diğer alt öğeyi etkilediği durumlarda kullanılır.

**Tuzaklar.** Ertelenmiş çizim kullanımında dikkat edilecek noktalar:

- Ertelenmiş alt öğe yerleşimde yer tuttuğu için `absolute`/`anchored` konumlandırma hâlâ doğru üst öğe sınırlarına bağlıdır.
- Üst katmanın fare olaylarını engellemesi isteniyorsa alt öğe içinde `.occlude()` veya `.block_mouse_except_scroll()` kullanılır.
- Öncelik değeri global bir z-index değildir; yalnızca aynı pencere ekran karesi içindeki ertelenmiş kuyruk için geçerlidir.

---
