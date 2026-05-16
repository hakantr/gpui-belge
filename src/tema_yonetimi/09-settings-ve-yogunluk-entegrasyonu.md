# Settings ve yoğunluk entegrasyonu

Runtime çalışır hale geldikten sonra kullanıcı ayarları devreye girer. Bu bölüm tema seçimi, font override akışları ve UI density sözleşmesinin runtime'a nasıl bağlandığını anlatır. Bu üç hat birlikte düşünülür; birindeki karar diğerlerinin davranışını doğrudan etkiler.

---

## 39. Settings entegrasyonu: `ThemeSettings`, `RegisterSetting`, `IntoGpui`, `ThemeSettingsProvider`, font runtime API'leri

### Settings / override / selector köprüsü

Zed-benzeri bir kontrol için tek bir `ad: String` yeterli değildir. Minimum settings modeli şu dört özelliği taşımalıdır:

1. **Static seçim:** Tek bir tema adı her modda kullanılır.
2. **Dynamic seçim:** `mode + light + dark` üçlüsü tanımlanır; `mode=system` durumunda OS modu hangi adın seçileceğini belirler.
3. **Aktif tema override'ı:** Geçerli temanın üstüne geçici veya deneysel `ThemeStyleContent` uygulanır.
4. **Tema bazlı override:** Belirli bir tema adı için özel bir override map'i tanımlanır.

Zed'e denk düşen settings sözleşmesi:

> **Owner zinciri (önemli):** `ThemeName`, `IconThemeName`, `ThemeAppearanceMode`, `FontFamilyName`, `DEFAULT_LIGHT_THEME` ve `DEFAULT_DARK_THEME` aslında **`settings_content` crate'inde** tanımlıdır (`settings_content/src/theme.rs:282`, `501`, `507`). `theme_settings` crate'i bunları `pub use settings::{...}` (`theme_settings/src/settings.rs:13`) ile yeniden ihraç eder; tüketici kod `theme_settings::ThemeName` adıyla erişir. Mirror tarafta tek kaynak `kvs_ayarlari_icerik` (veya muadili) crate'i olmalıdır. `kvs_tema_ayarlari` yalnızca `pub use` ile köprü kurar. Aynı tipin birden fazla yerde tanımlanması schema ve test çatışmasına yol açar.

> **Flatten ilişkisi:** Zed'de `SettingsContent.theme: Box<ThemeSettingsContent>` alanı `#[serde(flatten)]` ile işaretlidir (`settings_content/src/settings_content.rs:117-121`). Yani kullanıcı ayar dosyasında `theme:` veya `icon_theme:` **iç alan değildir** — `ui_font_size`, `theme`, `icon_theme`, `experimental.theme_overrides`, `theme_overrides`, `unstable.ui_density` gibi tüm alanlar `settings.json`'da **top-level alanlar** olarak yazılır. `SettingsContent` 25'ten fazla alt struct'ını flatten ile birleştirir (project, theme, extension, workspace, editor, remote vb.); kullanıcı tek bir düz JSON görür. Mirror tarafında da `kvs_ayarlari_icerik::AyarIcerik` içinde `pub theme: Box<TemaAyarContent>` alanının `#[serde(flatten)]` ile sarmalanması Zed paritesi için zorunludur. Aksi halde mevcut Zed kullanıcı ayar dosyaları çalışmaz.
>
> Aynı şekilde `UserSettingsContent` (`settings_content/src/settings_content.rs:409`) `content: Box<SettingsContent>` + `release_channel_overrides` + `platform_overrides` üçlüsünü flatten ile barındırır; üst seviye `profiles: IndexMap<String, SettingsProfile>` ise düz alan olarak kalır.

```rust
use std::{collections::HashMap, sync::Arc};

#[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
#[serde(transparent)]
pub struct ThemeName(pub Arc<str>);

#[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
#[serde(transparent)]
pub struct IconThemeName(pub Arc<str>);

#[derive(Clone, Copy, Debug, Default, Eq, PartialEq, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum ThemeAppearanceMode {
    Light,
    Dark,
    #[default]
    System,
}

#[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
#[serde(untagged)]
pub enum ThemeSelection {
    Static(ThemeName),
    Dynamic {
        #[serde(default)]
        mode: ThemeAppearanceMode,
        light: ThemeName,
        dark: ThemeName,
    },
}

#[derive(Clone, Debug, serde::Serialize, serde::Deserialize)]
#[serde(untagged)]
pub enum IconThemeSelection {
    Static(IconThemeName),
    Dynamic {
        #[serde(default)]
        mode: ThemeAppearanceMode,
        light: IconThemeName,
        dark: IconThemeName,
    },
}

pub const DEFAULT_LIGHT_THEME: &str = "One Light";
pub const DEFAULT_DARK_THEME: &str = "One Dark";

#[derive(Clone, Debug, Default, serde::Serialize, serde::Deserialize)]
#[serde(default)]
pub struct ThemeSettingsContent {
    // Font ve typography alanları Konu 39'de listelenir.
    pub theme: Option<ThemeSelection>,
    pub icon_theme: Option<IconThemeSelection>,

    #[serde(rename = "experimental.theme_overrides")]
    pub experimental_theme_overrides: Option<ThemeStyleContent>,

    pub theme_overrides: HashMap<String, ThemeStyleContent>,
}
```

Örnek bir kullanıcı config'i:

```jsonc
{
  "theme": {
    "mode": "system",
    "light": "One Light",
    "dark": "One Dark"
  },
  "icon_theme": {
    "mode": "system",
    "light": "Material Light",
    "dark": "Material Dark"
  },
  "experimental.theme_overrides": {
    "background": "#101216ff",
    "text": "#e6e8ebff"
  },
  "theme_overrides": {
    "One Dark": {
      "editor.active_line.background": "#222631ff"
    }
  }
}
```

Selection çözümleme fonksiyonları:

```rust
impl ThemeSelection {
    pub fn name(&self, system: Appearance) -> ThemeName {
        match self {
            Self::Static(name) => name.clone(),
            Self::Dynamic { mode, light, dark } => match mode {
                ThemeAppearanceMode::Light => light.clone(),
                ThemeAppearanceMode::Dark => dark.clone(),
                ThemeAppearanceMode::System => match system {
                    Appearance::Light => light.clone(),
                    Appearance::Dark => dark.clone(),
                },
            },
        }
    }

    pub fn mode(&self) -> Option<ThemeAppearanceMode> {
        match self {
            Self::Static(_) => None,
            Self::Dynamic { mode, .. } => Some(*mode),
        }
    }
}

impl IconThemeSelection {
    pub fn name(&self, system: Appearance) -> IconThemeName {
        match self {
            Self::Static(name) => name.clone(),
            Self::Dynamic { mode, light, dark } => match mode {
                ThemeAppearanceMode::Light => light.clone(),
                ThemeAppearanceMode::Dark => dark.clone(),
                ThemeAppearanceMode::System => match system {
                    Appearance::Light => light.clone(),
                    Appearance::Dark => dark.clone(),
                },
            },
        }
    }

    pub fn mode(&self) -> Option<ThemeAppearanceMode> {
        match self {
            Self::Static(_) => None,
            Self::Dynamic { mode, .. } => Some(*mode),
        }
    }
}
```

### `Settings` trait — `ThemeSettings::get_global(cx)` nereden gelir

Zed'in `theme_settings::ThemeSettings` tipi `Settings` trait'ini (`settings/src/settings_store.rs:60-100`) implement eder:

```rust
pub trait Settings: 'static + Send + Sync + Sized {
    /// Settings dosyasına her zaman yazılan alan adları (versiyon tag'leri
    /// gibi).
    const PRESERVED_KEYS: Option<&'static [&'static str]> = None;

    /// `SettingsContent` (default + user + project merged) → runtime tipi.
    fn from_settings(content: &SettingsContent) -> Self;

    /// `SettingsStore`'a kaydet (init sırasında).
    fn register(cx: &mut App);

    /// `path` ile path-scoped okuma (proje override için).
    fn get<'a>(path: Option<SettingsLocation>, cx: &'a App) -> &'a Self;

    /// Global okuma — path::None ile aynı.
    fn get_global(cx: &App) -> &Self;

    /// Yumuşak okuma — `SettingsStore` kuruluysa.
    fn try_get(cx: &App) -> Option<&Self>;

    /// AsyncApp tarafından senkron read (typed setting tipini callback'e ver).
    fn try_read_global<R>(
        cx: &AsyncApp,
        f: impl FnOnce(&Self) -> R,
    ) -> Option<R>;

    /// Runtime override — settings dosyası değişene kadar geçerli.
    fn override_global(settings: Self, cx: &mut App);
}
```

### `#[derive(RegisterSetting)]` ile auto-registration

Zed'in `Settings`-tipi auto-registration mekanizması iki bileşene dayanır (`settings_macros::derive_register_setting`, `settings/src/settings_store.rs:131-137, 412-416`):

```rust
// proc-macro üretimi (settings_macros/src/settings_macros.rs:85-105):
inventory::submit! {
    RegisteredSetting {
        settings_value: || Box::new(SettingValue::<#type_name> { ... }),
        from_settings: |content| Box::new(<#type_name as Settings>::from_settings(content)),
        id: || std::any::TypeId::of::<#type_name>(),
    }
}

// settings_store.rs içinde:
inventory::collect!(RegisteredSetting);

impl SettingsStore {
    fn load_settings_types(&mut self) {
        for registered_setting in inventory::iter::<RegisteredSetting>() {
            self.register_setting_internal(registered_setting);
        }
    }

    pub fn new(cx: &mut App, default_settings: &str) -> Self {
        let mut this = Self { /* ... */ };
        this.load_settings_types();   // ← link-time tüm setting tipleri burada toplanır
        this
    }
}
```

**Pratik sonuç:**

`ThemeSettings` aslında `#[derive(Clone, PartialEq, RegisterSetting)]` (`theme_settings/src/settings.rs:38`) ile işaretlenir. `theme_settings::init` **`ThemeSettings::register(cx)` çağırmaz**. Inventory crate'i, `submit!` makrosunun üretildiği yerde static registration yapar. `SettingsStore::new` constructor'ında `inventory::iter::<RegisteredSetting>()` üzerinden tüm linklenen setting tipleri toplanır. Yani üretim akışında setting tiplerini elle `register` etmek gerekmez; bu trait metodu yalnızca testlerde veya `SettingsStore` elle kurulurken kullanılır.

> **Önemli parite notu:** Mirror tarafta inventory pattern'inin alternatifi elle register etmektir (`kvs_ayarlari::init` veya benzeri). İki yol karıştırılmamalıdır: ya tüm tipler `#[derive(KaydetAyar)]` ile auto-register edilir ya da tamamı elle kaydedilir. Karışık bir mod, yani bazı tiplerin kayıtlı bazılarının kayıtsız olması, sessiz bug üretir.

### `ThemeSettings` alan görünürlükleri

`ThemeSettings` struct'ında alanların görünürlüğü Zed paritesinde **karışıktır** (`theme_settings/src/settings.rs:39-86`):

| Alan | Görünürlük | Erişim yolu |
|------|-----------|-------------|
| `ui_font_size: Pixels` | **private** | `theme_settings.ui_font_size(cx)` accessor |
| `ui_font: Font` | `pub` | doğrudan alan |
| `buffer_font_size: Pixels` | **private** | `theme_settings.buffer_font_size(cx)` accessor |
| `buffer_font: Font` | `pub` | doğrudan alan |
| `agent_ui_font_size: Option<Pixels>` | **private** | `theme_settings.agent_ui_font_size(cx)` |
| `agent_buffer_font_size: Option<Pixels>` | **private** | `theme_settings.agent_buffer_font_size(cx)` |
| `markdown_preview_font_family: Option<SharedString>` | **private** | `theme_settings.markdown_preview_font_family()` |
| `markdown_preview_code_font_family: Option<SharedString>` | **private** | `theme_settings.markdown_preview_code_font_family()` |
| `markdown_preview_theme: Option<ThemeSelection>` | `pub` | doğrudan alan |
| `buffer_line_height: BufferLineHeight` | `pub` | doğrudan alan |
| `theme: ThemeSelection` | `pub` | doğrudan alan |
| `experimental_theme_overrides: Option<ThemeStyleContent>` | `pub` | doğrudan alan |
| `theme_overrides: HashMap<String, ThemeStyleContent>` | `pub` | doğrudan alan |
| `icon_theme: IconThemeSelection` | `pub` | doğrudan alan |
| `ui_density: UiDensity` | `pub` | doğrudan alan |
| `unnecessary_code_fade: f32` | `pub` | doğrudan alan |

**Gerekçe:** Font size değerleri `*FontSize` override global'lerinden etkilenir (Konu 39). Accessor metotlar bu override'ı uygular ve **etkin değeri** döndürür. Doğrudan alan okuması ise settings dosyasındaki ham değeri verir. Bu yüzden font size alanları bilinçli olarak private tutulur. Okuyucu ya `.ui_font_size(cx)` accessor'ını kullanır ya da ham değer için `.ui_font_size_settings()` accessor'ına gider (Konu 43 tablosunda yer alır). Markdown preview font family alanları da private tutulur; düz markdown metni UI fontuna, inline code ve code block ise buffer fontuna fallback eder.

Mirror tarafta `TemaAyarlari` struct'ında font size'ların private tutulması ve accessor metotlarla okunması Zed paritesi açısından zorunludur. Aksi halde override drop davranışı yanlış uygulanır.

**Çalışma akışı:**

1. `SettingsStore::set_global(cx, store)` settings sisteminin init'inde kurulur.
2. `SettingsStore::new`'ün `load_settings_types`'ı inventory'den kayıtlı tipleri otomatik yükler; `ThemeSettings::register(cx)` üretim akışında çağrılmaz (yukarıdaki auto-registration bölümü).
3. `ThemeSettings::get_global(cx)` çalıştığında `cx.global::<SettingsStore>().get(None)` üzerinden cache'lenmiş güncel `&ThemeSettings` döner.
4. `SettingsLocation { worktree_id, path }` ile path-scoped lookup yapıldığında proje-local `.zed/settings.json` override'ları uygulanır.

**Mirror tarafında:** `kvs_tema` `Settings` trait'ine doğrudan bağımlı değildir (Konu 5 bağımlılık matrisi). Ancak `kvs_tema_ayarlari` crate'i `kvs_ayarlari::Settings` benzeri bir trait sözleşmesini takip eder. `ThemeSettingsProvider` (Konu 39) bu bağlantının soyutlanmış arayüzüdür. `kvs_tema`, provider'dan typography ve density okur; `Settings` trait'ini doğrudan kullanmaz. Auto-registration için `#[derive(KaydetAyar)]` benzeri bir macro mirror edilebilir; alternatif olarak `init` fonksiyonunda elle `Settings::register` çağrılır.

### `IntoGpui` trait — Settings → Runtime köprüsü

Zed, `*Content` tiplerini GPUI runtime tiplerine çevirirken **tek bir trait** kullanır: `settings::IntoGpui` (`settings/src/content_into_gpui.rs:12-15`).

```rust
pub trait IntoGpui {
    type Output;
    fn into_gpui(self) -> Self::Output;
}
```

Tüm impl'ler `settings` crate'inde toplanır (mirror tarafında `kvs_ayarlari` veya `kvs_ayarlari_icerik` köprü modülü):

| Source (Content) | Output (Runtime) | Davranış |
|------------------|-----------------|----------|
| `FontStyleContent` | `gpui::FontStyle` | Variant 1:1 (Normal/Italic/Oblique) |
| `FontWeightContent` | `gpui::FontWeight` | `FontWeight(self.0.clamp(100., 950.))` — CSS aralığında zorlama |
| `FontFeaturesContent` | `gpui::FontFeatures` | `FontFeatures(Arc::new(map.collect()))` |
| `WindowBackgroundContent` | `gpui::WindowBackgroundAppearance` | Variant 1:1 (Opaque/Transparent/Blurred) |
| `ModifiersContent` | `gpui::Modifiers` | Alan kopyala (`control, alt, shift, platform, function`) |
| `FontSize` | `gpui::Pixels` | `px(self.0)` |
| `FontFamilyName` | `gpui::SharedString` | `SharedString::from(self.0)` (klonsuz `Arc<str>` taşır) |

`ThemeSettings::from_settings` font-bazlı her alanda `into_gpui()` zinciri ile bu trait'i kullanır:

```rust
ui_font_size: clamp_font_size(content.ui_font_size.unwrap().into_gpui()),
ui_font: Font {
    family: content.ui_font_family.as_ref().unwrap().0.clone().into(),
    features: content.ui_font_features.clone().unwrap().into_gpui(),
    fallbacks: font_fallbacks_from_settings(content.ui_font_fallbacks.clone()),
    weight: content.ui_font_weight.unwrap().into_gpui(),
    style: Default::default(),
},
```

**Çok kritik bir davranış:** `ThemeSettings::from_settings` içinde birçok satır `.unwrap()` çağırır. Yani **`default.json` bu alanları doldurmak zorundadır**. `ui_font_size`, `ui_font_family`, `ui_font_features`, `ui_font_weight`, `buffer_font_family`, `buffer_font_features`, `buffer_font_weight`, `buffer_font_size`, `buffer_line_height`, `theme`, `icon_theme` ve `unnecessary_code_fade` boş kalırsa runtime tipi üretilirken panic oluşur. Mirror tarafta `kvs_default_settings.json` bu zorunlu alanları içermelidir; aksi halde `init` panic atar.

### Content/Runtime tip duplication ve `From` impls

Zed'in `theme_settings::settings` modülü runtime tarafında `ThemeSelection`, `IconThemeSelection`, `BufferLineHeight` gibi tipleri **yeniden tanımlar**. Bunlar `settings_content` tarafındaki Content tipleriyle aynı varyantlara sahiptir, ama farklı derive list'leri taşır. Aralarındaki köprü `From` implementasyonları üzerinden kurulur:

```rust
// theme_settings/src/settings.rs:136-145, 188-197, 350-359:
impl From<settings::ThemeSelection> for ThemeSelection {
    fn from(s: settings::ThemeSelection) -> Self { /* variant kopyala */ }
}

impl From<settings::IconThemeSelection> for IconThemeSelection { /* aynı */ }
impl From<settings::BufferLineHeight> for BufferLineHeight { /* variant kopyala */ }
```

`UiDensity` için ise `pub(crate) fn ui_density_from_settings(val) -> UiDensity` helper'ı bulunur (`From` trait kullanılmaz). Bu iki kat tip katmanı kasıtlıdır:

- **Content tipleri** (`settings_content::theme::*`): `JsonSchema, MergeFrom, serde::{Serialize, Deserialize}, strum::EnumDiscriminants` derive'larıyla birlikte gelir; JSON sözleşmesi, schema üretimi ve user/default/project cascade için ayarlanır.
- **Runtime tipleri** (`theme_settings::settings::*`): Daha az derive taşır, runtime hot path için optimize edilir. Selector UI ile `ThemeSettings.theme.name(appearance)` gibi metotlar burada yer alır.

Mirror tarafta bu duplication korunmalıdır: `kvs_ayarlari_icerik::TemaSecimi` (content) ve `kvs_tema_ayarlari::TemaSecimi` (runtime), aralarında `From` impl ile bağlanır. Tek tipe indirgeme yapılırsa serde derive'ı runtime tarafına taşınır ve selector UI'nın `EnumDiscriminants` kullanımı bozulabilir.

Tema uygulama akışı (`configured_theme` Zed'de **private** bir `fn`'dir, `theme_settings/src/theme_settings.rs:145`; aşağıdaki örnek mirror tarafında public bir yardımcı olarak yazılabilir):

```rust
pub fn configured_theme(settings: &ThemeSettingsContent, cx: &mut App) -> Arc<Theme> {
    let registry = ThemeRegistry::global(cx);
    let system = SystemAppearance::global(cx).0;
    let selection = settings.theme.clone().unwrap_or_else(default_theme_selection);
    let name = selection.name(system);

    let mut theme = registry
        .get(&name.0)
        .or_else(|_| registry.get(default_theme_name(system)))
        .unwrap_or_else(|_| registry.get("Kvs Default Dark").unwrap());

    theme = apply_theme_overrides(theme, settings);
    theme
}

pub fn apply_theme_overrides(
    mut theme: Arc<Theme>,
    settings: &ThemeSettingsContent,
) -> Arc<Theme> {
    if let Some(overrides) = &settings.experimental_theme_overrides {
        let mut clone = (*theme).clone();
        modify_theme(&mut clone, overrides);
        theme = Arc::new(clone);
    }

    if let Some(overrides) = settings.theme_overrides.get(theme.name.as_ref()) {
        let mut clone = (*theme).clone();
        modify_theme(&mut clone, overrides);
        theme = Arc::new(clone);
    }

    theme
}
```

`modify_theme` aynı düşük seviyeli refinement araçlarını kullanır, ama tam bir `refine_theme` pipeline'ı değildir. `window_background_appearance` override edilir, `status_colors_refinement` ve `theme_colors_refinement` uygulanır, player ve accent listeleri merge edilir, syntax override'ları mevcut syntax üstüne bindirilir. Override işlemi registry'deki orijinal `Arc<Theme>` örneğini değiştirmez; yalnızca bir klon üzerinde çalışır.

> **Önemli fark:** Zed `ThemeSettings::modify_theme` içinde `apply_status_color_defaults` ve `apply_theme_color_defaults` çağırmaz. Yani settings seviyesindeki `theme_overrides`, fg-only status değerinden otomatik background üretmez ve `element_selection_background` değerini player selection'dan türetmez. Bu iki türetme yalnızca tam user theme yüklemesindeki `refine_theme` akışına aittir. Buna karşılık syntax override'ları gerçekten `SyntaxTheme::merge(...)` ile mevcut syntax üstüne field-bazlı bindirilir.

Settings observer (Zed paritesi `theme_settings::init`, `theme_settings/src/theme_settings.rs:85-142`):

```rust
pub fn observe_tema_ayarlari(cx: &mut App) {
    let settings = TemaAyarlari::get_global(cx);
    let mut prev_buffer_font_size_settings = settings.buffer_font_size_settings();
    let mut prev_ui_font_size_settings = settings.ui_font_size_settings();
    let mut prev_agent_ui_font_size_settings = settings.agent_ui_font_size_settings();
    let mut prev_agent_buffer_font_size_settings = settings.agent_buffer_font_size_settings();
    let mut prev_theme_name = settings.theme.name(SystemAppearance::global(cx).0);
    let mut prev_icon_theme_name = settings.icon_theme.name(SystemAppearance::global(cx).0);
    let mut prev_theme_overrides = (
        settings.experimental_theme_overrides.clone(),
        settings.theme_overrides.clone(),
    );

    cx.observe_global::<AyarStore>(move |cx| {
        let settings = TemaAyarlari::get_global(cx);
        let buffer_font_size_settings = settings.buffer_font_size_settings();
        let ui_font_size_settings = settings.ui_font_size_settings();
        let agent_ui_font_size_settings = settings.agent_ui_font_size_settings();
        let agent_buffer_font_size_settings = settings.agent_buffer_font_size_settings();
        let theme_name = settings.theme.name(SystemAppearance::global(cx).0);
        let icon_theme_name = settings.icon_theme.name(SystemAppearance::global(cx).0);
        let theme_overrides = (
            settings.experimental_theme_overrides.clone(),
            settings.theme_overrides.clone(),
        );

        // Settings dosyasındaki baz font size değişirse runtime override
        // global'ini sıfırla — kullanıcının `cmd-+` ile büyüttüğü değer
        // settings dosyası "hakikat kaynağı"na yenilirse drop olur.
        if buffer_font_size_settings != prev_buffer_font_size_settings {
            prev_buffer_font_size_settings = buffer_font_size_settings;
            reset_buffer_font_size(cx);
        }
        if ui_font_size_settings != prev_ui_font_size_settings {
            prev_ui_font_size_settings = ui_font_size_settings;
            reset_ui_font_size(cx);
        }
        if agent_ui_font_size_settings != prev_agent_ui_font_size_settings {
            prev_agent_ui_font_size_settings = agent_ui_font_size_settings;
            reset_agent_ui_font_size(cx);
        }
        if agent_buffer_font_size_settings != prev_agent_buffer_font_size_settings {
            prev_agent_buffer_font_size_settings = agent_buffer_font_size_settings;
            reset_agent_buffer_font_size(cx);
        }

        if theme_name != prev_theme_name || theme_overrides != prev_theme_overrides {
            prev_theme_name = theme_name;
            prev_theme_overrides = theme_overrides;
            reload_theme(cx);
        }
        if icon_theme_name != prev_icon_theme_name {
            prev_icon_theme_name = icon_theme_name;
            reload_icon_theme(cx);
        }
    }).detach();
}
```

> **Tüm 7 değişken zorunludur:** Observer 4 font size, 2 theme name ve 1 theme_overrides alanını izler. Font size'lar izlenmezse, kullanıcı settings dosyasında `buffer_font_size`'ı değiştirdiğinde runtime eski override değerini göstermeye devam edebilir.

Tema seçici davranışı:

```text
liste kaynağı:
  ThemeRegistry::list() -> Vec<ThemeMeta { name, appearance }>

preview:
  seçici içinde highlight değişince GlobalTheme::update_theme ile
  geçici tema uygulanır, refresh_windows çağrılır

confirm:
  settings dosyası ThemeSelection olarak güncellenir:
    - Static ise seçilen ad tek değer olur
    - Dynamic ise seçilen temanın appearance'ına göre light/dark slot'u güncellenir
    - mode=system ve seçilen tema sistem görünümünden farklıysa mode light/dark'a çekilir

dismiss/cancel:
  açılıştaki tema adı saklanır; seçici kapanınca confirm edilmediyse
  önizleme öncesi tema geri yüklenir ve refresh_windows çağrılır
```

Bu modelle uygulama, Zed'deki gibi iki davranışı da sunar: kullanıcı ister tek tema seçer, ister sistem moduna göre light/dark temaları ayrı tutar.

### Settings mutator helper'ları (Zed paritesi)

Zed `crates/theme_settings/src/settings.rs` içinde **runtime global'i değil**, kullanıcı ayar dosyasının `SettingsContent` AST'ini güvenli biçimde mutate eden üç public helper sunar:

```rust
// theme_settings::settings içinde:
pub fn set_theme(
    current: &mut SettingsContent,
    theme_name: impl Into<Arc<str>>,
    theme_appearance: Appearance,
    system_appearance: Appearance,
);

pub fn set_icon_theme(
    current: &mut SettingsContent,
    icon_theme_name: IconThemeName,
    appearance: Appearance,
);

pub fn set_mode(content: &mut SettingsContent, mode: ThemeAppearanceMode);
```

| Fonksiyon | İş yaptığı yer | Karar mantığı |
|-----------|----------------|---------------|
| `set_theme` | `settings.theme.theme` (`Option<ThemeSelection>`) | `Static` ise adı değiştirir, `Dynamic` ise `theme_appearance`'a göre `light` veya `dark` slot'unu günceller. `mode == System` iken seçilen appearance sistem appearance'ından farklıysa `mode`'u seçilen tarafa kilitler |
| `set_icon_theme` | `settings.theme.icon_theme` | `Dynamic` modda mevcut mode'a göre `light` veya `dark` slot'unu yazar; `Static` durumunda tek slot'u günceller. `Option<IconThemeSelection>` `None` ise `Static` ile başlatır |
| `set_mode` | `settings.theme.theme` | Mevcut `Static` seçimi `Dynamic { mode = System, light = DEFAULT_LIGHT_THEME, dark = DEFAULT_DARK_THEME }` ile değiştirir; mevcut `Dynamic` ise yalnızca `mode`'u günceller; `None` durumunda `Dynamic`'i baştan kurar |

**`kvs_tema` karşılığı:** Bu üç fonksiyon `kvs_tema` runtime API'sinin değil, selector / settings UI köprüsünün sorumluluğudur. Mirror crate yapısında ya `kvs_tema_ayarlari` ya da `kvs_secici` modülünde tutulur. Selector confirm akışında dosya yazma sırası şöyle kurulur:

```rust
pub fn confirm_selection(
    secilen: &ThemeMeta,
    cx: &mut App,
) -> anyhow::Result<()> {
    let system = SystemAppearance::global(cx).0;

    // 1. Önce in-memory SettingsContent'i mutate et.
    let mut content = SettingsStore::global(cx).user_settings_content().clone();
    set_theme(&mut content, secilen.name.clone(), secilen.appearance, system);

    // 2. Diske persist et (file watcher Konu 26 ile reload'u tetikler).
    SettingsStore::global(cx).write_user_settings(content)?;

    // 3. Observer (Konu 38) reload_theme'i çağırır; explicit
    //    GlobalTheme::update_theme + refresh_windows burada GEREKMEZ.
    Ok(())
}
```

**Tuzak:** Selector preview için `GlobalTheme::update_theme` + `refresh_windows` çağrıldıktan sonra kullanıcı confirm yerine dismiss seçerse, settings dosyası yazılmamış olur ama runtime hâlâ önizleme temasını gösterir. Cancel akışında preview öncesi tema adı saklanmalı ve `GlobalTheme::update_theme(cx, eski)` ile geri yüklenmelidir.

### `reload_theme` / `reload_icon_theme` — observer reaksiyonu

Zed `crates/theme_settings/src/theme_settings.rs` içinde iki public reload helper'ı tanımlar:

```rust
pub fn reload_theme(cx: &mut App);
pub fn reload_icon_theme(cx: &mut App);
```

Davranış (`theme_settings.rs:185-196`):

1. `configured_theme(cx)` (veya `configured_icon_theme(cx)`) ile aktif seçim ve override'lar yeniden çözülür.
2. `GlobalTheme::update_theme` veya `update_icon_theme` ile global yazılır.
3. `cx.refresh_windows()` çağrılır.

Settings observer (`init` içindeki `cx.observe_global::<SettingsStore>`) font size, theme name, icon theme name veya theme override'ların değiştiğini fark ettiğinde ilgili reload helper'ını çağırır. Yani Settings'i mutate etmek otomatik olarak runtime'a yansır; selector confirm akışında ek bir `update_theme` çağrısına gerek kalmaz (önizleme yazılmadığı sürece).

`kvs_tema` mirror tarafında bu iki fonksiyon `pub fn temayi_yeniden_yukle(cx)` ve `pub fn icon_temayi_yeniden_yukle(cx)` olarak yer alır; observer'ı kuran `init` fonksiyonu da Zed'deki `theme_settings::init`'in karşılığıdır.

### Sistem mod takipli otomatik tema

```rust
pub fn observe_system_mod_ile_tema_takibi(
    window: &mut Window,
    cx: &mut Context<impl 'static>,
) {
    cx.observe_window_appearance(window, |_, window, cx| {
        let kategori = match window.appearance() {
            WindowAppearance::Dark | WindowAppearance::VibrantDark => Appearance::Dark,
            WindowAppearance::Light | WindowAppearance::VibrantLight => Appearance::Light,
        };

        // SystemAppearance güncelle
        cx.set_global(GlobalSystemAppearance(SystemAppearance(kategori)));

        // Mevcut tema'nın appearance'ı sistemle uyumlu mu?
        let mevcut = cx.theme();
        if mevcut.appearance != kategori {
            let ad = match kategori {
                Appearance::Dark => "Kvs Default Dark",
                Appearance::Light => "Kvs Default Light",
            };
            let _ = temayi_degistir(ad, cx);
        }
    }).detach();
}
```

> **Kullanıcı tercihini ezme uyarısı:** Bu fonksiyon sistem değişiminde otomatik tema değiştirir. Bu, kullanıcının manuel seçiminin sistem değişiminde kaybolması anlamına gelebilir. Production akışında `ayar.mod_takibi: bool` bayrağı ile koşullu çalıştırılması yerinde olur.

### Tema reload

Kullanıcı tema dosyasını editörden değiştirdiğinde uygulamanın yeniden okumaya geçmesi:

```rust
pub fn temayi_yeniden_yukle(
    yol: &Path,
    cx: &mut App,
) -> anyhow::Result<()> {
    let bytes = std::fs::read(yol)?;
    let family: ThemeFamilyContent = serde_json_lenient::from_slice(&bytes)?;

    let baseline_dark = fallback::kvs_default_dark();
    let baseline_light = fallback::kvs_default_light();

    let registry = ThemeRegistry::global(cx);
    let themes: Vec<Theme> = family
        .themes
        .into_iter()
        .map(|theme_content| {
            let baseline = match theme_content.appearance {
                AppearanceContent::Dark => &baseline_dark,
                AppearanceContent::Light => &baseline_light,
            };
            Theme::from_content(theme_content, baseline)
        })
        .collect();
    registry.insert_themes(themes);  // Aynı isim üzerine yazar

    // Aktif tema yeniden yüklendi mi? Re-set ile observer'ları tetikle.
    let aktif_ad = cx.theme().name.clone();
    if let Ok(yeni) = registry.get(&aktif_ad) {
        GlobalTheme::update_theme(cx, yeni);
        cx.refresh_windows();
    }

    Ok(())
}
```

**Akış:**

1. Disk'ten okunur ve parse edilir.
2. Her tema varyantı için uygun baseline seçilir (light → light baseline).
3. `registry.insert` çağrısı eski kaydın üstüne yazar — aynı isim güncellenir.
4. Aktif tema yeniden yüklendiyse `GlobalTheme::update_theme` ile global güncellenir ve `refresh_windows` çağrılır.

### Performans

| Operasyon | Süre | Hot path? |
|-----------|------|-----------|
| `registry.get(name)` | O(1) HashMap lookup | Sık (her tema değişimde) |
| `GlobalTheme::update_theme` | Global update + observer trigger | Sık |
| `refresh_windows` | Tüm açık view ağaçları | Sık |
| `Theme::from_content` (reload) | ~25–60 µs (Konu 32) | Nadir |
| Tek tema değişimi toplam | ~2–5 ms (next frame'de görünür) | Kullanıcı tetikler |

### Tuzaklar

1. **`registry.get` üzerine `unwrap`**: Hata UI'da görünür kılınmalı, panic'e dönüşmemelidir. `?` ile propagate ya da bir match deseni kullanılır.
2. **Sistem mod takipli akışta kullanıcı tercihinin ezilmesi**: `ayar.mod_takibi` bayrağı ile koşullu çalıştırma yerinde olur.
3. **Async reload'da `cx` lifetime'ı**: `cx.spawn` içinde `cx` `AsyncApp`'tir; sync bağlamaya `cx.update(|cx| ...)` ile geçiş yapılır.

---


### `ThemeSettingsProvider` — settings entegrasyon trait'i

**Kaynak:** `crates/theme/src/theme_settings_provider.rs:9`.

Hedeflenen Zed referansında `crates/theme`, `crates/theme_settings`'i **doğrudan tüketmez**. Bunun yerine `ThemeSettingsProvider` adında bir trait sunar. Settings crate'i bu trait'i implement eder ve `crates/theme` çalışma zamanında provider'ı sorgular. Böylece bağımlılık yönü temiz kalır: tema crate'i settings'e bağımlı değildir; settings crate'i tema'ya bir hizmet sunar.

```rust
use gpui::{App, Font, Pixels};

pub trait ThemeSettingsProvider: Send + Sync + 'static {
    fn ui_font<'a>(&'a self, cx: &'a App) -> &'a Font;
    fn buffer_font<'a>(&'a self, cx: &'a App) -> &'a Font;
    fn ui_font_size(&self, cx: &App) -> Pixels;
    fn buffer_font_size(&self, cx: &App) -> Pixels;
    fn ui_density(&self, cx: &App) -> UiDensity;
}

pub fn set_theme_settings_provider(provider: Box<dyn ThemeSettingsProvider>, cx: &mut App);
pub fn theme_settings(cx: &App) -> &dyn ThemeSettingsProvider;
```

**Sözleşme sınırı:** Bu trait aktif tema adını veya aktif icon tema adını döndürmez. Zed'de provider yalnızca typography ve density okumaları için vardır. Selector state'i `ThemeSettingsContent.theme` ve `ThemeSettingsContent.icon_theme` alanlarından çözülür. Provider trait `agent_font_size`, `active_theme_name` ya da `active_icon_theme_name` gibi metotlar içermez.

**`kvs_tema`'da karşılığı:**

```rust
// kvs_tema/src/settings_provider.rs
use gpui::{App, Font, Pixels};

pub trait TemaAyarSaglayici: Send + Sync + 'static {
    fn ui_font<'a>(&'a self, cx: &'a App) -> &'a Font;
    fn buffer_font<'a>(&'a self, cx: &'a App) -> &'a Font;
    fn ui_font_size(&self, cx: &App) -> Pixels;
    fn buffer_font_size(&self, cx: &App) -> Pixels;
    fn ui_density(&self, cx: &App) -> UiDensity;
}

struct GlobalTemaAyarSaglayici(Box<dyn TemaAyarSaglayici>);
impl Global for GlobalTemaAyarSaglayici {}

pub fn set_tema_ayar_saglayici(provider: Box<dyn TemaAyarSaglayici>, cx: &mut App) {
    cx.set_global(GlobalTemaAyarSaglayici(provider));
}

pub fn tema_ayarlari(cx: &App) -> &dyn TemaAyarSaglayici {
    &*cx.global::<GlobalTemaAyarSaglayici>().0
}
```

**Bağlama akışı:**

```rust
// kvs_uygulama/src/main.rs
struct KvsAyarSaglayici;

impl TemaAyarSaglayici for KvsAyarSaglayici {
    fn ui_font<'a>(&'a self, cx: &'a App) -> &'a Font {
        &kvs_ayarlari::get(cx).ui_font
    }
    fn buffer_font<'a>(&'a self, cx: &'a App) -> &'a Font {
        &kvs_ayarlari::get(cx).buffer_font
    }
    fn ui_font_size(&self, cx: &App) -> Pixels {
        kvs_ayarlari::get(cx).ui_font_size
    }
    fn buffer_font_size(&self, cx: &App) -> Pixels {
        kvs_ayarlari::get(cx).buffer_font_size
    }
    fn ui_density(&self, cx: &App) -> UiDensity {
        kvs_ayarlari::get(cx).ui_density
    }
}

fn main() {
    Application::new().run(|cx| {
        kvs_tema::init(LoadThemes::All(Box::new(KvsAssets)), cx);
        kvs_ayarlari::init(cx);
        kvs_tema::set_tema_ayar_saglayici(Box::new(KvsAyarSaglayici), cx);
        // ...
    });
}
```

**Neden bir trait?**

- Tema crate'i settings crate'inin tipini bilmez; yalnızca davranışını sözleşme olarak kabul eder.
- Test ortamında `MockTemaAyarSaglayici` enjekte edilebilir; gerçek bir settings store'u kurmaya gerek kalmaz.
- Settings dosya formatı değişirse (`config.toml` → `settings.json`) trait imzası değişmeden kalır.

**Konu 38 (`temayi_degistir`) ile ilişki:** `temayi_degistir` çağrıldığında ayar dosyasının da güncellenmesi istenirse `tema_ayarlari(cx)` üzerinden mutable bir API tasarlanır; Zed'de bunu `update_settings_file` üstlenir. `kvs_tema` settings crate'inin sözleşmesine bağımlı olmadığı için bu çağrı **tüketici tarafında** kalır.

**`ThemeSettingsContent` alan modeli:**

`crates/settings_content/src/theme.rs` tarafındaki settings şeması provider'dan daha geniştir; kullanıcı ayar dosyası burada temsil edilir. `ThemeSettingsContent` şu alanları taşır:

```text
ui_font_size, ui_font_family, ui_font_fallbacks, ui_font_features,
ui_font_weight, buffer_font_family, buffer_font_fallbacks,
buffer_font_size, buffer_font_weight, buffer_line_height,
buffer_font_features, agent_ui_font_size, agent_buffer_font_size,
markdown_preview_font_family, markdown_preview_code_font_family,
markdown_preview_theme, theme, icon_theme, ui_density,
unnecessary_code_fade, experimental_theme_overrides, theme_overrides
```

Bu alanların yardımcı tipleri de şemaya dahildir:

| Tip | Rol | Kritik sözleşme |
|-----|-----|-----------------|
| `ThemeSettingsContent` | Kullanıcı settings dosyasındaki tema/font/density alanları | 22 alan; `#[serde(default)]` ve `MergeFrom` davranışı korunur |
| `FontSize` | `f32` pixel newtype'ı | Serialize edilirken iki ondalık basamak tutar |
| `FontFamilyName` | font family adı | `#[serde(transparent)]`, `Arc<str>` |
| `FontFeaturesContent` | OpenType feature map'i | 4 karakter alfanumerik key; boolean veya unsigned integer value |
| `BufferLineHeight` | `comfortable`, `standard`, `custom(f32)` | custom değer `>= 1.0` olmalıdır |
| `CodeFade` | gereksiz kod fade oranı | schema aralığı `0.0..=0.9` |
| `DEFAULT_LIGHT_THEME` / `DEFAULT_DARK_THEME` | settings fallback adları | `"One Light"` / `"One Dark"` tek kaynak olarak korunur |

`agent_ui_font_size` ve `agent_buffer_font_size` provider trait'inde yer almaz; agent panel ayarı olarak settings katmanında kalır. `theme`, `icon_theme`, `markdown_preview_theme`, `experimental.theme_overrides` ve `theme_overrides` selector ve override akışına gider; typography helper'ları ise provider üzerinden `ui_font`, `buffer_font`, `ui_font_size`, `buffer_font_size` ve `ui_density` değerlerini okur. `markdown_preview_code_font_family` provider trait'ine eklenmez; markdown preview tüketicisi `ThemeSettings` üzerinden okur ve değer boş kaldığında `buffer_font.family` kullanır.

Tema renklerini tüketen her ayar `ThemeSettingsContent` içine girmez. `completion_menu_item_kind` bunun yeni örneğidir: schema sahibi `EditorSettingsContent`'tir, değerleri `off` ve `symbol` olur, default `off`'tur. `symbol` açıldığında completion menüsü aktif syntax theme'den capture rengi okur; bu yüzden tema dokümantasyonunda ele alınır, ama `ThemeSettingsProvider` veya `ThemeSettingsContent` sözleşmesine eklenmez. Eski ad, alias veya geriye uyumluluk katmanı tanımlanmaz.

### Font ayarları runtime API'leri (`adjust_*`, `reset_*`, override global'leri)

**Kaynak modüller:** `crates/theme_settings/src/settings.rs` ve `crates/theme_settings/src/theme_settings.rs`.

Zed font ölçeklemesini iki katmanlı çalıştırır: ayar dosyasındaki taban değer (`ThemeSettings.{ui,buffer,agent_ui,agent_buffer}_font_size`) ve **runtime override global'leri**. Override global'i set edildiğinde `ThemeSettings::*_font_size(cx)` accessor'ı önce global'i okur; yoksa settings değerine düşer. Böylece kullanıcı `cmd-+`/`cmd--` ile font'u geçici olarak büyütebilir ve settings dosyası yazılmaz.

```rust
// Override global'leri (Pixels newtype'ları):
struct BufferFontSize(Pixels);               // private, settings.rs:96
pub(crate) struct UiFontSize(Pixels);        // crate-içi, settings.rs:102
pub struct AgentUiFontSize(Pixels);          // public, settings.rs:108
pub struct AgentBufferFontSize(Pixels);      // public, settings.rs:114

impl Global for BufferFontSize {}      // ... her biri için
```

Public yüzey:

```rust
// Düzenle (callback ile)
pub fn adjust_buffer_font_size(cx: &mut App, f: impl FnOnce(Pixels) -> Pixels);
pub fn adjust_ui_font_size(cx: &mut App, f: impl FnOnce(Pixels) -> Pixels);
pub fn adjust_agent_ui_font_size(cx: &mut App, f: impl FnOnce(Pixels) -> Pixels);
pub fn adjust_agent_buffer_font_size(cx: &mut App, f: impl FnOnce(Pixels) -> Pixels);

// Override'ı kaldır → settings değerine düş
pub fn reset_buffer_font_size(cx: &mut App);
pub fn reset_ui_font_size(cx: &mut App);
pub fn reset_agent_ui_font_size(cx: &mut App);
pub fn reset_agent_buffer_font_size(cx: &mut App);

// ±1 px convenience (theme_settings.rs:420, 426)
pub fn increase_buffer_font_size(cx: &mut App);
pub fn decrease_buffer_font_size(cx: &mut App);

// Yardımcılar (settings.rs)
pub fn clamp_font_size(size: Pixels) -> Pixels;
pub fn adjusted_font_size(size: Pixels, cx: &App) -> Pixels;
pub fn observe_buffer_font_size_adjustment<V: 'static>(
    cx: &mut Context<V>,
    f: impl 'static + Fn(&mut V, &mut Context<V>),
) -> Subscription;
pub fn setup_ui_font(window: &mut Window, cx: &mut App) -> gpui::Font;
```

`adjust_*` her zaman aynı 4 adımı izler:

1. `ThemeSettings::get_global(cx).*_font_size(cx)` (veya `*_font_size_settings()`) ile **mevcut baz değer** okunur.
2. `cx.try_global::<*FontSize>().map_or(base, |g| g.0)` ile override varsa o değer, yoksa baz değer alınır.
3. Callback çağrılır; sonuç `clamp_font_size` ile `[MIN_FONT_SIZE, MAX_FONT_SIZE]` aralığına sıkıştırılır ve `cx.set_global(*FontSize(...))` ile yazılır.
4. `cx.refresh_windows()` çağrılır.

`reset_*` ise `cx.has_global::<*FontSize>()` durumunda `remove_global` ve ardından `refresh_windows` çalıştırır. Override yoksa no-op'tur; gereksiz redraw üretmez.

**Sayısal sabitler** (`theme_settings/src/settings.rs:18-20`):

```rust
const MIN_FONT_SIZE: Pixels = px(6.0);
const MAX_FONT_SIZE: Pixels = px(100.0);
const MIN_LINE_HEIGHT: f32 = 1.0;
```

`clamp_font_size` bu iki const'a göre sıkıştırma yapar. `MIN_LINE_HEIGHT` ise `ThemeSettings::line_height()` accessor'ında kullanılır (`theme_settings/src/settings.rs:451-453`):

```rust
pub fn line_height(&self) -> f32 {
    f32::max(self.buffer_line_height.value(), MIN_LINE_HEIGHT)
}
```

Yani `BufferLineHeight::Custom(0.5)` gibi geçersiz değerler bile accessor seviyesinde `1.0`'a yükseltilir. `deserialize_line_height` parse aşamasında zaten 1.0 alt sınırını zorlar; accessor tarafındaki kontrol ise in-memory override veya bug durumlarına karşı ikinci savunmadır. Mirror tarafta aynı çift koruma uygulanmalıdır.

**Settings observer ilişkisi:** `theme_settings::init` içindeki observer, ayar dosyasındaki taban değer değiştiğinde override'ı **otomatik olarak sıfırlar** (`reset_*` çağırır). Yani kullanıcı `cmd-+` ile büyüttüğü font'u settings dosyası üzerinden düzenlediğinde override drop edilir. Settings dosyası hakikat kaynağı rolünü korur.

**`kvs_tema` karşılığı:** Bu API ailesi `kvs_tema` runtime crate'inin değil, **settings/UI köprüsünün** sorumluluğudur. Mirror tarafta üç strateji vardır:

| Strateji | Açıklama | Ne zaman |
|----------|----------|----------|
| Provider trait'ini genişletme | `TemaAyarSaglayici`'a `adjust_*`/`reset_*` eklenir | `kvs_tema` tüketicilerinin font değişimini dinlemesi gerekiyorsa |
| Sade newtype mirror | `BufferFontSize` vb. global'leri `kvs_tema_ayarlari` crate'inde tutulur, `adjust_*`/`reset_*` orada implement edilir | Settings UI'sı bağımsız bir crate olduğunda |
| Atlama | UI yoksa hiç mirror edilmez | Font picker kapsam dışı bırakıldıysa |

Sözleşme parite bayrağı şudur: bu fonksiyonlar `kvs_tema` public API'sinde yer almaz.

---

## 40. `UiDensity` — UI yoğunluk ayarı

**Kaynak:** `crates/theme/src/ui_density.rs:21`.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum UiDensity {
    Compact,
    #[default]
    Default,
    Comfortable,
}
```

**Rol:** Kullanıcının UI'da tercih ettiği yoğunluğu temsil eder. Buton padding'leri, liste item yükseklikleri ve panel iç boşlukları bu enuma göre ölçeklenir.

**Tema sözleşmesindeki yeri:** `UiDensity` `Theme` içinde **yer almaz**. Ayrı bir kullanıcı tercihi olarak `TemaAyarSaglayici` üzerinden okunur. `ThemeColors` ile karıştırılmamalıdır; bu bir renk değil, boyut tercihidir.

> **Content/Runtime tip duplication:** `UiDensity` Zed'de **iki yerde** tanımlıdır:
>
> - `settings_content::theme::UiDensity` (`settings_content/src/theme.rs:374`): content tipi, `JsonSchema + MergeFrom + Serialize + Deserialize` derive'larıyla beraber.
> - `theme::ui_density::UiDensity` (`theme/src/ui_density.rs:21`): runtime tipi.
>
> Aralarındaki köprü `theme_settings::settings::ui_density_from_settings` adındaki `pub(crate)` bir helper'dır (`theme_settings/src/settings.rs:22-28`); `From` trait kullanılmaz; çünkü iki tipin değişik derive zincirleri arasındaki dönüşüm `theme_settings` crate'i içinde özel kalır. `ThemeSettings::from_settings` bu helper'ı çağırır: `ui_density: ui_density_from_settings(content.ui_density.unwrap_or_default())`.
>
> Mirror tarafta aynı duplication zorunlu değildir; tek bir `UiDensity` tipi kullanılabilir. Ancak `JsonSchema`/`MergeFrom` derive zincirini runtime hot path'ine eklemek istenmiyorsa ayrım korunur.

**Tüketici kullanım deseni:**

```rust
pub fn density_padding(density: UiDensity) -> Pixels {
    match density {
        UiDensity::Compact     => px(6.0),
        UiDensity::Default     => px(8.0),
        UiDensity::Comfortable => px(12.0),
    }
}

impl Render for Toolbar {
    fn render(&mut self, _w: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let density = kvs_tema::tema_ayarlari(cx).ui_density(cx);
        let colors = cx.theme().colors();

        div()
            .p(density_padding(density))
            .bg(colors.background)
            .child("...")
    }
}
```

**`bilesen_rehberi.md` ile köprü:** `DynamicSpacing::BaseXX.px(cx)` helper'ı zaten `UiDensity`'i bilir — `ui::ui_density(cx)` ile şu anki yoğunluk sorgulanır. Bağımsız bir component crate kullanılıyorsa `tema_ayarlari(cx).ui_density(cx)` çağrısının GPUI spacing helper'larına bağlanması gerekir.

**JSON kullanıcı ayarı:**

```jsonc
{
  "unstable.ui_density": "comfortable"
}
```

> **JSON anahtarı `"unstable.ui_density"`'dir** (`settings_content/src/theme.rs:166`). `ThemeSettingsContent.ui_density` alanı `#[serde(rename = "unstable.ui_density")]` ile işaretlenir. Düz bir `"ui_density"` anahtarı **tanınmaz**, parse aşamasında `None` kalır ve default değer (`UiDensity::Default`) etkin olur. Mirror tarafta aynı rename konulmalıdır; çünkü hedeflenen Zed sözleşmesinde geçerli anahtar budur. `"ui_density"` gibi alternatif anahtarlar desteklenmez.

---
