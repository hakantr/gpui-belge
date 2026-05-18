# Stil, Geometri ve Renkler

---

## Stil ve Yerleşim Haritası

GPUI stil sistemi CSS ve Tailwind'e benzeyen fluent metotlardan oluşur. Arka planda Rust tipleri olduğu için neyin hangi değeri aldığı daha nettir. Fluent zinciri okurken metotları şu gruplara ayırmak işi kolaylaştırır:

- Boyut: `w`, `h`, `size`, `min_w`, `max_w`, `flex_basis`, `size_full`, `h_auto`, `relative(f32)`, `px`, `rems`.
- Yerleşim: `flex`, `grid`, `flex_row`, `flex_col`, `items_*`, `justify_*`, `content_*`, `gap_*`, `flex_wrap`, `flex_grow`, `flex_shrink`.
- Konum: `relative`, `absolute`, `inset_*`, `top/right/bottom/left`, `z_index`.
- Taşma: düz stil taşma ve kalıcı veri tutan `.overflow_*_scroll()` ailesi.
- Arka plan ve kenarlık: `bg`, `border_*`, `border_color`, `rounded_*`, `box_shadow`, `opacity`.
- Metin: `text_color`, `text_bg`, `text_size`, `text_*`, `font_family`, `font_weight`, `italic`, `line_height`, `text_ellipsis`, `line_clamp`.
- Etkileşim: `hover`, `active`, `focus`, `focus_visible`, `cursor_*`, `track_focus`, `key_context`, action, tuş ve fare dinleyicileri.
- Grup stilleme: `.group("name")`, `group_hover(...)`, `group_active(...)` ve `group_drag_over::<T>(...)`, aynı isimli etkileşim grubuna göre uygulanır.
- Grid yerleşimi: kapsayıcı için `grid_cols`, `grid_cols_min_content`, `grid_cols_max_content`, `grid_rows`; alt öğe için `col_start`, `col_end`, `col_span`, `col_span_full`, `row_start`, `row_end`, `row_span`, `row_span_full` kullanılır. Altta `GridTemplate`, `TemplateColumnMinSize` ve `GridPlacement::{Line, Span, Auto}` yatar.

Yukarıdaki fluent metotların büyük çoğunluğu `Styled` trait gövdesinde tek tek yazılmaz; bir grup proc macro tarafından üretilir. `crates/gpui/src/styled.rs` içinde bu makrolar tek satırda çağrılır ve `gpui_macros` crate'i her çağrı için onlarca metot üretir. Hangi makronun hangi metotları ürettiği aşağıdaki tablodan takip edilebilir:

| Proc macro (`gpui_macros::...!()`) | Üretilen fluent metotlar (Styled trait üyesi olarak) |
|---|---|
| `visibility_style_methods!()` | `visible()`, `invisible()` |
| `margin_style_methods!()` | `m_*` ile birlikte `mt_`, `mb_`, `my_`, `mx_`, `ml_`, `mr_` ve her birinin spacing scale + `auto` varyantları (`mt_auto()` gibi) |
| `padding_style_methods!()` | `p_*`, `pt_`, `pb_`, `py_`, `px_`, `pl_`, `pr_` (margin ailesinin padding karşılığı) |
| `position_style_methods!()` | `relative()`, `absolute()` ve konumlanmış element ofset önekleri: `inset`, `top`, `bottom`, `left`, `right` |
| `overflow_style_methods!()` | `overflow_hidden()`, `overflow_x_hidden()`, `overflow_y_hidden()` |
| `cursor_style_methods!()` | `cursor(CursorStyle)`, `cursor_default()`, `cursor_pointer()`, `cursor_text()`, `cursor_move()`, `cursor_not_allowed()`, `cursor_context_menu()`, `cursor_crosshair()`, `cursor_vertical_text()`, `cursor_alias()`, `cursor_copy()`, `cursor_no_drop()`, `cursor_grab()`, `cursor_grabbing()` ve yeniden boyutlandırma ailesi: `cursor_ew_resize()`, `cursor_ns_resize()`, `cursor_nesw_resize()`, `cursor_nwse_resize()`, `cursor_col_resize()`, `cursor_row_resize()`, `cursor_n_resize()`, `cursor_e_resize()`, `cursor_s_resize()`, `cursor_w_resize()` |
| `border_style_methods!()` | `border_color(C)` ve `border_*` genişlik önekleri (`border_*`, `border_t_*`, `border_r_*`, `border_b_*`, `border_l_*`, `border_x_*`, `border_y_*`) × sonek tablosu (`_0`, `_1`, `_2`, `_4`, `_8`, vb.) |
| `box_shadow_style_methods!()` | `shadow(Vec<BoxShadow>)`, `shadow_none()`, `shadow_2xs()`, `shadow_xs()`, `shadow_sm()`, `shadow_md()`, `shadow_lg()`, `shadow_xl()`, `shadow_2xl()` |

Bu makrolar, `gpui_macros` crate'inden genel proc macro olarak dışa aktarılır ve `gpui::{visibility_style_methods, margin_style_methods, ...}` üzerinden de yeniden dışa aktarılır. Uygulama kodunun bunları doğrudan çağırması gerekmez; fluent metotlar zaten `Styled` trait'inin parçasıdır ve her `Styled` uygulaması bu metotlara otomatik sahip olur.

`Styled` trait içinde ayrıca `gpui_macros::style_helpers!()` çağrısı vardır. Bu proc macro `#[doc(hidden)]` olarak işaretlendiği ve `gpui` crate'inden yeniden dışa aktarılmadığı için `target/doc/gpui/all.html` makro listesinde görünmez. `w_*`, `h_*`, `size_*`, `min_size_*`, `min_w_*`, `min_h_*`, `max_size_*`, `max_w_*`, `max_h_*`, `gap_*`, `gap_x_*`, `gap_y_*` ve `rounded_*` aileleri bu iç makrodan gelir.

Özel bir element için `Styled` uygulandığında trait'in tüm metotları otomatik gelir; bu makroları yeniden çağırmak gerekmez. Yalnızca GPUI'nın kendisi gibi yeni bir stil çatısı (`style framework`) yazılırken, paralel bir `Styled` benzeri trait için bu makrolar yeni trait'in içinde çağrılabilir. `method_visibility` parametresi genel (`public`) veya `pub(crate)` ayarına izin verir.

**Pratik kararlar.** Stil zinciri kurulurken faydalı olan birkaç yönlendirici:

- Görünüm veriye bağlıysa `Render` içinde koşullu `.when(...)` kullanılır; stil sonradan emir kipinde (`imperative`) değiştirilmeye çalışılmaz.
- Scroll, klavye odağı, ipucu ve animasyon gibi kalıcı veriye sahip elementlerde id stabil olmalıdır; aksi halde veri ekran kareleri arasında kaybolur.
- Üst öğenin yerleşim genişliği belirsiz olduğunda metin taşması, görsel en-boy oranı ve `absolute` alt öğenin konumu beklenen sonucu vermeyebilir; bu nedenle üst öğenin boyutu önceden belirgin bırakılmalıdır.
- Kart, araç çubuğu veya liste gibi tekrar eden UI parçalarında boyutlar `min/max/aspect_ratio` ile sabitlenir; hover ya da yükleme durumlarının yerleşimi oynatmaması tasarımın temel kuralıdır.

## Geometri Tipleri ve Birim Yönetimi

`crates/gpui/src/geometry.rs`.

GPUI üç farklı piksel birimi kullanır. Ekran ölçeği değiştiğinde hangi birimin hangi katmanda kullanıldığını bilmek birçok hatayı baştan önler:

- `Pixels(f32)` — ölçekten bağımsız mantıksal piksel. UI kodunda neredeyse her zaman bu birim kullanılır.
- `ScaledPixels(f32)` — `Pixels * window.scale_factor()`. Çizim aracına iletilen değerdir.
- `DevicePixels(i32)` — fiziksel cihaz pikseli; asset ve doku (`texture`) boyutlarında kullanılır.

Yardımcı yapıcıları sıklıkla bu kalıpta görünür:

```rust
let p = px(12.0);                  // Pixels
let r = rems(1.5);                 // Rems
let p2 = point(px(10.), px(20.));  // Point<Pixels>
let s = size(px(100.), px(40.));   // Size<Pixels>
let b = Bounds::from_corners(point(px(0.), px(0.)), point(px(100.), px(100.)));
```

**Diğer birimler.** Pixels dışındaki uzunluk tipleri farklı senaryolar için ayrılmıştır:

- `Rems(f32)` — kök yazı tipi boyutuna görelidir (Zed'de `theme.ui_font_size` ile bağlıdır). `.text_sm()` veya `.gap_2()` gibi makro üretimi yardımcılar genellikle Rems üzerinden Pixels üretir.
- `AbsoluteLength` — `Pixels` veya `Rems`.
- `DefiniteLength` — `Absolute(AbsoluteLength)` veya `Fraction(f32)`.
- `Length` — `Definite(DefiniteLength)` veya `Auto`.

Stil API'leri Length türünü kabul ettiği için farklı birimler aynı zincirde karışık kullanılabilir:

```rust
div().w(px(120.))           // Pixels
    .min_h(rems(2.))        // Rems
    .flex_basis(relative(0.5)) // Fraction
    .h_auto()
```

Jenerik kapsayıcı tipleri `Point<T>`, `Size<T>`, `Bounds<T>`, `Edges<T>`, `Corners<T>`, çoğu metot için aritmetik destekler (`+`, `-`, `*`, `/`).

**Kaynaktaki doğrudan metot yüzeyi.** Geometri tipleri pek çok yardımcı metoda sahiptir; aşağıdaki liste hangi tipin hangi araçları açtığını özetler:

- `Point<T>`: `map`, `scale`, `magnitude`, `relative_to`, `max`, `min`, `clamp`.
- `Size<T>`: `new`, `map`, `center`, `scale`, `max`, `min`, `full`, `auto`, `to_pixels`, `to_device_pixels`.
- `Bounds<T>`: `centered`, `maximized`, `new`, `from_corners`, `from_anchor_and_size`, `centered_at`, `top_center`, `bottom_center`, `left_center`, `right_center`, `intersects`, `center`, `half_perimeter`, `dilate`, `extend`, `inset`, `space_within`, `top`, `bottom`, `left`, `right`, `top_right`, `bottom_right`, `bottom_left`, `corner`, `contains`, `is_contained_within`, `map`, `map_origin`, `map_size`, `localize`, `is_empty`, `scale`, `to_device_pixels`, `to_pixels`.
- `Edges<T>`: `all`, `map`, `any`, `auto`, `zero`, `to_pixels`, `scale`, `max`. Birden fazla `zero` veya `to_pixels` uygulaması farklı jenerik özelleştirmelerden gelir.
- `Corners<T>`: `all`, `corner`, `to_pixels`, `scale`, `max`, `clamp_radii_for_quad_size`, `map`.
- Birim sarmalayıcıları: `Pixels::{as_f32, floor, round, ceil, scale, pow, abs, signum, to_f64}`, `ScaledPixels::{as_f32, floor, round, ceil}`, `Rems::{is_zero, to_pixels, to_rems}`, `DevicePixels::to_bytes`.

**Hazır oran sabitleri.** GPUI kullanışlı bir oran sabiti sunar:

- `phi() -> DefiniteLength` (`geometry.rs:3698`), altın oranı `relative(1.618_034)` olarak döndürür — yani üst öğenin **1.618 katı**, %50 değil. GPUI, varsayılan `TextStyle::line_height` değeri olarak `phi()` kullanır (`style.rs:451`); bir yazı tipi için satır yüksekliği `font_size * 1.618` olur. Yerleşim oranlamada (örneğin altın oranla iki sütun) üst öğenin katı olarak ifade gerekiyorsa aynı sabit kullanılabilir.

**Tuzaklar.** Geometri tarafında sıkça yapılan yanılgılar:

- `Bounds::contains(point)`, yarı açık aralıklara göre çalışır; sınır pikseli `false` dönebilir.
- `Pixels` ile `ScaledPixels` aritmetiği, `From` veya `Into` üzerinden açık dönüşüm ister; örtük çevirme yapılmaz.
- `point(x, y)` argüman sırası önce X sonra Y'dir; `size(width, height)` de aynı sırayı izler.

## Renkler, Gradient ve Background

`crates/gpui/src/color.rs` ve `colors.rs`.

GPUI renkleri iki temel tipte ifade eder:

- `Rgba { r, g, b, a }` — bileşenler 0.0 ile 1.0 arasında.
- `Hsla { h, s, l, a }` — bileşenler 0.0 ile 1.0 arasında.

Yapıcı çağrıları sıklıkla şu biçimde görünür:

```rust
let red = rgb(0xff0000);                    // Rgba, alfa 1.0
let translucent = rgba(0xff000080);         // 0xRRGGBBAA
let h = hsla(0.0, 1.0, 0.5, 1.0);           // saf kırmızı
let grey = opaque_grey(0.5, 1.0);           // gri yardımcısı
```

**Hazır renk sabitleri.** Tümü `pub const fn ... -> Hsla` biçiminde tanımlıdır (`color.rs:344+`):

| Fonksiyon | HSLA değeri | Not |
|---|---|---|
| `black()` | `(0.0, 0.0, 0.0, 1.0)` | Saf siyah |
| `white()` | `(0.0, 0.0, 1.0, 1.0)` | Saf beyaz |
| `transparent_black()` | `(0.0, 0.0, 0.0, 0.0)` | Tam saydam siyah — gradient ucu olarak kullanışlı |
| `transparent_white()` | `(0.0, 0.0, 1.0, 0.0)` | Tam saydam beyaz |
| `red()` | `(0.0, 1.0, 0.5, 1.0)` | %100 doygun kırmızı |
| `blue()` | `(0.666…, 1.0, 0.5, 1.0)` | %100 doygun mavi |
| `yellow()` | `(0.166…, 1.0, 0.5, 1.0)` | %100 doygun sarı |
| `green()` | `(0.333…, 1.0, **0.25**, 1.0)` | Diğerlerinden farklı: lightness 0.25 (koyu yeşil) |

Bu sabitler Zed tasarım sisteminden bağımsızdır; tema renklerine ihtiyaç duyulduğunda `cx.theme().colors()` kullanılır. Debug placeholder, GPU shader testi veya tema-bağımsız palet örnekleri gerektiğinde bu hazır sabitler iş görür. `transparent_black()` `linear_gradient` ucu olarak en yaygın kullanılan tek parça çağrıdır (örneğin fade-out maskeleri için).

**Sık kullanılan metotlar** (`color.rs:472+`):

- `is_transparent()`, `is_opaque()`
- `opacity(factor)` — alfayı çarpar.
- `alpha(a)` — alfayı doğrudan ayarlar.
- `fade_out(factor)` — alfayı in-place olarak azaltır.
- `blend(other)` — önceden çarpılmış alfa (`pre-multiplied alpha`) ile karıştırır.
- `grayscale()` — doygunluğu sıfırlar.
- `to_rgb()` — Hsla'dan Rgba'ya dönüşüm yapar.

**Background tipi.** Background yalnızca düz renk değildir; gradient ve desen de aynı çatı altındadır (`color.rs:763+`):

```rust
solid_background(rgb(0xffffff))
linear_gradient(
    angle_deg,
    linear_color_stop(rgb(0x000000), 0.0),
    linear_color_stop(rgb(0xffffff), 1.0),
)
checkerboard(rgb(0xeeeeee), 8.0)
pattern_slash(rgb(0xff0000), 2.0, 6.0)
```

`linear_gradient(...).color_space(ColorSpace::Oklab)` ile renk uzayı seçilebilir; `opacity(factor)` her stop'a uygulanır. `Background::as_solid()` yalnızca düz renk background için `Some(Hsla)` döndürür; gradient veya pattern için `None` döner.

`.bg(impl Into<Background>)`, her stil fluent API'sinde mevcuttur. Düz `Hsla` da `Into<Background>` uyguladığı için `.bg(theme.colors().panel_background)` çağrısı tipik bir kullanımdır.

**Pratik notlar.** Renk ve gradient kullanırken karşılaşılan yaygın durumlar:

- Alfa 0 olan bir rengin opak arka plan üstünde sonucu yine opak görünür; bu yüzden gerçekten saydam bir alan istenmiyorsa temadaki opak renk tercih edilir.
- Gradient durakları (`stop`) `0.0` ve `1.0` arasında sıralı verilmelidir; aksi halde GPU `shader`'ı beklenmedik bir dağılım çizebilir.
- Hsla'da hue 1.0'a sarılmaz, kırpılır (`clamp`); döndürme için `hue + delta`, modulo 1.0 ile hesaplanmalıdır.

## SharedString, SharedUri ve Ucuz Klonlanan Tipler

`SharedString` GPUI'nin `gpui_shared_string` re-export'udur; `SharedUri` ise `crates/gpui/src/shared_uri.rs` içinde bu string tipini sarar.

UI ağacı her çizimde yeniden oluşturulduğu için string ve URI kopyalama maliyeti hızla birikebilir. GPUI bu yükü azaltmak için `Arc` tabanlı tipler sunar:

- `SharedString` — `&'static str` veya `Arc<str>`. `Clone` ucuzdur (referans sayısı). `Display`, `AsRef<str>` ve `Into<SharedString>` uygulamaları hazırdır. `&'static str`, `String` ve `Cow<'_, str>` ücretsiz dönüşür.
- `SharedUri` — aynı stratejiyle URI tutar; `ImageSource::Resource(Resource::Uri(...))` `SharedUri` bekler.

Render içinde her seferinde `String` üretip kopyalamak yerine entity verisinde `SharedString` saklamak yaygın bir desendir:

```rust
struct Header { title: SharedString }

impl Header {
    fn set_title(&mut self, title: impl Into<SharedString>, cx: &mut Context<Self>) {
        self.title = title.into();
        cx.notify();
    }
}

impl Render for Header {
    fn render(&mut self, _: &mut Window, _: &mut Context<Self>) -> impl IntoElement {
        div().child(self.title.clone())
    }
}
```

**İlgili ucuz klon tipleri.** Aynı maliyet hassasiyetiyle hazırlanmış birkaç tip daha vardır:

- `Arc<str>`, `Arc<Path>`, `Arc<[T]>` — GPUI sıkça `Arc` tabanlı dilim ve dosya yolu bekler.
- `Hsla` ve `Rgba` — `Copy` tipler oldukları için doğrudan değer geçirilir.
- `ElementId` — `Clone`'dur ve içinde iç ID veya metin varyantları taşır.

**Tuzaklar.** Ucuz klon tiplerinden faydalanırken atlanan noktalar:

- `SharedString::from(String)` çağrısı bir kez bellek ayırması (`allocation`) yapar; sonraki klonlar ucuzdur. Sık çalışan yollarda tekrar tekrar `String` üretmekten kaçınılmalıdır.
- `to_string()` çağrısı yeni bir `String` bellek ayırması üretir; gerekmiyorsa `as_ref()` veya `Display` üzerinden yazmak daha ekonomiktir.
- Biçim metni (`format string`) her çizimde çalışıyorsa `format!` sonucu da her ekran karesinde bellek ayırması üretir; sonucu önbelleğe almak için entity verisinde tutmak gerekir.

## WindowAppearance ve Tema Modu

`crates/gpui/src/platform.rs:1604` içinde tanımlıdır:

```rust
pub enum WindowAppearance {
    Light,        // macOS: aqua
    VibrantLight, // macOS: NSAppearanceNameVibrantLight
    Dark,         // macOS: darkAqua
    VibrantDark,  // macOS: NSAppearanceNameVibrantDark
}
```

`Vibrant` varyantları, macOS `NSAppearance` değerleriyle doğrudan eşleşir. Diğer platformlar bu enum'u yine taşır; ancak `vibrancy`'nin gerçek etkisi platform uygulamasına bağlıdır. Sistem açık veya koyu kipi tercih ettiğinde GPUI bunu platform görünümü olarak yansıtır; kullanıcı elle tema üzerine yazmadığı sürece Zed teması bu sinyali takip eder.

**Erişim.** Tema kipini okumak ve değişimini izlemek için birkaç yol vardır:

- `cx.window_appearance() -> WindowAppearance` — uygulama geneli platform tercihi.
- `window.appearance() -> WindowAppearance` — pencerenin gerçek görünümü (üst öğe üzerine yazabilir).
- `window.observe_window_appearance(|window, cx| ...)` — entity verisine gerek olmayan doğrudan pencere gözlemcisi.
- `cx.observe_window_appearance(window, |this, window, cx| ...)` — `Context<T>` içinden değişimi view verisi ile birlikte izler.
- `window.observe_button_layout_changed(...)` ve `cx.observe_button_layout_changed(window, ...)` — platform pencere kontrol butonu düzeni değiştiğinde çalışır.

Zed örüntüsü `crates/zed/src/main.rs` içinde tema seçimine şu şekilde bağlanır:

```rust
cx.observe_window_appearance(window, |_, window, cx| {
    let appearance = window.appearance();
    *SystemAppearance::global_mut(cx) = SystemAppearance(appearance.into());
    theme_settings::reload_theme(cx);
    theme_settings::reload_icon_theme(cx);
}).detach();
```

**Tuzaklar.** WindowAppearance ile çalışırken dikkat edilecek noktalar:

- macOS dışında `VibrantLight` ve `VibrantDark` değerleri üretilmez; ancak eşleştirme tablolarında yine de dört değerin tamamı ele alınmalıdır.
- Sistem teması pencere açıldıktan sonra değişirse `window_background_appearance` otomatik tetiklenmez; tema akışında elle `window.set_background_appearance(...)` çağrısı gerekir.
- `Vibrant*` kipleriyle birlikte `WindowBackgroundAppearance::Blurred` kullanılırsa macOS'ta bulanıklığın üzerine ek bir `vibrancy` bindirilir; tasarım sisteminde bu katmanlardan birinin seçilmesi tutarlılık açısından önemlidir.

---
