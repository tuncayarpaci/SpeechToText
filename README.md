# Whisper SaaS Speech-to-Text API

🇹🇷 [Türkçe](#türkçe) · 🇬🇧 [English](#english)

---

## Türkçe

Bu proje, [faster-whisper](https://github.com/SYSTRAN/faster-whisper) (CTranslate2 tabanlı Whisper implementasyonu) kullanan, GPU üzerinde çalışan, SQLite ile API anahtarı doğrulaması yapan bir ses-metin dönüşümü (transkripsiyon) REST API servisidir. Girdi olarak bir ses dosyası alır; çıktı olarak tam metni, kelime bazlı zaman damgalarını (`start`/`end`) ve model güven skorlarını (`probability`) döner.

### Mimari

Depo iki ayrı çalışma modu içerir:

- **`main_api.py`** — FastAPI tabanlı HTTP servisi ve uygulamanın giriş noktası (`uvicorn` ile `0.0.0.0:3333` üzerinde çalışır). API anahtarı doğrulaması, model yönetimi ve GPU kuyruğu mantığı burada yer alır.
- **`speechtotext.py`** — Bağımsız, komut satırından çalıştırılan bir gerçek-zamanlı mikrofon deşifre betiği (`sounddevice` ile ses yakalar, `faster-whisper` ile anlık transkripsiyon yapıp `konusma_kayitlari.txt` dosyasına yazar). API servisinin bir parçası değildir, ayrı bir araçtır.
- **`database.py`** — SQLite (`users.db`) üzerinde `users` tablosunu yönetir: `generate_new_key()` her kullanıcı için `tk_` önekli, `secrets.token_urlsafe(32)` ile üretilmiş benzersiz bir API anahtarı oluşturur; `validate_key()` gelen anahtarı sorgulayıp kullanıcı adını döner. Modül import edildiğinde `users.db` yoksa otomatik olarak oluşturulur.

#### GPU Kuyruğu

`main_api.py` içinde global bir `asyncio.Lock()` (`gpu_lock`) tanımlıdır. `/transcribe` endpoint'ine gelen her istek bu kilidi bekler; aynı anda yalnızca bir istek GPU üzerinde işlenir, diğerleri sırada bekler. Yanıt gövdesinde isteğin ne kadar süre kuyrukta beklediği (`queue_wait_time`) ve fiili işlem süresi (`processing_time`) ayrı ayrı raporlanır. Bu, tek GPU'lu bir ortamda modelin aynı anda birden fazla istekle çakışıp bellek taşırmasını (OOM) engellemek için kullanılan basit bir sıralama mekanizmasıdır; gerçek bir kuyruk/worker sistemi (Celery, Redis vb.) değildir — istekler süreç içinde art arda işlenir.

Whisper modelleri **lazy loading** ile yüklenir: `get_model()` fonksiyonu istenen `model_type` (`tiny`, `medium`, `large` vb.) ilk kez talep edildiğinde `WhisperModel(..., device="cuda", compute_type="int8_float16")` ile GPU belleğine yüklenir ve `loaded_models` sözlüğünde önbelleğe alınır; sonraki isteklerde tekrar yüklenmez.

### Kurulum ve Çalıştırma (Docker)

`Dockerfile`, `nvidia/cuda:12.2.2-cudnn8-runtime-ubuntu22.04` temel imajı üzerine Python 3.10, `ffmpeg` ve `requirements.txt` bağımlılıklarını kurar; `main_api.py`, `database.py` ve `templates/` klasörünü kopyalar, `3333` portunu açar ve konteyneri `python3 main_api.py` ile başlatır.

`docker-compose.yml`:

```bash
docker compose up --build
```

Notlar:
- **GPU desteği zorunludur.** Compose dosyası `deploy.resources.reservations.devices` ile `nvidia` sürücüsünü ve tüm GPU'ları (`count: all`) talep eder; host makinede NVIDIA sürücüleri ve [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) kurulu olmalıdır. `main_api.py` doğrudan `device="cuda"` kullanır ve CPU'ya otomatik geri dönüş (fallback) içermez — GPU olmadan başarısız olur.
- `./users.db` ve `./kayitlar/` klasörleri konteynere bind-mount edilir; böylece API anahtarları ve kayıtlar konteyner yeniden başlatıldığında/silindiğinde kaybolmaz.
- `restart: always` ile servis konteyner/host yeniden başlatıldığında otomatik ayağa kalkar.

### Ortam Değişkenleri / Konfigürasyon

Servis için tanımlı bir ortam değişkeni veya `.env` dosyası bulunmuyor; tüm ayarlar (port `3333`, dil `tr`, `compute_type="int8_float16"` vb.) kod içinde sabit (hardcoded) değerlerdir. Yalnızca `speechtotext.py` (bağımsız betik) içinde, `cudnn` kütüphane yolu bulunamazsa `LD_LIBRARY_PATH` değişkenine manuel bir yol eklenir; bu, Docker akışını etkilemez.

### API Kullanımı

| Endpoint | Metot | Açıklama |
|---|---|---|
| `/generate-key?username=<ad>` | `POST` | Verilen kullanıcı adı için yeni bir API anahtarı (`tk_...`) üretir ve SQLite'a kaydeder. Kullanıcı adı zaten kayıtlıysa `400` döner. |
| `/transcribe` | `POST` | `multipart/form-data` ile ses dosyası (`file`) alır; opsiyonel `model_type` (varsayılan `medium`) query parametresi ile model boyutu seçilir. `x-api-key` header'ı zorunludur (`database.validate_key` ile doğrulanır); eksikse `401`, geçersizse `403` döner. Yanıt: `text`, `model`, `user`, `queue_wait_time`, `processing_time`, `words` (kelime bazlı zaman damgası ve olasılık listesi). |

`templates/index.html`, tarayıcıdan mikrofonla `ws://.../ws/<api_key>` adresine bağlanıp canlı deşifre gösteren bir demo arayüzüdür. **Ancak `main_api.py` içinde şu an bir `/ws/...` WebSocket endpoint'i tanımlı değildir** (kod yalnızca `WebSocket`/`Request`/`Jinja2Templates` gibi sınıfları import eder ama kullanmaz); bu şablon dosyası herhangi bir route tarafından servis edilmiyor ve gerçek zamanlı WebSocket akışı henüz uygulanmamıştır.

### Bilinen Sınırlamalar

- Gerçek zamanlı WebSocket transkripsiyonu (`templates/index.html`'in beklediği `/ws/{api_key}`) API'de henüz uygulanmamıştır.
- `/generate-key` endpoint'i herhangi bir yetkilendirme gerektirmez; isteyen herkes yeni bir API anahtarı oluşturabilir — üretim ortamı için bir admin koruması eklenmesi gerekir.
- Tek `asyncio.Lock` tüm istekleri sıraya aldığından, isteklerin tamamı sıralı (seri) işlenir; çoklu GPU veya paralel işlem desteği yoktur.
- `main_api.py` içinde `device="cuda"` sabittir; CPU fallback yalnızca bağımsız `speechtotext.py` betiğinde mevcuttur, API servisinde yoktur.
- Transkripsiyon dili kod içinde `language="tr"` olarak sabitlenmiştir, dinamik dil seçimi yoktur.
- `users.db` dosyası hassas veriler içerdiği için `.gitignore`'da hariç tutulmuştur; üretim ortamında API anahtarlarını paylaşmayın.

---

## English

This project is a speech-to-text (transcription) REST API service that uses [faster-whisper](https://github.com/SYSTRAN/faster-whisper) (a CTranslate2-based Whisper implementation), runs on GPU, and performs API key authentication with SQLite. It takes an audio file as input and returns the full text, word-level timestamps (`start`/`end`), and model confidence scores (`probability`) as output.

### Architecture

The repository contains two separate modes of operation:

- **`main_api.py`** — The FastAPI-based HTTP service and the application's entry point (runs on `0.0.0.0:3333` via `uvicorn`). API key authentication, model management, and GPU queue logic live here.
- **`speechtotext.py`** — A standalone, command-line real-time microphone transcription script (captures audio with `sounddevice`, performs live transcription with `faster-whisper`, and writes to the `konusma_kayitlari.txt` file). It is not part of the API service — it is a separate tool.
- **`database.py`** — Manages the `users` table in SQLite (`users.db`): `generate_new_key()` creates a unique API key with a `tk_` prefix, generated via `secrets.token_urlsafe(32)`, for each user; `validate_key()` looks up the incoming key and returns the username. When the module is imported, `users.db` is created automatically if it does not exist.

#### GPU Queue

A global `asyncio.Lock()` (`gpu_lock`) is defined in `main_api.py`. Every request to the `/transcribe` endpoint waits on this lock; only one request is processed on the GPU at a time, while the rest wait in line. The response body separately reports how long the request waited in the queue (`queue_wait_time`) and the actual processing time (`processing_time`). This is a simple sequencing mechanism used in a single-GPU environment to prevent the model from colliding with multiple requests at once and overflowing memory (OOM); it is not a real queue/worker system (Celery, Redis, etc.) — requests are processed one after another within the process.

Whisper models are loaded with **lazy loading**: the `get_model()` function loads the requested `model_type` (`tiny`, `medium`, `large`, etc.) into GPU memory the first time it is requested, using `WhisperModel(..., device="cuda", compute_type="int8_float16")`, and caches it in the `loaded_models` dictionary; it is not reloaded on subsequent requests.

### Setup and Running (Docker)

The `Dockerfile` installs Python 3.10, `ffmpeg`, and the `requirements.txt` dependencies on top of the `nvidia/cuda:12.2.2-cudnn8-runtime-ubuntu22.04` base image; copies `main_api.py`, `database.py`, and the `templates/` folder, exposes port `3333`, and starts the container with `python3 main_api.py`.

`docker-compose.yml`:

```bash
docker compose up --build
```

Notes:
- **GPU support is mandatory.** The compose file requests the `nvidia` driver and all GPUs (`count: all`) via `deploy.resources.reservations.devices`; the NVIDIA drivers and the [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) must be installed on the host machine. `main_api.py` uses `device="cuda"` directly and has no automatic CPU fallback — it fails without a GPU.
- The `./users.db` and `./kayitlar/` folders are bind-mounted into the container, so API keys and recordings are not lost when the container is restarted or removed.
- With `restart: always`, the service automatically comes back up when the container/host is restarted.

### Environment Variables / Configuration

There is no environment variable or `.env` file defined for the service; all settings (port `3333`, language `tr`, `compute_type="int8_float16"`, etc.) are hardcoded values in the code. Only in `speechtotext.py` (the standalone script), if the `cudnn` library path cannot be found, a path is manually appended to the `LD_LIBRARY_PATH` variable; this does not affect the Docker flow.

### API Usage

| Endpoint | Method | Description |
|---|---|---|
| `/generate-key?username=<name>` | `POST` | Generates a new API key (`tk_...`) for the given username and saves it to SQLite. Returns `400` if the username is already registered. |
| `/transcribe` | `POST` | Takes an audio file (`file`) via `multipart/form-data`; the model size is selected via the optional `model_type` query parameter (default `medium`). The `x-api-key` header is required (validated via `database.validate_key`); returns `401` if missing, `403` if invalid. Response: `text`, `model`, `user`, `queue_wait_time`, `processing_time`, `words` (word-level timestamp and probability list). |

`templates/index.html` is a demo interface that connects to `ws://.../ws/<api_key>` from the browser's microphone and shows live transcription. **However, `main_api.py` currently does not define a `/ws/...` WebSocket endpoint** (the code only imports classes like `WebSocket`/`Request`/`Jinja2Templates` but does not use them); this template file is not served by any route, and real-time WebSocket streaming has not yet been implemented.

### Known Limitations

- Real-time WebSocket transcription (the `/ws/{api_key}` that `templates/index.html` expects) has not yet been implemented in the API.
- The `/generate-key` endpoint does not require any authorization; anyone can create a new API key — an admin safeguard should be added for a production environment.
- Since a single `asyncio.Lock` queues all requests, all requests are processed sequentially (serially); there is no support for multiple GPUs or parallel processing.
- `device="cuda"` is hardcoded in `main_api.py`; CPU fallback is only available in the standalone `speechtotext.py` script, not in the API service.
- The transcription language is hardcoded in the code as `language="tr"`; there is no dynamic language selection.
- The `users.db` file is excluded in `.gitignore` because it contains sensitive data; do not share API keys in a production environment.
