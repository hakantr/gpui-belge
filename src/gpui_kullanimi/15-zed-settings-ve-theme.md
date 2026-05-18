# Zed Settings ve Theme

---

## Kalıcılaştırma, Ayarlar ve Tema Akışı (Zed Tarafı)

Bu konu GPUI çekirdeği değildir; ancak yeni pencere veya UI eklenirken tema ve ayar akışını bilmek gerekir. Ana dosyalar: `crates/settings`, `crates/settings_content`, `crates/theme`, `crates/theme_settings`.

**Akış.** Ayarlar sisteminin ana adımları şu sırayla işler:

1. Kullanıcı `~/.config/zed/settings.json` veya proje `.zed/settings.json` dosyasına yazar.
2. `SettingsStore` global'i değişimi yayar.
3. `Settings` trait'leri (`WorkspaceSettings`, `ThemeSettings`, ...) kendi bölümlerini ayrıştırır.
4. UI tarafları `cx.observe_global::<SettingsStore>(...)` ile değişimi izler ve yeniden çizer.

Yeni bir ayar eklenirken önce `settings_content` içindeki JSON içerik modeline alan eklenir, sonra çalışma zamanı ayar tipi `Settings` trait'ini uygular. Bu repoda gerçek trait, ilişkili içerik tipi veya `load` yöntemi yerine `from_settings` kullanır:

```rust
#[derive(Clone, Deserialize, RegisterSetting)]
pub struct YourSettings {
    pub enabled: bool,
}

impl Settings for YourSettings {
    fn from_settings(content: &settings::SettingsContent) -> Self {
        let section = &content.your_feature;
        Self {
            enabled: section.enabled.unwrap_or(false),
        }
    }
}
```

`RegisterSetting` derive'ı envanter üzerinden tipi `SettingsStore` içine kaydeder. Elle kayıt gerektiğinde uygulama başlangıcında `YourSettings::register(cx)` de kullanılabilir. Okuma tarafındaki standart giriş noktaları `Settings::get_global(cx)`, `Settings::get(path, cx)` ve `Settings::try_get(cx)` şeklindedir.

Agent ayar içeriğinde `agent.default_model` yanında `agent.subagent_model: Option<LanguageModelSelection>` de bulunur. `spawn_agent` ile açılan subagent thread'i bu ayar ayarlanmışsa onu kullanır; ayar yoksa üst agent'ın model seçimiyle devam eder. Agent ve multi-workspace davranışında `AgentSettings::enabled` de önemli bir ayar sinyalidir.

**Ayar alanları.** Aşağıdaki alanlar ayar dünyasında karşılaşılan sürprizleri içerir:

- `editor.completion_menu_item_kind` — `"off"` varsayılan, `"symbol"` seçildiğinde completion menü satırlarında LSP item kind için sözdizimi teması renginden beslenen tek harfli bir badge gösterilir.
- `git.show_stage_restore_buttons` — varsayılan `true`; editör diff hunk kontrolünde Stage/Unstage ve Restore butonlarını gösterip göstermemeyi yönetir.
- `theme.markdown_preview_code_font_family` — markdown önizlemedeki kod fontunu ayırır; ayar yoksa buffer fontuna düşer.
- Agent tool permission varsayılanlarında `skill` aracı yer alır. Ayarlar UI'sındaki tool permission sayfasında bu araç için regex açıklaması skill'in mutlak `SKILL.md` path'i üzerinden yapılır.
- Amazon Bedrock dil modeli ayarlarında `guardrail_identifier` ve `guardrail_version` alanları vardır; version belirtilmediğinde sağlayıcı tarafında `"DRAFT"` yedeği beklenir.
- `git.path_style` yanında Git UI ve diff görünümü için yeni `git` ayarları okunurken çalışma zamanı `ProjectSettings` modelinin `settings_content` ile senkron tutulduğu kontrol edilmelidir; varsayılan JSON'a eklenmeyen alan `from_settings` içinde beklenmedik bir varsayılana düşer.
- Ayarlar UI'sındaki aramada sorgu boşken sonuç üretilmez ve sorgudaki kelimelerin tamamının aday ayar ile eşleşmesi gerekir. Kendi ayar panelinizde Zed davranışına paralel kalmak için kelimelerden herhangi birinin eşleşmesini yeterli sayan eski arama mantığını kullanmayın.

**Completion menu API yüzeyi.** Completion akışında dikkat edilecek iki nokta vardır:

- `CompletionsMenu::entries` doğrudan `Box<[StringMatch]>` değil, `CompletionMenuEntry::{Match, Divider, GroupHeader}` dizisidir. Test veya UI kodu entry'lerden completion adayına erişirken `entry.as_match()` kullanmalıdır.
- `project::Completion` üreticileri `group: Option<CompletionGroup>` alanını doldurmalıdır; grup değiştiğinde menu divider ve isteğe bağlı bir grup başlığı eklenir.

**Tema renkleri.** Tema bilgisine bağlam üzerinden ulaşılır:

```rust
let colors = cx.theme().colors();
let panel_bg = colors.panel_background;
let border = colors.border;
```

`cx.theme()` aktif `ThemeVariant` (light/dark) döndürür. `.colors()`, `.players()`, `.syntax()` alt bölümlerini taşır. `theme::ActiveTheme` uzantı trait'i `App` üzerinde olduğu için `cx.theme()` doğrudan çalışır.

**Kalıcılaştırılan örnekler.** Zed'in farklı katmanları farklı kalıcılık mekanizmalarını kullanır:

- Pencere bounds (bkz. "Pencere Sınırlarının Saklanması ve Geri Yüklenmesi" başlığı).
- Açık projeler ve yakın zamanlı: `crates/recent_projects`.
- Workspace serileştirme: `crates/workspace/src/persistence.rs` ve `db` crate'i (SQLite tabanlı).
- Vim modu, panel boyutları ve dock durumu: workspace serileştirme.

**Tuzaklar.** Ayarlar tarafında yapılan tipik hatalar:

- `cx.theme()` panel açılırken `None` döndürmez; ancak `cx.global::<ThemeRegistry>()` henüz yüklenmemişse yedek tema döner.
- Ayar serileştirmesi `SettingsContent` merge akışına bağlıdır; kullanıcı/genel, proje ve dile özel kaynaklar `SettingsStore` içinde yeniden hesaplanır.
- Yeni ayar eklenirken `settings_content` schema güncellenmelidir; aksi halde JSON schema doğrulaması yeni alanı tanımaz.

## SettingsStore: Kayıt, Okuma ve Üzerine Yazma

`crates/settings/src/settings_store.rs`. `SettingsStore` Zed'in tüm ayar kaynaklarını tek bir tip-güvenli store içinde birleştirir.

**Ayar kayıt yolları.** İki kayıt yolu vardır; biri derive yoluyla otomatik, diğeri elle çağrılır:

```rust
// Derive ile envanter üzerinden otomatik kayıt:
#[derive(Clone, Deserialize, RegisterSetting)]
pub struct YourSettings { pub enabled: bool }

impl Settings for YourSettings {
    fn from_settings(content: &SettingsContent) -> Self {
        Self { enabled: content.your_feature.as_ref()
            .and_then(|f| f.enabled).unwrap_or(false) }
    }
}

// Elle kayıt (her zaman çalışan path):
YourSettings::register(cx);
```

`RegisterSetting` derive'ı `inventory::collect!` ile derleme zamanında bir topluluk yaratır. Store kurulumu `SettingsStore::new(cx, &settings::default_settings())` imzasıyla yapılır; Zed'in normal başlangıç akışında bunu `settings::init(cx)` çağırır ve oluşan store'u `cx.set_global(settings)` ile global hale getirir.

**Okuma.** Ayarı çalışma zamanında almak için kullanılan API'ler:

- `YourSettings::get_global(cx)` — aktif global değer.
- `YourSettings::get(Some(SettingsLocation { worktree_id, path }), cx)` — worktree veya `.zed/settings.json` üzerine yazmaları dahil değer.
- `YourSettings::try_get(cx)` — store kaydedilmemişse `None`.
- `YourSettings::try_read_global(async_cx, |s| ...)` — async bağlam içinde okur.

**Yazma.** Ayarın çalışma zamanında değiştirilmesi veya kalıcı kaydedilmesi için yardımcılar vardır:

- `YourSettings::override_global(value, cx)` — programatik üzerine yazma; kalıcılaştırılmaz, yalnız çalışma zamanı durumunu değiştirir.
- `settings::update_settings_file(fs, cx, |content, cx| { ... })` — kullanıcı JSON'una kalıcı yazma yoludur. Dosya okuma/yazma, ayrıştırma ve store güncelleme akışı `SettingsStore::update_settings_file(...)` üzerinden tamamlanır.
- `SettingsStore::update_user_settings(...)` yalnızca `test-support` altında mevcuttur; uygulama kodunda kalıcı yazma için kullanılmaz.

**Gözlemci akışı.** Ayar değişimini dinlemek:

```rust
cx.observe_global::<SettingsStore>(|cx| {
    let theme = ThemeSettings::get_global(cx);
    apply_theme(theme, cx);
}).detach();
```

`SettingsStore` global'i her dosya değişimi veya programatik üzerine yazmadan sonra bildirilir; gözlemci geri çağrısı içinde değer zaten yeni durumdadır.

**Tuzaklar.** SettingsStore kullanımında karşılaşılan hatalar:

- `from_settings` panic ediyorsa varsayılan JSON eksiktir; her alan `assets/settings/default.json` içinde tanımlı olmalıdır.
- Dile özel ayar gerekiyorsa `LanguageSettings::get(Some(location), cx)` çağrısı worktree özel üzerine yazmayı otomatik getirir.
- Gözlemcide `cx.notify()` çağrısı entity'yi yeniden çizmek için gereklidir; `observe_global` sadece geri çağrıyı çalıştırır, view'i geçersizleştirmez.

## ThemeRegistry, ThemeFamily ve Tema Yükleme

`crates/theme/src/registry.rs`, `crates/theme/src/theme.rs`, `crates/theme_settings/src/theme_settings.rs`.

**Tema veri modeli.** Tema dünyası birkaç temel tipten oluşur:

- `ThemeFamily { name, author, themes: Vec<Theme> }` — bir paket içindeki light ve dark varyantlar.
- `Theme { name, appearance, styles }` — belirli bir varyant.
- `ThemeColors`, `StatusColors`, `PlayerColors`, `SyntaxTheme` — alt kategoriler.
- `Appearance::Light | Dark` — temanın nominal görünüm modu.

**`ThemeRegistry`.** Tüm yüklü temalar bu registry üzerinde toplanır ve ihtiyaç duyulduğunda buradan okunur:

- `ThemeRegistry::global(cx) -> Arc<Self>` — aktif registry.
- `ThemeRegistry::default_global(cx)` ve `try_global(cx)` — init ve test kodunda registry erişimi.
- `registry.assets()` — paketlenmiş tema ve icon asset kaynağı.
- `registry.list_names() -> Vec<SharedString>` — yüklü tema adları.
- `registry.list() -> Vec<ThemeMeta>` — tema adı ve appearance meta bilgisi.
- `registry.get(name) -> Result<Arc<Theme>, ThemeNotFoundError>`.
- `registry.insert_theme_families(families)` veya `insert_themes(themes)` — tema ekleme.
- `registry.remove_user_themes(&names)` — verilen tema adlarını temizler.
- `registry.list_icon_themes()`, `get_icon_theme(name)`, `default_icon_theme()`, `load_icon_theme(content, icons_root_dir)`, `remove_icon_themes(&names)` — icon theme yönetimi.
- `registry.extensions_loaded()` ve `set_extensions_loaded()` — extension temalarının yüklenip yüklenmediği bayrağı.

**Aktif tema akışı.** Bir view içinden tema bileşenlerine ulaşmak:

```rust
let theme = cx.theme();        // &Arc<Theme>
let colors = theme.colors();   // &ThemeColors
let status = theme.status();   // &StatusColors
let players = theme.players(); // &PlayerColors
let syntax = theme.syntax();   // &SyntaxTheme
```

`cx.theme()` uzantı trait `theme::ActiveTheme` ile sağlanır; `App` üzerinde çalışır. Aktif tema `ThemeSettings` içindeki seçimden ve `SystemAppearance`'tan hesaplanır:

```rust
pub fn reload_theme(cx: &mut App) {
    let theme = configured_theme(cx);
    GlobalTheme::update_theme(cx, theme);
    cx.refresh_windows();
}
```

`reload_icon_theme(cx)` aynı modeli icon theme için uygular. `theme::init(...)` registry'yi, `SystemAppearance`'ı, font ailesi önbelleğini ve yedek `GlobalTheme`'i kurar; `theme_settings::init(...)` bunun üstüne ayar gözlemcilerini ve paketlenmiş/kullanıcı tema yükleme akışını bağlar.

**Tema ayar sağlayıcısı.** Theme crate'i font ve density bilgisini ayrı bir sağlayıcı üzerinden okur; bu sayede kendi içinde somut settings crate'ine bağımlı olmaz:

- `theme::set_theme_settings_provider(provider, cx)` — UI font, buffer font, font size ve UI density kaynağını global olarak bağlar.
- `theme::theme_settings(cx) -> &dyn ThemeSettingsProvider` — theme crate'inin somut settings crate'ine bağımlı olmadan font ve density okumasını sağlar.
- `UiDensity::{Compact, Default, Comfortable}` ve `spacing_ratio()` aralık ölçeğini verir; Zed tarafında sağlayıcı uygulaması `theme_settings` crate'inde yer alır.

**Özel tema yükleme.** Bir kullanıcı temasını programatik olarak eklemek için:

```rust
theme_settings::load_user_theme(&ThemeRegistry::global(cx), bytes)?;
theme_settings::reload_theme(cx);
```

`load_user_theme` JSON'u `ThemeFamilyContent` olarak seriden çıkarır, `refine_theme_family` ile gerçek `ThemeFamily` üretir ve `insert_theme_families` çağırır. `crates/theme_importer/` VS Code temalarından `theme_settings::ThemeContent` üretmek için yardımcılar içerir. Zed tarafında `load_user_themes_in_background` ve izleyici akışı dosya değişiminden sonra `theme_settings::reload_theme(cx)` çağırır.

**Markdown önizleme tema yüzeyi.** Önizleme tarafında tema seçimleri bir kaç ek alanla devreye girer:

- `ThemeSettingsContent::markdown_preview_code_font_family` ayarı markdown önizleme içindeki satır içi kod ve kod bloğu fontunu belirler; ayar yoksa buffer fontuna düşer. Normal `markdown_preview_font_family` yalnızca önizleme metin fontunu kontrol eder.
- Mermaid çizim akışı Zed temasından renk ve font üretir; tema değiştiğinde markdown entity'sinin `invalidate_mermaid_cache(cx)` çağırarak diyagram önbelleğini temizlemesi ve yeniden çizmesi gerekir.

**Tuzaklar.** Tema akışında karşılaşılan hatalar:

- `cx.theme()` ilk karede yedek temayı döndürebilir; gözlemci ile `SystemAppearance` veya `SettingsStore` dinlenip yeniden çizim yapılmazsa ilk çizim yedek tema ile kalır.
- `ThemeColors` tüm token'ları içerir; kullanıcı temada eksik bırakılan token `null` olduğunda varsayılan light/dark temadan yedek alınır.
- `Theme.styles.colors.background` yerine doğrudan `theme.colors().background` kullanılır; `styles` alanı iç yerleşimdir.

---
