# Ürün titlebar'ı ve uygulamaya bağlama

Platform kabuğu hazır hale geldikten sonra sıra üst katmana gelir. Ürünün kendi başlık içeriği, sidebar bilgisi, menüleri ve uygulama shell'i bu kabuğa bağlanır. Bu bölüm, "platform kabuğu çalışıyor" noktasından "kullanıcının gördüğü tam başlık çubuğu hazır" noktasına geçişi anlatır.

## 15. Sidebar ve workspace etkileşimi

`PlatformTitleBar`, isteğe bağlı olarak bir `MultiWorkspace` zayıf referansı alabilir. Bu referansın tek amacı vardır: başlık çubuğundaki pencere kontrollerinin sidebar ile görsel olarak çakışmasını önlemek. Bu görev için yapılanlar şunlardır:

- Sol sidebar açıksa, sol taraftaki pencere kontrolleri gizlenir.
- Sağ sidebar açıksa, sağ taraftaki pencere kontrolleri gizlenir.
- CSD köşe yuvarlama, sidebar'ın temas ettiği tarafta kapatılır; böylece köşedeki gölge ve yuvarlama paneli kesip rahatsız edici bir görünüm vermez.

Zed'de bu bilgi `SidebarRenderState { open, side }` tipinde gelir. Port hedefinde sol veya sağ panel varsa, aynı soyutlamanın daha küçük bir tipe indirgenmesi yeterli olur:

```rust
#[derive(Default, Clone, Copy)]
struct ShellSidebarState {
    open: bool,
    side: SidebarSide,
}
```

Sidebar kavramının hiç bulunmadığı bir uygulamada bu alan tamamen kaldırılabilir. Alternatif olarak her zaman default state döndüren bir implementation da verilebilir.

`PlatformTitleBar::is_multi_workspace_enabled(cx)` fonksiyonu Zed'de ilginç biçimde yalnız `DisableAiSettings` ayarı üzerinden değer üretir. İsim ürün açısından kafa karıştırıcıdır; davranış ise aslında bir feature flag gibi çalışır. Fakat pencere kontrollerinin gizlenmesinde kullanılan gerçek `SidebarRenderState`, `MultiWorkspace::sidebar_render_state(cx)` üzerinden gelir; burada `open` değeri `self.sidebar_open() && self.multi_workspace_enabled(cx)` şeklinde hesaplanır ve `MultiWorkspace::multi_workspace_enabled(cx)` hem `DisableAiSettings::disable_ai` hem de `AgentSettings::enabled` değerine bakar. Yani Zed'de ürün başlığı tarafındaki helper ile sidebar state'inin tam kaynakları aynı değildir. Port hedefinde bu kontrolün `AppSettings::multi_workspace` veya `ShellSettings::sidebar_enabled` gibi doğrudan anlaşılır tek bir ayar ailesine bağlanması daha iyi olur.

Zed'in ürün modelinde klasör ve proje açma davranışı şöyle çalışır: varsayılan olarak yeni pencere açılmaz. Klasör veya proje, mevcut pencerenin threads sidebar'ına eklenir. `File > Open`, `File > Open Recent`, klasör sürükleme ve komut satırında `zed ~/project` çağrısı gibi yolların hepsi aynı pencere içinde workspace değişikliğine yol açabilir. Yeni pencere açmak için Open Recent'ta Cmd/Ctrl+Enter ya da CLI tarafında `zed -n` kullanılır. `cli_default_open_behavior` varsayılan olarak `existing_window` kaldığı sürece, CLI üzerinden yapılan açılışlar da mevcut pencere/sidebar yolunu takip eder.

Yeni Zed akışında `OpenMode::Activate` da `OpenMode::NewWindow` gibi pencereyi öne alır. Bu, üst bar için küçük ama önemli bir sözleşmedir: proje mevcut pencereye eklense bile "aktif proje değişti" olayı pasif bir state değişimi gibi ele alınmaz. Port hedefinde proje/sidebar aktivasyonu yapıldığında titlebar başlığı, aktif worktree bilgisi ve pencere active/inactive renkleri aynı render turunda güncellenmelidir. Eski davranışı taklit etmek için `Activate` modunda pencere aktivasyonunu atlayan bir uyumluluk yolu tutulmaz.

Bu davranış `PlatformTitleBar` render sözleşmesini değiştirmez. Zayıf `MultiWorkspace` referansı platform kabuğu için yalnızca tek bir işe yarar: sidebar tarafındaki pencere kontrol çakışmasını çözmek. Ürün titlebar'ı için ise kural farklıdır. Aktif proje veya workspace, pencere değişmeden de değişebilir. Bu yüzden proje adı, worktree bilgisi, sidebar tarafı ve başlık içeriği `Window` lifecycle'ına bağlanmaz. Aktif `MultiWorkspace::workspace()` durumuna gözlemci yerleştirilerek güncellenir. Aksi halde proje değiştiği halde başlıkta güncel olmayan isim görünmeye devam eder.

"Sidebar açık mı?" sorusu ile "açık proje var mı?" sorusu birbirine karıştırılmaz. Boş workspace'lerde yeni thread veya terminal oluşturma işlemi no-op olabilir. Buna rağmen sidebar'ın açık/kapalı durumu ve sol/sağ konumu, titlebar kontrol çakışmasını çözmek için ayrı bir render state olarak tutulur. Bu iki state aynı bayrak altında birleştirilirse pencere kontrolleri yanlış durumda gizlenir.

## 16. Başlık çubuğuna içerik yerleştirme

`PlatformTitleBar` kendi başına yalnızca platform kabuğunu sağlar. Kullanıcının gördüğü gerçek başlık içeriği bu tipin sorumluluğunda değildir. Zed'in gerçek ürün başlığı `crates/title_bar/src/title_bar.rs` dosyasındaki `TitleBar` tarafından üretilir. Bu üst katman platform kabuğuna child olarak şunları geçirir:

- Uygulama menüsü.
- Proje adı / recent projects popover.
- Git branch adı ve durum ikonu.
- Restricted mode göstergesi.
- Kullanıcı menüsü, sign-in butonu, plan chip, update bildirimi.
- Collab/screen share göstergeleri.
- Feature flag'e bağlı onboarding/announcement banner'ları.
- Update bildirimi tooltip'i (`Update to Version: ...` gibi).

Update tooltip'inin metin biçimi, `crates/title_bar/src/update_version.rs:66-75` aralığındaki `version_tooltip_message` fonksiyonunda oluşturulur. Sürüm semantik ise `SemanticVersion::to_string()` çıktısı kullanılır. Commit SHA durumunda ise `AppCommitSha::full()` ile kısaltılmamış 40 karakterlik hash döner. Tooltip metni her durumda `"Update to Version:"` önekiyle başlar. Port hedefinde tooltip kabuğu yazılırken bu uzun string'in tek satıra sığacağı varsayılmamalıdır. `Tooltip::text` veya muadili bir mekanizmada genişlik sınırı düşünülür. Aksi halde tooltip taşıp ekran kenarında okunmaz hale gelebilir.

Skills duyurusu, ürün titlebar'ındaki güncel banner örneğidir. `TitleBar::new(...)` artık `banner: None` ile başlamaz; `OnboardingBanner::new(...)` sonucu `Some(...)` olarak kurulur ve görünürlüğü `.visible_when(|cx| cx.has_flag::<SkillsFeatureFlag>())` predicate'iyle belirlenir. Banner etiketi migration durumundan bağımsız olarak sabittir: prefix `"Introducing:"`, ana metin `"Skills"` olur. Kullanıcının Rules taşıyacak içeriği olup olmadığı banner metninde değil, tıklamayla açılan migration bilgi modalında anlatılır. Port hedefinde eski Claude Agent veya ACP duyuru kalıntıları için uyumluluk yolu bırakılmaz; bu alan tek bir güncel duyuru modeli olarak ele alınır.

Port hedefinde aynı kalıbın kurulması tavsiye edilir. Platform titlebar yalnızca bir shell olarak tutulur. Ürünün anlamlı varlıklarının tamamı üst seviyede bir `AppTitleBar` veya `ShellTitleBar` entity'sinde üretilir. Bu ayrım korunmadan platform kabuğunun içine ürün varlıkları doldurulmaya başlanırsa, önceki bölümlerde anlatılan katman ayrımı hızla bulanıklaşır.

Önerilen sorumluluk ayrımı şu şekildedir:

| Katman | Sorumluluk |
| :-- | :-- |
| `PlatformTitleBar` | Platform davranışı, drag alanı, pencere kontrolleri, native tabs. |
| `AppTitleBar` | Uygulama adı, aktif workspace/doküman, menüler, kullanıcı aksiyonları. |
| `AppShell` | Pencere layout'u, CSD sarmalı, titlebar + içerik kompozisyonu. |
| `AppState` | Workspace, doküman, user session, ayar ve lifecycle action'ları. |

Başlık çubuğu içeriğinde `justify_between` modifier'ı kullanıldığı için child element'leri sol, orta ve sağ grup olarak vermek pratiktir. Bu yaklaşım hem render kalıbına uyar hem de element yerleşimini bir bakışta okunur kılar:

```rust
let children = [
    h_flex()
        .id("title-left")
        .gap_2()
        .child(app_menu)
        .child(project_picker)
        .into_any_element(),
    h_flex()
        .id("title-right")
        .gap_1()
        .child(sync_status)
        .child(user_menu)
        .into_any_element(),
];

self.platform_titlebar.update(cx, |title_bar, _| {
    title_bar.set_children(children);
});
```

Interaktif child element'lerde dikkat edilmesi gereken birkaç nokta vardır:

- Butonların ve popover tetikleyicilerin tamamı click ve mouse down propagation'ını durdurmalıdır. Bunlar drag yüzeyiyle aynı katmanda durduğu için propagation engellenmezse pencere sürüklemesi tetiklenebilir.
- Uzun metinler ya `truncate()` modifier'ı ile kısaltılır ya da sabit bir `max_w(...)` değeriyle sınırlandırılır. Aksi halde uzun proje ya da dosya adları başlık çubuğunu taşırır.
- Sağ tarafta platform pencere butonlarının bulunabileceği akılda tutulmalıdır. Ürün butonlarının sağ padding'i ve flex shrink davranışı bu olasılığa göre test edilir. Aksi halde pencere butonlarıyla çakışma yaşanır.
- Fullscreen modunda native pencere kontrolleri değişebileceği için macOS ve Windows davranışları ayrı ayrı doğrulanır. Tek bir platformda iyi çalışan layout, diğer platformda fullscreen geçişinde bozulabilir.

## 17. Kendi uygulamaya dahil etme

### Doğrudan Zed crate'iyle kullanım

Zed'in workspace crate'leri, settings kayıtları ve tema altyapısı projede zaten varsa entegrasyon iskeleti aşağıdaki gibidir. Bu yol, Zed'in uygulama başlangıç kurulumuna oldukça yakın bir ortam bekler. Zed'den bağımsız bir GPUI uygulamasında doğrudan kullanım pek pratik değildir; o senaryoda port yaklaşımı daha uygundur.

```rust
use gpui::{App, Context, Entity, Render, Window, div};
use platform_title_bar::PlatformTitleBar;
use ui::prelude::*;

pub fn init(cx: &mut App) {
    PlatformTitleBar::init(cx);
}

pub struct AppShell {
    title_bar: Entity<PlatformTitleBar>,
}

impl AppShell {
    pub fn new(cx: &mut Context<Self>) -> Self {
        Self {
            title_bar: cx.new(|cx| PlatformTitleBar::new("app-title-bar", cx)),
        }
    }
}

impl Render for AppShell {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let title = div()
            .id("title")
            .text_sm()
            .child("My GPUI App")
            .into_any_element();

        self.title_bar.update(cx, |title_bar, _| {
            title_bar.set_children([title]);
        });

        v_flex()
            .size_full()
            .child(self.title_bar.clone())
            .child(div().id("content").flex_1())
    }
}
```

Bu örnekte `set_children` çağrısının `render` fonksiyonu içinde yer aldığına dikkat edilmelidir. Bunun nedeni daha önce anlatıldığı gibi, Zed kaynağında child listesinin render sırasında `mem::take` ile tüketilmesidir. Entity oluşturulurken bir kez child vermek yeterli olmaz; sonraki render'da içerik tamamen kaybolur.

### Bağımsız GPUI uygulamasına port

Zed dışında bir uygulamada doğrudan `platform_title_bar` crate'ine bağımlanmak genellikle ağır gelir. Pek çok ek crate'in de projeye sürüklenmesine yol açar. Port edilirken aşağıdaki tabloda gösterilen değişimler yapılır:

| Zed bağımlılığı | Port karşılığı |
| :-- | :-- |
| `workspace::CloseWindow` | Uygulamanızın `CloseWindow`, `CloseDocument`, `CloseProject` veya `QuitRequested` action'ı. |
| `zed_actions::OpenRecent { create_new_window: true }` | Uygulamanızın `NewWindow` veya `OpenWorkspace` action'ı. |
| `WorkspaceSettings::use_system_window_tabs` | Uygulama ayarınızdaki native tab seçeneği. |
| `ItemSettings::{close_position, show_close_button}` | Sekme kapatma butonu konumu ve görünürlüğü için kendi ayar tipiniz. |
| `MultiWorkspace` ve `SidebarRenderState` | Sol/sağ panel açık mı bilgisini veren kendi shell state'iniz. |
| `DisableAiSettings` | Multi workspace veya sidebar davranışını açıp kapatan kendi feature flag'iniz. |
| `cx.theme().colors().title_bar_background` | Kendi tema sisteminizdeki titlebar token'ı. |

Pratik port sınırı şu şekilde özetlenebilir:

- `platform_title_bar.rs` dosyasının içinden Zed workspace bağımlılıkları temizlenir; bu dosya yalnızca platform sözleşmesini ve `TitleBarController` sözleşmesini bilmeli, hiçbir workspace tipiyle doğrudan ilgilenmemelidir.
- `system_window_tabs.rs` içindeki action ve settings kullanımları ürünün kendi action ve settings tipleriyle değiştirilir. Bu maliyetli görünüyorsa native tab desteği proje kararına göre kapalı başlatılır ve ihtiyaç doğduğunda etkinleştirilir. Buradaki bağımlılıklar diğer parçalara göre daha geniştir.
- `platforms/platform_linux.rs` ve `platforms/platform_windows.rs` dosyaları diğer parçalara göre daha taşınabilirdir. Çoğu uygulamada bu dosyalar yalnız küçük değişikliklerle çalışır; özellikle Windows caption davranışı çoğunlukla olduğu gibi kalır.

---
