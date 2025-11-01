

---

# **Proje Adı:** Film Öneri Sitesi (Flask + TMDB)

**Geliştirici:** Hakan Polat
**Teknolojiler:** Python 3.11+, Flask, Jinja2, Tailwind (CDN), Vanilla JS, TMDB API, PostgreSQL (psycopg3), python-dotenv


---

## 1) Proje Özeti (güncel)

Kullanıcılar **popüler/yeni** filmleri görür, **tür/yıl/puan** filtreleriyle keşif yapar, film detaylarını inceler, **YouTube fragmanını** izler. **Arama kutusunda canlı öneri** (autocomplete) açılır panel olarak gelir.
Kullanıcı **girişi** ile şu sosyal özellikler devreye girer:

* **Favoriler:** Detay sayfasından tek tıkla ekle/çıkar, “Favoriler” sayfası.
* **Like / Dislike:** Detay sayfasında beğendim/beğenmedim (toggle; sayacı var).
* **Yorum + Duygu Analizi:** (altyapı hazır; `sentiment.py` ile POS/NEG/NEU etiketleniyor).

**Veri kaynağı:** TMDB REST API. Kimlik ve sosyal veriler **PostgreSQL**’de tutulur.

**Öne çıkanlar**

* Ana sayfada *Yeni Filmler* ve *Trendler*.
* Sağda *Film Robotu* (genre, yıl, sort, puan) + sayfalama.
* Detayda poster, rozetler (tür/süre/puan), **Fragman modalı**, **Favori** ve **Like/Dislike**.
* Aramada sonuç listesi + sayfalama; **canlı öneri** (8 sonuç).

---

## 2) Dizin Yapısı (güncel)

```
BitirmeProjesiYeni/
├─ static/
│  └─ main.js                 # Discover, modal, canlı öneri, UI işlemleri
├─ templates/
│  ├─ base.html               # Ortak layout + canlı arama paneli + navbar Favoriler
│  ├─ index.html              # Ana sayfa (Yeni, Trend, Film Robotu)
│  ├─ detail.html             # Detay (fragman, favori, like/dislike, yorum alanı)
│  ├─ favorites.html          # Favoriler sayfası (grid)
│  ├─ login.html              # Giriş formu
│  └─ register.html           # Kayıt formu
├─ app.py                     # Flask uygulaması, PostgreSQL, auth, favori/rating uçları
├─ sentiment.py               # Duygu analizi (POS/NEG/NEU) entegrasyon noktası
├─ .env                       # TMDB_API_KEY, DB bağlantısı, APP_SECRET
├─ requirements.txt           # Bağımlılıklar
└─ .gitignore / README.md     # Standart
```

---

## 3) Bağımlılıklar (güncel)

`requirements.txt` önerisi:

```
flask==3.0.3
requests==2.32.3
psycopg[binary]==3.2.1
python-dotenv==1.0.1
Werkzeug==3.0.4
transformers==4.44.2
torch>=2.2
```

> `transformers/torch` yorumların duygu analizi içindir; çekirdek akış bunlar olmadan da çalışır.

---

## 4) Kurulum ve Çalıştırma

1. **Sanal ortam**

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

2. **.env** (örnek)

```dotenv
TMDB_API_KEY=xxx
APP_SECRET=dev-secret-change-me

# PostgreSQL (heroku/neon varsa direkt DATABASE_URL kullan)
DATABASE_URL=postgresql://postgres:password@localhost:5432/filmsite
# Alternatif tekil env'ler:
# PGHOST=localhost
# PGPORT=5432
# PGUSER=postgres
# PGPASSWORD=password
# PGDATABASE=filmsite
```

3. **Çalıştırma**

```bash
python app.py
# http://127.0.0.1:5000
```

---

## 5) Mimari (güncel)

* **Sunum:** Tailwind + Jinja2 şablonlar; `static/main.js` ile dinamik etkileşimler.
* **Uygulama:** Flask route’ları; kimlik doğrulama; favori/oylama; yorum + sentiment; TMDB yardımcıları; tür listesi için `@lru_cache`.
* **Veri Katmanı:** PostgreSQL (psycopg3). Kısa ömürlü bağlantı, transaction bazlı kullanım.
* **Harici API:** TMDB (dil: `tr-TR`; videolarda fallback `en-US`).

---

## 6) Veritabanı Tasarımı

**Tablolar**

* `users (id, username, email, password_hash, created_at)`
* `comments (id, movie_id, user_id→users.id, content, is_spoiler, created_at, sentiment_label, sentiment_score)`
* `favorites (id, user_id→users.id, movie_id, created_at, UNIQUE(user_id, movie_id))`
* `ratings (id, user_id→users.id, movie_id, value∈{-1,1}, created_at, UNIQUE(user_id, movie_id))`

**Notlar**

* `favorites` ve `ratings` **UNIQUE(user_id, movie_id)** ile tekrarı engeller.
* `ratings.value`: `1=like`, `-1=dislike`. Aynı değere ikinci kez basılırsa **kaldır (toggle)**; farklı değerse **update**.

---

## 7) Backend Uçları (güncel)

* `GET /` → Ana sayfa (Yeni, Trend, Film Robotu verisi)

* `GET /movie/<id>` → Detay (fragman seçimi: TR > EN > diğer; önerilen filmler; yorum ve istatistikler)

* `POST /movie/<id>/comment` → Yorum ekle (login zorunlu; sentiment etiketlenir)

* `POST /movie/<id>/favorite` → Favoriye ekle/çıkar (toggle)

* `POST /movie/<id>/rate` → Like/Dislike (toggle + sayım)

* `GET /favorites` → Kullanıcının favori filmleri (TMDB’den küçük payload’larla çekilir)

* `GET /search?q=…` → Arama sayfası

* `GET /api/search_suggest?q=…` → **Canlı öneri** (ilk 8 sonuç)

* `GET /api/discover` → Film Robotu verisi (genre, year, sort_by, vote_gte, page)

* Auth:

  * `GET|POST /register`, `GET|POST /login`, `GET /logout`

---

## 8) Frontend (güncel)

* **Navbar:** `Film izle`, `Listeler`, **Favoriler** (sadece login ise).
* **Arama kutusu:** **canlı öneri** paneli; yön tuşları/Enter/Escape destekli; tıklayınca kapanır.
* **Detay sayfası:**

  * Başlığın yanında **Favorilere Ekle / Favoride** butonu (durumlu).
  * Poster altında **👍 Beğendim / 👎 Beğenmedim** butonları, kullanıcı seçim rengi ve **global sayaç**.
  * **Fragmanı izle** modalı (YouTube).
  * Yorum alanı (login zorunlu), spoiler uyarısı, sentiment rozeti.
* **Favoriler sayfası:** Grid kartlar; başlık, yıl, puan.

---

## 9) Güvenlik & Performans

* **APP_SECRET**: production’da güçlü ve gizli tut.
* **Parola Hash:** `Werkzeug.generate_password_hash` / `check_password_hash`.
* **Rate limit/caching:** TMDB için basit önbellek (genre) var; prod’da Redis/HTTP cache önerilir.
* **Input doğrulama:** `sort_by` ve benzeri parametreler beyaz liste ile sınırlı.
* **Timeout:** `requests` 15 sn; prod’da retry/backoff önerilir.
* **SQL Güvenliği:** Parametrik sorgular (psycopg) kullanılıyor.

---

## 10) Manuel Test Senaryoları (güncel)

1. **Kayıt/Giriş/Çıkış** akışı.
2. **Favori Toggle:** Detay → Favorilere Ekle; tekrar tıkla → kaldır. `/favorites`’te görünmeli/kaybolmalı.
3. **Like/Dislike Toggle:**

   * Like’a bas → sayaç +1, buton yeşil.
   * Tekrar Like → oy kaldır (sayaç eski hal).
   * Dislike’a bas → Like kalkıp Dislike aktif olmalı (veya tersi).
4. **Canlı Öneri:** “superman” yaz → panel 8 sonuç listelesin; ok tuşlarıyla gez; Enter → detay sayfasına git.
5. **Discover:** Filtre/sayfalama, kapat → “Yeni Filmler” geri gelsin.
6. **Fragman Modal:** Aç/kapat; dışına tıklayınca kapanmalı.
7. **Yorum:** Spoiler kutusu açık/kapalı; sentiment rozeti görünsün.
8. **Yetki Kontrolü:** Login olmadan favori/oy/yorum uçlarına POST → login’e yönlen.

---

## 11) Dağıtım (Öneri)

* **Gunicorn:** `gunicorn 'app:app' --workers 2 --timeout 30`
* **Reverse Proxy:** Nginx (gzip + cache headers)
* **Env:** `.env` gizli; `debug=False`
* **DB:** PostgreSQL (Neon/Render/ElephantSQL uygun); `DATABASE_URL` ver
* **Statikler:** Nginx üzerinden; uzun `Cache-Control`
* **Günlükleme:** Erişim/hata logları; logrotasyon

---

## 12) Yol Haritası (güncel)

* [ ] Favorilerde **sayfalı TMDB toplu fetch** (istek sayısını azalt).
* [ ] **İleri analizler:** Favori + rating’e göre **kişiselleştirilmiş öneri** (içerik tabanlı/SBERT).
* [ ] **E-posta doğrulama / parola sıfırlama**.
* [ ] **Rate limit & global error banner** (TMDB hata/limit).
* [ ] **Redis cache** (Trend/Now Playing/Discover response’ları).
* [ ] **Unit/Integration tests** (pytest + requests-mock).
* [ ] **CI/CD** (GitHub Actions).

---

## 13) SSS (kısa)

* **IMDB puanı nereden?** TMDB `vote_average` alanı görselde “IMDB” etiketiyle gösteriliyor; TMDB puanıdır.
* **Favori/Like neden DB’de?** Kullanıcıya özgü, kalıcı ve sorgulanabilir olması için.
* **Neden PostgreSQL?** Güvenilir, ilişkisel; UNIQUE/foreign key/transaction gereksinimleri için uygun.

---

