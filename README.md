

**Proje Adı:** Film Öneri Sitesi (Flask + TMDB)
**Geliştirici:** Hakan Polat
**Teknolojiler:** Python 3.11+, Flask, Jinja2, Tailwind (CDN), Vanilla JS, TMDB API
**Repo/Dizin:** `BitirmeProjesiYeni/`
**Tarih:** 2025-10-20


---

## 1) Proje Özeti

Kullanıcılar popüler/yeni filmleri görebilir, tür/yıl/puan filtreleriyle keşif yapabilir, film detaylarına erişebilir ve arama kutusunda canlı öneri (suggestion) alabilir. Veri kaynağı **TMDB**’nin herkese açık REST API’sidir.

**Başlıca özellikler**

* Ana sayfada *Yeni Filmler* ve *Trendler*.
* Kenar çubuğunda *Film Robotu* (genre, yıl, sort, puan filtreleri) + sayfalama.
* Film detayında poster, özet, türler, puan, süre ve YouTube fragman modalı.
* Arama sayfasında sonuç listesi + sayfalama.
* Arama kutusunda canlı öneri (ilk 5 sonuç).



---

## 2) Dizin Yapısı

```
BitirmeProjesiYeni/
├─ .venv/                    # Sanal ortam 
├─ static/
│  └─ main.js               # Ön yüz davranışları (discover, modal, suggestion)
├─ templates/
│  ├─ base.html             # Ortak şablon (header, arama, modal, script)
│  ├─ index.html            # Ana sayfa (Yeni, Trend ve Film Robotu)
│  ├─ detail.html           # Film detay sayfası (fragman modalı dahil)
│  ├─ login.html            # (şablon hazır) Giriş formu
│  ├─ register.html         # (şablon hazır) Kayıt formu
│  └─ search.html           # Arama sonuçları
├─ .env                     # Ortam değişkenleri (TMDB_API_KEY vb.)
├─ .gitignore
├─ app.py                   # Flask uygulaması ve HTTP uçları
├─ README.md                # Kısa proje özeti / talimatlar (isteğe bağlı)
├─ requirements.txt         # Bağımlılıklar (sürüm sabitleme)
├─ sentiment.py             # (hazır) Duygu analizi için yer tutucu
└─ site.db                  # (hazır) SQLite veritabanı (şu an aktif kullanılmıyor)
```

---

## 3) Bağımlılıklar ve Sürümler

`requirements.txt` içeriği:

```
flask==3.0.3
requests==2.32.3
Werkzeug==3.0.4
transformers==4.44.2
torch>=2.2
```

**Notlar**

* Projenin çalışan çekirdeği **Flask + requests**’tir. `transformers/torch` şu sürümde aktif kullanılmıyor; ileride duygu analizi entegrasyonu için eklidir.
* Tailwind CSS CDN ile `base.html` içinde yüklenir (ek kurulum gerekmez).

---

## 4) Kurulum ve Çalıştırma

### 4.1. Ön Koşullar

* Python 3.11+
* TMDB API anahtarı: [https://www.themoviedb.org/](https://www.themoviedb.org/) (ücretsiz hesap)

### 4.2. Adımlar

1. Sanal ortam:

   ```bash
   python -m venv .venv
   source .venv/bin/activate   # Windows: .venv\Scripts\activate
   ```
2. Bağımlılıklar:

   ```bash
   pip install -r requirements.txt
   ```
3. Ortam değişkeni (`.env` dosyası):

   ```dotenv
   TMDB_API_KEY=buraya_kendi_anahtariniz
   ```
4. Çalıştırma (geliştirme):

   ```bash
   python app.py
   # Uygulama: http://127.0.0.1:5000
   ```

> **Hata ayıklama:** `AssertionError: TMDB_API_KEY ortam değişkenini` hatası alırsanız `.env` dosyanızı ve anahtarın doğruluğunu kontrol edin.

---

## 5) Yapılandırma

* `TMDB_API_KEY` (zorunlu): TMDB API çağrıları için kullanılır.
* `FLASK_ENV=development` (opsiyonel): Geliştirme modunda yeniden yükleme.
* `PORT`/`HOST` (ops.): Üretimde ters proxy arkasında farklı ayarlanabilir.

---

## 6) Mimari Genel Bakış

**Katmanlar**

* **Sunum (Templates + static/main.js):** Tailwind ile responsive arayüz, vanilla JS ile etkileşimler.
* **Uygulama (Flask/app.py):** HTTP uçları, şablon render, basit cache (genres).
* **Harici Servis (TMDB):** Tüm film verileri REST API üzerinden sağlanır.

**İstek Akışı (Örnek: Discover):**

1. Kullanıcı filtreleri seçer → `main.js` `GET /api/discover?genre_id=..&year=..&sort_by=..&vote_gte=..&page=..` çağırır.
2. Flask `tmdb_get('/discover/movie', params)` ile TMDB’yi çağırır.
3. JSON sonuçları aynen döner → `main.js` kartları üretip DOM’a basar.

**Önbellekleme:**

* Tür listesi (`/genre/movie/list`) `@lru_cache(maxsize=512)` ile süreç içi cache edilir.

---

## 7) Backend (Flask) Detayları

### 7.1. Yardımcılar

```py
TMDB_BASE = "https://api.themoviedb.org/3"

def tmdb_get(path, params=None):
    params = params or {}
    params["api_key"] = TMDB_KEY
    params["language"] = params.get("language", "tr-TR")
    r = requests.get(f"{TMDB_BASE}{path}", params=params, timeout=15)
    r.raise_for_status()
    return r.json()
```

* Tüm TMDB çağrıları ortak noktadan geçer; dil varsayılanı **tr-TR**.
* `raise_for_status()` ile HTTP hataları erken patlatılır (Flask, 500 dönebilir; prod’da yakalayıp kullanıcı dostu mesaj önerilir).

### 7.2. Uçlar (Routes)

* `GET /` (**home**)

  * **Input:** `page` (vars.: 1)
  * **TMDB:** `/movie/now_playing`, `/trending/movie/week`, `/genre/movie/list`
  * **View:** `index.html` (yeni, trend, film robotu)

* `GET /movie/<int:movie_id>` (**movie_detail**)

  * **TMDB:** `/movie/{id}?append_to_response=videos,credits,release_dates`
  * YouTube video seçimi: TR > EN > diğer
  * **View:** `detail.html` (poster, özet, rozetler, fragman modal)

* `GET /search` (**search page**)

  * **Input:** `q` (zorunlu), `page`
  * **TMDB:** `/search/movie` (adult:false)
  * **View:** `search.html` (grid + sayfalama)

* `GET /api/search_suggest` (**canlı öneri**)

  * **Input:** `q` (zorunlu)
  * **Çıkış:** İlk 5 sonuç (`[{id,title,poster_path,vote_average,release_date},..]`)

* `GET /api/discover` (**film robotu API**)

  * **Input:** `genre_id, year, sort_by=popularity.desc|vote_average.desc|release_date.desc, vote_gte, page`
  * **Çıkış:** TMDB `/discover/movie` cevabı aynen döner (pagination alanları dahil)

---

## 8) Frontend — Şablonlar

* **`base.html`**: Üst menü, arama formu (canlı öneri için yer), Trailer modal kapları ve `main.js` dahil edilir.
* **`index.html`**: Sol ana kolon *Yeni Filmler*; sağda *Trend Filmler* ve *Film Robotu*.
* **`detail.html`**: Poster, rozetler (tür, süre, puan), özet, *Fragmanı izle* butonu ve *Benzer Filmler*. (Yorum/istatistik blokları şablonda yer tutucudur.)
* **`search.html`**: Arama sonuç grid’i ve sayfa düğmeleri.
* **`login.html`/`register.html`**: Görsel şablon hazır; backend uçları **bu sürümde yok**.

---

## 9) Frontend — `static/main.js`

* **`fetchJson(url)`**: Basit GET yardımcı fonksiyon.
* **`movieCard(m)`**: Poster, başlık, yıl, puan rozeti içeren kart HTML’i üretir.
* **Discover akışı**: Filtre seç → `GET /api/discover` → sonuçları ana alana bas; *Yeni Filmler* bloğunu gizle; sayfalama ve “Kapat” ile geri dönüş.
* **Trailer modal**: `#watchTrailer` butonu tıklandığında YouTube `iframe`’i autoplay ile açılır; modal dışına tıklayınca kapanır.
* **Canlı arama önerisi**: Input’a yazdıkça `/api/search_suggest` çağrılır; sonuçlar aşağı açılır listede gösterilir; dışarı tıkla → kapanır.

---

## 10) Veritabanı ve Duygu Analizi Durumu

* **PostgresSQL `site.db`** mevcut; fakat bu sürümde Flask kodunda bağ yok.
* **`sentiment.py`** ve `transformers/torch` bağımlılıkları ileride *yorumlara duygu analizi* eklemek için hazırlanmış.
* Şablonda `comments` ve `stats` alanları (yer tutucu) görülebilir; şu an veri gelmez.



---

## 11) Güvenlik, Performans, Uyumluluk

* **API Anahtarı** `.env` içinde tutulmalı; depo dışına çıkarılmalı.
* **Rate limit**: TMDB’nin oran limitlerine uyun; yoğun isteklerde local cache veya server-side sayfalama kullanın.
* **`requests` timeout**: 15s; üretimde daha düşük değer + tekrar deneme (retry) önerilir.
* **Giriş/Şifre** (gelecek): `werkzeug.security` ile `generate_password_hash`/`check_password_hash` kullanın.
* **XSS/Injection**: Jinja autoescape açık; URL parametreleri server tarafında beyaz listeye göre doğrulanmalı (`sort_by` gibi).

---

## 12) Manuel Test Senaryoları (Örnek)

1. **Ana sayfa yüklenir** → Yeni/Trend listeleri görünür.
2. **Discover**: Tür=Action, Yıl=2022, Puan=7+, Sırala=Popularity → sonuçlar gelir, sayfalama çalışır.
3. **Arama**: “Inception” → sonuçlar + sayfa geçişleri.
4. **Canlı öneri**: Arama kutusuna “batm” → 5 öneri listelenir.
5. **Detay**: Bir film → Fragmanı izle → modal aç/kapat.
6. **Hata**: İnterneti kes → kullanıcıya dost hata (gelecek geliştirme: global error banner).

---

## 13) Dağıtım (Prod) Önerisi

* **Uygulama Sunucusu**: `gunicorn 'app:app' --workers 2 --timeout 30`
* **Reverse Proxy**: Nginx (gzip, caching headers, HTTPS)
* **Env**: `.env` prod sunucuda; `debug=False`
* **Statik Dosyalar**: Nginx üzerinden servis etme; uzun `Cache-Control`
* **Günlükleme**: Gunicorn erişim/hata logları + rotasyon
* **Docker (ops.)**: Küçük bir `Dockerfile` ile imaj üretimi

---

## 14) Bilinen Eksikler & Yol Haritası

* [ ] **Kimlik doğrulama** (register/login/logout, session yönetimi)
* [ ] **Yorumlar** (SQLite/PostgreSQL; CRUD; spoiler işaretleme)
* [ ] **Duygu analizi** (`sentiment.py` → Transformers ile POS/NEG/NEU etiketleme; istatistiklerin hesaplanması)
* [ ] **Hata yönetimi** (TMDB hata/limit durumlarında kullanıcı dost mesajlar)
* [ ] **Cache** (Tür listesi dışında; popüler/trend sonuçları için 5–15 dk in-memory/Redis)
* [ ] **Birim/Entegrasyon testleri** (pytest + requests-mock)
* [ ] **CI/CD** (GitHub Actions; lint/test; prod’a otomatik dağıtım)

---

## 15) SSS (Kısa)

* **Neden `transformers/torch` var?** Gelecekte yorum duygu analizi için. Şu an çekirdek çalışması için gerekmez.
* **Dil neden `tr-TR`?** TMDB isteklerinde varsayılan Türkçe içerik hedeflenmiştir. Gerektiğinde İngilizceye failover yapılır (fragman seçiminde olduğu gibi).
* **Veritabanı gerekli mi?** Bu sürümde hayır. Yorumlar/oturumlar eklenecekse gereklidir.

---




