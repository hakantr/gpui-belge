# Metin Sistemi

---

## Metin, Font ve Ölçüm

Metin sisteminin ana tipleri `crates/gpui/src/text_system.rs`, `style.rs` ve `elements/text.rs` içinde toplanır. Bir metni doğru çizebilmek için stil, font ve ölçüm tipleri birlikte çalışır. Her birinin sorumluluğu ayrıdır:

- `TextStyle` — renk, font ailesi, font boyutu, satır yüksekliği, kalınlık/stil, dekorasyon, boşluk, taşma, hizalama ve satır kısıtlaması gibi metin genelindeki görünüm parametrelerini taşır.
- `HighlightStyle` — belirli aralıklara uygulanacak kısmi (`partial`) stildir; taban stilin üstüne biner.
- `TextRun` — UTF-8 byte uzunluğu, font ve renk/dekorasyon bilgisini taşır. Run'ların toplam uzunluğu metnin byte uzunluğunu tam olarak karşılamak zorundadır.
- `StyledText` — `SharedString` ile birlikte run, vurgu ve font üzerine yazmalarını birleştirerek çizilir.
- `InteractiveText` — karakter veya aralık bazlı tıklama, hover ve tooltip davranışı sağlar.
- `Font`, `FontWeight`, `FontStyle`, `FontFeatures`, `FontFallbacks` — font seçimi ve özelliklerini tanımlayan yardımcı tiplerdir.

Pratikte stil ve vurgu birleşimi şu kalıpta görünür:

```rust
let text = StyledText::new("Error: missing field")
    .with_highlights([(0..5, HighlightStyle {
        color: Some(rgb(0xff0000).into()),
        font_weight: Some(FontWeight::BOLD),
        ..Default::default()
    })]);

div()
    .text_size(rems(0.875))
    .font_family(".SystemUIFont")
    .line_height(relative(1.4))
    .child(text)
```

**Metin ölçümü ve yerleşim.** Metnin ne kadar yer kapladığı ve aktif stil değeri aşağıdaki noktalardan okunur:

- `window.text_style()` — o anda kalıtılan (`inherited`) aktif metin stilini verir.
- `window.text_system()` — pencereye bağlı `WindowTextSystem` örneği.
- `App::text_system()` — global text system'a erişimi sağlar.
- `TextStyle::to_run(len)` — kalıtılan stilden run üretir.
- `TextStyle::line_height_in_pixels(rem_size)` — line-height değerini piksele çevirir.
- `window.line_height()` — aktif metin stiline göre satır yüksekliğini döndürür.

**Tuzaklar.** Metin sisteminde dikkat edilmesi gereken birkaç önemli nokta:

- Vurgu aralıkları byte aralığıdır; mutlaka UTF-8 karakter sınırlarına oturmalıdır. Çok-byte'lı karakterlerin ortasına denk gelen aralık hata üretir.
- `SharedString` kopyalama maliyetini azaltır; çizim alt öğelerinde `String` yerine bu tip tercih edilir.
- `text_ellipsis`, `line_clamp` ve `white_space` gibi taşma davranışları yerleşim genişliğine bağlıdır; üst öğenin genişliği belirsizse kırpma beklenen biçimde çalışmaz.
- Uygulamanın genel metin çizim kipi `cx.set_text_rendering_mode(...)` ile `PlatformDefault`, `Subpixel` ve `Grayscale` arasında seçilir. Subpixel akışında her glif için yatayda `gpui::SUBPIXEL_VARIANTS_X: u8 = 4`, dikeyde `gpui::SUBPIXEL_VARIANTS_Y: u8 = 1` farklı varyant rasterize edilir (`text_system.rs:45,48`); başka bir deyişle glif atlası boyutu yatay subpixel konumuna duyarlıdır, dikey konumda değildir.
- WGPU/Linux metin arka ucu (`CosmicTextSystem`) `Font.fallbacks` değerini font önbellek anahtarına dahil eder ve `layout_line` içinde kullanıcı yedek zincirini grapheme cluster sınırlarını koruyarak uygular. ASCII karakterlerinde birincil font tercih edilir; combining mark ve ZWJ emoji cluster'ları yedek aralığının içinde bölünmez. Özel font yedek ayarı incelenirken yalnızca aile adı değil yedek listesi de önbellek/ölçüm girdisi sayılmalıdır.

## StyledText, TextLayout ve InteractiveText

Basit bir metin doğrudan `SharedString` olarak bir elementin alt öğesi şeklinde verilebilir. Ölçüm, vurgu, font üzerine yazma veya tıklanabilir aralık gerektiğinde `StyledText` devreye girer. Tıklama ve hover gerekiyorsa `InteractiveText` kullanılır.

**`StyledText` kullanımı.** Vurgu ve font üzerine yazmaları fluent zincire eklenir:

```rust
let text = StyledText::new("Open settings")
    .with_highlights(vec![(0..4, highlight_style)])
    .with_font_family_overrides(vec![(5..13, "ZedMono".into())]);

let layout = text.layout().clone();
```

Önceden hesaplanmış `TextRun` listesi varsa vurguyu gecikmeli (`delayed`) uygulamak yerine `.with_runs(runs)` çağrısı yapılır. `with_default_highlights(&default_style, ranges)` ise üst öğe stili yerine açık bir `TextStyle`'ı baz alarak run üretir.

**Ölçüm sonrası `TextLayout`.** Yerleşim veya prepaint tamamlandıktan sonra elde edilen yerleşim nesnesi metin koordinatları üzerinde sorgu yapmaya izin verir:

- `index_for_position(point) -> Result<usize, usize>` — piksel konumundan UTF-8 byte index'i.
- `position_for_index(index) -> Option<Point<Pixels>>` — byte index'ten piksel koordinatı.
- `line_layout_for_index(index)`, `bounds()`, `line_height()`, `len()`, `text()`, `wrapped_text()`.

`TextLayout` değerleri yerleşim veya prepaint tamamlanmadan okunursa panic üretebilir. Bu nedenle ölçüm sonuçlarına ihtiyaç duyan kod olay işleyici ya da yerleşim sonrası akışta çalışır. Çizim sırasında henüz ölçülmemiş bir yerleşim nesnesine güvenilmez.

**`InteractiveText`.** Tıklama, hover ve tooltip ekleyen sarmalayıcıdır:

```rust
InteractiveText::new("settings-link", StyledText::new("Open settings"))
    .on_click(vec![0..13], |range_index, window, cx| {
        window.dispatch_action(OpenSettings.boxed_clone(), cx);
    })
    .on_hover(|index, event, window, cx| {
        update_hover(index, event, window, cx);
    })
    .tooltip(|index, window, cx| build_tooltip(index, window, cx))
```

Aralıklar yine byte index aralıklarıdır; Unicode metinde karakter sınırlarını yanlış hesaplamak hover ve tıklama eşleşmesini bozar. `on_click` yalnızca mouse down ile mouse up aynı verilen aralık içinde kaldığında dinleyiciyi tetikler. Yani bir aralıkta basıp başka bir aralıkta bırakmak tıklama sayılmaz.

**Markdown çizim davranışı.** Markdown ekosistemi metin sisteminin üzerine özel davranışlar bindirir; bunların bilinmesi sürpriz davranışları azaltır:

- Markdown görsel çizimi `StyledImage::with_fallback` kullanarak yüklenemeyen görsel için tıklanabilir bir "Failed to Load: ..." yedeği üretir. Yedek etiketi önce alt metin değerini, yoksa hedef URL'yi kullanır; tıklama ile `cx.open_url` çağrılır.
- Mermaid kod blokları yalnızca kapalı fenced block biçimindeyse diyagram olarak çıkarılır. ` ```mermaid` etiketinin yanı sıra `.mermaid` veya `.mmd` uzantılı kaynak yolu işaret edilen bloklar da diyagram olarak sayılır.
- Mermaid diyagram arayüzü önizleme ve kod sekmelerini, ayrıca kopyalama butonunu gösterebilir; çizim başarısızsa veya henüz tamamlanmadıysa kaynak kodu görünümü yedek olarak çizilir.

---
