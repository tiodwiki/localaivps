# openwebui-llamacpp

Docker Compose untuk menjalankan **Open WebUI** + **llama.cpp `llama-server`**
(penginference model **GGUF** yang *OpenAI-compatible*) di VPS **aarch64 (ARM)**, siap di-deploy via **Dokploy**.

Tidak ada model bawaan — kamu **pull & kelola model langsung dari GUI Open WebUI**
(`Settings → Models → Manage`), termasuk model yang menempel link Hugging Face
seperti `ibm-granite/granite-4.2-3b-GGUF`. Tanpa SSH, tanpa CLI, tanpa restart container.

---

## Arsitektur

| Service          | Image                                   | Peran                                                                 |
|------------------|-----------------------------------------|-----------------------------------------------------------------------|
| `llama-server`   | `ghcr.io/ggml-org/llama.cpp:server`     | Inference GGUF, **router mode**: load/unload/switch banyak model tanpa restart. Hanya di network internal (tidak expose). |
| `open-webui`     | `ghcr.io/open-webui/open-webui:main`    | UI chat + **control panel model** (download/load/unload/delete). Akses dari browser port `3000`. |

Poin penting:

- **Router mode** (`--models-dir /models`) meniru gaya Ollama: model terdeteksi otomatis,
  termuat on-demand saat diminta, dan bisa di-unload saat tidak dipakai supaya hemat RAM.
- **Download model dari HF disimpan ke volume mount** — env `LLAMA_CACHE=/models/cache`
  dan `HF_HOME=/models/hf-cache` memastikan semua unduhan masuk volume `data`
  (di-`mount` ke `/models`), bukan cache tersembunyi.
- Tuned untuk **VPS ARM 2-core / 12GB RAM**: `--threads 2`, `--ctx-size 4096`, `--flash-attn`,
  `--models-max 2` (maksimal 2 model termuat bersamaan).

---

## Deployment step-by-step

Panduan end-to-end (repo → push GitHub → install Dokploy → deploy → chat pertama
+ kelola model via GUI) ada di **[WALKTHROUGH.md](WALKTHROUGH.md)**.

---

## Deploy di Dokploy (env, domain, volume, verifikasi log)

> Perlu sudah: repo ini ter-push ke GitHub (lihat bagian "Hubungkan repo ke GitHub"),
> akun Git provider tersambung di Dokploy, dan VPS tamu memakai Ubuntu 24.04 + Dokploy.
> Ikut detail penuh di **[WALKTHROUGH.md](WALKTHROUGH.md)**.

### 1. Buat service **Docker Compose** dan hubungkan repo

1. Login Dokploy → **+ New Project** → beri nama (mis. `llm`).
2. Di dalam project → **+ New Service** → pilih **Docker Compose** (bukan *single container*).
3. Set **source / Git Provider**: pilih GitHub → repo `openwebui-llamacpp` → **Branch** `main`.

Dokploy membaca `compose.yaml` langsung dari repo untuk di-`docker compose up`.

### 2. Isi environment (tab **Environment**)

Dokploy menyimpan variabel tab ini ke file `.env` di folder project, lalu mengganti
`${VAR}` yang dipakai di `compose.yaml` saat deploy. Salin nilai dari `.env.example`
(**jangan** ikut-tempel baris `[TEMPLATE]`/komentar — cukup pasangan `KEY=VALUE`):

- **`LLAMA_API_KEY`** — **wajib.** Token apa pun (mis. `openssl rand -hex 32`).
  Dipakai llama-server sebagai kunci API sekaligus `OPENAI_API_KEY` Open WebUI.
- Opsional (hanya bila ingin menimpa): `PORT=3000`, `CTX_SIZE=4096`, `THREADS=2`.
  (Tanpa diisi pun bekerja karena compose punya default `:-`.)

`LLAMA_CACHE` & `HF_HOME` tidak perlu diisi — sudah ditetapkan di `compose.yaml`
supaya semua unduhan model tersimpan ke dalam volume.

### 3. Domain & publikasi (tab **Domains**)

Cara paling simpel & direkomendasikan Dokploy (Traefik disuntik otomatis):

1. Buka tab **Domains** pada service → **Add Domain**.
2. Pilih service **`open-webui`**, port di dalam container **`8080`**, isi subdomain
   mis. `chat.vpsmu.com`, centang **HTTPS** (Let's Encrypt otomatis).
3. Pastikan **DNS A record** `chat -> <IP_VPS>` sudah dibuat di DNS provider.
4. Klik **Preview Compose** untuk memeriksa label Traefik & network yang akan disuntik,
   lalu simpan.

> Akses jadi `https://chat.vpsmu.com`. Kalau belum punya domain, bisa juga dipublikasikan
> lewat port host: pastikan `PORT=3000` terisi maka browser membuka `http://<IP_VPS>:3000`
> (mapping `8080` container ↔ `3000` host).
>
> `llama-server` **tidak** dipublic — ia hanya diakses antar-container via network
> internal `http://llama-server:8080`.

### 4. Volume & persistensi

`compose.yaml` memakai **satu Docker named volume `data`** untuk semua data persisten,
alih-alih bind mount relatif `./models`/`./data`:

- `data:/models` → tempat semua file GGUF + cache HF.
- `data:/app/backend/data` → database & chat Open WebUI (keduanya satu volume `data`).

> **Kenapa bukan `./models`?** Saat Dokploy *AutoDeploy*, repo di-clone ulang pada setiap
> deploy dan isi folder relatif ikut dihapus — bind mount `./models`/`./data` akan
> **kehilangan semua model & chat** saat kamu push perubahan / redeploy.
> Named volume disimpan Docker (persisten antar deploy) dan bisa di-backup lewat fitur
> **Volume Backups** Dokploy (ke S3) dengan mudah.
>
> **Tidak perlu mengisi apa pun di field volume Dokploy** — pada service **Docker Compose**,
> volume tidak di-set lewat UI Dokploy, melainkan **seluruhnya dideklarasikan di `compose.yaml`**
> (Dokploy membaca & membuat named volume `data` otomatis saat deploy). UI Dokploy utk Compose
> hanya punya tab **Volume Backups** (khusus backup volume, bukan mengatur mount).
>
> Baru kalau kamu beralih ke bind mount `../files/…`, Dokploy memakai **Advanced → Mounts/File Mounts**.

### 5. Deploy & verifikasi log

1. Klik **Deploy** pada service → tunggu build & pull image selesai di tab **Deployments**.
2. Buka tab **Logs** → pilih service-nya:
   - **`llama-server`**: cari baris `server is listening on http://0.0.0.0:8080` (router mode aktif).
   - **`open-webui`**: cari baris `Uvicorn running on http://0.0.0.0:8080`.
   - Tidak ada `error`/`Traceback`/*exit* di kedua service.
3. Cek endpoint dari dalam container `llama-server` (lewat tab terminal Dokploy atau SSH +
   `docker compose exec`):
   ```bash
   wget -qO- http://localhost:8080/health; echo
   wget -qO- http://localhost:8080/v1/models
   ```
   → `{"status":"ok"}` berarti server hidup; `v1/models` menampilkan model yang terdeteksi.
4. Buka `https://chat.vpsmu.com`, daftarkan akun admin pertama, dan lanjut ke bagian
   "Penggunaan" di bawah untuk mengunduh model via GUI.

---

## Penggunaan: pull & kelola model via GUI

1. Buka Open WebUI → `http://<VPS_IP>:3000`.
2. Buat akun admin pertama kali, lalu buka **Settings (⚙️) → Admin Panel → Models → Manage**.
3. Di panel itu kamu bisa:
   - **Download**: isi nama repo HF (contoh `ibm-granite/granite-4.2-3b-GGUF`)
     → llama-server mengunduh GGUF ke volume `data` (ter-mount di `/models`).
   - **Load / Switch**: model tersedia di dropdown chat; pilih → termuat otomatis.
   - **Unload**: klik eject pada model untuk melepas dari RAM tanpa restart.
   - **Delete**: hapus model dari server (dan dari disk) untuk membebaskan storage.
4. Selesai — langsung dipakai untuk chat.

> **Catatan provider:** untuk menampilkan panel *Manage* model, koneksi OpenAI-nya
> harus ber-provider **llama.cpp**. Umumnya Open WebUI mendeteksi otomatis dari URL
> (`http://llama-server:8080/v1`). Bila perlu, set manual di:
> **Settings → Admin Panel → Connections → OpenAI** → pilih **llama.cpp** pada dropdown *Provider*.

### Model rekomendasi utk VPS ini

- **ibm-granite/granite-4.2-3b-GGUF** — Granite 3B, ringan, cocok untuk CPU 2-core.
  Pilih kuantisasi `Q4_K_M` (~2.2 GB) agar nyaman di 12 GB RAM.
- Model lain ±3B / Q4 juga aman. Hindari model >8B / Q8 pada VPS tanpa GPU.

---

## Dikembangkan & dijalankan di mana?

Repositori ini **tidak menjalankan apa pun di mesin dev cukup sebagai konteks kode** —
deployment berjalan penuh di VPS-mu lewat Dokploy.

---

## Hubungkan repo ke GitHub

Kalau repo ini belum punya remote (mis. baru di-clone/dibuat lokal), hubungkan ke GitHub:

```bash
# di dalam folder repo
git remote add origin https://github.com/<user>/openwebui-llamacpp.git
git branch -M main
git push -u origin main
```

Kalau belum ada repo-nya di GitHub: buat repository kosong (tanpa README/license agar tidak bentrok),
lalu jalankan tiga perintah di atas.

---

## Referensi

- [Open WebUI — starting with llama.cpp](https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-llama-cpp/)
- [llama.cpp server (router / model management)](https://huggingface.co/blog/ggml-org/model-management-in-llamacpp)