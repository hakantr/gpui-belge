# Dış API ve test ortamı

Bu bölüm iki işi netleştirir: tüketiciye açılacak public API sınırı ve test ortamında tema mock düzeni. İyi çizilmiş bir public API sınırı, mevcut Zed sözleşmesinde hangi parçaların bilinçli olarak açıldığını gösterir. İyi kurulmuş bir test ortamı ise bileşenlerin tema değişimi karşısında doğru davrandığını sürdürülebilir biçimde doğrular.

---

## 43. Public API kataloğu ve crate-içi sınır

`kvs_tema` ve `kvs_syntax_tema` crate'lerinin **dışa açtığı** sözleşme ile **iç implementasyon detayları** arasındaki ayrımı netleştirir. Bu sınırın amacı, tüketicinin yalnızca mevcut uygulama için desteklenen yüzeye bağlanmasını sağlamaktır.

### Sınır felsefesi

| Kategori | Public mi? | Kim kullanır | Not |
|----------|------------|--------------|-----|
| **Public API** | ✓ | Tüketici UI bileşenleri | Uygulamanın desteklediği açık yüzey |
| **Schema API** | Kısmen | Tema yükleme/import kodu | Mevcut Zed JSON sözleşmesine bağlı |
| **Crate-içi (`pub(crate)`)** | ✗ | Yalnızca `kvs_tema` modülleri | Dış tüketiciye açılmaz |

### `kvs_tema` public API kataloğu

**Veri tipleri (public):**

```rust
pub use crate::Theme;            // accessor metotlarıyla okunur (Konu 12)
pub use crate::ThemeFamily;
pub use crate::Appearance;

pub use crate::styles::ThemeColors;
pub use crate::styles::StatusColors;
pub use crate::styles::PlayerColors;
pub use crate::styles::PlayerColor;
pub use crate::styles::AccentColors;
pub use crate::styles::SystemColors;

// Icon tema sözleşmesi (Konu 18)
pub use crate::icon_theme::{
    IconTheme, IconThemeFamily, IconDefinition,
    DirectoryIcons, ChevronIcons,
};
```

> **`ThemeStyles` listede yer almaz.** `Theme.styles` alanı `pub(crate)` olduğundan tüketici okumalarını accessor üzerinden yapar (`theme.colors()`, `theme.status()`). `ThemeStyles` tipinin tüketici tarafından import edilmesi bu nedenle anlamlı değildir.

**Runtime (public):**

```rust
pub use crate::runtime::ActiveTheme;             // cx.theme() için trait
pub use crate::runtime::GlobalTheme;
pub use crate::runtime::SystemAppearance;
pub use crate::runtime::init;
pub use crate::runtime::temayi_degistir;         // tek-yönlü helper (Konu 38)
pub use crate::runtime::temayi_yeniden_yukle;    // disk reload (Konu 38)
pub use crate::runtime::observe_system_appearance;  // pencere observer (Konu 35)
```

**Registry (public):**

```rust
pub use crate::registry::{
    ThemeRegistry, ThemeMeta, ThemeNotFoundError, IconThemeNotFoundError,
};
```

**Fallback (public, namespace altında):**

```rust
// kvs_tema::fallback::kvs_default_dark()
// kvs_tema::fallback::kvs_default_light()
pub mod fallback;   // pub modül; içeride sadece public fonksiyonlar
```

**Schema (Zed sözleşmesine bağlı — tek tek ihraç):**

```rust
// Glob YASAK — yeni iç tip eklendiğinde istemeden public olmaması için
// tek tek ihraç:
pub use crate::schema::{
    AppearanceContent, FontStyleContent, FontWeightContent,
    HighlightStyleContent, PlayerColorContent, StatusColorsContent,
    ThemeColorsContent, ThemeContent, ThemeFamilyContent,
    ThemeStyleContent, WindowBackgroundContent,
    try_parse_color,
};

// Icon tema Content tipleri (Konu 18)
pub use crate::icon_theme::{
    IconThemeContent, IconThemeFamilyContent,
    DirectoryIconsContent, ChevronIconsContent, IconDefinitionContent,
};
```

> **Schema public kullanım için ana kapı değildir:** Bu tipler mevcut Zed JSON sözleşmesini taşır. Tüketici doğrudan `ThemeColorsContent` kullanırsa JSON ayrıntılarına bağlanmış olur. Mecburiyet yoksa tema yükleme işlemleri `Theme::from_content` ve registry helper'ları üzerinden yürütülmelidir.

**`kvs_syntax_tema` public API:**

```rust
pub use SyntaxTheme;
```

Tek bir tip yeterli olur. `HighlightStyle` zaten GPUI'den gelir; re-export gereksizdir.

### Crate-içi (NON-public) modüller

| Modül | İçerik | Erişim |
|-------|--------|--------|
| `refinement.rs` | `theme_colors_refinement`, `status_colors_refinement`, `apply_status_color_defaults`, `color()` helper | `pub(crate)` — yalnızca `Theme::from_content` çağırır |
| `runtime` iç detayları | `GlobalThemeRegistry`, `GlobalSystemAppearance` newtype'ları | `pub(crate)` veya `pub(super)` |

**Neden gizli tutulur?** Refinement davranışı tema üretim hattının iç adımıdır. Tüketici bu fonksiyonları doğrudan çağırırsa, `Theme::from_content` ile tek yerde tutulması gereken sıra ve default kuralları dağılır.

### Lib kökü yapısı (`kvs_tema/src/kvs_tema.rs`)

Konu 4'teki iskeletle aynı yapıdır. Burada özet biçimde sunulur:

```rust
//! kvs_tema — Zed-uyumlu, lisans-temiz tema sistemi.

// Crate-içi (gizli)
pub(crate) mod refinement;          // Bölüm VII

// Namespace altında public (kvs_tema::fallback::*)
pub mod fallback;

// Private dosya modülleri, içeriği seçici biçimde ihraç edilir
mod icon_theme;
mod registry;
mod runtime;
mod schema;
mod styles;

// Glob ihraç — public modüller
pub use crate::icon_theme::*;
pub use crate::registry::*;
pub use crate::runtime::*;
pub use crate::styles::*;

// Schema — KARARSIZ; tek tek ihraç, glob asla
pub use crate::schema::{
    AppearanceContent, FontStyleContent, FontWeightContent,
    HighlightStyleContent, PlayerColorContent, StatusColorsContent,
    ThemeColorsContent, ThemeContent, ThemeFamilyContent,
    ThemeStyleContent, WindowBackgroundContent,
    try_parse_color,
};

// Lib kökünde tanımlı tipler (zaten public)
// → Theme, ThemeFamily, Appearance
// → ThemeStyles tipi crate-içi; ihraç edilmez (Konu 12)
```

**İzlenen kalıp:**

- `pub(crate) mod refinement` — `refinement.rs` dışarıya kapalıdır, içeride paylaşılır.
- `pub use module::*` — public modüller için tümünü ihraç eden bir yol; her iç tipin public olması isteniyorsa glob, istenmiyorsa tek tek kullanılır.
- `pub use schema::{...}` — schema modülü için **glob YASAK**; çünkü iç implementasyon tipleri (örneğin `serde` helper'ları) istenmeden public hale gelmemelidir.
- `pub mod fallback` — fallback fonksiyonları bir namespace altında toplanır (`kvs_tema::fallback::kvs_default_dark`). `fallback.rs` içinde yalnızca bu iki fonksiyon `pub`'tır; gerisi `pub(super)` veya private kalır.

### `prelude` modülü (önerilen)

```rust
// kvs_tema/src/prelude.rs
//! kvs_tema prelude — sık kullanılan tipleri tek import'a indirir.

pub use crate::runtime::ActiveTheme;
pub use crate::{Appearance, Theme, ThemeFamily};
pub use crate::styles::{
    AccentColors, PlayerColor, PlayerColors,
    StatusColors, SystemColors, ThemeColors,
};
```

Tüketici tarafından kullanım:

```rust
use kvs_tema::prelude::*;
// Theme, ActiveTheme, ThemeColors, vb. hepsi mevcut
```

> **Prelude'a `ThemeRegistry`/`GlobalTheme` eklenmemiştir.** Bu tipler uygulama init ve admin kodu için anlam taşır; UI bileşenleri kullanmaz. Render path prelude'unun hafif tutulması yerinde olur.

### Referans güncelleme notu

`kvs_tema` hedeflediği Zed referansını değiştirdiğinde public export listesi, schema tipleri, fixture dosyaları ve snapshot testleri aynı çalışma içinde güncellenmelidir. Sözleşme değişimini yalnızca bir yerde yapmak yeterli değildir; aksi halde belge, test ve runtime modeli farklı Zed durumlarını tarif eder.

### Tüketici sözleşmesi

Tüketici kodun güvenle yapabileceği işlemler:

✓ `cx.theme().colors().background` okunabilir ✓ `kvs_tema::Theme`, `kvs_tema::ThemeColors` import edilebilir ✓ `ThemeRegistry::global(cx)` ile registry sorgulanabilir ✓ `kvs_tema::init(cx)` çağrılabilir ✓ `kvs_tema::temayi_degistir(ad, cx)` ile tema değiştirilebilir ✓ `kvs_tema::temayi_yeniden_yukle(yol, cx)` ile disk'ten reload yapılabilir ✓ `kvs_tema::observe_system_appearance(window, cx)` ile sistem mod takibi kurulabilir ✓ `kvs_tema::SystemAppearance::global(cx)` ile sistem mod sorgulanabilir ✓ `GlobalTheme::icon_theme(cx)` ile aktif icon tema okunabilir ✓ `ThemeRegistry::global(cx).list_icon_themes()` ile icon tema seçenekleri listelenebilir

Tüketicinin yapamayacağı işlemler (compile hatası alır):

✗ `theme.styles.colors.X` zinciri — `styles` `pub(crate)`, accessor şarttır ✗ `GlobalThemeRegistry` veya `GlobalSystemAppearance` newtype'larına doğrudan dokunulması (private)

Tüketicinin yapmaması gereken işlemler (compile geçer ama kötü pratik):

✗ `kvs_tema::ThemeColorsContent` üzerinde UI davranışı kurmak — schema JSON ayrıntısıdır (Konu 19). ✗ `kvs_tema::try_parse_color(...)` çağrısının doğrudan yapılması — `Theme::from_content` zaten bu işlemi sarmalar. ✗ Tema yüklenirken `baseline`'ın yanlış appearance ile seçilmesi (Konu 32).

### `pub(crate)` ile gerçek izolasyon

`pub use crate::refinement::*` yazıldığında **`refinement` modülü public hale gelir**; istenmediği halde dışa açılır. `pub(crate) mod refinement;` ile modülün gizli tutulması ve `Theme::from_content` impl bloğu içinde doğrudan çağrılması doğru yaklaşımdır:

```rust
// kvs_tema.rs
pub(crate) mod refinement;

// Theme::from_content impl içinde (crate-içi)
use crate::refinement::{theme_colors_refinement, ...};
```

`refinement.rs`'in fonksiyonları `pub` olabilir; modülün kendisi `pub(crate)` olduğu için dış dünyaya görünmez kalır. Bu, pratik bir API izolasyonu sağlar.

### Tüketici API sınırı örneği

`cargo doc --no-deps` çalıştırıldığında **yalnızca istenen tiplerin** görünmesi beklenir:

```sh
cargo doc -p kvs_tema --no-deps --open
```

Doc sayfasında `theme_colors_refinement` veya `apply_status_color_defaults` görünüyorsa `pub use` zincirinin gözden geçirilmesi gerekir; bir sızıntı söz konusudur.

### Tuzaklar

1. **`pub use crate::*` içinde `refinement`'in yakalanması**: `pub use crate::refinement::*` yazıp ardından modülün `pub(crate)` olarak tutulması bir compile hatasına yol açar ("re-exporting private module"). İki tarafın tutarlı olması gerekir.
2. **`Theme.styles` alanının public yapılması (`pub`)**: Bu durumda alan public hale gelir ve iç düzen sızar. Accessor metotları sunulduktan sonra `styles` `pub(crate)` olarak tutulur ve okumalar yalnızca accessor üzerinden yapılır.
3. **`schema::*` glob ile public ihraç**: Schema tiplerinin tamamı dışa açılır; iç tipler de otomatik olarak public olur. Bunun yerine tek tek `pub use` yapılması doğru tercihtir:
   ```rust
   pub use crate::schema::{
       ThemeFamilyContent, ThemeContent, ThemeStyleContent,
       ThemeColorsContent, StatusColorsContent, PlayerColorContent,
       HighlightStyleContent, AppearanceContent, WindowBackgroundContent,
       FontStyleContent, FontWeightContent,
       try_parse_color,
   };
   ```
4. **Public yüzeyi belirsiz bırakmak**: Tüketici hangi modüle bağlanacağını bilmiyorsa schema ve refinement detaylarına kayar. Public export listesi bu yüzden açık tutulmalıdır.
5. **`prelude::*` glob importlarının endüstri görüşü**: Bazı stil rehberleri glob'u kabul etmez. Bu konudaki karar tüketiciye bırakılır; prelude sunulur, kullanılması veya elle import edilmesi geliştiricinin tercihinde kalır.
6. **`pub` accessor metotlarının yazılmaması**: `theme.colors().background` gibi accessor olmadan `styles` alanına doğrudan erişim zorunlu hale gelir; iç düzen değiştiğinde tüm tüketici kodu kırılır. Accessor yazılması şarttır.
7. **Public API'ye `kvs_default_dark` doğrudan koymak**: `pub use fallback::*` yerine namespace altında tutmak (`pub mod fallback { pub use crate::fallback::{...}; }`) niyetin açık görünmesini sağlar — tüketici `kvs_tema::fallback::kvs_default_dark` ifadesini yazarak amaca işaret eder.

### Son public yüzey — küçük ama atlanabilir parçalar

Aşağıdaki öğeler ana sözleşme kadar büyük görünmeyebilir, ama tema crate'i birebir mirror edilecekse public yüzey kararlarına dahil edilmelidir. Listeyi isim ezberiyle değil, owner/metot/alan ilişkisiyle okumak daha yararlıdır: `content = runtime`, `reflection ⊆ runtime`.

| Öğe | Kaynak | Karar |
| :-- | :-- | :-- |
| `DEFAULT_DARK_THEME` (`theme.rs:45`) ve `DEFAULT_LIGHT_THEME`, `DEFAULT_DARK_THEME` (`settings_content/src/theme.rs:282-283`) | `theme.rs` + `settings_content` | İki ayrı tanım vardır: `theme.rs` yalnızca `DARK`'ı re-export eder, `settings_content` ikisini de tanımlar (`'One Light'` / `'One Dark'`). Mirror'da **tek kaynak**: `kvs_ayarlari_icerik::theme` sabitleri, `kvs_tema` `pub use` ile yeniden açar |
| `CLIENT_SIDE_DECORATION_ROUNDING`, `CLIENT_SIDE_DECORATION_SHADOW` (`px(10.0)`) | `theme.rs:48-50` | Tema renk sözleşmesi değildir; client-side window decoration shadow/corner radius token'ıdır. `kvs_pencere`/`kvs_ui` tarafına ayrılır; değer Zed referansıyla birebir tutulur (`px(10.0)`) |
| `DEFAULT_ICON_THEME_NAME` | `icon_theme.rs` | Aktif icon theme fallback seçimi için gereklidir; icon theme registry mirror edildiğinde taşınmalıdır |
| `ThemeNotFoundError`, `IconThemeNotFoundError` | `registry.rs` | Registry public API'sinin typed error yüzeyidir; string error'a düşürülmemesi gerekir |
| `zed_default_themes()` | `fallback_themes.rs` | Test/fallback family üretir; GPL kod gövdesi kopyalanmaz, bağımsız bir `kvs_default_themes()` yazılır |
| `default_color_scales()` | `default_colors.rs` | 33 scale set'lik palet matrisidir; yalnızca `ColorScale` mirror kararı verildiğinde taşınır |
| `Theme::darken(color, light_amount, dark_amount)` | `theme.rs` | Appearance'a göre lightness azaltan bir yardımcıdır; bileşen tarafında kullanılacaksa helper olarak mirror edilebilir |
| `SystemAppearance::global_mut(cx)` | `theme.rs` | Sistem görünümünü testte veya platform event'inde güncellemek için bir kapıdır |
| `ThemeRegistry::insert_themes`, `.clear()` | `registry.rs` | Test, import ve kullanıcı tema yenileme akışında gerekir |
| `ThemeRegistry::register_test_themes`, `.register_test_icon_themes` | `registry.rs` | `test-support` gated helper'dır; üretim API'si olarak expose edilmez |
| `Redistributable değildir: FontFamilyCache` | `font_family_cache.rs` | Sözleşmenin dışında kalır, ancak public metotları 43.8'de not edilmiştir |
| `ThemeSettingsProvider::ui_font`, `.buffer_font` | `theme_settings_provider.rs` | Provider font objesini de verir; yalnızca font size/density ile sınırlı değildir |
| `ThemeSettingsContent` typography alanları | `settings_content/src/theme.rs` | `FontSize`, `FontFamilyName`, `FontFeaturesContent`, `BufferLineHeight`, `CodeFade` Konu 39'a eklenmiştir |
| `ThemeSelection::mode`, `IconThemeSelection::mode` | `theme_settings/src/settings.rs` | Selector UI'nın hangi slot'un aktif olduğunu göstermesi için gereklidir |
| `GlobalTheme::new`, `.update_theme`, `.update_icon_theme`, `.theme`, `.icon_theme` | `theme.rs` | Zed public API'sinin tamamıdır. `set_theme_and_icon`, `set_theme`, `set_icon_theme` metotları **yoktur** — set yerine `cx.set_global(GlobalTheme::new(...))` ile init yapılır, sonra `update_*` ile değişim yürütülür |
| `theme_settings::settings::set_theme`, `set_icon_theme`, `set_mode` | `theme_settings/src/settings.rs:233-315` | Kullanıcı `SettingsContent`'ini mutate eden public mutator helper'lardır. Runtime global'i **değil**; selector confirm akışında dosya yazımından önce çağrılır. Mirror tarafında `kvs_secici` veya `kvs_tema_ayarlari` crate'inde yer alır |
| `theme_settings::settings::appearance_to_mode`, `default_theme` | `theme_settings/src/settings.rs:30, 89` | `Appearance` ↔ `ThemeAppearanceMode` köprüsü ve fallback ad seçimi; selector/observer akışında kullanılır |
| `theme_settings::theme_settings::reload_theme`, `reload_icon_theme`, `load_user_theme`, `deserialize_user_theme`, `refine_theme_family`, `refine_theme`, `merge_player_colors`, `merge_accent_colors`, `increase_buffer_font_size`, `decrease_buffer_font_size` | `theme_settings/src/theme_settings.rs:185-428` | Zed'in JSON → Theme pipeline'ının ve runtime tema reload akışının kanonik fonksiyonlarıdır. Konu 32 + Konu 38'de ayrıntılı işlenir |
| `theme_settings::settings::adjust_buffer_font_size`, `reset_buffer_font_size`, `adjust_ui_font_size`, `reset_ui_font_size`, `adjust_agent_ui_font_size`, `reset_agent_ui_font_size`, `adjust_agent_buffer_font_size`, `reset_agent_buffer_font_size`, `clamp_font_size`, `adjusted_font_size`, `observe_buffer_font_size_adjustment`, `setup_ui_font` | `theme_settings/src/settings.rs` | Runtime font ölçekleme override global'leri ve helper'larıdır (Konu 39). Settings UI ve shortcut tarafında kullanılır |
| `ThemeSettings::*_font_size_settings`, `markdown_preview_font_family`, `markdown_preview_code_font_family` | `theme_settings/src/settings.rs:412-455` | Settings dosyasındaki baz değerleri override global'lerinden ayıran accessor'lardır. Markdown preview text fontu UI fontuna, preview code fontu buffer fontuna fallback eder. `theme_settings::init` observer'ı `*_settings()` değerlerini izleyip runtime font override global'lerini sıfırlar |
| `AgentUiFontSize`, `AgentBufferFontSize` newtype global'leri | `theme_settings/src/settings.rs:108, 114` | Agent panel font override'ı için `Global`-impl'lenmiş `Pixels` newtype'larıdır (Konu 39). `BufferFontSize` private, `UiFontSize` `pub(crate)`'tir; root `pub use` listesine girmez |
| `theme_settings::schema::syntax_overrides`, `theme_colors_refinement`, `status_colors_refinement` | `theme_settings/src/schema.rs:40, 237, 65` | Content → Refinement dönüşüm fonksiyonlarının kanonik adlarıdır. **`theme_colors_refinement` 3 parametrelidir** (`this`, `status_colors`, `is_light`); fallback zincirleri ve diff-hunk opacity'leri Konu 30'da tam tablo halinde verilir |
| `theme_colors_refinement` fallback zincirleri | `theme_settings/src/schema.rs:237-876` | `version_control_* → status.*`, `minimap_thumb_* → scrollbar_thumb_*` (alpha max `0.7`), `panel_overlay_* → panel_background` (+ `element_hover` blend), document highlight / Vim / Helix fallback'leri ve `editor_diff_hunk_* → version_control_* × LIGHT/DARK_DIFF_HUNK_*_OPACITY`. Konu 30 tablosu eksiksiz listeler |
| `LIGHT_DIFF_HUNK_FILLED_OPACITY`, `DARK_DIFF_HUNK_FILLED_OPACITY`, `LIGHT_DIFF_HUNK_HOLLOW_BACKGROUND_OPACITY`, `DARK_DIFF_HUNK_HOLLOW_BACKGROUND_OPACITY`, `LIGHT_DIFF_HUNK_HOLLOW_BORDER_OPACITY`, `DARK_DIFF_HUNK_HOLLOW_BORDER_OPACITY` | `theme_settings/src/schema.rs:16-21` | Referans değerler sırasıyla `0.16 / 0.12 / 0.08 / 0.06 / 0.48 / 0.36`. `pub(crate)` sabitlerdir, tüketici görünür sözleşme değildir; sayıların mirror'da `kvs_tema_ayarlari` modülünde aynı tutulması gerekir |
| `apply_theme_color_defaults` kaynak rengi | `fallback_themes.rs:47-58` | `text_accent × 0.25` **değil** — `player_colors.local().selection`; alpha 1.0 ise 0.25'e çekilir, aksi halde olduğu gibi atanır. Konu 31'de ayrıntılıdır |
| `refine_theme` adım sırası | `theme_settings/src/theme_settings.rs:275-353` | `status_colors_refinement → apply_status_color_defaults → status.refine → merge_player_colors → theme_colors_refinement(c, &status_ref, is_light) → apply_theme_color_defaults(&player) → colors.refine → merge_accent_colors → inline syntax map + Arc::new(SyntaxTheme::new(...))`. Konu 32'de ayrıntılı |
| `pub fn syntax_overrides(style) -> Vec<(String, HighlightStyle)>` | `theme_settings/src/schema.rs:40` | Override map'i `IndexMap<String, HighlightStyleContent>` → `Vec<(String, gpui::HighlightStyle)>` çevirir. 4 alan (`color`, `background_color`, `font_style`, `font_weight`) parse edilir; underline/strikethrough/fade_out `Default::default()`'tan gelir. `ThemeSettings::modify_theme` içindeki `SyntaxTheme::merge`'in beklediği imzadır |
| `ThemeSettings::modify_theme` override akışı | `theme_settings/src/settings.rs:472-491` | Full `refine_theme` değildir: `apply_status_color_defaults` veya `apply_theme_color_defaults` çağırmaz; status/theme refinement'ı doğrudan uygular, player/accent merge eder ve syntax için `SyntaxTheme::merge(base, syntax_overrides(...))` kullanır |
| `SyntaxTheme::merge(base, overrides) -> Arc<Self>` | `syntax_theme/src/syntax_theme.rs:96` | **Field-bazlı merge**: aynı capture varsa `new.<f>.or(existing.<f>)` ile birleştirir; capture yeni ise listenin sonuna eklenir. Override boşsa baseline klonsuz döner. Konu 18 + settings seviyesi theme override akışında kanonik kullanımdır |
| `ThemeName`, `IconThemeName`, `ThemeAppearanceMode`, `FontFamilyName` re-export zinciri | `settings_content/src/theme.rs:282, 501, 507` → `theme_settings::settings::pub use settings::{...}:13` | Owner `settings_content`'tir; `theme_settings` `pub use` ile köprü kurar. Mirror'da `kvs_ayarlari_icerik` (veya muadili) tek kaynaktır, `kvs_tema_ayarlari` yalnızca `pub use` ile yeniden açar — aksi halde aynı tipin iki yerde tanımlanması bug'ı doğar |
| `Theme.styles` görünürlüğü | `theme.rs:216` | Zed'de **`pub styles: ThemeStyles`** (alan-bazlı). Yerel tasarım accessor disiplini için `pub(crate)` seçilebilir; bu yerel bir sıkılaştırmadır. Zed paritesi `pub`'tır |
| `ThemeFamily` alan görünürlüğü | `theme.rs:192-204` | `id`, `name`, `author`, `themes`, `scales` alanlarının tamamı `pub`'tır. `scales: ColorScales` alanı hedeflenen referansta 33 paleti taşır |
| `PlayerColors::color_for_participant(idx)` | `styles/players.rs:147` | Collab kullanıcı renkleri için `modulo (len - 1) + 1` offset'li lookup; **local slot'u (idx 0) atlar**, participant 0 listenin idx 1'inden başlar. Konu 15'te işlenmiştir, 43.9'da resmi imzası verilir |
| `PlayerColors::agent()`, `.absent()` davranışı | `styles/players.rs:130-136` | İkisi de `self.0.last().unwrap()` döner — yani **agent ve absent player aynı slot'tur** (listenin son elementi). Fallback temalarda son slot'un boş bırakılmamasının nedeni budur |
| `theme::init` davranışı | `theme.rs:96-116` | `SystemAppearance::init → ThemeRegistry::set_global(assets, cx) → FontFamilyCache::init_global → default_global → themes.get(DEFAULT_DARK_THEME) (yoksa list().next()) → default_icon_theme().unwrap() → cx.set_global(GlobalTheme { theme, icon_theme })`. `GlobalTheme::new` çağrısı yerine doğrudan struct literal kullanılır (her ikisi de mümkündür) |
| `Appearance::is_light`, `From<WindowAppearance>` | `theme.rs:63-78` | `is_light` `matches!(self, Light)`'tan kısadır. `From<WindowAppearance>`: `Dark` ve `VibrantDark` → `Dark`; `Light` ve `VibrantLight` → `Light` (vibrant varyantları normal mapping'e düşer) |
| `SystemAppearance::Default = Dark` | `theme.rs:142-146` | Sistem görünümü alınamadığında default Dark değeri alınır. Mirror tarafında aynı varsayılan tutulmalıdır; aksi halde init önce ekrana light tema gelir, sonra sistem dark olduğunda dark'a sıçrama oluşur |
| `theme_settings::settings::BufferLineHeight` (`Comfortable`, `Standard`, `Custom(f32)`), `value()` | `theme_settings/src/settings.rs:340-369` | `Standard = 1.3`, `Comfortable = 1.618`, `Custom` doğrudan değer alır. Settings tarafında `f32 >= 1.0` kısıtı vardır (`settings_content/src/theme.rs:452`) |
| `try_parse_color` (`theme/src/schema.rs:17`) | `theme/src/schema.rs` | Konu 21'de ayrıntılıdır; 43.9'da public helper olarak teyit edilir |
| `gpui::Rgba::try_from(&str)` desteklediği formatlar | `gpui/src/color.rs:162-256` | **Dört format**: `#rgb` (3 hane, çiftleme), `#rgba` (4 hane, çiftleme), `#rrggbb` (6 hane), `#rrggbbaa` (8 hane). `#` zorunludur; trim yapılır. Konu 21'de ayrıntılı |
| `AppearanceContent` (`theme/src/schema.rs:11`) | `theme/src/schema.rs` | JSON `appearance` alanının enum mirror'ı (`Light` / `Dark`); Konu 19'da kullanılır |
| `theme_settings::init` observer 7 değişken tracking'i | `theme_settings/src/theme_settings.rs:85-142` | Settings değişiminde `reset_*_font_size` (4), `reload_theme` (theme_name veya overrides) (1+1=2), `reload_icon_theme` (1). Toplam 7 izleme alanı vardır; eksik tracking font size değerlerinin stale kalmasına yol açar. Konu 38'de ayrıntılı |
| `theme_settings::init` 2 katmanlı init paritesi | `theme_settings/src/theme_settings.rs:64-83` | `theme::init` (registry + fallback dark + GlobalTheme) çağrıldıktan sonra `set_theme_settings_provider`, `load_bundled_themes` (LoadThemes::All ise), `configured_theme` ile settings'ten gelen seçim çözülür ve `GlobalTheme::update_theme` + `update_icon_theme` ile aktif tema atanır. Mirror'da iki ayrı `init` fonksiyonunun korunması gerekir; tek `kvs_tema::init`'te birleştirilmesi bağımlılık matrisini bozar |
| `Theme.styles.syntax` baseline davranışı | `theme_settings/src/theme_settings.rs:313-331` | `refine_theme` `Arc::new(SyntaxTheme::new(syntax_overrides))` çağırır — yani baseline syntax kullanılmaz, **tema JSON'unda syntax bloğu boşsa sonuç boş syntax theme** olur. Yazarın `syntax: { ... }` doldurması beklenir; aksi halde editor highlight'sız kalır. Konu 32'de ayrıntılı |
| `ThemeSettings::modify_theme` görünürlüğü | `theme_settings/src/settings.rs:472` | Aslında `fn` — **private**. `pub fn apply_theme_overrides(&self, arc_theme: Arc<Theme>) -> Arc<Theme>` üzerinden çağrılır; tüketici doğrudan `modify_theme` çağıramaz. Public yüzey `apply_theme_overrides`'tır |
| `ThemeRegistry::new` davranışı | `theme/src/registry.rs:101-123` | Constructor **zaten dolu döner**: `insert_theme_families([zed_default_themes()])` çağrısı ve `default_icon_theme()` (`DEFAULT_ICON_THEME_NAME`) icon haritasına yerleştirme yapar. Mirror tarafında da aynı garanti tutulmalıdır; aksi halde `default_icon_theme()` çağrısı `Err` döner |
| `ThemeRegistry::list_names()` sıralama | `theme/src/registry.rs:181-185` | `Vec<SharedString>` döner ve **sıralıdır** (`names.sort()`). Buna karşılık `list()` (`Vec<ThemeMeta>`) **sıralı değildir** — HashMap.values() üzerinden doğrudan map'lenir. Selector UI sıralı liste isterse `list_names` veya manuel sort gerekir |
| `ThemeRegistry::clear()` davranışı | `theme/src/registry.rs:176-178` | Yalnızca `themes` HashMap'ini temizler — `icon_themes` HashMap'ine **dokunmaz**. Test/reset senaryolarında icon themes ayrıca `remove_icon_themes(...)` ile temizlenmelidir |
| `ThemeRegistry::load_icon_theme` baseline merge davranışı | `theme/src/registry.rs:250-330`, `file_icons/src/file_icons.rs:89-164` | Yüklenen icon theme'in `file_stems`, `file_suffixes`, `named_directory_icons` haritaları **default icon theme üstüne extend edilir**. `file_icons`, `directory_icons`, `chevron_icons` constructor'da default'tan kopyalanmaz; UI lookup eksik dosya tipi, klasör ve chevron path'lerinde `file_icons` crate'i üzerinden default icon theme'e düşer. Her tema için yeni UUID atanır |
| `icon_theme_schema` re-export sınırı | `theme/src/theme.rs:15, 36`; `theme/src/icon_theme_schema.rs:10-49` | Modülün kendisi `mod icon_theme_schema` ile private'tır; root `pub use crate::icon_theme_schema::*` yalnızca `IconThemeFamilyContent`, `IconThemeContent`, `DirectoryIconsContent`, `ChevronIconsContent`, `IconDefinitionContent` tiplerini açar. Mirror'da `kvs_tema::icon_theme_schema` path'i ancak bilinçli bir tasarım kararıysa public yapılmalıdır |
| `SyntaxTheme::one_dark()` | `syntax_theme/src/syntax_theme.rs:134-220` | `#[cfg(feature = "bundled-themes")]` altında public helper'dır. Bundled `one/one.json` içinden `"One Dark"` syntax theme'i yükler; feature kapalıyken public API'de bulunmaz. Üretim tema yükleme hattı yerine test/dev fixture olarak düşünülmesi yerinde olur |
| `Refineable` trait yüzeyi tam katalog | `refineable/src/refineable.rs:29-131` | `Refineable`: `refine`, `refined`, `from_cascade`, `is_superset_of`, `subtract`; `IsEmpty`: `is_empty`; `Cascade<S>` metotları: `reserve`, `base`, `set`, `merged`; `CascadeSlot` yalnız slot handle'ıdır. Tema sistemi `from_cascade`/`Cascade` kullanmaz ama refineable trait sözleşmesi tam mirror edilmelidir |
| `#[with_fallible_options]` macro | `settings_macros/src/settings_macros.rs:110-152` | `Option<T>` alanlarına otomatik olarak `#[serde(default, skip_serializing_if = "Option::is_none", deserialize_with = "crate::fallible_options::deserialize")]` ekler. `ThemeSettingsContent`, `ThemeStyleContent`, `ThemeColorsContent`, `StatusColorsContent` gibi tiplerde kullanılır; parse hatası alan `None`'a düşer ve hata thread-local hata listesine eklenir |
| `fallible_options::parse_json`, crate-private `deserialize` | `settings_content/src/fallible_options.rs:11-65` | Top-level parse fonksiyonu (`parse_json::<T>(json) -> (Option<T>, ParseStatus)`) public re-export edilir; thread-local `ERRORS` listesi parse sırasında biriken hataları toplar. `deserialize` ise `pub(crate)` internal helper'dır, macro'nun eklediği serde attribute'u kullanır |
| `RootUserSettings::parse_json`, `parse_json_with_comments` | `settings_content/src/settings_content.rs:333-364` | `SettingsContent`, `Option<SettingsContent>` ve `UserSettingsContent` için public root parse trait'idir. `parse_json` fallible option tolerant hattını, `parse_json_with_comments` ise normal `settings_json::parse_json_with_comments` sonucunu döner |
| `HighlightStyleContent::treat_error_as_none` istisnası | `settings_content/src/theme.rs:1104-1144` | `HighlightStyleContent` `#[with_fallible_options]` kullanmaz. Yalnızca `background_color`, `font_style`, `font_weight` alanlarında yerel `treat_error_as_none` bulunur; `color` alanında bulunmaz. Bu yüzden `font_style` bilinmeyen enum olduğunda `None` döner, ama `"color": 3` bir deserialize hatası üretir |
| Tema dosyası parse yolu | `theme_settings/src/theme_settings.rs:212-234` | Bundled tema asset'leri `serde_json::from_slice` ile, kullanıcı tema dosyaları `serde_json_lenient::from_slice` ile parse edilir. Bu hatlar `RootUserSettings::parse_json` değildir; `with_fallible_options` hataları `ParseStatus` olarak toplamaz, normal serde hatası dönebilir |
| `MergeFrom` trait + derive | `settings_content/src/merge_from.rs:17-174`, `settings_macros::MergeFrom` | Primitive overwrite, `Option<T>` recursive merge, `Vec<T>` overwrite (concat değil), `HashMap`/`BTreeMap`/`IndexMap` key-bazlı recursive merge, `HashSet`/`BTreeSet` union, `serde_json::Value` Object recursive. Konu 20'de davranış matrisi tam tablo olarak verilir |
| `ThemeSettingsContent.ui_density` JSON rename | `settings_content/src/theme.rs:166-167` | `#[serde(rename = "unstable.ui_density")]` — kullanıcı JSON'unda **`"unstable.ui_density"`** yazmalıdır; `"ui_density"` çalışmaz. Konu 40'ta ayrıntılıdır |
| `ThemeSettingsContent` schemars tag'leri | `settings_content/src/theme.rs:124-171` | `ui_font_fallbacks`, `buffer_font_fallbacks` `#[schemars(extend("uniqueItems" = true))]`; `unnecessary_code_fade` `#[schemars(range(min = 0.0, max = 0.9))]`; her font alanı `#[schemars(default = "...")]` ile default value fonksiyonu işaretlidir |
| `FontSize` newtype derive list'i | `settings_content/src/theme.rs:186-200` | `Clone, Copy, Debug, Serialize, Deserialize, JsonSchema, MergeFrom, PartialEq, PartialOrd, derive_more::FromStr` ve `#[serde(transparent)]`. Serialize iki ondalık basamakla yapılır (`serialize_f32_with_two_decimal_places`). Display formatı `{:.2}`'dir |
| `UnderlineStyle`, `StrikethroughStyle` (`gpui::style:783-804`) | `gpui/src/style.rs` | `UnderlineStyle { thickness, color, wavy }`, `StrikethroughStyle { thickness, color }`. İkisi de `Refineable + Copy + Eq + Hash + JsonSchema` türevlidir. Tema JSON sözleşmesinde syntax stillerinin underline/strikethrough alanları **yoktur**; `refine_theme` her ikisini de `Default::default()` (nötr) bırakır. Konu 7'de işlendi |
| `HighlightStyle.fade_out` `Hash` impl | `gpui/src/style.rs:562-574` | `f32` `Hash` türevini taşımaz; `to_be_bytes()` ile `u32`'ye çevrilip hash'lenir. Mirror tarafında elle implement edilir veya GPUI'nin `HighlightStyle`'ı doğrudan kullanılır |
| `ThemeRegistry::default()` | `theme/src/registry.rs:327-331` | `Default for ThemeRegistry` impl'i `Self::new(Box::new(()))` — boş asset source ile constructor. Test'te `ThemeRegistry::default()` kısa yol olarak iş görür; asset gerektiren testler `Box::new(()) as Box<dyn AssetSource>` parametresini açık geçirir |
| `FileIcons::get_icon`, `get_icon_for_type` | `file_icons/src/file_icons.rs:20-99` | `get_icon(path, cx)` 6 katmanlıdır: tam ad → dot-suffix loop → `multiple_extensions` → `extension_or_hidden_file_name` → ham `extension` → `"default"` tipi. Her katmanda `file_stems` veya `file_suffixes` aktif temasında lookup, sonra `get_icon_for_type(typ, cx)` ile `file_icons` aktif → default fallback yapılır |
| `FileIcons::get_folder_icon`, `get_chevron_icon` | `file_icons/src/file_icons.rs:102-165` | Klasör fallback'i: `named_directory_icons` (klasör adına özel) → private `get_generic_folder_icon` ile `directory_icons` (jenerik); her ikisinde de aktif → default tema fallback. Chevron: yalnızca aktif → default. Expanded/collapsed slot ayrımı her katmanda korunur |
| `ParseStatus` 3 variantlı | `settings_content/src/settings_content.rs:75-83` | `Success`, **`Unchanged`** (dosya değişmediği için parse atlandı), `Failed { error: String }`. `Unchanged`, settings file watcher hattında "skip" sinyali için kritik bir varianttır |
| `SettingsContent.theme` flatten ilişkisi | `settings_content/src/settings_content.rs:114-145` | `pub theme: Box<ThemeSettingsContent>` **`#[serde(flatten)]`** ile işaretlidir. Kullanıcı `settings.json`'da `ui_font_size`, `theme`, `icon_theme`, `unstable.ui_density` vb. **top-level** alanlar olarak yazar — iç `"theme": { ... }` bloğu **yoktur**. `SettingsContent` 25'ten fazla alt struct'ı flatten ile birleştirir; `Box` heap'e taşıyarak stack overflow'tan kaçınır |
| `UserSettingsContent` yapısı | `settings_content/src/settings_content.rs:407-421` | `content: Box<SettingsContent>` flatten + `release_channel_overrides` flatten + `platform_overrides` flatten + `profiles: IndexMap<String, SettingsProfile>` düz alan. Yani `~/.config/zed/settings.json` `SettingsContent` alanlarını + override bloklarını + `"profiles": {...}` alanını taşır |
| `settings_overrides!` macro | `settings_content/src/settings_content.rs:40-65` | `Option<Box<SettingsContent>>` alanlı override struct'lar üretir + `OVERRIDE_KEYS: &[&str]` derive eder + `get_by_key(key) -> Option<&SettingsContent>` accessor sunar. Release channel ve platform override'ları bu pattern ile tanımlıdır |
| `Settings` trait (`settings_store`) | `settings/src/settings_store.rs:60-129` | Tam public yüzeyi şu metotlardan oluşur: `const PRESERVED_KEYS: Option<&'static [&'static str]> = None`, `from_settings(content: &SettingsContent) -> Self`, `register(cx: &mut App)`, `get<'a>(path: Option<SettingsLocation>, cx: &'a App) -> &'a Self`, `get_global(cx: &App) -> &Self`, `try_get(cx: &App) -> Option<&Self>`, `try_read_global<R>(cx: &AsyncApp, f: impl FnOnce(&Self) -> R) -> Option<R>`, `override_global(settings: Self, cx: &mut App)`. `ThemeSettings::get_global(cx)` bu trait'ten gelir; `from_settings` `SettingsContent.theme` flatten'ından typed `ThemeSettings` üretir |
| `SettingsStore` global'i | `settings/src/settings_store.rs` | `register_setting::<T: Settings>()` ile typed setting tipini dispatch tablosuna ekler; ayar dosyası değiştiğinde her kayıtlı tipin `from_settings`'i çağrılır. `cx.global::<SettingsStore>().get(None)` cache'lenmiş bir `&ThemeSettings` döner — accessor klonsuzdur ve hot path'tedir |
| `settings_json::parse_json_with_comments` | `settings_json/src/settings_json.rs:743-746` | `serde_json_lenient::Deserializer` + `serde_path_to_error::deserialize`. Hata mesajları **field path'ini de gösterir** (örn. `theme.colors.background: invalid hex`); mirror tarafında da `serde_path_to_error` kullanılması kullanıcı deneyimini iyileştirir |
| `MergeFromTrait` re-export | `settings_content/src/settings_content.rs:23` | `merge_from::MergeFrom` `pub use ... as MergeFromTrait` ile yeniden ihraç edilir. Aynı trait için iki ad: `MergeFrom` (derive macro + trait) ve `MergeFromTrait` (alias). Mirror tarafında tek bir ada karar verilmesi ve çakışma testlerinin yazılması yerinde olur |
| `HighlightStyleContent::is_empty()` | `settings_content/src/theme.rs:1128-1134` | 4 alanın hepsi `None` ise `true` döner. Selector preview, snapshot test ve `MergeFrom` "no-op detection" akışında kullanılır. `Refineable.is_empty` trait'inin elle muadilidir |
| `Refineable` derive `Option<T>` sarmalama kuralı | `refineable/derive_refineable/src/derive_refineable.rs:512-548` | Düz `T` → `Option<T>`; `Option<T>` aynen `Option<T>` (tekrar sarmalanmaz); `#[refineable] U` → `URefinement` (nested recursive). Konu 11'de tablo eklenmiştir |
| `configured_theme`, `configured_icon_theme` görünürlüğü | `theme_settings/src/theme_settings.rs:145, 166` | Aslında `fn` — **private**. Mirror tarafında `pub fn` yapmak yerel bir API genişletmesi anlamına gelir. Zed paritesinde `init` ve `reload_theme` bu helper'ları çağırır, tüketici doğrudan erişmez |
| `load_bundled_themes` hata davranışı | `theme_settings/src/theme_settings.rs:199-222` | Her tema asset'i `log_err()` ile sarılır; parse hatası **panic etmez**, log'lanır ve sonraki asset'e geçilir. Bu sayede tek bir bozuk bundled tema tüm Zed başlatmasını engellemez. Mirror tarafında aynı davranış zorunludur |
| `#[derive(RegisterSetting)]` auto-registration | `settings_macros/src/settings_macros.rs:85-105`, `settings/src/settings_store.rs:131-137, 412-416` | Setting tipini `inventory::submit!` ile static registry'ye ekler. `SettingsStore::new` → `load_settings_types` → `inventory::iter` ile **link-time** olarak otomatik kaydedilir. `ThemeSettings` `#[derive(Clone, PartialEq, RegisterSetting)]` ile işaretlidir; `theme_settings::init` **elle register çağırmaz**. Konu 39'da tam akış verilir |
| `inventory` crate auto-registration | `Cargo.toml` (settings, settings_macros) | Rust'ın link-time static registration için kullandığı crate'dir. Setting tipi tanımının yapıldığı yerde `submit!`, toplama tarafında `collect!` kullanılır. Mirror'da alternatif (elle register listesi) seçildiğinde iki pattern'in karıştırılmaması gerekir; aksi halde sessiz kayıt eksiklikleri oluşur |
| `SettingsStore::override_global<T>` davranışı | `settings/src/settings_store.rs:475-483` | Test ve runtime override için kullanılır. `setting_values[TypeId::of::<T>()].set_global_value(Box::new(value))`. Doc note: "The given value will be overwritten if the user settings file changes" — yani settings dosyası değiştiğinde override drop edilir (observer akışı bunu garanti altına alır). Tip kayıtlı değilse `panic!("unregistered setting type ...")` üretir |
| `ThemeSettings` alan görünürlükleri | `theme_settings/src/settings.rs:39-89` | Font size alanları (`ui_font_size`, `buffer_font_size`, `agent_*_font_size`, `markdown_preview_font_family`, `markdown_preview_code_font_family`) **private**'tır; accessor metotlar (`ui_font_size(cx)`, `buffer_font_size(cx)` vb.) override-aware okuma yapar. Diğer alanlar (`ui_font`, `buffer_font`, `theme`, `icon_theme`, `theme_overrides`, `experimental_theme_overrides`, `buffer_line_height`, `markdown_preview_theme`, `ui_density`, `unnecessary_code_fade`) `pub`'tır. Konu 39'da tam tablo verilir |
| `theme` crate'i `#![deny(missing_docs)]` | `theme/src/theme.rs:1` | Crate-level lint zorlamasıdır: tüm public öğeler doc yorumu gerektirir. Mirror tarafında aynısı uygulandığında public API kalitesi artar; yeni bir alan eklendiğinde "doc yok" hatası sayesinde sözleşme atlanmaz |
| `theme` crate mod görünürlükleri | `theme/src/theme.rs:11-21, 32-42` | Tüm modüller (`default_colors`, `fallback_themes`, `font_family_cache`, `icon_theme`, `icon_theme_schema`, `registry`, `scale`, `schema`, `styles`, `theme_settings_provider`, `ui_density`) **private** `mod`'dur. Public yüzey yalnızca `pub use crate::<mod>::*` re-export ile gelir. `fallback_themes` istisnasında yalnızca `apply_status_color_defaults` ve `apply_theme_color_defaults` `pub use` ile açılır — diğer iç fonksiyonlar (`zed_default_dark` vb.) `pub(crate)` kalır |
| `ui::is_light(cx)` public helper | `ui/src/utils.rs:23-25` | `cx.theme().appearance.is_light()` çağırır. UI tüketicileri için tekrar eden bir pattern; `kvs_ui` veya `kvs_bilesen` mirror'ında benzeri sağlanabilir. `ui::prelude` `pub use theme::ActiveTheme` ile trait'i tüketici crate'lere açar |
| `settings::IntoGpui` trait + 7 impl | `settings/src/content_into_gpui.rs:12-85` | `*Content` tiplerini GPUI runtime tiplerine çeviren tek köprüdür. İmpl edilen tipler: `FontStyleContent`, `FontWeightContent` (100-950 clamp), `FontFeaturesContent`, `WindowBackgroundContent`, `ModifiersContent`, `FontSize` (`px(self.0)`), `FontFamilyName` (`SharedString::from(self.0)`). `ThemeSettings::from_settings` her font alanında bu trait'i çağırır. Mirror tarafında `KvsIntoRuntime` veya benzeri bir trait gerekir |
| `markdown_preview_code_font_family` settings alanı | `settings_content/src/theme.rs:155`, `theme_settings/src/settings.rs:62, 418` | Markdown preview içindeki inline code ve code block fontunu ayrı seçer. Boş bırakıldığında `buffer_font.family` kullanılır; düz preview metni `markdown_preview_font_family` boş ise `ui_font.family` kullanmaya devam eder. Provider trait'i değişmez |
| Mermaid tema tüketimi | `markdown/src/mermaid.rs` | Mermaid renderer teması aktif `ThemeColors`, `AccentColors`, `PlayerColors`, `Appearance` ve `ThemeSettings::ui_font` üstünden üretilir. `Hsla` renkleri renderer'a alpha'sız `#rrggbb` olarak verilir. Flowchart, sequence, pie ve git diyagramları için ayrı slot'lar (`primary_*`, `sequence_actor_*`, `pie_colors`, `git_colors`, `git_inv_colors`) doldurulur. `pie_section_text_color` ve `git_branch_label_colors` sabit `#fff`'tir. Light tema'da `relative_luminance >= 0.35`, dark tema'da `<= 0.18` hedefi tutturulana kadar `accent_fill_and_text` lightness'i adım adım kaydırır. `accent0..accentN` class'ları player slot'larından üretilir; pie renkleri `AccentColors::color_for_index` ile beslenir. Diyagram fenced blok kapanmış (`metadata.is_fenced_closed`) olmalıdır; `~~~src` `.mermaid`/`.mmd` dosyaları da render hattına girer |
| `Markdown::invalidate_mermaid_cache` | `markdown/src/markdown.rs` | `pub fn invalidate_mermaid_cache(&mut self, cx: &mut Context<Self>)`. `options.render_mermaid_diagrams` açıkken `mermaid_state` cache'ini temizler, parsed markdown'u yeniden render kuyruğuna atar ve `cx.notify()` çağırır. Tema observer'larından çağrılır; aksi halde tema değişimi mermaid SVG cache'inde kalır. `mermaid_showing_code: HashSet<usize>` Preview/Code sekme durumunu offset bazında tutar; `toggle_mermaid_tab(offset)` ve `is_mermaid_showing_code(offset)` `pub(crate)` yardımcılarıyla yönetilir |
| `MarkdownOptions::render_mermaid_diagrams`, `CopyButtonVisibility` | `markdown/src/markdown.rs`, `markdown/src/mermaid.rs` | `MarkdownOptions { render_mermaid_diagrams: true, ..Default::default() }` mermaid pipeline'ını açar; kapatıldığında hem cache hem `mermaid_showing_code` boşaltılır. `CopyButtonVisibility::{Hidden, VisibleOnHover}` blok renderer'ından okunur (`CodeBlockRenderer::Default { copy_button_visibility, .. }`); `Hidden` seçildiğinde Mermaid sekme başlığı ve kopya butonu çizilmez. Bu yüzey markdown render hattının parçasıdır, tema crate'inde değildir |
| `gpui::Hsla::opaque_strategy`, `Arbitrary for Hsla` | `gpui/src/color.rs` | `proptest` feature'ı altında renk property testleri için resmi generator'dır. Kontrast/türetme helper'larında sentetik örneklerin yanına property test eklenebilir |
| `CompletionMenuItemKind` ve syntax rengi | `editor/src/completions_menu.rs`, `settings_content/src/editor.rs` | `EditorSettingsContent.completion_menu_item_kind` enum'u `off` / `symbol` değerlerini taşır. `symbol` açıkken completion kind rozetleri `constructor`, `constant`, `enum`, `function.method`, `function`, `namespace`, `operator`, `property`, `string`, `type`, `variable`, `variant`, `keyword` capture'larından beslenir; noktalı capture yoksa parent capture fallback'i denenir. Bu ayar `ThemeSettingsContent` alanı değildir |
| Content/Runtime tip duplication ve `From` impls | `theme_settings/src/settings.rs:136, 188, 350` ↔ `settings_content/src/theme.rs:267, 309, 438` | `ThemeSelection`, `IconThemeSelection`, `BufferLineHeight` **iki yerde tanımlıdır**: settings_content (Content, `JsonSchema/MergeFrom/EnumDiscriminants`) ve theme_settings (Runtime, daha az derive). Aralarında `From<settings::*>` impl bulunur. `UiDensity` için `ui_density_from_settings` `pub(crate)` helper kullanılır (`From` değil). Konu 39 ve 43.3'te açıklanmıştır |
| `ThemeSettings::from_settings` panic kontratı | `theme_settings/src/settings.rs:619-666` | Her font ve theme alanında `.unwrap()` çağrılır — yani `default.json` zorunlu alanları (`ui_font_size/family/features/weight`, `buffer_font_*`, `buffer_line_height`, `theme`, `icon_theme`, `unnecessary_code_fade`) doldurmak zorundadır. Boş olduğunda init panic eder. Mirror'da `kvs_default_settings.json` aynı zorunlu set'i taşımalıdır |
| `ThemeSelection::Default`, `IconThemeSelection` derive list'i | `settings_content/src/theme.rs:254-322` | Content tipleri `JsonSchema, MergeFrom, EnumDiscriminants, VariantArray, VariantNames, FromRepr` derive eder. `ThemeSelection::default() = Dynamic { mode: System, light: "One Light", dark: "One Dark" }`. Selector UI tabbed picker için `strum::VariantArray` ve `EnumDiscriminants` kullanılır |
| `ThemeAppearanceMode` derive ve default | `settings_content/src/theme.rs:329-354` | `Copy + Default + strum::VariantArray + strum::VariantNames` derive'larıyla birlikte gelir; `#[default] System` varyantını taşır. `serde(rename_all = "snake_case")` ile JSON'da `"light"`, `"dark"`, `"system"` biçiminde görünür |
| `BufferLineHeight::Custom` 1.0 alt sınır deserializer | `settings_content/src/theme.rs:445-460` | `#[serde(deserialize_with = "deserialize_line_height")] f32` özel fonksiyonu kullanılır; 1.0 altındaki değer **deserialize hatası** üretir. Çift savunma: parse'ta 1.0 alt sınır + runtime `ThemeSettings::line_height()` `f32::max(.., MIN_LINE_HEIGHT)` |
| `ThemeSettings::line_height()` + `MIN_LINE_HEIGHT` | `theme_settings/src/settings.rs:20, 451-453` | `MIN_LINE_HEIGHT = 1.0`. Buffer renderer satır yüksekliği için bu accessor kullanılır; ham `BufferLineHeight.value()` doğrudan döndürülmez — geçersiz değerlerden korunmak amacıyla clamp yapılır |
| `ColorScales` `IntoIterator` impl | `theme/src/scale.rs:194-235` | 33 paleti **sabit bir sırada** (`gray, mauve, slate, sage, olive, sand, gold, bronze, brown, yellow, amber, orange, tomato, red, ruby, crimson, pink, plum, purple, violet, iris, indigo, blue, cyan, teal, jade, green, grass, lime, mint, sky, black, white`) `Vec<ColorScaleSet>` olarak yayar. Snapshot test ve color picker için kanonik bir dolaşımdır. Konu 17'de işlenmiştir |
| `FontFamilyName` impls | `settings_content/src/theme.rs:399-421` | `#[serde(transparent)]` newtype + `AsRef<str>` + `From<String>` + `From<FontFamilyName> for String`. `Arc<str>` taşır, klonsuz `SharedString`'e dönüşür (`IntoGpui::into_gpui()`) |
| `FontWeightContent::into_gpui()` 100-950 clamp | `settings/src/content_into_gpui.rs:32-34` | `FontWeight(self.0.clamp(100., 950.))` — CSS font-weight aralığında zorlama yapar. `FontWeightContent::{THIN, EXTRA_LIGHT, ..., BLACK}` const'larının GPUI `FontWeight` const'larıyla eşleştiği test ile doğrulanır (`content_into_gpui.rs:92-103`) |
| `settings::private` modülü | `settings/src/settings.rs:21-25` | `pub mod private { pub use inventory; pub use crate::settings_store::{RegisteredSetting, SettingValue}; }`. `RegisterSetting` derive macro `settings::private::inventory::submit!` çağırır. Mirror tarafında `kvs_ayarlari::private` benzeri bir modül zorunludur; aksi halde macro çıktısı derlenmez |
| `theme_settings` re-export listesi | `theme_settings/src/theme_settings.rs:24-37` | `schema::{FontStyleContent, FontWeightContent, HighlightStyleContent, StatusColorsContent, ThemeColorsContent, ThemeContent, ThemeFamilyContent, ThemeStyleContent, WindowBackgroundContent, status_colors_refinement, syntax_overrides, theme_colors_refinement}` ve `settings::{AgentBufferFontSize, AgentUiFontSize, BufferLineHeight, FontFamilyName, IconThemeName, IconThemeSelection, ThemeAppearanceMode, ThemeName, ThemeSelection, ThemeSettings, adjust_*, reset_*, adjusted_font_size, appearance_to_mode, clamp_font_size, default_theme, observe_buffer_font_size_adjustment, set_icon_theme, set_mode, set_theme, setup_ui_font}` + `pub use theme::UiDensity`. Mirror tarafında bu sembollerin tek kaynak crate'inden re-export edilmesi gerekir |

Küçük metot yüzeyleri:

```rust
AccentColors::color_for_index(index)
PlayerColors::agent()
PlayerColors::absent()
PlayerColors::read_only()
ThemeColors::to_vec()
UiDensity::spacing_ratio()
ColorScale::step_1()
ColorScale::step_2()
ColorScale::step_3()
ColorScale::step_4()
ColorScale::step_5()
ColorScale::step_6()
ColorScale::step_7()
ColorScale::step_8()
ColorScale::step_9()
ColorScale::step_10()
ColorScale::step_11()
ColorScale::step_12()
ColorScale::step(step)
ColorScaleSet::new(name, light, light_alpha, dark, dark_alpha)
ColorScaleSet::name()
ColorScaleSet::light()
ColorScaleSet::light_alpha()
ColorScaleSet::dark()
ColorScaleSet::dark_alpha()
ColorScaleSet::step(cx, step)
ColorScaleSet::step_alpha(cx, step)
ThemeSettingsProvider::ui_font(cx)
ThemeSettingsProvider::buffer_font(cx)
ThemeSelection::mode()
IconThemeSelection::mode()
ThemeSelection::name(system_appearance)
IconThemeSelection::name(system_appearance)
ThemeSettings::buffer_font_size(cx)
ThemeSettings::ui_font_size(cx)
ThemeSettings::agent_ui_font_size(cx)
ThemeSettings::agent_buffer_font_size(cx)
ThemeSettings::markdown_preview_font_family()
ThemeSettings::markdown_preview_code_font_family()
Markdown::invalidate_mermaid_cache(cx)
ThemeSettings::buffer_font_size_settings()
ThemeSettings::ui_font_size_settings()
ThemeSettings::agent_ui_font_size_settings()
ThemeSettings::agent_buffer_font_size_settings()
ThemeSettings::line_height()
ThemeSettings::apply_theme_overrides(arc_theme)
Theme::darken(color, light_amount, dark_amount)
BufferLineHeight::value()
default_theme(appearance) -> &'static str
appearance_to_mode(appearance) -> ThemeAppearanceMode
clamp_font_size(size) -> Pixels
adjusted_font_size(size, cx) -> Pixels
```

Bu metotlar rehberin ana akışını değiştirmez. Ancak tema editörü, collab renkleri, density ölçümü ve snapshot karşılaştırması yazıldığında doğrudan public API ihtiyacına dönüşürler.

---

## 44. Test ortamında tema mock'lama

UI bileşenleri test edilirken **tüm tema sistemini** init etmek çoğu zaman gerekmez. Yalnızca bileşenin renk ihtiyacını karşılayacak kadar bir mock tema kurmak yeterli olabilir.

### Strateji 1: Tam init (`kvs_tema::init`)

En basit yol. Test'in başında tüm runtime kurulur:

```rust
use gpui::TestAppContext;
use kvs_tema::ActiveTheme;

#[gpui::test]
fn button_uses_theme_colors(cx: &mut TestAppContext) {
    cx.update(|cx| {
        kvs_tema::init(cx);  // Fallback temaları kurar

        let theme = cx.theme();
        assert_eq!(theme.appearance, kvs_tema::Appearance::Dark);
        assert_ne!(theme.colors().background, gpui::Hsla::default());
    });
}
```

**Avantaj:** Production akışına en yakın yöntemdir. Init bug'ları erken aşamada yakalanır.

**Dezavantaj:** Her test fallback tema oluşumu için ~50 µs harcar. Yüzlerce test koşulduğunda bu süre birikir.

### Strateji 2: Manuel kurulum (özel tema değerleri)

Test'in özel renk değerlerine ihtiyacı varsa kurulum elle yapılır:

**Bu strateji `kvs_tema` crate'inin içinden çalışır.** `Theme { styles: ... }` literal'i `pub(crate)` alanlara erişim gerektirir. `kvs_tema/tests/` (entegrasyon testleri) **dış crate** sayılır; oradan struct literal yazılamaz. Bu manuel kurulum yalnızca `kvs_tema/src/test.rs` (Strateji 3) veya `kvs_tema` modülünün kendi birim testlerinde kullanılır.

```rust
// kvs_tema/src/test.rs içinde (crate-içi)
use std::sync::Arc;
use gpui::hsla;
use crate::{
    Theme, ThemeStyles, ThemeColors, StatusColors, PlayerColors,
    AccentColors, SystemColors, Appearance,
};

pub(crate) fn test_theme(bg: gpui::Hsla, fg: gpui::Hsla) -> Theme {
    Theme {
        id: "test".into(),
        name: "Test Theme".into(),
        appearance: Appearance::Dark,
        styles: ThemeStyles {
            window_background_appearance: gpui::WindowBackgroundAppearance::Opaque,
            system: SystemColors::default(),
            colors: ThemeColors {
                background: bg,
                text: fg,
                // ... diğer alanlar için dummy değer
                ..fallback_colors_dark()  // ← kalanlar fallback'ten
            },
            status: fallback_status_dark(),
            player: PlayerColors::default(),
            accents: AccentColors::default(),
            syntax: Arc::new(kvs_syntax_tema::SyntaxTheme::new(Vec::<(String, HighlightStyle)>::new())),
        },
    }
}
```

> **`..fallback_colors_dark()` yapılabilmesi için:** `ThemeColors`'a `Default` türevi eklenmesi veya `pub(crate) fn fallback_colors_dark() -> ThemeColors` adında bir yardımcı tanımlanması gerekir. Mevcut yapıda `ThemeColors`'ın `Default` türevi bulunmaz; tüm alanlar zorunludur. Bu nedenle test helper'ı yazılması beklenir.
>
> **Dış crate'ten test:** `kvs_ui/tests/` içinde bu strateji çalışmaz. `feature = "test-util"` üzerinden Strateji 3'ün public helper'ları çağrılır.

### Strateji 3: `kvs_tema` test helper'ı

`kvs_tema/src/test.rs` (yeni modül, `#[cfg(any(test, feature = "test-util"))]`):

```rust
#[cfg(any(test, feature = "test-util"))]
pub mod test {
    use crate::*;
    use std::sync::Arc;

    /// Test için minimal tema kurar.
    pub fn init_test(cx: &mut gpui::App) {
        // ThemeRegistry::new artık AssetSource zorunlu — testte `()` kullanılır.
        let registry = Arc::new(ThemeRegistry::new(Box::new(()) as Box<dyn gpui::AssetSource>));
        registry.insert_themes([test_theme()]);
        let theme = registry.get("Test").unwrap();
        let icon_theme = registry.default_icon_theme().unwrap();
        // ThemeRegistry::set_global Zed'de pub(crate); test helper'ı
        // kvs_tema'nın test-support feature'ı altında public açar veya
        // doğrudan GlobalThemeRegistry newtype'ını cx.set_global ile kurar.
        cx.set_global(GlobalThemeRegistry(registry.clone()));
        cx.set_global(GlobalTheme::new(theme, icon_theme));
    }

    /// Yeniden kullanılabilir test tema'sı.
    pub fn test_theme() -> Theme {
        // Tüm alanları açık ama belirgin renklerle doldur
        // (debug için kolayca ayırt edilir)
        let red = gpui::hsla(0.0, 1.0, 0.5, 1.0);
        let green = gpui::hsla(0.333, 1.0, 0.5, 1.0);
        Theme {
            id: "test".into(),
            name: "Test".into(),
            appearance: Appearance::Dark,
            styles: ThemeStyles {
                colors: ThemeColors {
                    background: red,
                    border: green,
                    text: green,
                    // ... her alan benzersiz renkte (debug için)
                },
                // ...
            },
        }
    }
}
```

Tüketici tarafında (`kvs_ui/tests/...`):

```rust
[dev-dependencies]
kvs_tema = { path = "../kvs_tema", features = ["test-util"] }
```

```rust
use kvs_tema::test::init_test;

#[gpui::test]
fn render_uses_test_theme(cx: &mut TestAppContext) {
    cx.update(|cx| {
        init_test(cx);

        let theme = cx.theme();
        assert_eq!(theme.name.as_ref(), "Test");
    });
}
```

### Snapshot test deseni

Tema değerlerinin **belirli yerlerde** kullanıldığını doğrulamak için:

```rust
use kvs_tema::test::{init_test, test_theme};

#[gpui::test]
fn button_renders_with_theme_background(cx: &mut TestAppContext) {
    cx.update(|cx| {
        init_test(cx);

        let theme = test_theme();
        let expected_bg = theme.colors().element_background;

        let button = cx.new(|_| Button::new("OK".into()));

        // Element ağacını inspect et (test API'ları rehber.md #75'te)
        // Veya rendered scene'i query et:
        let render_output = /* ... */;
        assert!(render_output.contains_bg(expected_bg));
    });
}
```

> **GPUI test API'ları:** `TestAppContext::simulate_input`, `find_view` vb. rehber.md #75'te ayrıntılı işlenir. Render output'unun test edilmesi tema'nın doğru okunduğunu doğrular; tema'nın **kendi** doğruluğu için Bölüm VI fixture testleri yeterli olur.

### Tema değişimi simülasyonu

```rust
#[gpui::test]
fn ui_updates_on_theme_change(cx: &mut TestAppContext) {
    cx.update(|cx| {
        kvs_tema::init(cx);
        let initial = cx.theme().name.clone();

        // Light'a geç
        kvs_tema::temayi_degistir("Kvs Default Light", cx).unwrap();

        let new_theme = cx.theme();
        assert_ne!(new_theme.name, initial);
        assert_eq!(new_theme.appearance, kvs_tema::Appearance::Light);
    });
}
```

### Test'te `refresh_windows`

`TestAppContext` headless bir bağlamdır; açık pencere bulunmaz. `cx.refresh_windows()` hata üretmez ama görünür bir etkisi de olmaz. Tema değişimi test edilirken `cx.theme()` çağrısı **yeni değeri** döndürür; global hemen güncellenir.

UI bileşeninin gerçekten yeniden render edilip edilmediğini doğrulamak için `VisualTestContext` gerekir (rehber.md #75) — pencereyi açar, render eder ve snapshot karşılaştırması yapar.

### Tema mock'lamayı yanlış yapmak

```rust
// YANLIŞ: cx olmadan, struct literal ile tema oluşturmak
// (dış crate'te zaten derlenmez — `styles` pub(crate))
let theme = kvs_tema::fallback::kvs_default_dark();
let bg = theme.colors().background;
let button_bg = bg;  // ← bileşen rendering'i ile bağlantısı yok
```

Bu test yalnızca `Theme` struct'ının kendi alanlarını doğrular. **Tüketici kodun `cx.theme()` çağrısının doğru çalıştığını test etmez.** Bir UI testi hedefleniyorsa `TestAppContext::update` üzerinden global kurulum yapılmalıdır.

### Test ortamı sınır listesi

| Test türü | Strateji | Bileşen |
|-----------|----------|---------|
| Sözleşme doğrulama (parse, refinement) | `#[test]` + fixture | Bölüm VI/Konu 28 |
| Runtime init/registry | `#[gpui::test]` + `init` | Strateji 1 |
| Bileşen tema okuma | `#[gpui::test]` + custom theme | Strateji 2 veya 3 |
| Visual snapshot (render output) | `VisualTestContext` | rehber.md #75 |
| Tema değişim akışı | `#[gpui::test]` + `temayi_degistir` | Konu 38 + bu konu |

### Tuzaklar

1. **`kvs_tema::init(cx)` çağrılmadan `cx.theme()` çağırmak**: Panic atar. Her testin ilk satırının init veya manuel `cx.set_global(GlobalTheme::new(...))` olması beklenir.
2. **`TestAppContext::run` yerine `update`**: Tema testleri sync çalışır; `update` doğru tercihtir. `run` async bir event loop kurar ve burada gerekmez.
3. **`test_theme()` çağrısının her testte yeniden yapılması**: `set_global` her çağrıda üstüne yazar; cumulative bug üretmez ama performans maliyeti vardır. Test fixture olarak `Arc<Theme>` paylaşılabilir.
4. **`feature = "test-util"`'ın production'da açık bırakılması**: `#[cfg(any(test, feature = "test-util"))]` koşulu kurulur; release build'de kapalı kalmalıdır.
5. **Test tema'sının tüm alanlarının sıfır bırakılması**: `unsafe { zeroed() }` ile doldurulan bir test = production'da görünmez bir UI'a karşılık gelir. Test sırasında bile tüm alanların açık değerle doldurulması, bug yakalama gücünü yükseltir.
6. **`refresh_windows`'un test'te etkisiz sanılması**: `TestAppContext` üzerinde etkisi olmaz ama global state güncel kalır; test mantığı `cx.theme()` ile yeni değeri okur. `VisualTestContext` kullanıldığında ise render gerçek anlamda tetiklenir.
7. **Mock tema'nın `Theme.id` değerinin aynı bırakılması**: Test'te birden fazla tema kurulduğunda farklı `id`'lerin (`"test-1"`, `"test-2"`) verilmesi gerekir; aksi halde `Theme.id` üzerinden yapılan equality testi yanlış sonuç verebilir.

---
