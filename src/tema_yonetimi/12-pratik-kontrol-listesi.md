# Pratik kontrol listesi

Son bölüm, önceki adımlarda alınan kararları hata sınıflarına göre hızlıca taranabilir bir kontrol listesine çevirir. Her madde özet niteliğindedir; detay gerektiğinde ilgili konuya dönülmelidir.

---

## 45. Yaygın tuzaklar

Önceki bölümlerde geçen tuzaklar **tek bir yerde** toplanır. Her madde kısa bir özet içerir ve bölüm referansıyla bağlanır. Ayrıntılı tartışma için ilgili konuya dönülmelidir.

### Sözleşme katmanı (Bölüm IV ve V)

1. **Bilinmeyen alanların sessizce atlanması.** Mevcut Zed sözleşmesinde olmayan alanlar açık hataya düşmelidir. `style` alanı `flatten` kullandığı için bu kontrol açık anahtar allowlist'i ile yapılır. → Konu 22.
2. **JSON'da snake_case beklemek.** Zed `border.variant` yazar; `#[serde(rename = "border.variant")]` şarttır, yoksa alan sessizce boş kalır. → Konu 23.
3. **`Option<Hsla>` yerine `Option<String>` kullanmanın gerekçesi.** Sıkı tipli bir parse hatası tüm temayı bozar; iki katmanlı opsiyonellik (Content katmanı string + Refinement katmanı Hsla) sözleşmenin temelidir. → Konu 20.
4. **Bir alan grubunu "şimdilik gerek yok" diye atlamak.** `terminal_ansi`, debugger, diff hunk, vcs, vim, icon theme — UI'da okunmuyor olsa bile struct'ta bulunmalıdır. Aksi halde sözleşme delinir. → Konu 2 (Temel ilke), Konu 13.
5. **`#[refineable(...)]` attribute'unu atlamak.** Refinement tipi yalnızca `Default + Clone` türevini taşır; serde/JSON deserialize yolu kapanır. → Konu 11.

### Refinement (Bölüm VII)

6. **Türetme adımının atlanıp doğrudan `refine`'a geçilmesi.** Status alanlarında kullanıcı `error` yazar ama `error_background` yazmazsa, baseline'dan gelen karışım renkler ortaya çıkar. `apply_status_color_defaults` çağrısı şarttır. → Konu 31.
7. **`color()` helper'ında hata mesajının yutulması.** Production debug için `inspect_err` ile log eklenebilir; default'ta kapalı tutulur. → Konu 30.
8. **Baseline'ın yanlış appearance ile seçilmesi.** Light tema yüklenirken dark baseline kullanılması uyumsuz bir görüntü doğurur. Baseline `content.appearance` değerine göre seçilmelidir. → Konu 32.

### Runtime (Bölüm VIII)

9. **`cx.refresh_windows()`'ın çağrılmaması.** Tema değişir ama UI eski renkte kalır — en yaygın tema bug'ıdır. `GlobalTheme::update_theme + refresh_windows` her zaman bir çift olarak çağrılmalıdır. → Konu 38.
10. **`cx.notify()` ile yetinmek.** Yalnızca tek view'ı yeniler; oysa tema değişimi tüm pencerelerde geçerli olur. → Konu 38.
11. **`kvs_tema::init`'in atlanması.** `cx.theme()` panic atar. Bu çağrı uygulama girişinin **ilk** satırı olarak konumlanır. → Konu 36.
12. **`update_global` içinde `set_global` çağrılması.** Re-entrancy panic'i üretir. Update callback'i içinde yalnızca field mutate edilebilir. → Konu 34.
13. **`observe_window_appearance`'da `.detach()` unutulması.** Subscription drop edildiğinde observer ölür. → Konu 35.

### GPUI tipleri (Bölüm III)

14. **Hue'nun 0–360 aralığında yazılması.** GPUI hue 0–1 normalizedir; `hsla(210.0, ...)` aslında `hsla(210.0 / 360.0, ...)` biçiminde olmalıdır. → Konu 6.
15. **`Hsla::default()` ile struct doldurmak.** `(0,0,0,0)` görünmez bir değer üretir. 150 + 42 alanın tamamı açık değerlerle doldurulmalıdır. → Konu 6, 25.
16. **`opacity` ile `alpha` arasındaki farkın karıştırılması.** `opacity(x)` mevcut alpha'yı `* x` ile çarpar; `alpha(x)` ise doğrudan set eder. → Konu 6.
17. **`use kvs_tema::ActiveTheme;` import'unun unutulması.** `cx.theme()` çağrısı "method not found" hatası verir. Prelude kullanılması pratik bir çözüm sağlar. → Konu 41.

### Etkileşim (Bölüm X)

18. **`.id()` çağrısının atlanması.** Interactivity stateful'dur; ID olmadan hover/active çalışmaz ve sessiz bir başarısızlık ortaya çıkar. → Konu 42.
19. **`element_selected` ile `element_selection_background` karıştırılması.** İlki liste öğesi seçimini, ikincisi metin highlight'ını temsil eder. → Konu 42.
20. **Ghost ile element grubunun karıştırılması.** Toolbar'da `element_*` kullanıldığında yüzey rengiyle dolar ve tasarım dili kayar. → Konu 42.
21. **Mermaid kaynağında tema dışı stil taşımak.** `%%{init}%%`, elle `classDef` veya rastgele hex renkler aktif tema ile çakışır. Vurgu için `accent0..accent7` kullanılmalı; cache tema/settings değişiminde invalid edilmelidir. → Konu 41.
22. **Completion rozet ayarını tema ayarı sanmak.** `completion_menu_item_kind` `EditorSettingsContent` alanıdır; `ThemeSettingsContent` veya provider trait'ine eklenmez. Renk tüketimi syntax theme üstünden yapılır. → Konu 39, 41.
23. **Markdown preview metin fontu ile code fontunu birleştirmek.** Düz preview metni `markdown_preview_font_family`, inline code ve code block ise `markdown_preview_code_font_family` kullanır. İki alanın fallback hedefleri farklıdır. → Konu 39, 41.

### Bundling / lisans (Bölüm VI)

24. **`palette` sürümünün Zed'le aynı tutulmaması.** Renk dönüşümü kayar; fixture testleri kırılır. → Konu 5, 21.
25. **`refineable` dep'inin fork'lanmadan production'a alınması.** `publish = false` ayarı crates.io yayını engeller; vendor veya fork şarttır. → Konu 3, 26.
26. **Zed `default_colors.rs` HSL değerlerinin birebir kopyalanması.** GPL-3 ihlalidir. Bağımsız anchor değerleri seçilmelidir. → Konu 3, 25.
27. **Lisans dosyasını "sonradan eklerim" demek.** Bundled tema'ya atıf eklenmediğinde telif ihlali oluşur. → Konu 27.
28. **GPL lisanslı fixture'ın alınması.** Tema JSON'undaki HSL değerleri bile telif kapsamına girer; yalnızca MIT/Apache gibi uyumlu lisanslı temalar fixture'a alınır. → Konu 27.

### Test (Bölüm VI ve XI)

29. **`#[gpui::test]` `cx.update` içinde init yapmamak.** Test başında `kvs_tema::init(cx)` veya `init_test(cx)` çağrısı gerekir. → Konu 44.
30. **Hsla'nın `assert_eq!` ile karşılaştırılması.** Float eşitliği yanıltıcıdır; epsilon karşılaştırması tercih edilir. → Konu 28.
31. **`feature = "test-util"`'ın production'da açık bırakılması.** `#[cfg(any(test, feature = "test-util"))]` koşulu kurulur; release build'inde kapatılmalıdır. → Konu 44.

### API yüzeyi (Bölüm XI)

32. **`refinement.rs` modülünün public yapılması.** `pub use crate::refinement::*` sözleşmeyi sızdırır. `pub(crate) mod refinement;` zorunludur. → Konu 43.
33. **`schema::*` glob ile public ihraç.** Yeni iç tip otomatik olarak public hale gelir. Tek tek `pub use` yazılmalıdır. → Konu 43.
34. **`Theme.styles` alanının doğrudan public olması.** İç düzen sızıntısına yol açar; accessor metotlarının (`theme.colors()`) tercih edilmesi gerekir. → Konu 12, 41.

### Özet karar matrisi

Önceki pratik bölümünde toplanan "İleri öğeler" başlığı ilgili katmanlara dağıtılmıştır. Aşağıdaki tablo, mirror kararı için kısa kontrol listesidir:

| Zed öğesi | `kvs_tema` mirror gerekli mi? | Ne zaman gerekli? |
|-----------|-------------------------------|------------------|
| `LoadThemes` | Önerilir | Init API'si kapsamdaysa |
| `ThemeSettingsProvider` | Settings entegrasyonu için **gerekli** | Tema selector veya runtime ayar kapsamdaysa |
| `UiDensity` | Tema değil, settings — yine de mirror edilmesi gerekir | Spacing tutarlılığı için |
| `all_theme_colors` / `ThemeColorField` | Tema editörü/preview için **gerekli**, diğer durumlarda opsiyonel | Tema editörü yazıldığında |
| `ColorScale` ailesi | **Çoğu uygulama için gereksiz** | Yalnızca geniş tema varyant matrisi gerektiğinde |
| `apply_theme_color_defaults` | Gerekli (Konu 31'in ikiz fonksiyonu) | Tema refinement akışı kurulduğunda |
| `deserialize_icon_theme` | Trivial bir helper; sarmalanması önerilir | Icon tema yüklendiğinde |
| `FontFamilyCache` | Hayır — sözleşme dışıdır | — |
| `DiagnosticColors` | Editor render path'i kullanıyorsa **gerekli** | Editor entegre olduğunda |
| Registry sabitleri / typed error'lar | Registry mirror ediliyorsa **gerekli** | Registry kapsamdaysa |
| `default_color_scales` / `zed_default_themes` | Karara bağlıdır | Fallback ve scale mirror kararı verildiğinde |

Yukarıdaki öğeler kapsam kararına göre mirror edilir. Public API'ye alınmayan parçalar tüketici sözleşmesinin dışında kalır.

> **Referans:** Bölüm IV/Konu 14 (DiagnosticColors detayı), Bölüm VII/Konu 31 (apply_status_color_defaults + apply_theme_color_defaults), Bölüm VIII/Konu 34–38 ile Bölüm IX/Konu 39 (runtime/settings), Bölüm VI/Konu 25 (fallback tasarımı).

---

# Son

Bu rehber, `kvs_tema` ve `kvs_syntax_tema` crate'lerinin **tüm yüzeyini** 12 bölüm ve 45 konu altında, uygulama kurulum sırasını izleyerek toplar. Üç temel kural baştan sona geçerli kalır:

1. **Veri sözleşmesinde dışlama yok** — Zed'in tüm alanları mirror edilir (Konu 2).
2. **Lisans-temiz çalışma** — kod gövdesi GPL'den kopyalanmaz; yalnızca sözleşme paritesi korunur (Konu 3).
3. **Sözleşme sınırı** — public API ile crate-içi detaylar birbirinden ayrılır; tüketici yalnızca açıkça desteklenen yüzeye bağlanır.

Beklenmedik bir durumla karşılaşıldığında ilk bakılacak yer Bölüm XII'deki tuzaklar listesidir. Daha geniş tartışma için ilgili konuya geri dönülmelidir.

---
