# Etkileşim ve Olaylar

---

## Klavye Odağı, Odak Kaybı ve Klavye Olayları

Klavye odağı GPUI'de `FocusHandle` ile temsil edilir. Bir view'un odak alıp verebilmesi için kendine ait bir handle tutması ve çizim sırasında bu handle'ı elemente bağlaması gerekir.

```rust
struct View {
    focus_handle: FocusHandle,
}

impl View {
    fn new(cx: &mut Context<Self>) -> Self {
        Self {
            focus_handle: cx.focus_handle(),
        }
    }
}
```

Çizim zincirinde handle element'e bağlanır; isteğe bağlı olarak `focus-visible` stili eklenir:

```rust
div()
    .track_focus(&self.focus_handle)
    .focus_visible(|style| style.border_color(cx.theme().colors().border_focused))
```

Programatik olarak odak vermek için handle'ın kendisi veya `cx.focus_view` çağrısı kullanılır:

```rust
self.focus_handle.focus(window, cx);
// veya
cx.focus_view(&child_entity, window);
```

**Odak sorguları.** Mevcut odak durumunu kontrol etmek için üç temel soru ve üç karşılık gelen metot vardır:

- `focus_handle.is_focused(window)` — handle doğrudan odakta mı?
- `focus_handle.contains_focused(window, cx)` — bu handle veya altındaki bir düğüm odakta mı?
- `focus_handle.within_focused(window, cx)` — bu handle odakta olan düğümün içinde mi?

**Odak olayları.** Odakla ilgili değişimleri dinlemek için ayrı abonelik metotları mevcuttur:

- `cx.on_focus(handle, window, ...)` — handle doğrudan odak aldı.
- `cx.on_focus_in(handle, window, ...)` — handle veya bir alt öğe odak aldı.
- `cx.on_blur(handle, window, ...)` — handle odak kaybetti.
- `cx.on_focus_out(handle, window, |this, event, window, cx| ...)` — handle veya bir alt öğe odak dışına çıktı; geri çağrı view verisini alır ve `FocusOutEvent` içinden odağı kaybeden handle'a (`event.blurred`) erişilebilir.
- `window.on_focus_out(handle, cx, |event, window, cx| ...)` — aynı olayın view verisi almayan, daha düşük seviyeli `Window` varyantıdır; sonucu `Subscription` olarak döner.
- `cx.on_focus_lost(window, ...)` — pencere içinde hiçbir handle odakta kalmadığında çalışır.

**Klavye action akışı.** Tuşların action'a bağlanması birkaç adımdan oluşur; bu adımlar her özel kısayol için tekrarlanır:

1. `actions!(namespace, [ActionA, ActionB])` veya `#[derive(Action)]` + `#[action(...)]` ile action tanımı yapılır.
2. Element ağacında `.key_context("context-name")` belirtilir; bu sayede action yalnızca uygun bağlamda yönlendirilir.
3. `cx.bind_keys([KeyBinding::new("cmd-k", ActionA, Some("context-name"))])` ile kısayol kaydedilir.
4. Dinleyici için `.on_action(...)`, `.capture_action(...)` veya `cx.on_action(...)` kullanılır.

**Olay yayılımı.** GPUI olay yayılımı varsayılan olarak yukarı doğru ilerler. İki yardımcı bu davranışı kontrol eder:

- Fare ve tuş olay dinleyicileri olayı varsayılan olarak yukarıya iletir.
- `cx.stop_propagation()`, arkadaki veya üstteki dinleyicilere olayın ulaşmasını keser.
- Action `bubble` aşamasında dinleyiciler varsayılan olarak yayılımı durdurur; gerekirse `cx.propagate()` ile devam ettirilebilir.

## Fare, Sürükle-Bırak ve Hitbox

Element seviyesindeki etkileşim API'leri tek bir fluent zincir içinde toplanır; aşağıdaki metotlar farklı fare olaylarına ve sürükle-bırak kalıplarına karşılık gelir:

- `.on_click(...)`
- `.on_mouse_down(...)`, `.on_mouse_up(...)`, `.on_mouse_move(...)`
- `.on_mouse_down_out(...)`, `.on_mouse_up_out(...)`
- `.on_scroll_wheel(...)`
- `.on_pinch(...)`
- `.on_drag_move::<T>(...)`
- `.drag_over::<T>(...)`
- `.on_drop::<T>(...)`
- `.can_drop(...)`
- `.occlude()` veya `.block_mouse_except_scroll()`
- `.cursor_pointer()`, `.cursor(...)`

Pencere kontrol hitbox'ı isteniyorsa fluent API üzerinden işaretlenir:

```rust
h_flex()
    .window_control_area(WindowControlArea::Drag)
```

Özel yeniden boyutlandırma ve imleç davranışı için `canvas` ile hitbox eklemek, Zed'deki istemci tarafı pencere süslemesi (`client decoration`) deseninin tipik bir örneğidir:

```rust
canvas(
    |bounds, window, _cx| {
        window.insert_hitbox(bounds, HitboxBehavior::Normal)
    },
    |_bounds, hitbox, window, _cx| {
        window.set_cursor_style(CursorStyle::ResizeLeftRight, &hitbox);
    },
)
```

Burada `canvas` imzası `prepaint: FnOnce(Bounds<Pixels>, &mut Window, &mut App) -> T` ve `paint: FnOnce(Bounds<Pixels>, T, &mut Window, &mut App)` şeklindedir. İkinci closure'da ilk pozisyonel argüman `bounds`'tur (kullanılmıyorsa `_bounds`), ikinci argüman ise prepaint'in döndürdüğü değerdir (örnekteki `hitbox`). `set_cursor_style` hitbox'a referans aldığı için `&hitbox` şeklinde geçilir.

## Sürükleme ve Bırakma İçeriği Üretimi

`crates/gpui/src/elements/div.rs:572+` ve `1271+`.

GPUI'da sürükleme sırasında, sürüklenen elementin yerine ayrı bir hayalet (`ghost`) view oluşturulur ve fare ile birlikte bu view hareket eder:

```rust
div()
    .id("draggable")
    .on_drag(payload.clone(), |payload, mouse_offset, window, cx| {
        cx.new(|_| GhostView::for_payload(payload.clone(), mouse_offset))
    })
```

İmza şu şekildedir:

```rust
fn on_drag<T, W>(
    self,
    value: T,
    constructor: impl Fn(&T, Point<Pixels>, &mut Window, &mut App) -> Entity<W> + 'static,
) -> Self
where
    T: 'static,
    W: 'static + Render;
```

- `value: T` — sürükleme yükünün (`payload`) tipidir; alıcı tarafta `on_drop::<T>` ile aynı tipe bağlanır.
- `constructor` — her sürükleme başlangıcında hayalet view üreten yapıcıdır; fare uzaklığını yüke göre konumlandırır.
- `W: Render` — hayaletin kendi entity'sidir; standart çizim gibi davranır.

**Bırakma tarafı.** Alıcı element kabul edilebilirlik kontrolünü, stilini ve dinleyicisini ayrı ayrı tanımlar:

```rust
div()
    .drag_over::<MyPayload>(|style, payload, window, cx| {
        style.bg(rgb(0xeeeeee))
    })
    .can_drop(|payload, window, cx| {
        payload
            .downcast_ref::<MyPayload>()
            .is_some_and(|payload| payload.is_compatible(window, cx))
    })
    .on_drop::<MyPayload>(cx.listener(|this, payload: &MyPayload, window, cx| {
        this.accept(payload.clone());
        cx.notify();
    }))
```

**API.** Sürükle-bırak akışı için kullanılan başlıca metotlar şunlardır:

- `.on_drag::<T, W>(value, ctor)` — sürüklemeyi başlatır.
- `.drag_over::<T>(|style, payload, window, cx| -> StyleRefinement)` — hover sırasında uygulanan stil refinement'ı.
- `.can_drop(|payload: &dyn Any, window, cx| -> bool)` — bırakmanın kabul edilip edilmeyeceğine karar verir. Tip kontrolü için `downcast_ref::<T>()` kullanılır.
- `.on_drop::<T>(listener)` — bırakma tamamlandığında çalışır.
- `.on_drag_move::<T>(listener)` — sürükleme süresince fare konumu bilgisi verir.
- `cx.has_active_drag()` — uygulama genelinde aktif bir sürükleme olup olmadığını döner.
- `cx.active_drag_cursor_style()` — aktif sürükleme için imleç üzerine yazma değerini verir.
- `cx.stop_active_drag(window)` — aktif sürüklemeyi temizler, pencereyi yeniden çizim için işaretler ve gerçekten bir sürükleme varsa `true` döner. Escape veya iptal yollarında kullanılır.

**Harici sürükleme.** Dosya sisteminden sürükleyip bırakma akışı için `FileDropEvent` ve `ExternalPaths` tipleri kullanılır. Platform `FileDropEvent::Entered/Pending/Submit/Exited` üretir; `Window::dispatch_event` bu olayları dahili `active_drag` durumuna ve `ExternalPaths` yüküne çevirir. UI tarafında normal sürükle-bırak API'siyle yakalanır:

```rust
div()
    .on_drag_move::<ExternalPaths>(cx.listener(|this, event, window, cx| {
        let paths = event.drag(cx).paths();
        this.preview_external_drop(paths, event.bounds, window, cx);
    }))
    .on_drop(cx.listener(|this, paths: &ExternalPaths, window, cx| {
        this.handle_external_paths_drop(paths, window, cx);
    }))
```

`ExternalPaths::paths()` `&[PathBuf]` döner. Hayalet view, dosya ikonları olarak platform tarafından çizilir; GPUI tarafındaki `Render for ExternalPaths` bilerek `Empty` döndürür.

**Tuzaklar.** Sürükle-bırak yazarken karşılaşılan yaygın hatalar:

- Sürüklenen tip `T: 'static` olmalıdır; ödünç alma süresi (`lifetime`) taşıyan tipler kabul edilmez.
- Aynı element üzerinde `on_drag` iki kez çağrıldığında `panic` oluşur ("calling on_drag more than once on the same element is not supported").
- Hayalet view her sürüklemede yeni bir `cx.new(...)` ile yaratılır; yapıcı içinde yan etkiden kaçınılır.
- `can_drop` `false` döndüğünde `drag_over` ve `group_drag_over` stilleri uygulanmaz, `on_drop` çağrılmaz. Kabul edilmeyen hedef için ayrı bir görsel geri bildirim gösterilecekse `on_drag_move` kullanılır.

## Hitbox, İmleç, İşaretçi Yakalama ve Otomatik Kaydırma

Hitbox, fare çarpışma testinin (`hit-test`) ve imleç davranışının temelidir. Element dinleyicileri çoğu zaman hitbox'ı arka planda kurar; bu API doğrudan özel `canvas` veya özel element yazılırken devreye girer.

```rust
let hitbox = window.insert_hitbox(bounds, HitboxBehavior::Normal);
if hitbox.is_hovered(window) {
    window.set_cursor_style(CursorStyle::PointingHand, &hitbox);
}
```

**Davranış tipleri.** Hitbox'ın arka planda kalan başka hitbox'larla ilişkisi `HitboxBehavior` ile ifade edilir:

- `HitboxBehavior::Normal` — arkadaki hitbox'ları etkilemez.
- `HitboxBehavior::BlockMouse` — arkadaki fare, hover, ipucu (`tooltip`) ve scroll hitbox davranışlarını engeller. `.occlude()` bu davranışı kullanır.
- `HitboxBehavior::BlockMouseExceptScroll` — arkadaki fare etkileşimini engeller ama scroll'un geçmesine izin verir. `.block_mouse_except_scroll()` bu davranışı kullanır.

**İşaretçi yakalama.** Sürükleme veya yeniden boyutlandırma gibi senaryolarda fare element sınırlarının dışına çıksa bile olayları almaya devam etmek için işaretçi yakalama (`pointer capture`) kullanılır:

```rust
window.capture_pointer(hitbox.id);
// drag/resize bittiğinde
window.release_pointer();
```

Yakalama aktifken ilgili hitbox üzerinde durulmuş (`hovered`) sayılır. Yeniden boyutlandırma tutamacı ve sürükleme etkileşimlerinde fare element sınırlarının dışına çıksa bile hareket takip edilebilir. `window.captured_hitbox()` aktif yakalama id'sini döndürür; özel element hata ayıklaması veya iç içe sürükleme verisini ayrıştırma dışında genelde kullanılmaz.

**Otomatik kaydırma.** Sürükleme sırasında görünür alanın kenarına yaklaşıldığında otomatik kaydırma talep etmek için iki yardımcı vardır:

- `window.request_autoscroll(bounds)` — sürükleme sırasında görünür alan kenarına yakın bölge için otomatik kaydırma talep eder.
- `window.take_autoscroll()` — scroll kapsayıcısı tarafında bu talebi tüketir.

**İmleç.** İmleç stili hitbox veya pencere bağlamında ayarlanır:

- `window.set_cursor_style(style, &hitbox)` — hitbox üzerinde durulmuşsa imleç stilini ayarlar.
- `window.set_window_cursor_style(style)` — pencere genelindeki imleç durumunu ayarlar.
- `cx.set_active_drag_cursor_style(style, window)` — aktif sürükleme yükü için imleç üzerine yazma.
- `cx.active_drag_cursor_style()` — mevcut sürükleme imlecini okur.

**Tuzaklar.** Hitbox ve imleç tarafında dikkat edilecek noktalar:

- `Hitbox::is_hovered`, klavye girdi kipi sırasında `false` dönebilir; scroll dinleyicisi yazılırken `should_handle_scroll` tercih edilir.
- Üst katman elementleri `.occlude()` kullanmazsa arkadaki butonlar hover ve tıklama almaya devam edebilir.
- İşaretçi yakalama serbest bırakılmadığında sonraki fare hareketlerinde yanlış hitbox üstte kalabilir.

## Tab Sırası ve Klavye Navigasyonu

`crates/gpui/src/tab_stop.rs`, `window.rs:397`.

Tab navigasyonu `FocusHandle` üzerindeki iki bayrak yardımıyla kontrol edilir; ikisi de fluent zincirde okunur:

```rust
let handle = cx.focus_handle()
    .tab_stop(true)        // Tab tuşuyla durulabilir
    .tab_index(0);         // Sıralama yoluna katılır
```

**Sıralama kuralları.** Tab gezinme sırası, `TabStopMap` içindeki düğüm sıralamasına göre belirlenir:

1. Aynı grup içinde `tab_index` küçükten büyüğe sıralanır.
2. `tab_index` eşit olduğunda element ağaç sırası (DFS) belirleyicidir.
3. `tab_stop(false)` olan handle, sıradaki konumunu korur ama klavyeyle durak olmaz. Negatif `tab_index` özel olarak "devre dışı" anlamına gelmez; yalnızca sıralamada daha erken bir yol değeri üretir.

**Gruplar.** Bir grup tanımlamak için element tarafında `.tab_group()` kullanılır; grubun sırası gerekiyorsa aynı elemente `.tab_index(index)` verilir. `TabStopMap::begin_group` ve `end_group`, gezinme algoritmasının iç operasyonlarıdır; uygulama kodunda doğrudan çağrılmaz.

Düşük seviyeli karşılık `window.with_tab_group(Some(index), |window| ...)` çağrısıdır; `None` verilirse grup açılmadan closure çalışır. Normal bileşen kodunda `.tab_group()` fluent API'si tercih edilir.

**`Window` üzerindeki yardımcılar.** Tab ve Shift-Tab davranışı pencere üzerinden yapılır:

- `window.focus_next(cx)` / `window.focus_prev(cx)` — Tab veya Shift-Tab geldiğinde çağrılır.
- `window.focused(cx)` — o anki odak handle'ını verir.

**Özel girdi bileşeni.** Tab akışına dahil olacak özel bir girdi (`input`) bileşeni için:

```rust
div()
    .track_focus(&self.focus_handle)
    .on_action(cx.listener(|this, _: &menu::Confirm, window, cx| { ... }))
    .child(/* ... */)
```

`tab_stop(true)` olmadan handle yalnızca programatik olarak odak alır; klavyeyle ulaşılamaz. Erişilebilirlik ve form akışı için her interaktif elementin bir handle'a sahip olması beklenir.

## Metin Girdisi ve IME

Platform IME entegrasyonu `InputHandler` üzerinden çalışır. Düzenleyici benzeri metin alanlarının aşağıdaki metot ailesini sağlaması gerekir:

- `selected_text_range`
- `marked_text_range`
- `text_for_range`
- `replace_text_in_range`
- `replace_and_mark_text_in_range`
- `unmark_text`
- `bounds_for_range`
- `character_index_for_point`
- `accepts_text_input`

Ham `InputHandler` uygulaması yazıldığında ayrıca `prefers_ime_for_printable_keys` üzerine yazılabilir. Bununla birlikte yaygın view yolu olan `EntityInputHandler` + `ElementInputHandler` ikilisinde bu ayrı bir kanca (`hook`) değildir; mevcut sarmalayıcı, `prefers_ime_for_printable_keys` sorusunu `accepts_text_input` sonucunu kullanarak yanıtlar. IME ve kısayol önceliğinin `accepts_text_input`'tan bağımsız yönetilmesi gerekiyorsa doğrudan `InputHandler` uygulayan özel bir dinleyici yazılır.

IME aday penceresinin doğru konumda kalması için imleç hareketinden sonra:

```rust
window.invalidate_character_coordinates();
```

Zed'de form tipindeki tek satırlık girdi için doğrudan düzenleyici yazmak yerine `ui_input::InputField` kullanılır. Bu crate, düzenleyiciye (`editor`) bağlı olduğu için `ui` içinde değildir.

**`ui_input` genel yüzeyi.** Genel API üzerinde aşağıdaki öğeler bulunur:

- `pub use input_field::*`; ana bileşen `InputField`.
- `InputField::new(window, cx, placeholder_text)`, tek satırlık bir düzenleyici nesnesi ister ve yer tutucuyu (`placeholder`) hemen düzenleyiciye yazar.
- Builder ve metot zinciri: `.start_icon(IconName)`, `.label(...)`, `.label_size(LabelSize)`, `.label_min_width(Length)`, `.tab_index(isize)`, `.tab_stop(bool)`, `.masked(bool)`, `.is_empty(cx)`, `.editor()`, `.text(cx)`, `.clear(window, cx)`, `.set_text(text, window, cx)`, `.set_masked(masked, window, cx)`.
- `InputFieldStyle`, `pub` bir struct olarak görünür ancak alanları private'dır; dışarıdan stil üzerine yazma sözleşmesi değil, çizim içi tema anlık görüntüsüdür.
- `ErasedEditor` trait'i düzenleyici köprüsüdür; `text`, `set_text`, `clear`, `set_placeholder_text`, `move_selection_to_end`, `set_masked`, `focus_handle`, `subscribe`, `render`, `as_any` metotlarını içerir.
- `ErasedEditorEvent::{BufferEdited, Blurred}`, picker veya arama gibi üst bileşenlerin düzenleme ve odak kaybı akışını dinlemesi için yayınlanır.
- `ERASED_EDITOR_FACTORY: OnceLock<fn(&mut Window, &mut App) -> Arc<dyn ErasedEditor>>`, düzenleyici crate'i tarafından kurulur. Zed'de `crates/editor/src/editor.rs` init akışında bu fabrika, `Editor::single_line(window, cx)` döndüren `ErasedEditorImpl` ile atanır. Fabrika atanmamışken `InputField::new` `panic` üretir; bu nedenle uygulama init sırası, düzenleyici kurulumu tamamlandıktan sonra `InputField` üretimine güvenmelidir.

## Metin Girdisi Dinleyicisi ve IME Derin Akışı

Metin düzenleyen özel bir element yazılırken yalnızca tuş olayı dinlemek yeterli değildir. IME, ölü tuş (`dead key`), işaretli metin (`marked text`) ve aday penceresi için platforma `InputHandler` sağlanmalıdır.

**View tarafı.** Görece geniş bir trait yüzeyi vardır; sık kullanılan metotlar şu şekilde uygulanır:

```rust
impl EntityInputHandler for EditorLikeView {
    fn selected_text_range(
        &mut self,
        ignore_disabled_input: bool,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<UTF16Selection> {
        self.selection_utf16(ignore_disabled_input, window, cx)
    }

    fn marked_text_range(
        &self,
        window: &mut Window,
        cx: &mut Context<Self>,
    ) -> Option<Range<usize>> {
        self.marked_range_utf16(window, cx)
    }

    fn unmark_text(&mut self, window: &mut Window, cx: &mut Context<Self>) {
        self.clear_marked_text(window, cx);
    }

    // text_for_range, replace_text_in_range,
    // replace_and_mark_text_in_range, bounds_for_range,
    // character_index_for_point da uygulanır.
}
```

Element çizimi sırasında dinleyici pencereye kaydedilir:

```rust
window.handle_input(
    &focus_handle,
    ElementInputHandler::new(bounds, view_entity.clone()),
    cx,
);
```

**Kurallar.** IME entegrasyonunda sıkça gözden kaçan noktalar şunlardır:

- Aralık (`Range`) değerleri UTF-16 ofsetidir; Rust byte index'iyle karıştırılmaz.
- `bounds_for_range`, ekran veya aday penceresi konumlandırması için doğru mutlak sınırları döndürmelidir.
- İmleç veya seçim hareketinden sonra `window.invalidate_character_coordinates()` çağrılır; aksi halde IME paneli yeni konuma taşınmaz.
- `accepts_text_input` `false` olduğunda platformun metin eklemesi engellenebilir.
- Ham `InputHandler::prefers_ime_for_printable_keys` `true` olduğunda, ASCII dışı IME aktifken yazdırılabilir tuşlar kısayoldan önce IME'ye gider. `ElementInputHandler` sarmalı `EntityInputHandler` için GPUI bu kararı `accepts_text_input` üzerinden verir; trait'te ayrı bir üzerine yazma noktası yoktur.
- Pencere ekran karesi geçişinde platform girdi dinleyicisi `Vec<Option<_>>` slot'ları `.pop()` ile kısaltılmaz; `.take()` ile boş slot bırakılır ve bir sonraki ekran karesinde aynı slot'a geri yerleştirilir. `reuse_paint` önbelleğindeki `paint_range` indeksleri bu yüzden stabil kalır. Özel düşük seviyeli pencere veya ekran karesi kodu yazılırken girdi dinleyicisi dizisinin uzunluğu, indeks önbelleği varken değiştirilmez.

**Tuzaklar.** IME ile çalışırken sık yapılan hatalar:

- Yalnızca `.on_key_down` ile metin düzenleyici yazmak, IME ve ölü tuşlu (`dead key`) dillerde bozulur.
- UTF-16 aralığını doğrudan byte dilimine uygulamak, çok byte'lı karakterlerde `panic` ya da yanlış seçim üretir.
- Girdi dinleyicisi ekran karesine bağlıdır; odaktaki element çizilmediğinde platform girdi dinleyicisi de düşer.

## Keystroke, Modifiers ve Platform Bağımsız Kısayollar

`crates/gpui/src/platform/keystroke.rs`, klavye girdisinin normalize edilmiş modelini içerir. Keymap yalnızca action bağlama değildir; tamamlanmamış girdi, IME durumu ve gösterim metni de bu tiplerle taşınır.

**Ana tipler.** Klavye dünyasını ifade eden tipler birbirini destekleyecek şekilde tasarlanmıştır:

- `Keystroke { modifiers, key, key_char }` — gerçek tuş vuruşu. `key`, basılan tuşun ASCII karşılığıdır (örneğin `option-s` için `s`); `key_char` o tuşla üretilebilecek karakteri tutar (`option-s` için `Some("ß")`, `cmd-s` için `None`). ASCII'ye çevrilemeyen düzenlerde `key` yine ASCII karşılığı olur; asıl yazılan karakter `key_char`'a düşer. Ayrı bir `ime_key` alanı yoktur.
- `KeybindingKeystroke` — kısayol dosyalarında görünen görsel `modifier`/`key` ile eşleşme için kullanılan sarmalayıcı tip.
- `InvalidKeystrokeError` — ayrıştırma hatası. Hatanın `Display` çıktısı, `gpui::KEYSTROKE_PARSE_EXPECTED_MESSAGE: &str` sabitini şablon olarak kullanır (`platform/keystroke.rs:69`); kullanıcı keymap ayrıştırıcısında aynı beklenti cümlesinin gösterilmesi için bu sabite bağlanılır.
- `Modifiers` — `control`, `alt`, `shift`, `platform`, `function` alanları.
- `AsKeystroke` — hem `Keystroke` hem de görsel sarmalayıcılar üzerinden ortak keystroke erişimi sağlayan küçük trait.
- `Capslock { on }` — platform girdi anlık görüntüsünde Caps Lock durumunu taşır.

Tipik kullanım, ayrıştırma, geri biçimleme ve yönlendirme zincirinde görünür:

```rust
let keystroke = Keystroke::parse("cmd-shift-p")?;
let text = keystroke.unparse();
let handled = window.dispatch_keystroke(keystroke, cx);
```

**Modifier yardımcıları.** Sık kullanılan modifier kombinasyonları için yapıcı fonksiyonlar mevcuttur:

- `Modifiers::none()`, `command()`, `windows()`, `super_key()`, `secondary_key()`, `control()`, `alt()`, `shift()`, `function()`, `command_shift()`, `control_shift()`.
- `command()`, `windows()` ve `super_key()` aslında aynı işi yapar: `Modifiers { platform: true, .. }` üretir. Tek bir `platform` alanı, işletim sistemine göre command (macOS), windows (Windows) veya super (Linux) anlamına gelir; bu üç yapıcı fonksiyon yalnızca kavramsal vurgu için farklı isimlerle dışa aktarılır.
- `secondary_key()`, macOS'ta command, Linux ve Windows'ta control üretir; Zed'de platformdan bağımsız kısayol yazılırken çoğu durumda doğru seçim budur.
- `modified()`, `secondary()`, `number_of_modifiers()`, `is_subset_of(&other)`, girdi ayrıştırmada kullanılır.

**IME.** Bileşimsel girdi sırasında özel bayraklar devreye girer:

- `Keystroke::is_ime_in_progress()` — IME bileşim (`composition`) sırasında `true` döner.
- `window.dispatch_keystroke(...)`, test ve simülasyon yolunda `with_simulated_ime()` uygular; doğrudan düşük seviyeli olay üretilirken IME durumunun ayrıca düşünülmesi gerekir.

**`KeybindingKeystroke` yüzeyi.** Görsel ve gerçek keystroke ayrımı bu sarmalayıcı üzerinden yapılır:

- `KeybindingKeystroke::new_with_mapper(inner, use_key_equivalents, keyboard_mapper)` — platform klavye eşleyicisi üzerinden görsel `key` ve `modifier` üretir. `from_keystroke(keystroke)`, platform eşlemesi yapmadan sarar. Windows'ta `new(inner, display_modifiers, display_key)` yapıcısı da vardır; macOS ve Linux derlemelerinde bu yapıcı bulunmaz.
- `inner()`, `modifiers()`, `key()` okuyucuları (`getter`), görsel ile gerçek keystroke ayrımını saklar. Windows'ta `modifiers()` ve `key()` görsel değeri döndürebilir; gerçek GPUI girdisi için `inner()` okunur.
- `set_modifiers(...)`, `set_key(...)`, `remove_key_char()` ve `unparse()`, kısayol düzenleyici veya normalize edici akışında kullanılır. `remove_key_char()` yalnızca `inner.key_char = None` yapar; `key` alanına dokunmaz.

**Kısayol sorguları.** Kullanıcıya gösterilecek kısayol metni ve aktif girdi zinciri için `window` üzerinde yardımcılar mevcuttur:

- `window.bindings_for_action(&Action)` ve `window.keystroke_text_for(&Action)`, kullanıcıya gösterilecek kısayol metni için tercih edilir.
- `cx.all_bindings_for_input(&[Keystroke])` ve `window.possible_bindings_for_input(&[Keystroke])`, çoklu vuruş veya ön ek kısayolu durumlarında kullanılır.
- `window.pending_input_keystrokes()`, henüz tamamlanmamış girdi zincirini verir.

---
