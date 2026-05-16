# Action ve Keymap

---

## Action Sistemi Derinlemesine

`crates/gpui/src/action.rs`, `key_dispatch.rs`.

Action tanımı iki ana yolla yapılır. Seçim, action'ın veri taşıyıp taşımamasına göre yapılır.

**Veri taşımayan action.** Yalnız adı olan action'lar için makro tek satırda iş görür:

```rust
use gpui::actions;
actions!(my_namespace, [Save, Close, Reload]);
```

`actions!` makrosu her isim için bir unit struct ve `Action` implementasyonu üretir; namespace `my_namespace::Save` adıyla registry'ye kaydedilir.

**Veri taşıyan action.** Yanında veri götürmesi gereken action'lar için derive ve attribute kullanılır:

```rust
use gpui::Action;

#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema, Action)]
#[action(namespace = editor)]
pub struct GoToLine { pub line: u32 }
```

`#[action(namespace = ..., name = "...", no_json, no_register)]` attribute'leri bu derive üzerinde davranışı yönlendirir. Varsayılan olarak `Deserialize` derive'ı ve `JsonSchema` implementasyonu beklenir; tamamen kod içinde kullanılacak bir action için `no_json`, registry'ye kayıt istenmeyen durumda `no_register` seçilir.

**Dispatch.** Bir action'ı tetiklemenin başlıca yolları şunlardır:

- `window.dispatch_action(action.boxed_clone(), cx)` — focused element'ten root'a doğru bubble.
- `focus_handle.dispatch_action(&action, window, cx)` — belirli bir handle'dan başlatır.
- Keymap girdileri eşleştiğinde otomatik dispatch tetiklenir.

**Listener kaydı.** Action'ı dinleyen handler element üzerinde tanımlanır:

```rust
.on_action(cx.listener(|this, action: &GoToLine, window, cx| {
    this.go_to(action.line);
    cx.notify();
}))
.capture_action(cx.listener(handler)) // capture phase
```

**`DispatchPhase`.** Olaylar element ağacında iki ayrı fazda akar:

- `Capture` — root'tan focused element'e doğru.
- `Bubble` — focused element'ten root'a doğru. Varsayılan fazdır. Action handler'lar burada varsayılan olarak propagation'ı durdurur; aksi gerekirse handler içinde `cx.propagate()` çağrılır.

**Keybinding.** Tuş bağlama tanımı `bind_keys` çağrısıyla yapılır:

```rust
cx.bind_keys([
    KeyBinding::new("cmd-s", Save, Some("Workspace")),
    KeyBinding::new("ctrl-g", GoToLine { line: 0 }, Some("Editor")),
]);
```

**Context predicate grameri.** Bağlam ifadeleri keymap'in eşleşme mantığını kurar (`crates/gpui/src/keymap/context.rs:172,360+`):

- `Editor` — context stack'te `Editor` identifier'ı bulunuyor.
- `Editor && !ReadOnly` — birleştirme ve negasyon.
- `Workspace > Editor` — `>` operatörü "Editor parent dispatch path'inde Workspace altında" anlamına gelen descendant predicate'idir.
- `mode == insert` — eşitlik (`KeyContext::set("mode", "insert")` ile yazılan key/value değerine bakar).
- `mode != normal` — eşitsizlik.
- `(Editor || Terminal) && !ReadOnly` — parantezle gruplama.

Gerçek parser yalnızca şu operatörleri tanır: `>`, `&&`, `||`, `==`, `!=`, `!`. `in (a, b)`, `not in` veya fonksiyon çağrısı gibi sözdizimler yoktur. Vim modu gibi çoklu seçenek `mode == normal || mode == visual` biçiminde ifade edilir.

`.key_context("Editor")` çağrısı element ağacına context ekler; child'lar üst context'leri görür. Aynı binding birden fazla context'te eşleşirse en spesifik (en derin) olan kazanır.

**Tuzaklar.** Action tanımlama tarafında sık karşılaşılan hatalar:

- Action register edilmeden binding tanımlandığında keymap parse'da hata oluşur; `actions!` veya `#[derive(Action)]` mutlaka ana modülde derlenmiş olmalıdır.
- Bubble fazında handler `cx.propagate()` çağırmadığı sürece parent action handler'lara ulaşılmaz (varsayılan davranış).
- Aynı action ismi iki crate'te tanımlanırsa registry çakışması olur; namespace bu yüzden zorunludur.
- Zed çalışma zamanında bilinmeyen action ismi keymap'te warning log'u üretir, panic vermez.

## Action Makro Detayları ve register_action!

`#[derive(Action)]` ve `actions!` makrosu çoğu durumda yeterlidir; ancak action sözleşmesinin ek köşe taşları da vardır.

#### Yeni action yazarken kullanılan çekirdek yüzey (`crates/gpui/src/action.rs:117+`)

```rust
pub trait Action: Any + Send {
    fn boxed_clone(&self) -> Box<dyn Action>;
    fn partial_eq(&self, other: &dyn Action) -> bool;
    fn name(&self) -> &'static str;
    fn name_for_type() -> &'static str where Self: Sized;
    fn build(value: serde_json::Value) -> Result<Box<dyn Action>>
        where Self: Sized;
    fn action_json_schema(_: &mut SchemaGenerator) -> Option<Schema>
        where Self: Sized { None }
    fn documentation() -> Option<&'static str>
        where Self: Sized { None }
}
```

`name(&self)` çalışma zamanı adını verir; `name_for_type()` static adı verir. Runtime polymorphism söz konusu olduğunda ilki, registration sırasında ikincisi kullanılır.

#### `#[action(...)]` attribute'leri

`#[derive(Action)]` üzerinde tanımlanabilen attribute'ler şunlardır:

- `namespace = my_crate` — action adını `my_crate::Save` formuna çevirir.
- `name = "OpenFile"` — namespace içinde özel ad.
- `no_json` — `Deserialize`/`JsonSchema` derive zorunluluğunu kaldırır; `build()` her zaman hata döner, `action_json_schema()` `None` verir. Tamamen kod içi kullanılacak action'lar (örneğin `RangeAction { start: usize }`) için tercih edilir.
- `no_register` — inventory üzerinden otomatik kaydı atlar; trait'i elle uygularken veya koşullu kayıt yaparken gerekir.

#### `register_action!` makrosu

`#[derive(Action)]` kullanılmadan `Action`'u manuel implement edildiğinde, action'ın inventory'e dahil olabilmesi için ayrı bir kayıt makrosu vardır:

```rust
use gpui::register_action;

register_action!(Paste);
```

Bu makro yalnızca bir `inventory::submit!` çağrısı üretir; struct ya da impl tanımına dokunmaz. `no_register` ile birleştirildiğinde elle kaydın zamanı seçilebilir.

#### Action runtime API'leri

Action'ları çalışma zamanında sorgulamak ve tetiklemek için bazı yardımcılar sağlanır:

- `cx.is_action_available(&action) -> bool` — focused element path'inde bu action'ı dinleyen biri var mı? Menü item'larını pasifleştirmek için idealdir.
- `window.is_action_available(&action, cx)` — pencereye özel sürüm.
- `cx.dispatch_action(&action)` — focused window'a yayınlar.
- `window.dispatch_action(action.boxed_clone(), cx)` — pencereye özel sürüm.
- `cx.build_action(name, json_value)` — keymap entry'sinden çalışma zamanı action'ı üretir; schema yoksa `ActionBuildError` döner.

#### Tuzaklar

Action makrolarının kullanımındaki ince noktalar:

- `partial_eq` varsayılan olarak `PartialEq` impl'ini kullanır; derive eklenmemişse karşılaştırma yanlış sonuç verebilir.
- Aynı `name()` döndüren iki action register edildiğinde inventory startup'ta panic eder; namespace kullanımı çakışmaları önler.
- `no_json` ile işaretlenmiş bir action keymap dosyasından çağrılamaz; yalnızca kod içinden `dispatch_action` ile tetiklenir.

## Keymap, KeyContext ve Dispatch Stack

Action tanımı tek başına yeterli değildir. Keybinding'in çalışabilmesi için focused element'in dispatch path'inde uygun `KeyContext` bulunmalıdır.

**Context koyma.** Element bağlamı `key_context` ile bildirilir:

```rust
div()
    .track_focus(&self.focus_handle)
    .key_context("Editor mode=insert")
    .on_action(cx.listener(|this, _: &Save, window, cx| {
        this.save(window, cx);
    }))
```

**Binding ekleme.** Binding'ler `bind_keys` ile kaydedilir:

```rust
cx.bind_keys([
    KeyBinding::new("cmd-s", Save, Some("Editor")),
    KeyBinding::new("ctrl-g", GoToLine { line: 0 }, Some("Workspace && !Editor")),
]);
```

**Önemli parçalar.** Bu sistemin temel taşları şunlardır:

- `KeyContext::parse("Editor mode = insert")` — elementin bağlamını üretir.
- `KeyContext::new_with_defaults()` — varsayılan context set'iyle başlar. `primary()` ve `secondary()` parse edilen ana ve ek context girişlerine erişir. `is_empty()`, `clear()`, `extend(&other)`, `add(identifier)`, `set(key, value)`, `contains(key)` ve `get(key)` düşük seviyeli context inşa ve sorgu yüzeyidir.
- `KeyBindingContextPredicate` — binding tarafındaki predicate dilidir: `Editor`, `mode == insert`, `!Terminal`, `Workspace > Editor`, `A && B`, `A || B`.
- `KeyBindingContextPredicate::parse(source)` — predicate üretir. `eval(context_stack)` bool eşleşme, `depth_of(context_stack)` en derin eşleşme derinliği, `is_superset(&other)` ise keymap önceliği ve çakışma analizinde kullanılır. `eval_inner(...)` public görünür ama parser/validator gibi düşük seviyeli kodlar içindir; normal component kodu doğrudan çağırmaz.
- `Keymap::bindings_for_input(input, context_stack)` — eşleşen action'ları ve pending multi-stroke durumunu döndürür.
- `Keymap::possible_next_bindings_for_input(input, context_stack)` — mevcut chord prefix'ini takip edebilecek binding'leri öncelik sırasında verir.
- `Keymap::version() -> KeymapVersion` — binding seti değiştikçe artan sayaçtır; keybinding UI cache'lerinde invalidation anahtarı olarak kullanılabilir.
- `Keymap::new(bindings)`, `add_bindings(bindings)`, `bindings()`, `bindings_for_action(action)`, `all_bindings_for_input(input)` ve `clear()` ham keymap tablosunu kurma, sorgulama ve reset yüzeyidir. Uygulama akışında çoğunlukla `cx.bind_keys(...)` ve settings loader tercih edilir; bu metotlar test, validator, diagnostic ve özel keymap UI için kullanılır.
- `window.context_stack()` — focused node'dan root'a dispatch path'teki context'leri verir.
- `window.keystroke_text_for(&action)` — UI'da gösterilecek en yüksek öncelikli binding metni.
- `window.possible_bindings_for_input(&[keystroke])` — chord veya pending yardım UI'ları için kullanılır.
- `cx.key_bindings() -> Rc<RefCell<Keymap>>` — keymap'e düşük seviyeli erişim. Üretim kodunda mümkün olduğu kadar `bind_keys`, keymap dosyası ve validator akışı tercih edilir; bu handle test, diagnostic ve özel keymap UI'ı için uygundur.
- `cx.clear_key_bindings()` — tüm binding'leri temizler ve pencereleri refresh için planlar; normal uygulama akışında değil test ya da reset yollarında kullanılır.

**Öncelik.** Aynı tuşa birden çok binding düştüğünde sıralama şu kurallarla çözülür:

- Context path'te daha derin eşleşme daha yüksek önceliklidir.
- Aynı derinlikte sonra eklenen binding önce gelir; kullanıcı keymap'i built-in binding'leri bu yüzden ezebilir.
- `NoAction` ve `Unbind` binding'leri devre dışı bırakma için kullanılır.
- Printable input IME'ye gidecekse `InputHandler::prefers_ime_for_printable_keys` keybinding yakalamasının önceliğini düşürebilir. `EntityInputHandler` kullanan view'larda bu değer `ElementInputHandler` tarafından `accepts_text_input` sonucundan türetilir; ayrı bir karar isteniyorsa raw `InputHandler` yazılır.

**Tuzaklar.** Bu sistemde sık karşılaşılan hatalar:

- `.key_context(...)` bulunmayan bir subtree'de context predicate'li binding çalışmaz.
- Handler focus path'inde değilse action bubble oraya ulaşmaz; global handler için `cx.on_action(...)`, lokal handler için element üzerinde `.on_action(...)` kullanılır.
- `KeyBinding::new` parse hatasında panic verebilir; kullanıcı JSON'undan yükleme yapıldığında `KeyBinding::load` ve hata raporlama tercih edilir.
- `KeyBinding::load(keystrokes, action, context_predicate, use_key_equivalents, action_input, keyboard_mapper)` fallible bir loader'dır; `KeyBindingKeystroke` mapping'ini de burada kurar. `context_predicate` `Option<Rc<KeyBindingContextPredicate>>`, `action_input` ise `Option<SharedString>` alır ve hata durumunda `InvalidKeystrokeError` döner. Çalışma zamanında `with_meta(...)` ve `set_meta(...)` binding'in hangi keymap katmanından geldiğini taşır; `match_keystrokes(...)` tam veya pending eşleşmeyi verirken `keystrokes()`, `action()`, `predicate()`, `meta()` ve `action_input()` getter'ları diagnostic ve command/keymap UI'larını besler.

## DispatchPhase, Event Propagation ve DispatchEventResult

Mouse, key ve action olayları element ağacında iki faz içinde akar:

```rust
pub enum DispatchPhase {
    Capture, // root → focused
    Bubble,  // focused → root (default)
}
```

`Window::on_mouse_event`, `on_key_event` ve `on_modifiers_changed` listener'ları faza göre çağrılır. Element fluent API'lerinde `.on_*` ailesi bubble fazına, `.capture_*` ailesi ise capture fazına bağlanır.

**Kontrol bayrakları** (`crates/gpui/src/app.rs:2021+`):

- `cx.stop_propagation()` — aynı tipteki diğer handler'ların çağrılmasını keser (mouse'ta z-index'te alt katman, key'de ağaçta üst element).
- `cx.propagate()` — önceki `stop_propagation()` etkisini geri alır. Action handler'lar bubble fazında varsayılan olarak propagation'ı durdurduğu için parent'a düşürmek isteniyorsa handler içinden `cx.propagate()` çağrılır.
- `window.prevent_default()` ve `window.default_prevented()` — aynı dispatch içinde varsayılan element davranışını bastıran pencere bayrağıdır. En görünür kullanım örneği mouse down sırasında parent focus transferinin engellenmesidir.

**Platforma dönen sonuç.** Dispatch tamamlandığında platforma şu yapı döner:

```rust
pub struct DispatchEventResult {
    pub propagate: bool,        // hâlâ bubble ediliyorsa true
    pub default_prevented: bool, // GPUI default davranışı bastırıldı mı
}
```

`PlatformWindow::on_input` callback'i `Fn(PlatformInput) -> DispatchEventResult` döndürür. Mevcut platform backend'lerinde "event işlendi mi?" kararı esas olarak `propagate` üzerinden verilir (`!propagate` "handled" anlamına gelir). `default_prevented` GPUI dispatch ağacındaki varsayılan element davranışını ve test/diagnostic sonucunu taşır. Bunu genel bir platform iptal mekanizması gibi değil, handler'ın açıkça kontrol ettiği yerlerde anlamlandırmak gerekir.

**Pratik akış.** Tipik bir dispatch turunda şu adımlar izlenir:

1. Element listener tetiklenir, view state güncellenir, gerekirse `cx.notify()` çağrılır.
2. Listener event'i tüketmek isterse `cx.stop_propagation()` çağırır.
3. Action handler default davranışı korumak istiyorsa `cx.propagate()` ile bubble'ı yeniden açar.
4. Default focus transferi gibi GPUI içi bir davranış bastırılacaksa `window.prevent_default()` çağrılır.

**Tuzaklar.** Bu akışta dikkat edilmesi gerekenler:

- `capture_*` handler'ları focus path'i bilinmeden çalışır; pencere global shortcut veya observer için kullanılır, ancak state mutate edilmek isteniyorsa focused element çalışma zamanında kontrol edilmelidir.
- Action propagation davranışı mouse veya key event'lerinden ters çalışır. Yeni bir action handler'ında refleksle `stop_propagation` yazmak parent action'larını engelleyebilir.
- `default_prevented` genel bir platform cancellation API'si değildir; hangi davranışın durdurulduğunu anlamak için ilgili element veya pencere handler'ının `window.default_prevented()` kontrol edip etmediğine bakılır.

## Action ve Keymap Runtime Introspection

Action tanımlama ve dispatch önceki bölümlerde işlenmiştir. Zed komut paleti, keymap UI ve geliştirici diagnostikleri için çalışma zamanı introspection yüzeyinin de bilinmesi gerekir.

**Action registry.** Registry'ye kayıtlı action'lar üzerinde aşağıdaki yardımcılar sorgu yapar:

- `cx.build_action(name, data) -> Result<Box<dyn Action>, ActionBuildError>` — string action adı ve isteğe bağlı JSON verisinden çalışma zamanı action'ı üretir.
- `cx.all_action_names() -> &[&'static str]` — register edilmiş tüm action adlarını döndürür. Kayıtlı olmak, action'ın element ağacında o anda kullanılabilir olduğu anlamına gelmez.
- `cx.action_schemas(generator)` — internal olmayan action adlarını ve JSON schema'larını verir.
- `cx.action_schema_by_name(name, generator)` — tek action için schema döndürür; `None` action'ın yokluğunu, `Some(None)` action'ın varlığını ama schema bulunmayışını ifade eder.
- `cx.action_documentation()` — keymap editor, JSON schema ve geliştirici diagnostikleri için action açıklamalarını sağlar.

**Available action ve binding sorguları.** Bağlamdaki action durumunu ve ilgili binding'leri öğrenmek için:

- `window.available_actions(cx)` — focused element dispatch path'indeki action listener'larıyla global action listener'larını birleştirir. Menü ve komut UI'ında "bu action şu anda yapılabilir mi?" sorusunun pencereye özel cevabıdır.
- `window.on_action_when(condition, TypeId::of::<A>(), listener)` — paint fazında current dispatch node'una koşullu, düşük seviyeli action listener ekler. Element API'deki `.on_action(...)` ve `.capture_action(...)` genelde daha okunaklıdır; custom element yazılmıyorsa bu seviyeye inilmez.
- `cx.is_action_available(&action)` ve `window.is_action_available(&action, cx)` — bool kısayollar.
- `window.is_action_available_in(&action, focus_handle)` — action availability sorgusunu belirli bir focus handle dispatch path'inden yapar.
- `window.bindings_for_action(&action)` — focused context stack'e göre action'a giden binding'leri döner; display için son binding en yüksek öncelikli kabul edilir.
- `window.highest_precedence_binding_for_action(&action)` — aynı sorgunun tek sonuçlu, daha ucuz versiyonu.
- `window.bindings_for_action_in(&action, focus_handle)` ve `highest_precedence_binding_for_action_in(...)` — sorguyu belirli focus handle path'inden yapar.
- `window.bindings_for_action_in_context(&action, KeyContext)` — tek bir elle verilmiş context'e göre sorgu.
- `window.highest_precedence_binding_for_action_in_context(&action, KeyContext)` — aynı sorgunun tek sonuçlu en yüksek öncelik versiyonu.
- `cx.all_bindings_for_input(&[Keystroke])` — context'e bakmadan input dizisine kayıtlı tüm binding'leri listeler.
- `window.possible_bindings_for_input(&[Keystroke])` — multi-stroke veya prefix akışında current context stack'e göre sıradaki aday binding'leri verir. Tam eşleşen action dispatch sonucu için normal `window.dispatch_keystroke(...)` akışı kullanılır; public `Window::bindings_for_input` helper'ı yoktur.
- `window.pending_input_keystrokes()` ve `window.has_pending_keystrokes()` — tamamlanmamış key chord durumunu UI'da göstermek veya test etmek için.

**Keystroke global gözlem.** Tüm pencerelerdeki keystroke akışını yakalamak için iki kanca vardır; biri dispatch'ten sonra, diğeri dispatch'ten önce çalışır:

```rust
let after_dispatch = cx.observe_keystrokes(|event, window, cx| {
    log_key(event, window, cx);
});

let before_dispatch = cx.intercept_keystrokes(|event, window, cx| {
    if should_block(event) {
        cx.stop_propagation();
    }
});
```

- `observe_keystrokes` action ve event mekanizmaları çözüldükten sonra çalışır; propagation durdurulmuşsa çağrılmaz.
- `intercept_keystrokes` dispatch'ten önce çalışır; burada `cx.stop_propagation()` çağrısı action dispatch'i engeller.
- Her ikisi de `Subscription` döner; kaybedildiğinde observer düşer.

**Tuzaklar.** Introspection yüzeyini kullanırken atlanan noktalar:

- `all_action_names` içinde görünen bir action'ın o anda kullanılabilir olması garanti değildir; UI enable/disable için `available_actions` veya `is_action_available` tercih edilir.
- Binding gösteriminde context stack'i dikkate almayan `cx.all_bindings_for_input` yerine mümkün olduğunda window veya focus handle bazlı sorgu kullanılır.
- Interceptor'lar global etkilidir; modal'a özgü tuş engelleme yapılacaksa mümkün olduğunca element action veya capture handler ile sınırlı tutulur.

## Zed Keymap Dosyası, Validator ve Unbind Akışı

GPUI action ve keybinding modeli Zed'de `settings::keymap_file` üzerinden kullanıcı dosyasına bağlanır. Bu bölüm, runtime dispatch'ten farklı olarak JSON yükleme, şema ve dosya güncelleme tarafını kapsar.

**Dosya modeli.** Keymap JSON yapısı birkaç ana tip üzerinden okunur:

- `KeymapFile(Vec<KeymapSection>)` — üst seviye JSON array.
- `KeymapSection` — context predicate ile binding ve unbind map'lerini taşır. Alan görünürlüğü dengeli değildir: yalnız `pub context: String` dış erişime açıktır; `use_key_equivalents: bool`, `unbind: Option<IndexMap<...>>`, `bindings: Option<IndexMap<...>>` ve `unrecognized_fields: IndexMap<...>` alanları **private**'tır. `KeymapFile` parser'ı bu alanlara crate içinden doğrudan erişir; dış kod yalnızca `KeymapSection::bindings(&self)` getter'ını kullanabilir ve bu da `(keystroke, KeymapAction)` çiftlerini dönen tek public iterator'dır (`keymap_file.rs:101`). `unbind` ve `use_key_equivalents` için ayrı bir public getter şu sürümde bulunmuyor.
- `KeymapAction(Value)` — `null`, `"action::Name"` veya `["action::Name", { ...args... }]` biçimlerini temsil eder.
- `UnbindTargetAction(Value)` — `unbind` map'indeki hedef action değeri.
- `KeymapFileLoadResult::{Success, SomeFailedToLoad, JsonParseFailure}` — dosyanın kısmen yüklenebildiği senaryoyu açıkça ayırır.

**Yükleme.** İçeriği parse veya load etmenin iki ana yolu vardır:

```rust
let keymap = KeymapFile::parse(&contents)?;
let result = KeymapFile::load(&contents, cx);
```

`load_asset(asset_path, source, cx)` bundled keymap dosyalarını yükler ve `KeybindSource` meta verisini set edebilir. `load_panic_on_failure` yalnızca startup ve test gibi "asset bozuksa devam etmeyelim" yolları içindir.

**Base keymap.** Hangi temel keymap'in seçili olduğu enum'da tutulur:

- `BaseKeymap::{VSCode, JetBrains, SublimeText, Atom, TextMate, Emacs, Cursor, None}`.
- Base, default, vim ve user binding'leri `KeybindSource` meta verisi taşır; UI bu meta veriyle binding'in nereden geldiğini gösterir.

**Built-in keymap davranışı.** Aşağıdaki davranışlar Zed'in varsayılan keymap kurulumunda gözlenir:

- Agent panelindeki ACP thread'e özgü kısayollar `AcpThread` context'inde yaşar. Terminal alt bağlamı descendant predicate ile `AgentPanel > Terminal` kullanır; bu, `>` operatörünün gerçek dispatch path ilişkisi için kullanılmasının pratik örneğidir.
- Git paneli iki tab'lıdır: `git_panel::ActivateChangesTab` ve `git_panel::ActivateHistoryTab` için varsayılan binding macOS'ta `cmd-1` ile `cmd-2`, Linux/Windows'ta `ctrl-1` ile `ctrl-2`'dir.
- Worktree picker `worktree_picker::ForceDeleteWorktree` action'ını destekler. Varsayılan binding macOS'ta `cmd-alt-shift-backspace`, Linux/Windows'ta `ctrl-alt-shift-backspace`'tir; UI tarafında delete ikonunun üzerinde `alt` basılıysa force delete yolu çalışır.
- `buffer_search::UseSelectionForFind` seçimi, seçim yoksa cursor altındaki kelimeyi arama sorgusu olarak kullanır. Ayarın bypass edilmesi gereken çağrılar `SeedQuerySetting::Always` override'ı vermelidir.
- Vim kullanırken Helix tarzında kelimeye atlama `vim::HelixJumpToWord` aksiyonudur. Varsayılan key binding verilmez; kullanıcı normal/visual mod bağlamında örneğin `"g w"` bağlayabilir. Normal modda hedef kelimenin başına taşır, visual modda seçimi hedef kelime başına kadar genişletir. Vim'in varsayılan `g w` rewrap davranışını korumak isteyenler bu binding'i eklememelidir.

**Validator.** Belirli action tiplerine özel doğrulama mantığı eklenebilir:

- `KeyBindingValidator` — belirli action type'ı için binding doğrulaması yapar.
- `KeyBindingValidatorRegistration(pub fn() -> Box<dyn KeyBindingValidator>)` inventory ile toplanır.
- Validator hataları `MarkdownString` döner; keymap UI bunu kullanıcıya okunaklı bir hata olarak gösterebilir.

**Disable / unbind sentinel action'ları.** GPUI iki ayrı sentinel action sağlar (`gpui::action.rs:425-453`). İkisinin çalışma zamanı davranışı da **dispatch etmemek**'tir, ancak keymap dispatch tablosundaki görevleri farklıdır:

- **`zed::NoAction`** — `actions!(zed, [NoAction])` ile tanımlıdır, veri taşımaz. Keymap JSON'unda eylem değeri olarak `null` veya `"zed::NoAction"` yazılır:

  ```json
  { "context": "Editor", "bindings": { "cmd-p": null } }
  ```

  Aynı keystroke'a daha düşük öncelikle bağlanmış action'lar bu context için iptal edilir; eşleşme `Keymap::resolve_binding` tarafından `disabled_binding_matches_context(...)` çağrısıyla **context-aware** filtrelenir (`gpui/src/keymap.rs:120`). Yani `NoAction` belirli bir context predicate'i içinde o tuşu sessize alır.

- **`zed::Unbind(SharedString)`** — `derive(Action)` ile tanımlıdır, payload bir action adıdır. JSON formatı şu şekildedir:

  ```json
  ["zed::Unbind", "editor::NewLine"]
  ```

  Keymap parser'ı bu sentinel'ı gördüğünde aynı keystroke'a aynı action adıyla ileriden gelen tüm binding'leri **action context'inden bağımsız olarak** iptal eder (`keymap.rs:124-128`'deki `is_unbind` kolu). Yani `editor::NewLine` için `enter` tuşunun tüm context'lerde unbind edilmesi gerektiğinde kullanılır.

**Sentinel kontrol API'leri.** Sentinel action'ları çalışma zamanında tespit etmek için iki ufak yardımcı vardır:

- `gpui::is_no_action(&dyn Action) -> bool` (`action.rs:445`) — `as_any().is::<NoAction>()` üzerinden downcast eder. Custom keymap UI veya komut paleti listesinde "(disabled)" göstergesi koymak için uygundur.
- `gpui::is_unbind(&dyn Action) -> bool` (`action.rs:450`) — aynı şekilde `Unbind` instance'ını downcast eder.

`KeymapFile::update_keybinding` `KeybindUpdateOperation::Remove` için bu iki sentinel'ı bağlama göre üretir: user binding'leri `Remove` ile doğrudan dosyadan silinir; framework veya default binding'lerin sessize alınabilmesi için kullanıcı keymap'ine `null` (NoAction) ya da `["zed::Unbind", ...]` girdisi yazılır.

**Dosya güncelleme.** Keymap dosyasını programatik olarak değiştirmek için şu fonksiyon kullanılır:

```rust
let updated = KeymapFile::update_keybinding(
    operation,
    keymap_contents,
    tab_size,
    keyboard_mapper,
)?;
```

- `KeybindUpdateOperation::Add { source, from }` — yeni binding ekler.
- `Replace { source, target, target_keybind_source }` — user binding ise değiştirir; user dışı binding değişiyorsa add + suppression unbind'e dönüştürebilir.
- `Remove { target, target_keybind_source }` — user binding'i dosyadan siler; user dışı binding'i kaldırmak için `unbind` yazar.
- `KeybindUpdateTarget` action adı, isteğe bağlı action argümanları, context ve `KeybindingKeystroke` dizisini taşır.

**Tuzaklar.** Dosya güncellemeyle ilgili dikkat noktaları:

- `use_key_equivalents` yalnızca destekleyen platformlarda anlamlıdır; keyboard mapper sağlanmadan dosya güncelleme doğru keystroke string'ini üretemez.
- User dışı binding'i "silmek" gerçek kaynağı değiştirmez; kullanıcı keymap'ine suppression yapan bir `unbind` girdisi yazılır.
- Kullanıcı JSON'u bozuksa `update_keybinding` dosyayı değiştirmez; önce parse başarıyla geçmelidir.

---
