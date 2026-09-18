# Whisper SaaS Speech-to-Text API

Bu proje, [faster-whisper](https://github.com/SYSTRAN/faster-whisper) (CTranslate2 tabanlı Whisper implementasyonu) kullanan, GPU üzerinde çalışan, SQLite ile API anahtarı doğrulaması yapan bir ses-metin dönüşümü (transkripsiyon) REST API servisidir. Girdi olarak bir ses dosyası alır; çıktı olarak tam metni, kelime bazlı zaman damgalarını (`start`/`end`) ve model güven skorlarını (`probability`) döner.

## Mimari

Depo iki ayrı çalışma modu içerir:

- **`main_api.py`** — FastAPI tabanlı HTTP servisi ve uygulamanın giriş noktası (`uvicorn` ile `0.0.0.0:3333` üzerinde çalışır). API anahtarı doğrulaması, model yönetimi ve GPU kuyruğu mantığı burada yer alır.
- **`speechtotext.py`** — Bağımsız, komut satırından çalıştırılan bir gerçek-zamanlı mikrofon deşifre betiği (`sounddevice` ile ses yakalar, `faster-whisper` ile anlık transkripsiyon yapıp `konusma_kayitlari.txt` dosyasına yazar). API servisinin bir parçası değildir, ayrı bir araçtır.
- **`database.py`** — SQLite (`users.db`) üzerinde `users` tablosunu yönetir: `generate_new_key()` her kullanıcı için `tk_` önekli, `secrets.token_urlsafe(32)` ile üretilmiş benzersiz bir API anahtarı oluşturur; `validate_key()` gelen anahtarı sorgulayıp kullanıcı adını döner. Modül import edildiğinde `users.db` yoksa otomatik olarak oluşturulur.

### GPU Kuyruğu

`main_api.py` içinde global bir `asyncio.Lock()` (`gpu_lock`) tanımlıdır. `/transcribe` endpoint'ine gelen her istek bu kilidi bekler; aynı anda yalnızca bir istek GPU üzerinde işlenir, diğerleri sırada bekler. Yanıt gövdesinde isteğin ne kadar süre kuyrukta beklediği (`queue_wait_time`) ve fiili işlem süresi (`processing_time`) ayrı ayrı raporlanır. Bu, tek GPU'lu bir ortamda modelin aynı anda birden fazla istekle çakışıp bellek taşırmasını (OOM) engellemek için kullanılan basit bir sıralama mekanizmasıdır; gerçek bir kuyruk/worker sistemi (Celery, Redis vb.) değildir — istekler süreç içinde art arda işlenir.

Whisper modelleri **lazy loading** ile yüklenir: `get_model()` fonksiyonu istenen `model_type` (`tiny`, `medium`, `large` vb.) ilk kez talep edildiğinde `WhisperModel(..., device="cuda", compute_type="int8_float16")` ile GPU belleğine yüklenir ve `loaded_models` sözlüğünde önbelleğe alınır; sonraki isteklerde tekrar yüklenmez.

## Kurulum ve Çalıştırma (Docker)

`Dockerfile`, `nvidia/cuda:12.2.2-cudnn8-runtime-ubuntu22.04` temel imajı üzerine Python 3.10, `ffmpeg` ve `requirements.txt` bağımlılıklarını kurar; `main_api.py`, `database.py` ve `templates/` klasörünü kopyalar, `3333` portunu açar ve konteyneri `python3 main_api.py` ile başlatır.

`docker-compose.yml`:

```bash
docker compose up --build
```

Notlar:
- **GPU desteği zorunludur.** Compose dosyası `deploy.resources.reservations.devices` ile `nvidia` sürücüsünü ve tüm GPU'ları (`count: all`) talep eder; host makinede NVIDIA sürücüleri ve [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) kurulu olmalıdır. `main_api.py` doğrudan `device="cuda"` kullanır ve CPU'ya otomatik geri dönüş (fallback) içermez — GPU olmadan başarısız olur.
- `./users.db` ve `./kayitlar/` klasörleri konteynere bind-mount edilir; böylece API anahtarları ve kayıtlar konteyner yeniden başlatıldığında/silindiğinde kaybolmaz.
- `restart: always` ile servis konteyner/host yeniden başlatıldığında otomatik ayağa kalkar.

## Ortam Değişkenleri / Konfigürasyon

Servis için tanımlı bir ortam değişkeni veya `.env` dosyası bulunmuyor; tüm ayarlar (port `3333`, dil `tr`, `compute_type="int8_float16"` vb.) kod içinde sabit (hardcoded) değerlerdir. Yalnızca `speechtotext.py` (bağımsız betik) içinde, `cudnn` kütüphane yolu bulunamazsa `LD_LIBRARY_PATH` değişkenine manuel bir yol eklenir; bu, Docker akışını etkilemez.

## API Kullanımı

| Endpoint | Metot | Açıklama |
|---|---|---|
| `/generate-key?username=<ad>` | `POST` | Verilen kullanıcı adı için yeni bir API anahtarı (`tk_...`) üretir ve SQLite'a kaydeder. Kullanıcı adı zaten kayıtlıysa `400` döner. |
| `/transcribe` | `POST` | `multipart/form-data` ile ses dosyası (`file`) alır; opsiyonel `model_type` (varsayılan `medium`) query parametresi ile model boyutu seçilir. `x-api-key` header'ı zorunludur (`database.validate_key` ile doğrulanır); eksikse `401`, geçersizse `403` döner. Yanıt: `text`, `model`, `user`, `queue_wait_time`, `processing_time`, `words` (kelime bazlı zaman damgası ve olasılık listesi). |

`templates/index.html`, tarayıcıdan mikrofonla `ws://.../ws/<api_key>` adresine bağlanıp canlı deşifre gösteren bir demo arayüzüdür. **Ancak `main_api.py` içinde şu an bir `/ws/...` WebSocket endpoint'i tanımlı değildir** (kod yalnızca `WebSocket`/`Request`/`Jinja2Templates` gibi sınıfları import eder ama kullanmaz); bu şablon dosyası herhangi bir route tarafından servis edilmiyor ve gerçek zamanlı WebSocket akışı henüz uygulanmamıştır.

## Bilinen Sınırlamalar

- Gerçek zamanlı WebSocket transkripsiyonu (`templates/index.html`'in beklediği `/ws/{api_key}`) API'de henüz uygulanmamıştır.
- `/generate-key` endpoint'i herhangi bir yetkilendirme gerektirmez; isteyen herkes yeni bir API anahtarı oluşturabilir — üretim ortamı için bir admin koruması eklenmesi gerekir.
- Tek `asyncio.Lock` tüm istekleri sıraya aldığından, isteklerin tamamı sıralı (seri) işlenir; çoklu GPU veya paralel işlem desteği yoktur.
- `main_api.py` içinde `device="cuda"` sabittir; CPU fallback yalnızca bağımsız `speechtotext.py` betiğinde mevcuttur, API servisinde yoktur.
- Transkripsiyon dili kod içinde `language="tr"` olarak sabitlenmiştir, dinamik dil seçimi yoktur.
- `users.db` dosyası hassas veriler içerdiği için `.gitignore`'da hariç tutulmuştur; üretim ortamında API anahtarlarını paylaşmayın.
