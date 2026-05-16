# 12. Feedback ve Durum Göstergeleri

Feedback bileşenleri, kullanıcıya uygulamanın o anki durumunu anlatır. Bilgi, başarı, uyarı, hata, ilerleme, sayaç veya dikkat gerektiren kararlar bu grubun çatısı altına girer. Hepsi aynı tema token'larını paylaşır, fakat görsel yoğunlukları farklıdır. Bu yüzden önce mesajın ne kadar dikkat çekmesi gerektiğine karar verilir, sonra uygun yüzey seçilir:

- `Banner`: sayfa veya panel üstünde kısa ve akışı bölmeyen bir mesaj vermek için kullanılır.
- `Callout`: içerik akışı içinde, kullanıcının okuması ve gerekirse karar vermesi beklenen daha açıklayıcı bir mesaj için kullanılır.
- `Modal`: kendi modal içeriğini kurmak için bir shell sağlar.
- `AlertModal`: kısa karar akışları veya uyarı diyalogları için.
- `AnnouncementToast`: yeni özellik veya duyuru kartı; lifecycle'ı parent notification sistemi tarafından yönetilir.
- `CountBadge`, `Indicator`, `ProgressBar`, `CircularProgress`: küçük durum ve ilerleme göstergeleri.
- `ActivityIndicator`: workspace status alanında LSP, debug, git, filesystem, extension ve formatlama aktivitelerini tek bir kompakt trigger altında toplar.

## Severity

Kaynak:

- Tanım: `../zed/crates/ui/src/styles/severity.rs`
- Export: `ui::Severity`.
- Prelude: `ui::prelude::*` içinde otomatik gelir.

Ne zaman kullanılır:

- Bir mesajın tonunu `Info`, `Success`, `Warning` veya `Error` olarak tek bir enum üzerinden seçmek için.
- `Banner` ve `Callout` gibi bileşenlerde icon, background ve border renginin otomatik olarak eşleşmesi gerektiğinde.

Davranış:

- `Banner` ve `Callout`, severity değerinden icon ile status renklerini türetir.
- `Info` nötr ya da muted, `Success` yeşil, `Warning` sarı, `Error` kırmızı status token'larını kullanır.
- Severity, kullanıcıya gösterilen mesajın yerine geçmez. Mesaj yine kısa ve açık olmalıdır. Bir aksiyon gerekiyorsa, aksiyon ayrı bir button slot'una yerleştirilir.

## Banner

Kaynak:

- Tanım: `../zed/crates/ui/src/components/banner.rs`
- Export: `ui::Banner`.
- İlgili tipler: `ui::Severity`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for Banner`.

Ne zaman kullanılır:

- Sayfa veya panel içinde kısa bir bilgi, başarı, uyarı veya hata mesajı göstermek için.
- Kullanıcıyı akıştan koparmadan bir CTA veya düzeltme aksiyonu sunmak için.
- Bir içeriğin üstünde ya da ilgili bölümün başında non-blocking bir mesaj konumlandırmak için.

Ne zaman kullanılmaz:

- Uzun açıklama, bullet listesi veya ayrıntılı bir karar gerekiyorsa `Callout` daha uygun bir yüzeydir.
- Kullanıcının devam etmeden önce karar vermesi zorunluysa `AlertModal` daha doğru bir araçtır.
- Kısa süreli bir global bildirim lifecycle'ı gerekiyorsa, uygulama notification altyapısı ile uygun notification view birlikte kullanılır.

Temel API:

- `Banner::new()`.
- `.severity(Severity)`.
- `.action_slot(element)`.
- `.wrap_content(bool)`.
- ParentElement: `.child(...)`, `.children(...)`.

Davranış:

- Varsayılan severity değeri `Severity::Info`'dur.
- Severity'ye göre icon, background ve border rengi seçilir.
- `action_slot(...)` verildiğinde, banner sağ tarafta bir aksiyon alanı açar ve içerik padding'i bu yapıya göre düzenlenir.
- `.wrap_content(true)`, dar alanlarda içeriğin satıra kırılmasına izin verir.

Örnek:

```rust
use ui::{Banner, Button, Icon, IconName, IconSize, Severity, prelude::*};

fn render_sync_banner() -> impl IntoElement {
    Banner::new()
        .severity(Severity::Info)
        .child(Label::new("Sync in progress"))
        .action_slot(
            Button::new("view-sync", "View")
                .end_icon(Icon::new(IconName::ArrowUpRight).size(IconSize::Small)),
        )
}
```

Çok satırlı içerik gerekirse `wrap_content(true)` çağrısı satır kırılmasına izin verir:

```rust
use ui::{Banner, Severity, prelude::*};

fn render_provider_warning_banner() -> impl IntoElement {
    Banner::new()
        .severity(Severity::Warning)
        .wrap_content(true)
        .child(
            Label::new(
                "This setting is not available for the selected provider.",
            )
            .size(LabelSize::Small),
        )
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/extensions_ui/src/extensions_ui.rs`: extension upsell ve registry migration banner'ları.
- `../zed/crates/settings_ui/src/pages/tool_permissions_setup.rs`: ayar sayfası uyarıları.
- `../zed/crates/language_models/src/provider/opencode.rs`: provider durum mesajları.

Dikkat edilecek noktalar:

- Banner kısa kalmalıdır. Birden fazla paragraf veya liste gerektiren içerikler için `Callout` daha uygundur.
- `action_slot(...)` içinde birden çok aksiyon yer alacaksa, `h_flex().gap_1()` ile açık bir spacing kurulması okunabilirliği artırır.
- Banner, modal içindeki karar alanı gibi kullanılmamalıdır. Modal kararlarının footer aksiyonlarıyla verilmesi beklenir.

## Callout

Kaynak:

- Tanım: `../zed/crates/ui/src/components/callout.rs`
- Export: `ui::Callout`, `ui::BorderPosition`.
- İlgili tipler: `ui::Severity`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for Callout`.

Ne zaman kullanılır:

- İçerik içinde kullanıcının okuması gereken bir açıklamayı, sınırlamayı veya kararı göstermek için.
- Başlığı, açıklamayı, aksiyonu ve dismiss kontrolünü tek bir yüzeyde toplamak için.
- Markdown veya özel bir element gibi metin dışı bir açıklama içeriği gerekiyorsa `description_slot(...)` ile.

Ne zaman kullanılmaz:

- Yalnızca tek satırlık bir sayfa üstü mesaj için `Banner` çok daha uygundur.
- Global ve geçici bir bildirim için bir notification host kullanılır.
- Bloklayıcı bir karar gerekiyorsa `AlertModal` doğru yüzeydir.

Temel API:

- `Callout::new()`.
- `.severity(Severity)`.
- `.icon(IconName)`.
- `.title(text)`.
- `.description(text)`.
- `.description_slot(element)`.
- `.actions_slot(element)`.
- `.dismiss_action(element)`.
- `.line_height(px)`.
- `.border_position(BorderPosition::Top | BorderPosition::Bottom)`.

Davranış:

- Varsayılan severity `Severity::Info`'dur.
- `.icon(...)` çağrılmadığında icon alanı render edilmez; çağrıldığında ikon rengi severity'den türetilir.
- `.description_slot(...)` ile `.description(...)` aynı anda verildiğinde slot önceliklidir.
- Açıklama alanı `max_h_32()` ve `overflow_y_scroll()` özelliklerini kullanır; bu sayede uzun bir içerikte callout'un yüksekliği kontrol altında tutulur.
- Aksiyon ve dismiss slot'ları title satırının sağında render edilir.

Örnek:

```rust
use ui::{Button, Callout, IconButton, IconName, IconSize, Severity, prelude::*};

fn render_retry_callout() -> impl IntoElement {
    Callout::new()
        .severity(Severity::Warning)
        .icon(IconName::Warning)
        .title("Connection failed")
        .description("Retrying in 10 seconds. Check your network settings if this continues.")
        .actions_slot(Button::new("retry-now", "Retry now").label_size(LabelSize::Small))
        .dismiss_action(
            IconButton::new("dismiss-retry", IconName::Close).icon_size(IconSize::Small),
        )
}
```

Özel bir açıklama içeriği gerekiyorsa `description_slot(...)` ile çok satırlı veya kompozit bir yapı verilebilir:

```rust
use ui::{Callout, IconName, Severity, prelude::*};

fn render_permission_callout() -> impl IntoElement {
    Callout::new()
        .severity(Severity::Error)
        .icon(IconName::XCircle)
        .title("Permission denied")
        .description_slot(
            v_flex()
                .gap_1()
                .child(Label::new("The selected command cannot run in this workspace."))
                .child(Label::new("Open workspace settings to allow it.").color(Color::Muted)),
        )
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/agent_ui/src/conversation_view/thread_view.rs`: agent retry, token ve tool kullanımı uyarıları.
- `../zed/crates/zed/src/visual_test_runner.rs`: visual test durum mesajları.

Dikkat edilecek noktalar:

- `Callout` içeriği flow içinde yer alır; viewport'u kaplayan bir overlay gibi davranmaz.
- İkon gösterilmesi gerekiyorsa `.icon(...)` çağrısı açıkça yapılmalıdır.
- Description slot'una scroll yapan karmaşık bir içerik konulduğunda, içerideki metinlerin `min_w_0()` ve `.truncate()` davranışı ayrıca düşünülür.

## Modal

Kaynak:

- Tanım: `../zed/crates/ui/src/components/modal.rs`
- Export: `ui::Modal`, `ui::ModalHeader`, `ui::ModalRow`, `ui::ModalFooter`, `ui::Section`, `ui::SectionHeader`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: Doğrudan `impl Component` yok.

Ne zaman kullanılır:

- Modal içeriğini Zed'in header, section ve footer düzeniyle kurmak için.
- Çok bölümlü bir ayar, form veya seçim akışı oluşturulurken.
- Scroll handle'ı dışarıdan yönetilen bir modal body gerektiğinde.

Ne zaman kullanılmaz:

- Kısa bir uyarı ve iki aksiyonlu bir karar için `AlertModal` daha az kodla doğru davranışı verir.
- Modal dışı bir panel veya sayfa düzeni için `v_flex()` ile birlikte `Section` dışında bir layout daha uygun olur.

Temel API:

- `Modal::new(id, scroll_handle)`.
- `.header(ModalHeader)`.
- `.section(Section)`.
- `.footer(ModalFooter)`.
- `.show_dismiss(bool)`.
- `.show_back(bool)`.
- ParentElement: `.child(...)`, `.children(...)`.
- `ModalHeader::new().headline(...).description(...).icon(...).show_dismiss_button(...).show_back_button(...)`.
- `ModalFooter::new().start_slot(...).end_slot(...)`.
- `Section::new()`, `Section::new_contained()`, `.contained(bool)`, `.header(...)`, `.meta(...)`, `.padded(bool)`.
- `SectionHeader::new(label).end_slot(...)`.
- `ModalRow::new()`.

Davranış:

- Modal root `size_full()`, `flex_1()` ve `overflow_hidden()` kullanır; modal container'ı genellikle parent overlay tarafından sağlanır.
- `scroll_handle` verildiğinde body `overflow_y_scroll()` ve `track_scroll(...)` ile bağlanır.
- `show_dismiss(true)` ve `show_back(true)`, header'da Zed'in `menu::Cancel` aksiyonunu dispatch eden icon button'lar üretir.
- `Section::new_contained()` border'lı bir iç yüzey oluşturur; normal `Section` ise daha düz bir akış verir.

Örnek:

```rust
use ui::{
    Button, Modal, ModalFooter, ModalHeader, ModalRow, Section, SectionHeader, prelude::*,
};

fn render_project_settings_modal() -> impl IntoElement {
    Modal::new("project-settings-modal", None)
        .show_dismiss(true)
        .header(
            ModalHeader::new()
                .headline("Project Settings")
                .description("Changes apply to the current workspace."),
        )
        .section(
            Section::new()
                .header(SectionHeader::new("Behavior"))
                .child(
                    ModalRow::new()
                        .child(Label::new("Format on save").flex_1())
                        .child(Label::new("Enabled").color(Color::Muted)),
                ),
        )
        .footer(ModalFooter::new().end_slot(Button::new("save-settings", "Save")))
}
```

Modal lifecycle ve workspace entegrasyonu:

Zed UI tarafındaki `Modal` bileşeni yalnızca bir içerik shell'idir. Bir modal'ın açılıp kapanma davranışı bu bileşenin değil; asıl olarak `workspace::ModalLayer` ile `workspace::ModalView` trait'inin sorumluluğundadır.

```rust
use gpui::{Entity, ManagedView};
use ui::{Modal, ModalFooter, ModalHeader, Section, prelude::*};
use workspace::{ModalView, Workspace};

struct ProjectSettingsModal {
    focus_handle: gpui::FocusHandle,
}

impl gpui::EventEmitter<gpui::DismissEvent> for ProjectSettingsModal {}

impl gpui::Focusable for ProjectSettingsModal {
    fn focus_handle(&self, _cx: &App) -> gpui::FocusHandle {
        self.focus_handle.clone()
    }
}

impl ManagedView for ProjectSettingsModal {}
impl ModalView for ProjectSettingsModal {}

impl Render for ProjectSettingsModal {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        Modal::new("project-settings-modal", None)
            .header(ModalHeader::new().headline("Project Settings"))
            .section(Section::new().child(Label::new("…")))
            .footer(ModalFooter::new().end_slot(Button::new("close", "Close")))
    }
}

fn open_project_settings(
    workspace: &mut Workspace,
    window: &mut Window,
    cx: &mut Context<Workspace>,
) {
    workspace.toggle_modal::<ProjectSettingsModal, _>(window, cx, |_window, cx| {
        ProjectSettingsModal {
            focus_handle: cx.focus_handle(),
        }
    });
}
```

`ModalView` trait sözleşmesi şu maddeleri kapsar:

- `ManagedView` üzerinden gelen `Render + Focusable + EventEmitter<DismissEvent>` zorunluluğu.
- `on_before_dismiss(window, cx) -> DismissDecision`: kapanmadan önce bir validation veya kullanıcı onayı istenebilir. `DismissDecision::Pending` kapanmayı erteler; `DismissDecision::Dismiss(false)` ise iptal eder.
- `fade_out_background(&self) -> bool`: ekrandaki diğer içeriği soluklaştırmak için override edilebilir.
- `render_bare(&self) -> bool`: workspace tarafındaki `ModalLayer`'ın varsayılan elevation yüzeyini bypass etmek gerektiğinde kullanılır.

`Workspace::toggle_modal::<V, _>(window, cx, build_fn)` çağrısı, aynı modal türü zaten açıksa onu kapatır; farklı bir modal açıksa onu kapatıp yenisini açar. `ModalLayer`, dismiss event'ini dinler ve focus'u önceki elemana otomatik olarak geri verir.

Dikkat edilecek noktalar:

- `Modal` yalnızca bir içerik shell'idir; açma ve kapama lifecycle'ı modal host veya parent view tarafından yönetilir.
- Header'daki dismiss ve back button'ları `menu::Cancel` dispatch eder; bu aksiyonun parent context tarafından ele alınması beklenir.
- Section içinde çok sayıda ayar satırı yer alıyorsa, body için bir scroll handle verilmesi gerekir.
- `Modal`, bir `AlertModal` yerine kullanılsa bile yine de workspace üzerinden `toggle_modal` ile sunulur; ayrı bir overlay altyapısı kurmaya gerek yoktur.

## AlertModal

Kaynak:

- Tanım: `../zed/crates/ui/src/components/notification/alert_modal.rs`
- Export: `ui::AlertModal`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for AlertModal`.

Ne zaman kullanılır:

- Kullanıcıdan kısa bir onay veya iptal kararı almak için.
- Güvenlik, silme veya workspace trust gibi devam etmeden önce anlaşılması gereken uyarılar için.
- Özel bir header veya footer gerekse de modal iskeletini hızlıca kurmak için.

Ne zaman kullanılmaz:

- Non-blocking bir bilgi mesajı için `Banner` veya `Callout` çok daha doğrudan bir araçtır.
- Çok bölümlü bir ayar formu için `Modal` daha uygundur.
- Yeni özellik duyurusu için `AnnouncementToast` daha doğru bir yüzeydir.

Temel API:

- `AlertModal::new(id)`.
- `.title(text)`.
- `.header(element)`.
- `.footer(element)`.
- `.primary_action(label)`.
- `.dismiss_label(label)`.
- `.width(width)`.
- `.key_context(context)`.
- `.on_action::<A>(listener)`.
- `.track_focus(&focus_handle)`.
- ParentElement: `.child(...)`, `.children(...)`.

Davranış:

- Varsayılan genişlik `px(440.)`'tır.
- `.title(...)` verildiğinde küçük bir `Headline` içeren bir default header üretilir.
- `.primary_action(...)` veya `.dismiss_label(...)` verildiğinde bir default footer üretilir. Label verilmediği durumda primary `"Ok"`, dismiss ise `"Cancel"` olur.
- Default footer button'ları yalnızca görünümü kurar; karar akışının Zed action sistemi üzerinden `.on_action(...)` veya parent lifecycle ile bağlanması gerekir.
- `.header(...)` ve `.footer(...)` verildiğinde, default header veya footer yerine tamamen özel bir element render edilir.

Örnek:

```rust
use ui::{AlertModal, prelude::*};

fn render_delete_alert() -> impl IntoElement {
    AlertModal::new("delete-project-alert")
        .title("Delete project?")
        .child("This removes the project from the recent projects list.")
        .primary_action("Delete")
        .dismiss_label("Cancel")
}
```

Özel bir header gerektiğinde, hem `width(...)` hem de `header(...)` ile modal'a kendi görsel kimliği verilebilir:

```rust
use ui::{AlertModal, Icon, IconName, prelude::*};

fn render_restricted_workspace_alert(cx: &App) -> impl IntoElement {
    AlertModal::new("restricted-workspace-alert")
        .width(rems(40.))
        .header(
            v_flex()
                .p_3()
                .gap_1()
                .bg(cx.theme().colors().editor_background.opacity(0.5))
                .border_b_1()
                .border_color(cx.theme().colors().border_variant)
                .child(
                    h_flex()
                        .gap_2()
                        .child(Icon::new(IconName::Warning).color(Color::Warning))
                        .child(Label::new("Unrecognized Workspace")),
                ),
        )
        .child("Restricted mode prevents workspace commands from running automatically.")
        .primary_action("Trust Workspace")
        .dismiss_label("Stay Restricted")
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/workspace/src/security_modal.rs`: restricted workspace karar akışı; `key_context`, `track_focus` ve `.on_action(...)` birlikte kullanılır.
- `../zed/crates/ui/src/components/notification/alert_modal.rs`: basic ve custom header preview örnekleri.

Dikkat edilecek noktalar:

- AlertModal kısa ve karar odaklı tutulur. Birden fazla section gerekiyorsa doğru araç `Modal`'dır.
- Tehlikeli aksiyonlarda primary label'in net olması beklenir; `"Ok"` yerine `"Delete"` veya `"Trust Workspace"` gibi doğrudan eylemi anlatan bir metin tercih edilir.
- Focus ve keyboard action davranışı gerekiyorsa `key_context(...)` ile `track_focus(...)` bağlanmadan yalnızca görsel bir modal üretilmesi doğru değildir; aksi halde modal klavye ile etkileşemez.

## AnnouncementToast

Kaynak:

- Tanım: `../zed/crates/ui/src/components/notification/announcement_toast.rs`
- Export: `ui::AnnouncementToast`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for AnnouncementToast`.

Ne zaman kullanılır:

- Yeni özellik, önemli değişiklik veya üründeki görünür duyuruları kart biçiminde göstermek için.
- İllüstrasyon, başlık, açıklama, bullet listesi ve iki aksiyonlu duyuru gerektiğinde.

Ne zaman kullanılmaz:

- Hata, retry veya inline bir durum mesajı için `Banner` ya da `Callout` daha uygundur.
- Basit bir toast ihtiyacı için parent notification sisteminin daha küçük view'ı kullanılır.
- Kullanıcının devam etmeden karar vermesi gerekiyorsa `AlertModal` daha doğru bir yüzeydir.

Temel API:

- `AnnouncementToast::new()`.
- `.illustration(element)`.
- `.heading(text)`.
- `.description(text)`.
- `.bullet_item(element)`.
- `.bullet_items(items)`.
- `.primary_action_label(text)`.
- `.primary_on_click(handler)`.
- `.secondary_action_label(text)`.
- `.secondary_on_click(handler)`.
- `.dismiss_on_click(handler)`.

Davranış:

- Varsayılan primary label `"Try Now"`, secondary label ise `"Learn More"` olarak gelir.
- Click handler'ları boş bir default closure ile gelir; gerçek davranışın bağlanması parent view'in dismiss veya navigation callback'leri üzerinden olur.
- Root element `occlude()`, `relative()`, `w_full()` ve `elevation_3(cx)` kullanır.
- Sağ üstte bir close icon button render edilir; dismiss lifecycle'ı `.dismiss_on_click(...)` callback'ine bırakılır.

Örnek:

```rust
use ui::{AnnouncementToast, ListBulletItem, prelude::*};

fn render_feature_announcement() -> impl IntoElement {
    div().w_80().child(
        AnnouncementToast::new()
            .heading("Parallel agents")
            .description("Run multiple agent threads across projects.")
            .bullet_item(ListBulletItem::new("Launch agents in isolated worktrees"))
            .bullet_item(ListBulletItem::new("Review progress without changing tabs"))
            .primary_action_label("Try Now")
            .primary_on_click(|_, _window, cx| cx.open_url("https://zed.dev"))
            .secondary_action_label("Learn More")
            .secondary_on_click(|_, _window, cx| cx.open_url("https://zed.dev/docs"))
            .dismiss_on_click(|_, _window, _cx| {}),
    )
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/auto_update_ui/src/auto_update_ui.rs`: announcement toast notification view'ı; click handler'lar telemetry, URL ve dismiss callback'leriyle bağlanır.

Dikkat edilecek noktalar:

- `AnnouncementToast` tek başına notification lifecycle'ını yönetmez. Dismiss, suppress veya route davranışının parent notification view'ı içinde uygulanması gerekir.
- Bullet sayısı sınırlı tutulur; çok uzun bir duyuru kartı kullanıcıyı akıştan koparır.
- İllüstrasyon eklendiğinde toast'ın üstünde render edilir ve body'den border ile ayrılır.

## Notification Modülü

Kaynak:

- Modül: `../zed/crates/ui/src/components/notification.rs`
- Export: `ui::AlertModal`, `ui::AnnouncementToast`.
- Prelude: Hayır.

Mevcut `ui` kaynağında standalone bir `Notification` component'i yer almaz. `notification.rs` dosyası yalnızca `alert_modal` ve `announcement_toast` modüllerini re-export eder. Runtime bildirim kuyruğu, dismiss veya suppress event'leri ve notification trait'leri Zed'in daha üst seviyeli notification altyapısında tutulur.

Pratik sonuç şudur:

- UI component olarak `AlertModal` veya `AnnouncementToast` render edilir.
- Gösterme, saklama, kapatma ve tekrar göstermeme kararı parent notification view'ı içinde yönetilir.
- Toast içindeki click handler'larda gerekirse telemetry, URL açma ve dismiss akışı birlikte bağlanır.

## CountBadge

Kaynak:

- Tanım: `../zed/crates/ui/src/components/count_badge.rs`
- Export: `ui::CountBadge`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for CountBadge`.

Ne zaman kullanılır:

- İkon, tab veya kompakt bir toolbar item üzerinde küçük bir sayaç göstermek için.
- Bildirim, hata, değişiklik veya bekleyen öğe sayısını küçük bir alanda belirtmek için.

Ne zaman kullanılmaz:

- Sayısal değer ana içeriğin kendisiyse, bir `Label` veya tablo hücresi kullanılır.
- Durum sadece var/yok şeklindeyse `Indicator::dot()` daha sade bir ifadedir.

Temel API:

- `CountBadge::new(count)`.

Davranış:

- `count > 99` durumunda `"99+"` olarak gösterilir.
- `absolute()`, `top_0()` ve `right_0()` ile parent'ın sağ üst köşesine yerleşir.
- Parent element `relative()` değilse, badge beklenen konuma oturmaz.
- Background, editor background ile error status renginin blend edilmesiyle hesaplanır.

Örnek:

```rust
use ui::{CountBadge, IconButton, IconName, prelude::*};

fn render_notifications_button(count: usize) -> impl IntoElement {
    div()
        .relative()
        .child(IconButton::new("notifications", IconName::Bell))
        .when(count > 0, |this| this.child(CountBadge::new(count)))
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/workspace/src/dock.rs`: dock item üzerinde count badge.
- `../zed/crates/ui/src/components/count_badge.rs`: capped count preview.

Dikkat edilecek noktalar:

- Parent'ın hitbox'ı ile badge'in absolute konumunun birlikte düşünülmesi gerekir. Çok küçük icon button'larda badge, tıklanabilir alanı görsel olarak kalabalıklaştırabilir.
- Badge metni otomatik olarak capped olduğu için, gerçek tam sayının tooltip veya bir detay view'ında gösterilmesi gerekebilir.

## Indicator

Kaynak:

- Tanım: `../zed/crates/ui/src/components/indicator.rs`
- Export: `ui::Indicator`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for Indicator`.

Ne zaman kullanılır:

- Küçük bir durum noktası, üst bar veya icon tabanlı bir durum göstergesi gerektiğinde.
- Liste satırında bağlantı, breakpoint, conflict, active/inactive gibi hızlı taranabilir durumlar için.
- Bir icon button veya list item yanında, dikkat çekmeyen bir status işareti için.

Ne zaman kullanılmaz:

- İşlem ilerlemesi için `ProgressBar` ya da `CircularProgress` kullanılır.
- Metinsel bir açıklama gerekiyorsa yanına bir `Label` eklenir; indicator tek başına erişilebilir bir anlam taşımaz.

Temel API:

- `Indicator::dot()`.
- `Indicator::bar()`.
- `Indicator::icon(icon)`.
- `.color(Color)`.
- `.border_color(Color)`.

Davranış:

- Dot varyantı `w_1p5()`, `h_1p5()` ve `rounded_full()` kullanır.
- Bar varyantı `w_full()`, `h_1p5()` ve `rounded_t_sm()` kullanır; bu yüzden parent genişliği önemlidir.
- Icon indicator, ikonu `custom_size(rems_from_px(8.))` ile çok küçük bir biçimde render eder.
- `border_color(...)` yalnızca dot ve bar varyantları için bir border uygular.

Örnek:

```rust
use ui::{Indicator, prelude::*};

fn render_connection_state(connected: bool) -> impl IntoElement {
    h_flex()
        .gap_1()
        .items_center()
        .child(
            Indicator::dot().color(if connected {
                Color::Success
            } else {
                Color::Error
            }),
        )
        .child(Label::new(if connected { "Connected" } else { "Disconnected" }))
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/workspace/src/status_bar.rs`: status bar indicator.
- `../zed/crates/debugger_ui/src/dropdown_menus.rs`: debug session state.
- `../zed/crates/keymap_editor/src/keymap_editor.rs`: conflict indicator.
- `../zed/crates/title_bar/src/title_bar.rs`: title bar durum noktaları.

Dikkat edilecek noktalar:

- Rengin `Color::Success`, `Warning`, `Error`, `Info` veya `Muted` gibi semantik token'lardan seçilmesi tutarlılığı korur.
- Indicator tek bilgi kaynağı olarak kullanılmaz; özellikle error ve warning durumlarında tooltip veya label ile anlamın belirtilmesi gerekir.

## ProgressBar

Kaynak:

- Tanım: `../zed/crates/ui/src/components/progress/progress_bar.rs`
- Export: `ui::ProgressBar`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for ProgressBar`.

Ne zaman kullanılır:

- İşlemin belirli bir `value / max_value` oranı varsa.
- Yatay alanda dosya indirme, kullanım limiti, sync veya task progress göstermek için.

Ne zaman kullanılmaz:

- İlerleme oranı bilinmiyorsa `LoadingLabel` veya `SpinnerLabel` çok daha uygundur.
- Çok dar bir inline alanda ring görünümü daha doğal duracaksa `CircularProgress` tercih edilir.

Temel API:

- `ProgressBar::new(id, value, max_value, cx)`.
- `.value(value)`.
- `.max_value(max_value)`.
- `.bg_color(hsla)`.
- `.fg_color(hsla)`.
- `.over_color(hsla)`.

Davranış:

- Fill genişliği `(value / max_value).clamp(0.02, 1.0)` formülüyle hesaplanır.
- `value > max_value` durumunda fill rengi `over_color` olur.
- Varsayılan foreground renk `cx.theme().status().info`'dur.
- `max_value` pozitif bir değer olmalıdır; sıfır veya anlamsız bir max değer üretilmemelidir.

Örnek:

```rust
use ui::{ProgressBar, prelude::*};

fn render_usage_progress(used: f32, limit: f32, cx: &App) -> impl IntoElement {
    v_flex()
        .gap_1()
        .child(
            h_flex()
                .justify_between()
                .child(Label::new("Usage"))
                .child(Label::new(format!("{used:.0} / {limit:.0}")).color(Color::Muted)),
        )
        .child(ProgressBar::new("usage-progress", used, limit, cx))
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/edit_prediction_ui/src/edit_prediction_button.rs`: kullanım limiti progress bar'ı.
- `../zed/crates/ui/src/components/progress/progress_bar.rs`: empty, partial ve filled preview örnekleri.

Dikkat edilecek noktalar:

- `value` ve `max_value` aynı birimde olmalıdır.
- Progress bar'a sadece renk yüklenmemeli; yanında bir label veya tooltip ile bağlam verilmesi okunabilirliği artırır.
- `value > max_value` bilinçli bir over-limit durumudur. Normal "işlem tamamlandı" durumu için `value == max_value` kullanılır.

## CircularProgress

Kaynak:

- Tanım: `../zed/crates/ui/src/components/progress/circular_progress.rs`
- Export: `ui::CircularProgress`.
- Prelude: Hayır; ayrıca import edilir.
- Preview: `impl Component for CircularProgress`.

Ne zaman kullanılır:

- Dar veya inline bir alanda belirli bir ilerleme oranını ring olarak göstermek için.
- Token kullanımı, kompakt quota veya küçük status cluster'larında.

Ne zaman kullanılmaz:

- Geniş yatay bir alanda metinle birlikte ilerleme göstermek için `ProgressBar` daha okunaklı bir tercihtir.
- İlerleme oranı bilinmiyorsa spinner veya loading bileşeni kullanılır.

Temel API:

- `CircularProgress::new(value, max_value, size, cx)`.
- `.value(value)`.
- `.max_value(max_value)`.
- `.size(px)`.
- `.stroke_width(px)`.
- `.bg_color(hsla)`.
- `.progress_color(hsla)`.

Davranış:

- Canvas üzerinde bir background circle ve bir progress arc çizer.
- Progress üstten başlar ve saat yönünde ilerler.
- Progress oranı `(value / max_value).clamp(0.0, 1.0)` ile hesaplanır.
- `progress >= 0.999` olduğunda tam bir çember çizilir.
- Varsayılan stroke width `px(4.)`'tür.

Örnek:

```rust
use ui::{CircularProgress, prelude::*};

fn render_token_ring(used: f32, max: f32, cx: &App) -> impl IntoElement {
    h_flex()
        .gap_1()
        .items_center()
        .child(
            CircularProgress::new(used, max, px(18.), cx)
                .stroke_width(px(2.))
                .progress_color(cx.theme().status().info),
        )
        .child(Label::new(format!("{used:.0}/{max:.0}")).size(LabelSize::Small))
}
```

Zed içinden kullanım örnekleri:

- `../zed/crates/agent_ui/src/conversation_view/thread_view.rs`: token usage ring'leri.
- `../zed/crates/ui/src/components/progress/circular_progress.rs`: farklı yüzde değerleri için preview örnekleri.

Dikkat edilecek noktalar:

- Ring küçük olduğunda, bir label veya tooltip olmadan oranı okumak zorlaşır.
- `max_value` pozitif bir değer olmalıdır.
- Aynı ekranda çok sayıda animasyonlu veya sık güncellenen canvas progress kullanılıyorsa, repaint maliyeti hesaba katılır.

## ActivityIndicator

Kaynak:

- Tanım: `../zed/crates/activity_indicator/src/activity_indicator.rs`
- Export: `activity_indicator::ActivityIndicator`; `ui` crate kökünden re-export edilen genel bir component değildir.
- Render modeli: `workspace::StatusItemView` olarak workspace status alanına bağlanır.

Ne zaman kullanılır:

- Workspace genelinde devam eden LSP, debug, git, dosya sistemi, extension update veya formatlama hatası gibi aktiviteleri tek bir status trigger'ında göstermek için.
- Bir aktivite kullanıcı aksiyonu gerektiriyorsa aynı trigger üzerinden click handler veya popover menu vermek için.

Davranış:

- İçerik artık `ActivityIcon` ayrımıyla seçilir: bilinmeyen süreli işler `LoadingSpinner`, statik durumlar ise `Icon(IconName)` taşır.
- Spinner görünümü doğrudan `Button::loading(true)` üzerinden gelir; bu yüzden loading durumunda start icon yerine `IconName::LoadCircle` çizilir.
- Warning, download ve benzeri statik durumlarda trigger `Button::start_icon(...)` kullanır ve ikon `Color::Muted` ile çizilir.
- Language server work listesi trigger'a yalnızca içerikte özel bir click handler yoksa popover olarak bağlanır. Menü en az bir cancellable work varsa açılır; cancellable entry'ler `Close` ikonu ve `Cancel ...` label'ıyla render edilir.
- Environment error, formatting failure veya extension failure gibi durumlar kendi click handler'larını taşıyorsa language server work menüsü aynı trigger'a eklenmez.
- Extension install ve remove durumları loading spinner kullanır; extension upgrade ve download durumları download ikonu kullanır.

Dikkat edilecek noktalar:

- `ActivityIndicator`, genel amaçlı bir progress component'i değildir. Bir panel içinde belirli bir iş oranı göstermek için `ProgressBar`, kompakt oran için `CircularProgress`, bilinmeyen süreli yerel loading için `Button::loading(...)` veya `SpinnerLabel` tercih edilir.
- Popover menüsünde yalnızca iptal edilebilir işler aksiyon üretir. İptal edilemeyen işler tek başına menüyü açtırmaz; bu yüzden kullanıcıya tıklanabilir bir affordance gerekiyorsa ilgili durumun `on_click` callback'i açıkça bağlanmalıdır.

## Feedback Kompozisyon Örnekleri

Bir sync sürecinde hem genel bir banner hem de yüzdeyi gösteren bir progress bar bir arada görünebilir:

```rust
use ui::{Banner, ProgressBar, Severity, prelude::*};

fn render_sync_feedback(progress: f32, cx: &App) -> impl IntoElement {
    v_flex()
        .gap_2()
        .child(
            Banner::new()
                .severity(Severity::Info)
                .child(Label::new("Sync in progress"))
                .child(Label::new("Remote changes are being applied.").color(Color::Muted)),
        )
        .child(ProgressBar::new("sync-progress", progress, 1.0, cx))
}
```

Bir toolbar üzerinde hem bir count badge hem de küçük bir indicator birlikte yer alabilir. Aşağıdaki örnek hem sayaç hem de hata durumu için yan yana bir kullanım gösterir:

```rust
use ui::{CountBadge, IconButton, IconName, Indicator, prelude::*};

fn render_review_toolbar_item(issue_count: usize, has_errors: bool) -> impl IntoElement {
    h_flex()
        .gap_2()
        .items_center()
        .child(
            div()
                .relative()
                .child(IconButton::new("review", IconName::Check))
                .when(issue_count > 0, |this| this.child(CountBadge::new(issue_count))),
        )
        .child(
            Indicator::dot().color(if has_errors {
                Color::Error
            } else {
                Color::Success
            }),
        )
}
```

Bütün bu bileşenlerin kullanım kararı için kısa bir özet işe yarar:

- Kısa, non-blocking ve sayfa veya panel üstü bir mesaj için `Banner`.
- İçerik içinde açıklama, aksiyon ve dismiss bir arada gerekiyorsa `Callout`.
- Çok bölümlü bir modal içerik için `Modal`.
- Kısa bir karar veya uyarı diyalogu için `AlertModal`.
- Yeni özellik duyurusu için `AnnouncementToast` ile notification lifecycle birlikte.
- Bir ikon üzerine sayaç bindirmek için `CountBadge`.
- Var/yok veya nokta düzeyinde bir state için `Indicator`.
- Belirli bir yatay ilerleme için `ProgressBar`.
- Belirli bir kompakt ilerleme için `CircularProgress`.
