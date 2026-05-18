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

`actions!` makrosu her isim için bir `unit struct` ve `Action` uygulaması üretir; namespace `my_namespace::Save` adıyla kayıt defterine (`registry`) eklenir.

**Veri taşıyan action.** Yanında veri götürmesi gereken action'lar için derive ve attribute kullanılır:

```rust
use gpui::Action;

#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema, Action)]
#[action(namespace = editor)]
pub struct GoToLine { pub line: u32 }
```

`#[action(namespace = ..., name = "...", no_json, no_register)]` öznitelikleri bu derive üzerinde davranışı yönlendirir. Varsayılan olarak `Deserialize` derive'ı ve `JsonSchema` uygulaması beklenir; tamamen kod içinde kullanılacak bir action için `no_json`, kayda alınması istenmeyen durumda `no_register` seçilir.

**Yönlendirme.** Bir action'ı tetiklemenin başlıca yolları şunlardır:

- `window.dispatch_action(action.boxed_clone(), cx)` — odaktaki elementten root'a doğru yayılır.
- `focus_handle.dispatch_action(&action, window, cx)` — belirli bir handle'dan başlatır.
- Keymap girdileri eşleştiğinde otomatik yönlendirme tetiklenir.

**Dinleyici kaydı.** Action'ı dinleyen kod element üzerinde tanımlanır:

```rust
.on_action(cx.listener(|this, action: &GoToLine, window, cx| {
    this.go_to(action.line);
    cx.notify();
}))
.capture_action(cx.listener(handler)) // capture aşaması
```

**`DispatchPhase`.** Olaylar element ağacında iki ayrı aşamada akar:

- `Capture` — root'tan odaktaki elemente doğru.
- `Bubble` — odaktaki elementten root'a doğru. Varsayılan aşamadır. Action dinleyicileri burada varsayılan olarak yayılımı durdurur; aksi gerekirse dinleyici içinde `cx.propagate()` çağrılır.

**Kısayol bağlama.** Tuş bağlama tanımı `bind_keys` çağrısıyla yapılır:

```rust
cx.bind_keys([
    KeyBinding::new("cmd-s", Save, Some("Workspace")),
    KeyBinding::new("ctrl-g", GoToLine { line: 0 }, Some("Editor")),
]);
```

**Bağlam yüklemi (`predicate`) grameri.** Bağlam ifadeleri keymap'in eşleşme mantığını kurar (`crates/gpui/src/keymap/context.rs:172,360+`):

- `Editor` — bağlam yığınında `Editor` tanımlayıcısı bulunuyor.
- `Editor && !ReadOnly` — birleştirme ve negasyon.
- `Workspace > Editor` — `>` operatörü, "Editor'ün üst yönlendirme yolunda Workspace var" anlamına gelen alt öğe yüklemidir.
- `mode == insert` — eşitlik (`KeyContext::set("mode", "insert")` ile yazılan anahtar/değer çiftine bakar).
- `mode != normal` — eşitsizlik.
- `(Editor || Terminal) && !ReadOnly` — parantezle gruplama.

Gerçek ayrıştırıcı yalnızca şu operatörleri tanır: `>`, `&&`, `||`, `==`, `!=`, `!`. `in (a, b)`, `not in` veya fonksiyon çağrısı gibi söz dizimleri yoktur. Vim modu gibi çoklu seçenek `mode == normal || mode == visual` biçiminde ifade edilir.

`.key_context("Editor")` çağrısı element ağacına bağlam ekler; alt öğeler üst bağlamları görür. Aynı kısayol birden fazla bağlamda eşleşirse en özgül (en derin) olan kazanır.

**Tuzaklar.** Action tanımlama tarafında sık karşılaşılan hatalar:

- Action kayda alınmadan kısayol tanımlandığında keymap ayrıştırmasında hata oluşur; `actions!` veya `#[derive(Action)]` mutlaka ana modülde derlenmiş olmalıdır.
- Bubble aşamasında dinleyici `cx.propagate()` çağırmadığı sürece üst öğedeki action dinleyicilerine ulaşılmaz (varsayılan davranış).
- Aynı action ismi iki crate'te tanımlanırsa kayıt çakışması olur; namespace bu yüzden zorunludur.
- Zed çalışma zamanında bilinmeyen action ismi keymap'te uyarı kaydı üretir, `panic` vermez.

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

`name(&self)` çalışma zamanı adını verir; `name_for_type()` statik adı verir. Çalışma zamanı çok biçimliliği (`runtime polymorphism`) söz konusu olduğunda ilki, kayıt sırasında ikincisi kullanılır.

#### `#[action(...)]` öznitelikleri

`#[derive(Action)]` üzerinde tanımlanabilen öznitelikler şunlardır:

- `namespace = my_crate` — action adını `my_crate::Save` biçimine çevirir.
- `name = "OpenFile"` — namespace içinde özel ad.
- `no_json` — `Deserialize`/`JsonSchema` derive zorunluluğunu kaldırır; `build()` her zaman hata döner, `action_json_schema()` `None` verir. Tamamen kod içinde kullanılacak action'lar (örneğin `RangeAction { start: usize }`) için tercih edilir.
- `no_register` — inventory üzerinden otomatik kaydı atlar; trait'i elle uygularken veya koşula bağlı kayıt yaparken gerekir.

#### `register_action!` makrosu

`#[derive(Action)]` kullanılmadan `Action` elle uygulandığında, action'ın inventory'e dahil olabilmesi için ayrı bir kayıt makrosu vardır:

```rust
use gpui::register_action;

register_action!(Paste);
```

Bu makro yalnızca bir `inventory::submit!` çağrısı üretir; struct ya da impl tanımına dokunmaz. `no_register` ile birleştirildiğinde elle kaydın zamanı seçilebilir.

#### Action runtime API'leri

Action'ları çalışma zamanında sorgulamak ve tetiklemek için bazı yardımcılar sağlanır:

- `cx.is_action_available(&action) -> bool` — odaktaki element yolunda bu action'ı dinleyen biri var mı? Menü öğelerini pasifleştirmek için idealdir.
- `window.is_action_available(&action, cx)` — pencereye özel sürüm.
- `cx.dispatch_action(&action)` — odaktaki pencereye yayınlar.
- `window.dispatch_action(action.boxed_clone(), cx)` — pencereye özel sürüm.
- `cx.build_action(name, json_value)` — keymap girdisinden çalışma zamanı action'ı üretir; şema yoksa `ActionBuildError` döner.

#### Tuzaklar

Action makrolarının kullanımındaki ince noktalar:

- `partial_eq` varsayılan olarak `PartialEq` impl'ini kullanır; derive eklenmemişse karşılaştırma yanlış sonuç verebilir.
- Aynı `name()` döndüren iki action kayda alındığında, inventory başlangıçta `panic` üretir; namespace kullanımı çakışmaları önler.
- `no_json` ile işaretlenmiş bir action keymap dosyasından çağrılamaz; yalnızca kod içinden `dispatch_action` ile tetiklenir.

## Keymap, KeyContext ve Dispatch Stack

Action tanımı tek başına yeterli değildir. Kısayolun çalışabilmesi için odaktaki elementin yönlendirme yolunda uygun `KeyContext` bulunmalıdır.

**Bağlam ekleme.** Element bağlamı `key_context` ile bildirilir:

```rust
div()
    .track_focus(&self.focus_handle)
    .key_context("Editor mode=insert")
    .on_action(cx.listener(|this, _: &Save, window, cx| {
        this.save(window, cx);
    }))
```

**Kısayol ekleme.** Kısayollar `bind_keys` ile kaydedilir:

```rust
cx.bind_keys([
    KeyBinding::new("cmd-s", Save, Some("Editor")),
    KeyBinding::new("ctrl-g", GoToLine { line: 0 }, Some("Workspace && !Editor")),
]);
```

**Önemli parçalar.** Bu sistemin temel taşları şunlardır:

- `KeyContext::parse("Editor mode = insert")` — elementin bağlamını üretir.
- `KeyContext::new_with_defaults()` — varsayılan bağlam kümesiyle başlar. `primary()` ve `secondary()`, ayrıştırılan ana ve ek bağlam girişlerine erişir. `is_empty()`, `clear()`, `extend(&other)`, `add(identifier)`, `set(key, value)`, `contains(key)` ve `get(key)`, düşük seviyeli bağlam inşa ve sorgu yüzeyidir.
- `KeyBindingContextPredicate` — kısayol tarafındaki yüklem dilidir: `Editor`, `mode == insert`, `!Terminal`, `Workspace > Editor`, `A && B`, `A || B`.
- `KeyBindingContextPredicate::parse(source)` — yüklem üretir. `eval(context_stack)` bool eşleşme, `depth_of(context_stack)` en derin eşleşme derinliği, `is_superset(&other)` ise keymap önceliği ve çakışma analizinde kullanılır. `eval_inner(...)` public görünür ama ayrıştırıcı veya doğrulayıcı gibi düşük seviyeli kodlar içindir; normal bileşen kodu doğrudan çağırmaz.
- `Keymap::bindings_for_input(input, context_stack)` — eşleşen action'ları ve bekleyen çok vuruşlu (`multi-stroke`) durumu döndürür.
- `Keymap::possible_next_bindings_for_input(input, context_stack)` — mevcut akor önekini (`chord prefix`) takip edebilecek kısayolları öncelik sırasında verir.
- `Keymap::version() -> KeymapVersion` — kısayol seti değiştikçe artan sayaçtır; kısayol UI önbelleklerinde geçersizleştirme anahtarı olarak kullanılabilir.
- `Keymap::new(bindings)`, `add_bindings(bindings)`, `bindings()`, `bindings_for_action(action)`, `all_bindings_for_input(input)` ve `clear()`, ham keymap tablosunu kurma, sorgulama ve sıfırlama yüzeyidir. Uygulama akışında çoğunlukla `cx.bind_keys(...)` ve settings yükleyicisi tercih edilir; bu metotlar test, doğrulayıcı, tanılama ve özel keymap UI'ı için kullanılır.
- `window.context_stack()` — odaktaki düğümden root'a, yönlendirme yolundaki bağlamları verir.
- `window.keystroke_text_for(&action)` — UI'da gösterilecek en yüksek öncelikli kısayol metni.
- `window.possible_bindings_for_input(&[keystroke])` — akor veya bekleyen yardım UI'ları için kullanılır.
- `cx.key_bindings() -> Rc<RefCell<Keymap>>` — keymap'e düşük seviyeli erişim. Üretim kodunda mümkün olduğu kadar `bind_keys`, keymap dosyası ve doğrulayıcı akışı tercih edilir; bu handle test, tanılama ve özel keymap UI'ı için uygundur.
- `cx.clear_key_bindings()` — tüm kısayolları temizler ve pencereleri yeniden çizim için işaretler; normal uygulama akışında değil test veya sıfırlama yollarında kullanılır.

**Öncelik.** Aynı tuşa birden çok kısayol düştüğünde sıralama şu kurallarla çözülür:

- Bağlam yolunda daha derin eşleşme daha yüksek önceliklidir.
- Aynı derinlikte sonra eklenen kısayol önce gelir; kullanıcı keymap'i yerleşik kısayolları bu yüzden ezebilir.
- `NoAction` ve `Unbind` kısayolları devre dışı bırakma için kullanılır.
- Yazdırılabilir girdi IME'ye gidecekse `InputHandler::prefers_ime_for_printable_keys`, kısayol yakalamasının önceliğini düşürebilir. `EntityInputHandler` kullanan view'larda bu değer, `ElementInputHandler` tarafından `accepts_text_input` sonucundan türetilir; ayrı bir karar isteniyorsa ham `InputHandler` yazılır.

**Tuzaklar.** Bu sistemde sık karşılaşılan hatalar:

- `.key_context(...)` bulunmayan bir alt ağaçta bağlam yüklemli kısayol çalışmaz.
- Dinleyici odak yolunda değilse action yayılımı oraya ulaşmaz; genel dinleyici için `cx.on_action(...)`, yerel dinleyici için element üzerinde `.on_action(...)` kullanılır.
- `KeyBinding::new`, ayrıştırma hatasında `panic` verebilir; kullanıcı JSON'undan yükleme yapıldığında `KeyBinding::load` ve hata raporlama tercih edilir.
- `KeyBinding::load(keystrokes, action, context_predicate, use_key_equivalents, action_input, keyboard_mapper)`, başarısız olabilen bir yükleyicidir; `KeyBindingKeystroke` eşlemesini de burada kurar. `context_predicate` `Option<Rc<KeyBindingContextPredicate>>`, `action_input` ise `Option<SharedString>` alır ve hata durumunda `InvalidKeystrokeError` döner. Çalışma zamanında `with_meta(...)` ve `set_meta(...)`, kısayolun hangi keymap katmanından geldiğini taşır; `match_keystrokes(...)` tam veya bekleyen eşleşmeyi verirken `keystrokes()`, `action()`, `predicate()`, `meta()` ve `action_input()` okuyucuları, tanılama ve komut/keymap UI'larını besler.

## DispatchPhase, Olay Yayılımı ve DispatchEventResult

Fare, tuş ve action olayları element ağacında iki aşamada akar:

```rust
pub enum DispatchPhase {
    Capture, // root → odaktaki element
    Bubble,  // odaktaki element → root (varsayılan)
}
```

`Window::on_mouse_event`, `on_key_event` ve `on_modifiers_changed` dinleyicileri aşamaya göre çağrılır. Element fluent API'lerinde `.on_*` ailesi bubble aşamasına, `.capture_*` ailesi ise capture aşamasına bağlanır.

**Kontrol bayrakları** (`crates/gpui/src/app.rs:2021+`):

- `cx.stop_propagation()` — aynı tipteki diğer dinleyicilerin çağrılmasını keser (farede z-index'te alt katman, tuşta ağaçta üst element).
- `cx.propagate()` — önceki `stop_propagation()` etkisini geri alır. Action dinleyicileri bubble aşamasında varsayılan olarak yayılımı durdurduğu için üst öğeye düşürmek isteniyorsa dinleyici içinden `cx.propagate()` çağrılır.
- `window.prevent_default()` ve `window.default_prevented()` — aynı yönlendirme içinde varsayılan element davranışını bastıran pencere bayrağıdır. En görünür kullanım örneği, fare basma sırasında üst öğeye odak aktarımının engellenmesidir.

**Platforma dönen sonuç.** Yönlendirme tamamlandığında platforma şu yapı döner:

```rust
pub struct DispatchEventResult {
    pub propagate: bool,        // hâlâ bubble ediliyorsa true
    pub default_prevented: bool, // GPUI varsayılan davranışı bastırıldı mı
}
```

`PlatformWindow::on_input` geri çağrısı, `Fn(PlatformInput) -> DispatchEventResult` döndürür. Mevcut platform arka uçlarında (`backend`) "olay işlendi mi?" kararı esas olarak `propagate` üzerinden verilir (`!propagate`, "işlendi" anlamına gelir). `default_prevented`, GPUI yönlendirme ağacındaki varsayılan element davranışını ve test/tanılama sonucunu taşır. Bunu genel bir platform iptal mekanizması gibi değil, dinleyicinin açıkça kontrol ettiği yerlerde anlamlandırmak gerekir.

**Pratik akış.** Tipik bir yönlendirme turunda şu adımlar izlenir:

1. Element dinleyicisi tetiklenir, view verisi güncellenir, gerekirse `cx.notify()` çağrılır.
2. Dinleyici olayı tüketmek isterse `cx.stop_propagation()` çağırır.
3. Action dinleyicisi varsayılan davranışı korumak istiyorsa `cx.propagate()` ile yayılımı yeniden açar.
4. Varsayılan odak aktarımı gibi GPUI içi bir davranış bastırılacaksa `window.prevent_default()` çağrılır.

**Tuzaklar.** Bu akışta dikkat edilmesi gerekenler:

- `capture_*` dinleyicileri odak yolu bilinmeden çalışır; pencere genelinde kısayol veya gözlemci için kullanılır; ama veri değiştirilecekse odaktaki element çalışma zamanında kontrol edilmelidir.
- Action yayılımı davranışı, fare veya tuş olaylarından ters çalışır. Yeni bir action dinleyicisinde refleksle `stop_propagation` yazmak, üst öğedeki action'ları engelleyebilir.
- `default_prevented`, genel bir platform iptal API'si değildir; hangi davranışın durdurulduğunu anlamak için ilgili element veya pencere dinleyicisinin `window.default_prevented()` kontrol edip etmediğine bakılır.

## Action ve Keymap'in Çalışma Zamanı İncelemesi

Action tanımlama ve yönlendirme önceki bölümlerde işlenmiştir. Zed komut paleti, keymap UI'ı ve geliştirici tanılaması için çalışma zamanı inceleme (`introspection`) yüzeyinin de bilinmesi gerekir.

**Action kayıt defteri.** Kayıt defterindeki action'lar üzerinde aşağıdaki yardımcılar sorgu yapar:

- `cx.build_action(name, data) -> Result<Box<dyn Action>, ActionBuildError>` — metin (`string`) action adı ve isteğe bağlı JSON verisinden çalışma zamanı action'ı üretir.
- `cx.all_action_names() -> &[&'static str]` — kayda alınmış tüm action adlarını döndürür. Kayıtlı olmak, action'ın element ağacında o anda kullanılabilir olduğu anlamına gelmez.
- `cx.action_schemas(generator)` — iç kullanım dışı action adlarını ve JSON şemalarını verir.
- `cx.action_schema_by_name(name, generator)` — tek bir action için şema döndürür; `None` action'ın yokluğunu, `Some(None)` action'ın varlığını ama şema bulunmayışını ifade eder.
- `cx.action_documentation()` — keymap düzenleyicisi, JSON şeması ve geliştirici tanılaması için action açıklamalarını sağlar.

**Kullanılabilir action ve kısayol sorguları.** Bağlamdaki action durumunu ve ilgili kısayolları öğrenmek için:

- `window.available_actions(cx)` — odaktaki element yönlendirme yolundaki action dinleyicileriyle genel action dinleyicilerini birleştirir. Menü ve komut UI'ında "bu action şu anda yapılabilir mi?" sorusunun pencereye özel cevabıdır.
- `window.on_action_when(condition, TypeId::of::<A>(), listener)` — çizim aşamasında o anki yönlendirme düğümüne koşullu, düşük seviyeli action dinleyicisi ekler. Element API'sindeki `.on_action(...)` ve `.capture_action(...)` genelde daha okunaklıdır; özel element yazılmıyorsa bu seviyeye inilmez.
- `cx.is_action_available(&action)` ve `window.is_action_available(&action, cx)` — bool kısayollar.
- `window.is_action_available_in(&action, focus_handle)` — action kullanılabilirliği sorgusunu belirli bir odak handle'ının yönlendirme yolundan yapar.
- `window.bindings_for_action(&action)` — odaktaki bağlam yığınına göre action'a giden kısayolları döner; gösterim için son kısayol en yüksek öncelikli kabul edilir.
- `window.highest_precedence_binding_for_action(&action)` — aynı sorgunun tek sonuçlu, daha ucuz sürümü.
- `window.bindings_for_action_in(&action, focus_handle)` ve `highest_precedence_binding_for_action_in(...)` — sorguyu belirli bir odak handle'ı yolundan yapar.
- `window.bindings_for_action_in_context(&action, KeyContext)` — tek bir elle verilmiş bağlama göre sorgu.
- `window.highest_precedence_binding_for_action_in_context(&action, KeyContext)` — aynı sorgunun tek sonuçlu en yüksek öncelikli sürümü.
- `cx.all_bindings_for_input(&[Keystroke])` — bağlama bakmadan girdi dizisine kayıtlı tüm kısayolları listeler.
- `window.possible_bindings_for_input(&[Keystroke])` — çok vuruşlu veya önek akışında o anki bağlam yığınına göre sıradaki aday kısayolları verir. Tam eşleşen action yönlendirme sonucu için normal `window.dispatch_keystroke(...)` akışı kullanılır; public `Window::bindings_for_input` yardımcısı yoktur.
- `window.pending_input_keystrokes()` ve `window.has_pending_keystrokes()` — tamamlanmamış tuş akoru (`key chord`) durumunu UI'da göstermek veya test etmek için.

**Genel keystroke gözlemi.** Tüm pencerelerdeki keystroke akışını yakalamak için iki kanca vardır; biri yönlendirmeden sonra, diğeri yönlendirmeden önce çalışır:

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

- `observe_keystrokes`, action ve olay mekanizmaları çözüldükten sonra çalışır; yayılım durdurulmuşsa çağrılmaz.
- `intercept_keystrokes`, yönlendirmeden önce çalışır; burada `cx.stop_propagation()` çağrısı action yönlendirmesini engeller.
- Her ikisi de `Subscription` döner; bu değer elden çıkınca gözlemci düşer.

**Tuzaklar.** İnceleme yüzeyini kullanırken atlanan noktalar:

- `all_action_names` içinde görünen bir action'ın o anda kullanılabilir olması garanti değildir; UI öğesini etkin veya pasif yapmak için `available_actions` ya da `is_action_available` tercih edilir.
- Kısayol gösteriminde bağlam yığınını dikkate almayan `cx.all_bindings_for_input` yerine mümkün olduğunda pencere veya odak handle'ı bazlı sorgu kullanılır.
- Yakalayıcılar (`interceptor`) genel etkilidir; modal pencerelere özgü tuş engelleme yapılacaksa mümkün olduğunca element action veya capture dinleyicisi ile sınırlı tutulur.

## Zed Keymap Dosyası, Doğrulayıcı ve Unbind Akışı

GPUI action ve kısayol modeli Zed'de `settings::keymap_file` üzerinden kullanıcı dosyasına bağlanır. Bu bölüm, çalışma zamanı yönlendirmesinden farklı olarak JSON yükleme, şema ve dosya güncelleme tarafını kapsar.

**Dosya modeli.** Keymap JSON yapısı birkaç ana tip üzerinden okunur:

- `KeymapFile(Vec<KeymapSection>)` — üst seviye JSON array.
- `KeymapSection` — bağlam yüklemiyle birlikte kısayol ve unbind eşlemelerini taşır. Alan görünürlüğü dengeli değildir: yalnız `pub context: String` dış erişime açıktır; `use_key_equivalents: bool`, `unbind: Option<IndexMap<...>>`, `bindings: Option<IndexMap<...>>` ve `unrecognized_fields: IndexMap<...>` alanları **private**'tır. `KeymapFile` ayrıştırıcısı bu alanlara crate içinden doğrudan erişir; dış kod yalnızca `KeymapSection::bindings(&self)` okuyucusunu kullanabilir ve bu da `(keystroke, KeymapAction)` çiftlerini dönen tek public iteratördür (`keymap_file.rs:101`). `unbind` ve `use_key_equivalents` için ayrı bir public okuyucu bu sürümde bulunmuyor.
- `KeymapAction(Value)` — `null`, `"action::Name"` veya `["action::Name", { ...args... }]` biçimlerini temsil eder.
- `UnbindTargetAction(Value)` — `unbind` eşlemesindeki hedef action değeri.
- `KeymapFileLoadResult::{Success, SomeFailedToLoad, JsonParseFailure}` — dosyanın kısmen yüklenebildiği senaryoyu açıkça ayırır.

**Yükleme.** İçeriği ayrıştırmanın veya yüklemenin iki ana yolu vardır:

```rust
let keymap = KeymapFile::parse(&contents)?;
let result = KeymapFile::load(&contents, cx);
```

`load_asset(asset_path, source, cx)`, paketlenmiş (`bundled`) keymap dosyalarını yükler ve `KeybindSource` üst verisini ayarlayabilir. `load_panic_on_failure` yalnızca başlangıç ve test gibi "asset bozuksa devam etmeyelim" yolları içindir.

**Temel keymap.** Hangi temel keymap'in seçili olduğu enum'da tutulur:

- `BaseKeymap::{VSCode, JetBrains, SublimeText, Atom, TextMate, Emacs, Cursor, None}`.
- Temel, varsayılan, vim ve kullanıcı kısayolları `KeybindSource` üst verisi taşır; UI bu üst veriyle kısayolun nereden geldiğini gösterir.

**Yerleşik keymap davranışı.** Aşağıdaki davranışlar Zed'in varsayılan keymap kurulumunda gözlenir:

- Agent panelindeki ACP thread'ine özgü kısayollar `AcpThread` bağlamında yaşar. Terminal alt bağlamı, alt öğe yüklemi ile `AgentPanel > Terminal` kullanır; bu, `>` operatörünün gerçek yönlendirme yolu ilişkisi için kullanılmasının pratik örneğidir.
- Git paneli iki sekmelidir: `git_panel::ActivateChangesTab` ve `git_panel::ActivateHistoryTab` için varsayılan kısayol macOS'ta `cmd-1` ile `cmd-2`, Linux/Windows'ta `ctrl-1` ile `ctrl-2`'dir.
- Worktree seçici, `worktree_picker::ForceDeleteWorktree` action'ını destekler. Varsayılan kısayol macOS'ta `cmd-alt-shift-backspace`, Linux/Windows'ta `ctrl-alt-shift-backspace`'tir; UI tarafında silme ikonu üzerinde `alt` basılıysa zorla silme yolu çalışır.
- `buffer_search::UseSelectionForFind` seçimi, seçim yoksa imleç altındaki kelimeyi arama sorgusu olarak kullanır. Ayarın atlanması gereken çağrılar, `SeedQuerySetting::Always` üzerine yazmasını vermelidir.
- Vim kullanırken Helix tarzında kelimeye atlama `vim::HelixJumpToWord` aksiyonudur. Varsayılan kısayol verilmez; kullanıcı normal veya visual mod bağlamında örneğin `"g w"` bağlayabilir. Normal modda hedef kelimenin başına taşır, visual modda seçimi hedef kelime başına kadar genişletir. Vim'in varsayılan `g w` yeniden sarma (`rewrap`) davranışını korumak isteyenler bu kısayolu eklememelidir.

**Doğrulayıcı.** Belirli action tiplerine özel doğrulama mantığı eklenebilir:

- `KeyBindingValidator` — belirli bir action tipi için kısayol doğrulaması yapar.
- `KeyBindingValidatorRegistration(pub fn() -> Box<dyn KeyBindingValidator>)` inventory ile toplanır.
- Doğrulayıcı hataları `MarkdownString` döner; keymap UI bunu kullanıcıya okunaklı bir hata olarak gösterebilir.

**Devre dışı bırakma ve unbind nöbetçi action'ları (`sentinel`).** GPUI iki ayrı nöbetçi action sağlar (`gpui::action.rs:425-453`). İkisinin çalışma zamanı davranışı da **yönlendirme yapmamak**'tır; ancak keymap yönlendirme tablosundaki görevleri farklıdır:

- **`zed::NoAction`** — `actions!(zed, [NoAction])` ile tanımlıdır, veri taşımaz. Keymap JSON'unda eylem değeri olarak `null` veya `"zed::NoAction"` yazılır:

  ```json
  { "context": "Editor", "bindings": { "cmd-p": null } }
  ```

  Aynı keystroke'a daha düşük öncelikle bağlanmış action'lar bu bağlam için iptal edilir; eşleşme `Keymap::resolve_binding` tarafından `disabled_binding_matches_context(...)` çağrısıyla **bağlama duyarlı** şekilde süzülür (`gpui/src/keymap.rs:120`). Yani `NoAction`, belirli bir bağlam yüklemi içinde o tuşu sessize alır.

- **`zed::Unbind(SharedString)`** — `derive(Action)` ile tanımlıdır; yükü (`payload`) bir action adıdır. JSON biçimi şu şekildedir:

  ```json
  ["zed::Unbind", "editor::NewLine"]
  ```

  Keymap ayrıştırıcısı bu nöbetçiyi gördüğünde aynı keystroke'a aynı action adıyla ileriden gelen tüm kısayolları **action bağlamından bağımsız olarak** iptal eder (`keymap.rs:124-128`'deki `is_unbind` kolu). Yani `editor::NewLine` için `enter` tuşunun tüm bağlamlarda unbind edilmesi gerektiğinde kullanılır.

**Nöbetçi kontrol API'leri.** Nöbetçi action'ları çalışma zamanında tespit etmek için iki ufak yardımcı vardır:

- `gpui::is_no_action(&dyn Action) -> bool` (`action.rs:445`) — `as_any().is::<NoAction>()` üzerinden alta dönüştürme yapar. Özel keymap UI veya komut paleti listesinde "(disabled)" göstergesi koymak için uygundur.
- `gpui::is_unbind(&dyn Action) -> bool` (`action.rs:450`) — aynı şekilde `Unbind` nesnesini alta dönüştürür.

`KeymapFile::update_keybinding`, `KeybindUpdateOperation::Remove` için bu iki nöbetçiyi bağlama göre üretir: kullanıcı kısayolları `Remove` ile doğrudan dosyadan silinir; çatı (`framework`) veya varsayılan kısayolların sessize alınabilmesi için kullanıcı keymap'ine `null` (NoAction) ya da `["zed::Unbind", ...]` girdisi yazılır.

**Dosya güncelleme.** Keymap dosyasını programatik olarak değiştirmek için şu fonksiyon kullanılır:

```rust
let updated = KeymapFile::update_keybinding(
    operation,
    keymap_contents,
    tab_size,
    keyboard_mapper,
)?;
```

- `KeybindUpdateOperation::Add { source, from }` — yeni kısayol ekler.
- `Replace { source, target, target_keybind_source }` — kullanıcı kısayoluysa değiştirir; kullanıcı dışı kısayol değişiyorsa ekleme artı bastırma için unbind'e dönüştürebilir.
- `Remove { target, target_keybind_source }` — kullanıcı kısayolunu dosyadan siler; kullanıcı dışı bir kısayolu kaldırmak için `unbind` yazar.
- `KeybindUpdateTarget`, action adı, isteğe bağlı action argümanları, bağlam ve `KeybindingKeystroke` dizisini taşır.

**Tuzaklar.** Dosya güncellemeyle ilgili dikkat noktaları:

- `use_key_equivalents` yalnızca destekleyen platformlarda anlamlıdır; klavye eşleyicisi (`keyboard mapper`) sağlanmadan dosya güncellemesi doğru keystroke metnini üretemez.
- Kullanıcı dışı bir kısayolu "silmek" gerçek kaynağı değiştirmez; kullanıcı keymap'ine bastırma yapan bir `unbind` girdisi yazılır.
- Kullanıcı JSON'u bozuksa `update_keybinding` dosyayı değiştirmez; önce ayrıştırma başarıyla geçmelidir.

---
