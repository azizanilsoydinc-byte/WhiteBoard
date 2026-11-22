# Real-Time Collaborative Whiteboard

Excalidraw / Figma estetiğinde, gerçek zamanlı çok kullanıcılı, sonsuz tuvalli bir beyaz tahta. Framework yok: modüler Vanilla JS + HTML5 Canvas, arkada Node.js + Express + Socket.io, kalıcılık için MongoDB (yoksa otomatik RAM fallback).

## Özellikler

- **Vektör nesne modeli:** her çizim `id, type, x, y, width, height, points, color, strokeWidth, text, zIndex` alanlı bir JSON nesnesidir; seçilir, taşınır, boyutlandırılır, silinir.
- **Araçlar:** Seç/Taşı, Kaydır (el), Kalem (quadratic smoothing), Dikdörtgen, Elips, Çizgi, Ok, Yapışkan Not (çift tıkla düzenle), Metin, Nesne Silgisi.
- **Sonsuz tuval:** imleç odaklı zoom (fare tekerleği), `Space + sürükle` veya orta fare tuşu ile pan.
- **İşbirliği:** davet linkli odalar, canlı imleçler, avatarlar, **Follow Mode** (avatara tıkla → o kişinin ekranını takip et).
- **Kullanıcıya özel Undo/Redo:** `Ctrl+Z` / `Ctrl+Y`, yalnızca kendi işlemlerini geri alır.
- **Tema:** açık/koyu mod; tuval arka planı ve ızgara temaya uyar. "Otomatik" renk temaya göre siyah/beyaz olur.
- **Export:** yüksek çözünürlüklü PNG ve JSON yedeği.
- **Otomatik kayıt:** son değişiklikten 2 sn sonra (debounce) MongoDB'ye yazılır.

## Kurulum

```bash
npm install
cp .env.example .env     # Windows: copy .env.example .env
npm run dev              # veya: npm start
```

Tarayıcıda `http://localhost:3000` adresini aç. Aynı linki (veya "Davet linki" butonunu) ikinci bir sekmede/cihazda açarak işbirliğini dene.

`.env` içindeki `MONGODB_URI` boşsa ya da MongoDB'ye bağlanılamazsa uygulama çökmez; veriler sunucu çalıştığı sürece RAM'de tutulur.

> Gereksinim: Node.js 18+. Arayüz Tailwind ve Lucide'i CDN'den yükler, internet gerekir.

## Klavye Kısayolları

| Tuş | İşlev | Tuş | İşlev |
|---|---|---|---|
| V | Seç | N | Yapışkan not |
| H / Space | Kaydır | T | Metin |
| P | Kalem | E | Silgi |
| R / O | Dikdörtgen / Elips | `[` / `]` | Arkaya / öne gönder |
| L / A | Çizgi / Ok | Delete | Seçiliyi sil |
| Ctrl+Z / Ctrl+Y | Geri al / Yinele | Esc | Seçimi bırak |

## Mimari Kararlar

**Çift katmanlı canvas.** `static-canvas` yalnızca nesne listesi, kamera veya tema değiştiğinde yeniden çizilir (görünüm dışındaki nesneler atlanır). `interactive-canvas` aktif fırça darbesini, seçim kutusunu, silgi vurgusunu ve uzak imleçleri `requestAnimationFrame` döngüsüyle çizer.

**Kamera matematiği.** `screen = world * zoom + pan`. Zoom'da imlecin altındaki dünya noktası sabit tutulur: `pan = screen - world * newZoom`.

**Delta sync.** Ağda yalnızca `object:add`, `object:update {id, changes}`, `object:delete {id}` dolaşır. Taşıma/boyutlandırma sırasında güncellemeler ~40 ms'de bir gönderilir, bırakma anında son durum garanti gönderilir. `cursor:move` ve `view:update` throttle edilir; imleçler `volatile` yayınlanır.

**Command Pattern.** Her kullanıcının kendi geri/ileri yığını vardır. Komut, ters çevrilebilir op listesidir (`add`, `delete`, `update{before, after}`). Uzak kullanıcıların işlemleri yığına girmez; geri alma normal `object:*` olayları olarak yayınlanır, bu yüzden başkasının çizimi bozulmaz.

**Kalıcılık.** Sunucu, oda başına nesneleri bellekte tutar (`Map`). Her değişiklikte 2 sn'lik debounce zamanlayıcısı yenilenir; oda boşalınca anında kaydedilir ve bellekten atılır.

## Socket Protokolü

| Olay | Yön | Yük |
|---|---|---|
| `room:join` | C → S | `{roomId, name}` (ack ile `{you, users, objects}`) |
| `user:joined` / `user:left` | S → C | kullanıcı / id |
| `object:add` | ↔ | nesne |
| `object:update` | ↔ | `{id, changes}` |
| `object:delete` | ↔ | `{id}` |
| `cursor:move` | ↔ | `{x, y}` (dünya koordinatı) |
| `view:update` | ↔ | `{cx, cy, zoom}` (görünüm merkezi, Follow Mode için) |

## Klasör Yapısı

```
server/
  index.js            Express + Socket.io + DB bağlantısı
  socketHandler.js    oda yönetimi, delta sync, debounced auto-save
  models/Board.js     Mongoose modeli + RAM fallback
client/
  index.html
  css/style.css
  js/main.js            modülleri bağlar, ağ <-> motor köprüsü, Follow Mode
  js/canvasEngine.js    kamera, araçlar, hit-test, çift katmanlı render, export
  js/socketClient.js    Socket.io sarmalayıcı + throttle
  js/historyManager.js  Command Pattern undo/redo
  js/uiController.js    araç çubuğu, paneller, kısayollar, avatarlar, tema
```

## Yol Haritası

- Canlı (tamamlanmamış) fırça darbelerinin diğer kullanıcılara akıtılması
- Çoklu seçim ve gruplama
- JSON içe aktarma
- Oda şifresi / salt-okunur davet linki
