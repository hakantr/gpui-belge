# Türkçeleştirme Sözlüğü ve Disiplini

Bu dosya `src/gpui_kullanimi/` altındaki 18 bölümün Türkçeleştirme turunda kurulan glossary'yi, karar gerekçelerini ve sıkça başvurulan eşleşmeleri tek noktada toplar. Yeni bölümler eklendiğinde veya mevcut metin yeniden gözden geçirildiğinde aynı terimlerin aynı şekilde çevrilmesi için bir referanstır.

## Yapılan İş

- Bölüm 01 kullanıcı tarafından şablon olarak kuruldu; kalan 17 bölüm (02–18) aynı disiplinle denetlendi ve düzeltildi.
- Her bölümde **prose seviyesindeki İngilizce kelimeler** çevrildi, **API/tip/metod/enum isimleri ve kod örnekleri** olduğu gibi korundu.
- Kod yorumlarındaki prose ifadeleri de çevrildi (örn. `// prepaint: hitbox, yerleşim zamanlı durum`).
- Diff **simetrik** tutuldu (insertion sayısı = deletion sayısı) — böylece satır numaraları ve dosya yapısı bozulmadı.
- Çapraz bölüm referansları gerçek başlıklarla eşleştirildi (örn. bölüm 15'teki "Pencere Sınırlarının Saklanması ve Geri Yüklenmesi").
- Anlatım pasif veya 3. tekil tutuldu; sadeleştirme yerine derinleştirme tercih edildi.

## Çekirdek Prensipler

1. **Çeviri sınırı:** Prose'da geçen bir kelime İngilizce ise çevrilir; aynı kelime kod, tip adı, enum varyantı veya method imzasında ise dokunulmaz.
2. **Parantezli orijinal:** Türkçe karşılığı az bilinen terimler için ilk geçtiği yerde `kalıtılan (\`inherited\`)`, `gecikmeli (\`delayed\`)` gibi parantezle İngilizce orijinal verilir.
3. **Korunan teknik terimler:** Rust visibility (`pub`, `private`), `import`, `use`, `prelude`, `namespace`, `crate`, `trait`, `struct`, `enum` gibi dil kavramları çevrilmez; yerleşik UI/grafik terimleri (hover, tooltip, popover, modal, picker, dispatch, scrollbar, hitbox, paint, prepaint, viewport, MRU, fenced block, hit-test, debounce, async, executor, builder, factory, registry) korunur.
4. **Çapraz referans bütünlüğü:** Başka bölüme yapılan atıflar gerçek başlıkla eşleşmelidir; yoksa hedef başlık güncellenmelidir.

## Sözlük

### Element ve UI Hiyerarşisi
| İngilizce | Türkçe |
| --- | --- |
| child | alt öğe |
| parent | üst öğe |
| root | kök |
| container | kapsayıcı |
| wrapper | sarmalayıcı |
| nested | iç içe |
| sub-tree, child tree | alt ağaç |

### Çizim, Yerleşim ve Kare
| İngilizce | Türkçe |
| --- | --- |
| render, rendering, draw | çizim |
| frame (rendering) | kare / ekran karesi |
| layout | yerleşim |
| draw order | çizim sırası |
| rerender, redraw | yeniden çizim |
| invalidate | geçersizleştir- |
| notify | bildir- |
| paint, prepaint | korundu (faz adları) |

### Stil, Geometri ve Renk
| İngilizce | Türkçe |
| --- | --- |
| decoration | dekorasyon / süsleme |
| shadow | gölge |
| overflow | taşma |
| position | konum |
| inherent | doğrudan |
| mode | kip |
| override | üzerine yazma |
| default | varsayılan |
| layer | katman |
| clip, clipping | kırpma |
| mask | maske |
| transform | dönüşüm |
| border radius | kenarlık yarıçapı |
| pre-multiplied alpha | önceden çarpılmış alfa |
| slice | dilim |
| golden ratio | altın oran |
| manuel | elle |
| device pixel | cihaz piksel |
| border (prose, generic) | kenarlık |
| border (API method) | korundu |
| bounds, window bounds | sınırlar, pencere sınırları |
| bounds restore | sınır geri yükleme |
| anchor (konumlandırma) | anchor / referans noktası |
| anchor bounds | anchor sınırları |
| menu bounds | menü sınırları |

### Etkileşim ve Olaylar
| İngilizce | Türkçe |
| --- | --- |
| click | tıklama |
| mouse | fare |
| pointer | işaretçi |
| drag | sürükleme |
| drop | bırakma |
| drag/drop | sürükle-bırak |
| focus | odak |
| blur | odak kaybı |
| focus ring | odak halkası |
| focus tree | odak ağacı |
| focus traversal | odak gezinmesi |
| listener | dinleyici |
| callback | geri çağrı |
| emit | yay- |
| handler, event handler | (olay) işleyici |
| hover | korundu |
| dispatch | korundu |

### Durum, Bellek ve Yaşam Döngüsü
| İngilizce | Türkçe |
| --- | --- |
| state | durum |
| leak | sızıntı |
| ownership | sahiplik |
| subscription | abonelik |
| observer | gözlemci |
| retain | tutma |
| detach | ayırma |
| persistence | kalıcılık |
| persist | kalıcılaştır- |
| drop (elden çıkma) | elden çıkma |

### API ve Tip Yapıları
| İngilizce | Türkçe |
| --- | --- |
| constructor | yapıcı |
| accessor | erişim metodu |
| implementation, implementasyon | uygulama |
| implement | uygula- |
| blanket impl | blanket olarak uygula- |
| type erasure, type-erased | tip silme, tipi silinmiş |
| type alias | tip takma adı |
| re-export | yeniden dışa aktar- |
| export | dışa aktarım |
| public | genel |
| private | korundu (Rust visibility) |
| inventory | envanter |
| feature gate | özellik kapısı |
| reflection, reflectable | yansıma, yansıtılabilir |
| marker (trait) | işaretleyici |
| signature | imza |
| source location | kaynak konumu |
| builder, factory, registry | korundu |

### Asenkron ve İşlem
| İngilizce | Türkçe |
| --- | --- |
| foreground | ön plan |
| background | arka plan |
| background task | arka plan görevi |
| pending | bekleyen |
| effect cycle | etki döngüsü |
| executor, timer, debounce, async | korundu |
| sync | eşzamanlı |
| event loop | olay döngüsü |
| process | işlem |
| instance | örnek |
| channel | kanal |
| unbounded | sınırsız |
| runtime | çalışma zamanı |
| foreground/background executor | ön plan/arka plan executor'ı |

### Veri, Önbellek ve Format
| İngilizce | Türkçe |
| --- | --- |
| cache | önbellek |
| eviction | tahliye |
| decode | kodu çöz- |
| encoded | kodlanmış |
| parse | ayrıştır- |
| serialize, serialization | serileştir-, serileştirme |
| deserialize | seriden çıkar- |
| canonicalization | kanonikleştirme |
| clipboard | pano |
| format | biçim |

### Görsel, Asset ve Yükleme
| İngilizce | Türkçe |
| --- | --- |
| image | görsel |
| loader | yükleyici |
| loading | yükleme |
| fallback | yedek |
| provider | sağlayıcı |
| bundled | paketlenmiş |
| standalone | bağımsız |
| watcher | izleyici |
| delayed-fallback | gecikmeli yedek |
| asset cache | asset önbelleği |
| spinner | yükleme göstergesi |
| loading state | yükleme durumu |

### Metin, Font ve Tipografi
| İngilizce | Türkçe |
| --- | --- |
| text | metin |
| text rendering | metin çizim |
| text system | metin sistemi |
| font family | font ailesi |
| font size | font boyutu |
| UI font | UI fontu |
| buffer font | buffer fontu |
| line height | satır yüksekliği |
| whitespace | boşluk |
| align | hizalama |
| glyph | glif |
| inline code | satır içi kod |
| code block | kod bloğu |
| ellipsis | üç nokta |
| truncate, truncation | kırp-, kırpma |
| streaming text | akışlı metin |
| lazy content | tembel içerik |
| underline (paint) | alt çizgi |
| density, UI density | yoğunluk, UI yoğunluğu |

### Bileşen ve UI Yüzeyleri
| İngilizce | Türkçe |
| --- | --- |
| component | bileşen |
| button | buton |
| label | etiket |
| group | grup |
| header | başlık |
| separator | ayırıcı |
| preview | önizleme |
| link | bağlantı |
| hyperlink | köprü |
| settings | ayarlar |
| settings UI | Ayarlar UI'sı |
| theme | tema |
| syntax theme | sözdizimi teması |
| extension trait | uzantı trait |
| cursor position | imleç konumu |
| cursor (style) | imleç |
| selection | seçim |
| pinned | sabitlenmiş |
| controls | kontroller |
| hardcode | sabit kodlanmış |
| custom | özel |
| reusable | tekrar kullanılabilir |
| semantic | anlamsal |
| feedback | geri bildirim |
| feature, feature flag | özellik, özellik bayrağı |
| completion | tamamlama |
| completion menu | tamamlama menüsü |
| icon theme | ikon teması |
| start icon | başlangıç ikonu |
| application menu | application menüsü |
| anchor element | anchor element |

### Pencere, Platform ve Süsleme
| İngilizce | Türkçe |
| --- | --- |
| native | yerel |
| titlebar | başlık çubuğu |
| dialog | diyalog |
| client-side decoration | istemci tarafı süslemesi |
| client decoration | istemci süslemesi |
| blur, blurred | bulanıklık, bulanık |
| transparent | saydam |
| overlay | kaplama |
| cross-platform | çapraz platform |
| screen capture | ekran yakalama |
| headless | başsız |
| headless renderer | başsız çizim aracı |
| software emulation | yazılım öykünmesi |
| diagnostic | tanı |
| harness | koşum |
| debug | hata ayıklama |
| debug assert | hata ayıklama assert'i |
| debug bounds | hata ayıklama sınırları |
| popover, tooltip, modal, picker, scrollbar, hitbox, viewport, anchor | korundu |

### Komut Paleti, Picker ve Menü
| İngilizce | Türkçe |
| --- | --- |
| confirm | onay |
| secondary confirm | ikincil onay |
| hit count | hit sayısı |
| prefix | önek |
| Zed link | Zed bağlantısı |
| keymap editor | keymap editörü |
| binding | bağlama |
| context menu | bağlam menüsü |
| right-click menu | korundu (genel kullanım) |
| literal input | birebir girdi |
| match (sonuç) | eşleşme |

### Workspace, Sidebar ve Pane
| İngilizce | Türkçe |
| --- | --- |
| editor (prose) | editör |
| editor benzeri | editör benzeri |
| pinned tab | sabitlenmiş tab |
| navigation, navigation history | gezinme, gezinme geçmişi |
| nav entry | giriş |
| drop indicator | bırakma göstergesi |
| draft | taslak |
| empty-state | boş durum |
| root path | kök yolu |
| close davranışı | kapatma davranışı |
| terminal entry | terminal girişi |
| location | konum |
| recent (proje, liste) | yakın zamanlı |
| unload | boşalt- |
| Username/password | Kullanıcı adı/parola |
| URL-decode | URL kodu çözül- |

## Korunan (Çevrilmeyen) Listeler

Aşağıdaki terim aileleri prose'da bile İngilizce bırakıldı — ya teknik standart isim, ya da Türkçeleştirildiğinde okunabilirliği düşürdüğü için:

- **Rust dil/derleme:** `pub`, `private`, `crate`, `trait`, `struct`, `enum`, `impl`, `use`, `import`, `prelude`, `namespace`, `derive`, `match`, `let`, `mut`, `ref`, `panic`, `must_use`, `#[cfg(...)]`, `#[track_caller]`, `#[doc(hidden)]`, lifetime sözdizimi.
- **GPUI/Zed teknik:** `paint`, `prepaint`, `hitbox`, `popover`, `tooltip`, `modal`, `picker`, `viewport`, `anchor`, `anchored`, `subpixel`, `atlas`, `sprite`, `taffy`, `lyon`, `scheduler`, `executor`, `dispatch`, `dispatcher`, `MRU`, `fenced block`, `hit-test`, `debounce`, `throttle`, `defer`, `tiling`, `keystroke`, `keybinding`, `keymap`.
- **Standart/protokol:** `IPv6`, `JSON`, `JSON schema`, `SQLite`, `RWGB`, `RGBA`, `BGRA`, `HSLA`, `SVG`, `GIF`, `WebP`, `PNG`, `JPEG`, `LSP`, `Vim`, `DWM`, `Wayland`, `X11`, `NSVisualEffectView`, `AXDocument`, `compositor`, `MultiWorkspace`.
- **Unicode/tipografi standartları:** `grapheme cluster`, `combining mark`, `ZWJ`, `URL-decode`.
- **API method/struct/enum isimleri:** Her zaman korundu. Örnekler: `Application::with_platform`, `WindowOptions`, `WindowKind::Dialog`, `WindowControlArea::{Drag, Close, Max, Min}`, `WindowBackgroundAppearance::{Transparent, Blurred}`, `ResizeEdge`, `CommandPaletteFilter`, `PickerDelegate`, `SettingsStore`, `ThemeRegistry`, vb.

## Bölüm Başlıkları Özel Kararları

Çapraz referansların kırılmaması için bölüm başlıklarındaki tipik kalıplar:

| İngilizce kalıp | Türkçe kalıp |
| --- | --- |
| Runtime | Çalışma Zamanı |
| Snapshot | Anlık Görüntü |
| Layout | Yerleşim |
| Frame | Kare |
| Re-export | Yeniden Dışa Aktarım |
| Style Extension Trait'leri | Stil Uzantı Trait'leri |
| Settings ve Theme | Ayarları ve Tema |
| Context Menu | Bağlam Menüsü |
| Client-side | İstemci Tarafı |
| Completion menu | Tamamlama menüsü |
| Constants ve Type Aliases | Constants ve Tip Takma Adları |
| Layer Shell | korundu |

## Tekrar Eden Asimetri Çağrıları

Bazı kavramlar hem prose hem API yüzeyinde geçer; bu yüzden kararlar bilinçli olarak farklılaştırıldı:

- `View` (capitalized, kavram) — prose'da korundu; `view` (lowercase) genelde korundu, bazı yerlerde "view'lar arasında" gibi çekim alarak Türkçeleştirildi.
- `border` — `Style::border(...)` API'sinde korunur; prose'da "kenarlık" tercih edildi.
- `frame` — animasyon ve çizim bağlamında "kare"; `RenderImage` frame'leri için yine "kare"; `request_animation_frame` gibi API adlarında korundu.
- `cache` — prose'da "önbellek"; ancak `image_cache`, `RetainAllImageCache` gibi API isimlerinde korundu.
- `bounds` — `Bounds`, `window.bounds()`, `layout_bounds` gibi API/tip/metot adlarında korunur; prose'da "sınırlar" veya bağlama göre "pencere sınırları" tercih edilir.
- `anchor` — `Anchor` enum'u, `anchored()` elementi ve `.anchor(...)` API'sinde korunur; prose'da "anchor element" kalıbı korunabilir, ancak ölçüm sonucunda "anchor sınırları" gibi Türkçe çekim kullanılır.
- `completion` — `Completion`, `CompletionsMenu` ve alan adlarında korunur; prose'da "tamamlama", "tamamlama menüsü" kullanılır.
- `decoration` — pencere bağlamında "süsleme"; metin/yazı bağlamında "dekorasyon" (CSS terimi gibi).
- `paint` (faz adı) — fluent zinciri ve faz tartışmasında korundu; "çizim" yalnız prose render bağlamı için kullanıldı.

## Diff Disiplini

- Her bölümün toplam diff'i **simetrik** tutuldu: `git diff --stat` çıktısında insertion ve deletion sayıları eşit.
- Bunun pratik sonucu: dosyaların satır sayısı değişmedi; başka bölümlerdeki line referansları (örn. `text_system.rs:45,48`) ve içeride sayılan satır pozisyonları sabit kaldı.
- Bir satır çok kısaldığında bilgi düşmesin diye **derinleştirildi** (`feedback_rehber_derinlik` memory disiplini).

## Bu Dosyayı Güncelleme

Yeni bir bölüm eklenir veya mevcut bir bölümde yeni bir terim çevrilirse:

1. Burada uygun grupta yeni satır eklenir.
2. Eğer yeni bir asimetri kararı verildiyse "Tekrar Eden Asimetri Çağrıları" bölümüne not eklenir.
3. Korunan listede yeni bir terim eklenecekse gerekçesi bir cümleyle yazılır.
