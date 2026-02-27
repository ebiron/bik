# 🚨 BİK Analitik Tracker (v2) — Kritik Hata Raporu

| | |
|---|---|
| **Tarih** | 27 Şubat 2026 |
| **Raporlayan** | HaberPanelim v9 Teknik Ekip |
| **Tracker Versiyon** | v2 (NS01 collector.p.analitik.bik.gov.tr) |
| **Dosya** | [`t-1-kurtalangazetesi-com-0.js`](https://cdn-v2.p.analitik.bik.gov.tr/t-1-kurtalangazetesi-com-0.js) |

---

## Bug #1 — `filterDeepObjects` Fonksiyonu Tüm Fingerprint Verisini Siliyor

| | |
|---|---|
| **Önem Derecesi** | 🔴 **KRİTİK** — Fingerprint tamamen çalışmıyor |
| **Etki Alanı** | TÜM yayıncılar · TÜM tarayıcılar · TÜM cihazlar |

### Sorun Açıklaması

Tracker kodunda, collector'lar (`fonts`, `hardware`, `locales`, `math`, `permissions`, `plugins`) çalıştırıldıktan sonra sonuçlar `filterDeepObjects` adlı bir recursive filtre fonksiyonundan geçiriliyor. Bu fonksiyon **YALNIZCA** nested (iç içe) object'leri tutuyor. Ancak tüm collector çıktılarının leaf (yaprak) değerleri `string`, `number` veya `array` tipinde — **hiçbiri nested object DEĞİL**.

**Sonuç:**

```
Filtre TÜM veriyi siliyor
  → resolvedComponents = {}
  → JSON.stringify({}) = "{}"
  → MurmurHash3("{}") = "8fc360c824b22f7f24b22f7f24b22f7f"
```

> ⚠️ Bu fingerprint değeri **dünya üzerindeki her cihaz, her tarayıcı, her işletim sistemi için aynıdır**. Fingerprint tabanlı tekil ziyaretçi ayrımı **tamamen çalışmamaktadır**.

### Hatalı Kod

Minified koddan açılmış hali:

```javascript
const d = function e(t) {
    const n = {};
    for (const [i, r] of Object.entries(t))
        if ("object" == typeof r && !Array.isArray(r) && null !== r) {
            const t = e(r);
            Object.keys(t).length > 0 && (n[i] = t);
        }
    return n;
}(l);
```

### Neden Çalışmıyor?

Fonksiyon recursive olarak sadece `object` tipindeki ve `Array` olmayan değerleri tutuyor. Collector çıktılarının yapısı:

| Collector | Çıktı Formatı | Leaf Tipi | Sonuç |
|-----------|---------------|-----------|-------|
| `fonts` | `{ "Arial": 245.5, "Georgia": 198.2 }` | `number` | ❌ SİLİNİYOR |
| `hardware` | `{ videocard: {...}, architecture: 128 }` | `string/number` | ❌ SİLİNİYOR |
| `locales` | `{ languages: "tr-TR", timezone: "Europe/..." }` | `string` | ❌ SİLİNİYOR |
| `math` | `{ acos: 1.047..., sin: 0.003... }` | `number` | ❌ SİLİNİYOR |
| `permissions` | `{ camera: "prompt", mic: "denied" }` | `string` | ❌ SİLİNİYOR |
| `plugins` | `{ plugins: ["PDF\|...", "Chrome\|..."] }` | `array` | ❌ SİLİNİYOR |

Filtreleme mantığı:
- `STRING` → `typeof !== "object"` → **SİLİNİYOR**
- `NUMBER` → `typeof !== "object"` → **SİLİNİYOR**
- `ARRAY` → `Array.isArray === true` → **SİLİNİYOR**
- Boş obje → `Object.keys(t).length === 0` → **PARENT'TAN DA SİLİNİYOR**

### Kanıtlar

3 farklı tarayıcıda test edilmiştir:

| Tarayıcı | Fingerprint | resolvedComponents |
|-----------|-------------|--------------------|
| macOS Chrome | `8fc360c824b22f7f24b22f7f24b22f7f` | `{}` |
| macOS Safari | `8fc360c824b22f7f24b22f7f24b22f7f` | `{}` |
| Android Chrome | `8fc360c824b22f7f24b22f7f24b22f7f` | `{}` |

### Önerilen Düzeltme

```diff
- const d = filterDeepObjects(l);
+ const d = l;
```

---

## Bug #2 — Bot Tespiti Safari ve Mobil Tarayıcıları Yanlış İşaretliyor

| | |
|---|---|
| **Önem Derecesi** | 🟠 **YÜKSEK** — Gerçek kullanıcılar bot olarak sayılıyor |
| **Etki Alanı** | Tüm Safari · Tüm Firefox · Tüm mobil tarayıcılar |

### Sorun Açıklaması

Bot tespit fonksiyonundaki `hasUndetectedBehavior` kontrolü 3 koşulun `OR` birleşimini kullanıyor:

```javascript
t.hasUndetectedBehavior =
    void 0 === e.chrome ||           // (A) window.chrome yok mu?
    0 === navigator.plugins.length || // (B) plugin sayısı 0 mı?
    0 === navigator.languages.length  // (C) dil listesi boş mu?
```

### Sorunlu Koşullar

| Koşul | Sorun | Etkilenen Kitle |
|-------|-------|-----------------|
| **(A)** `void 0 === e.chrome` | `window.chrome` sadece Chrome/Chromium'da tanımlı. Safari ve Firefox'ta **tanımsız** → bot olarak işaretleniyor | ~%25-30 (Safari + Firefox) |
| **(B)** `0 === navigator.plugins.length` | Modern mobil tarayıcılarda `navigator.plugins` **boş** dönüyor | ~%60-70 (tüm mobil) |
| **(C)** `0 === navigator.languages.length` | ✅ Doğru kontrol — headless bot'larda dil listesi boş | Sorun yok |

### Etkisi

Bot olarak işaretlenen cihazlarda `hardware` collector'ı **exclude** ediliyor:

```javascript
new p(e ? { exclude: ["hardware"] } : {}).get()
```

Ayrıca sunucu tarafında:
- Safari ziyaretçileri → `is_bot = true` ❌
- Mobil ziyaretçiler → `is_bot = true` ❌
- Gerçek bot oranı istatistikleri **tamamen yanıltıcı**

### Kanıtlar

| Tarayıcı | `hasUndetectedBehavior` | Sebep |
|-----------|------------------------|-------|
| macOS Safari 17.x | `true` ❌ | `window.chrome = undefined` |
| Android Chrome 120 | `true` ❌ | `plugins.length = 0` |
| macOS Chrome 120 | `false` ✅ | Doğru çalışıyor |

### Önerilen Düzeltme

```diff
  t.hasUndetectedBehavior =
-     void 0 === e.chrome ||
-     0 === navigator.plugins.length ||
      0 === navigator.languages.length
```

Alternatif daha güvenilir bot tespit yöntemleri:
- `navigator.webdriver === true` (Selenium, Puppeteer, Playwright)
- `/HeadlessChrome/i.test(navigator.userAgent)`
- `window.outerWidth === 0 && window.outerHeight === 0`

---

## Bug #3 — Zincirleme Etki: `distinct_id` Tüm Kullanıcılar İçin Aynı

| | |
|---|---|
| **Önem Derecesi** | 🟡 **ORTA** — Bug #1'in doğrudan sonucu |
| **Etki Alanı** | Sunucu tarafı (collector backend) |

### Sorun Açıklaması

Collector backend'inde `distinct_id` üretimi:

```
1. İlk tercih  : tracker'dan gelen decoded['id'] (önceki response'tan dönen id)
2. İkinci tercih: decoded['fingerprint']
3. Son çare     : random_bytes(16) — rastgele UUID
```

İlk sayfa görüntülemede `decoded['id']` boş → fingerprint kullanılıyor.
Bug #1 nedeniyle fingerprint **herkesin aynı değeri** → `distinct_id` de herkes için aynı.

**Sonuç:**
- Tekil ziyaretçi sayısı her zaman **1** görünür
- Kullanıcı bazlı analiz yapılamaz
- Retention/cohort analizleri anlamsız

### Önerilen Düzeltme

Bug #1 düzeltildiğinde bu sorun **otomatik olarak çözülür**.
Ek güvenlik olarak backend'de fingerprint collision kontrolü eklenebilir.

---

## Etki Analizi

Bu bug'lar BİK Analitik tracker v2 kullanan **TÜM yayıncıları** etkilemektedir.

### Etkilenen Metrikler

| Metrik | Durum | Açıklama |
|--------|-------|----------|
| Fingerprint tabanlı tekil ziyaretçi | ❌ | Tamamen yanlış — herkes aynı |
| `distinct_id` tabanlı kullanıcı ayrımı | ❌ | Herkes aynı id |
| Bot/gerçek kullanıcı oranı | ❌ | Safari + mobil = bot |
| Cihaz bazlı fingerprint dağılımı | ❌ | Tek değer |
| Hardware collector verisi | ❌ | Bot'larda exclude ediliyor |

### Etkilenmeyen Metrikler

| Metrik | Durum |
|--------|-------|
| Sayfa görüntüleme sayısı | ✅ |
| Active seconds (süre takibi) | ✅ |
| Scroll/click event'leri | ✅ |
| URL/referrer verileri | ✅ |
| `session_id` | ✅ |

---

## Doğrulama Adımları

### Yöntem 1 — Network İnceleme

1. Herhangi bir yayıncı sitesine **farklı tarayıcılardan** girin
2. `DevTools > Network` → collector'a giden request'in payload'ında `d` alanını XOR decode edin (key = hostname)
3. Her tarayıcıda fingerprint değerinin aynı olduğunu doğrulayın: `8fc360c824b22f7f24b22f7f24b22f7f`

### Yöntem 2 — Console Testi

Aşağıdaki kodu herhangi bir tarayıcı console'unda çalıştırın:

```javascript
const filter = function e(t) {
    const n = {};
    for (const [i, r] of Object.entries(t))
        if ("object" == typeof r && !Array.isArray(r) && null !== r) {
            const t = e(r);
            Object.keys(t).length > 0 && (n[i] = t);
        }
    return n;
};

// Test 1: fonts collector çıktısı
console.log(filter({ "Arial": 245.5, "Georgia": 198.2 }));
// Sonuç: {} (boş obje)

// Test 2: permissions collector çıktısı
console.log(filter({ camera: "prompt", microphone: "denied" }));
// Sonuç: {} (boş obje)

// Test 3: hardware collector çıktısı
console.log(filter({
    videocard: { vendor: "Google Inc.", renderer: "ANGLE..." },
    architecture: 128,
    deviceMemory: "8",
    jsHeapSizeLimit: 4294705152
}));
// Sonuç: {} (boş obje — videocard nested ama leaf'leri string → silinir)
```

---

## İletişim

Bu rapor teknik detaylarla birlikte hazırlanmıştır.

📧 **mtalmac@gmail.com**
