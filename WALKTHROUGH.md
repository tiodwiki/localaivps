# Walkthrough Deployment — Open WebUI + llama.cpp (Dokploy)

Panduan langkah-demi-langkah, dari repo kosong sampai chat pertama dengan Granite 3B di Open WebUI.
Ditujukan untuk VPS ARM (aarch64), 2 core, 12 GB RAM (Oracle Cloud / KVM).

---

## 0. Prasyarat

- VPS **aarch64** Ubuntu 24.04, RAM ≥ 8 GB (rekomendasi 12 GB), disk kosong **≥ 15 GB** (untuk model).
- **Docker** terpasang di VPS.
- **Dokploy** terpasang di VPS (lihat Bagian 2 bila belum).
- Akun **GitHub**.
- Token API sebarang untuk `LLAMA_API_KEY` → buat cepat: `openssl rand -hex 32`.

---

## 1. Hubungkan repo ke GitHub

Lewat terminal mesin kamu (folder repo):

```bash
# 1. Buat repository BARU (kosong, tanpa README/license) di GitHub:
#    github.com -> New repository -> nama: openwebui-llamacpp

# 2. Hubungkan
git remote add origin https://github.com/<USERNAME>/openwebui-llamacpp.git
git branch -M main

# 3. Push
git push -u origin main
```

> Setelah push, segala perubahan kamu (`git add -A && git commit -m "..." && git push`) akan
> otomatis di-deploy ulang oleh Dokploy sesuai pengaturan di bawah.

---

## 2. (Bila belum ada) Install Dokploy di VPS

SSH ke VPS, lalu jalankan installer resmi:

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

- Ikuti petunjuk; Dokploy butuh domain (mis. `dokploy.vpsmu.com`) + **DNS A record** yang mengarah
  ke IP VPS agar reverse-proxy (Traefik) bisa memberi sertifikat HTTPS.
- Setelah selesai, buka `https://dokploy.vpsmu.com`, buat akun admin, dan isi email SMTP bila diminta.

Dokploy mendukung ARM — tidak ada konfirmasi khusus yang perlu diubah.

---

## 3. Deploy stack di Dokploy

### 3.1. Buat Project Compose

1. Login Dokploy.
2. Klik **+ New Project** → beri nama (mis. `llm`).
3. Di project tersebut, **+ New Service** → **Build Method: `Docker Compose`**
   (pilih ini, **bukan** `Dockerfile`/`Nixpacks`) → **Source: Git Provider** (GitHub) →
   pilih repo `openwebui-llamacpp`.
4. Pastikan **Compose Path** mengarah ke `compose.yaml` (field ini default `docker-compose.yml`
   — karena repo ini namanya `compose.yaml`, set jadi `./compose.yaml` bila perlu).

### 3.2. Config deploy (Compose)

Dokploy membaca `compose.yaml` di repo. Di pengaturan service:

- **Git Branch**: `main`.
- **Build / Command**: biarkan default (compose langsung jalan; tidak ada Dockerfile tambahan).
- **Environment (tab Environment)**: isi dari `.env.example`. Dokploy menulisnya ke `.env`
  di folder project lalu mengganti `${VAR}` yang dipakai di `compose.yaml`. Pasang pasangan
  `KETIK=ISI` (lewati baris `[TEMPLATE]`/komentar). Wajib:
  ```
  LLAMA_API_KEY=isi_token_openssl_rand_hex_32_abcdef0123...
  ```
  Opsional (bila ingin menimpa default di compose): `PORT=3000`, `CTX_SIZE=4096`, `THREADS=2`.
  `LLAMA_CACHE` & `HF_HOME` sudah ditetapkan di compose → jangan diisi.

### 3.3. Public access — domain (tab **Domains**)

Dokploy menjalankan Traefik sebagai reverse-proxy dan menyuntik label-nya otomatis:

- Buka tab **Domains** pada service → **Add Domain**.
- Pilih service **`open-webui`**, port container **`8080`**, subdomain mis. `chat.vpsmu.com`,
  centang **HTTPS** (Let's Encrypt).
- Pastikan **DNS A record** `chat -> <IP_VPS>` sudah dibuat. Gunakan **Preview Compose**
  di bawah untuk melihat label Traefik + network yang disuntik sebelum menyimpan.

> Akses jadi `https://chat.vpsmu.com`. Tanpa domain, publikasikan lewat port host:
> isi `PORT=3000` → buka `http://<IP_VPS>:3000` (mapping `8080` container ↔ `3000` host).
>
> `llama-server` **tidak** perlu dipublic — dia hanya diakses oleh `open-webui`
> via network internal `http://llama-server:8080`.

### 3.4. Volume & persistensi data

`compose.yaml` memakai **satu Docker named volume `data`** (bukan bind mount `./`),
tepat untuk Dokploy — dipakai untuk semua data persisten:

- `data:/models` → tempat semua GGUF + cache HF (`LLAMA_CACHE=/models/cache`).
- `data:/app/backend/data` → database & chat Open WebUI (satu volume `data` yang sama).

> **Jangan ganti ke `./models`/`./data`.** Saat Dokploy menjalankan **AutoDeploy**, ia me-`git clone`
> ulang repo pada setiap deploy → folder relatif `./` benar-benar dihapus, sehingga **semua model &
> chat hilang tiap kali kamu push & redeploy**. Named volume disimpan Docker (persisten) dan bisa
> di-backup otomatis lewat fitur **Volume Backups** Dokploy (mis. ke S3).
>
> **Tidak ada field volume yang perlu diisi di UI Dokploy.** Pada service **Docker Compose**,
> volume mount dideklarasikan **sepenuhnya di `compose.yaml`** — Dokploy membaca file itu dan
> membuat named volume `data` otomatis saat deploy pertama (tidak perlu dibuat manual).
> Tab Dokploy yg tersedia utk Compose hanya **Volume Backups** (backup named volume, bukan
> mengatur mount); kalau nanti pakai bind mount, baru lewat **Advanced → Mounts/File Mounts**.
> Isi volume `data` bisa dicek lewat tab terminal/exec tiap container (`/models`, `/app/backend/data`),
> dan di-backup lewat **Volume Backups**.

### 3.5. Deploy

Klik **Deploy** → pantau tab **Deployments** sampai pull image & `up` selesai → lanjut ke Bagian 5
untuk verifikasi log lengkap.

---

## 4. Setup pertama & kelola model via GUI

### 4.1. Buat akun admin

Buka `https://chat.vpsmu.com` (atau `http://<IP>:3000`). Daftarkan **admin pertama**.

### 4.2. Pastikan koneksi OpenAI ber-provider llama.cpp

Open WebUI perlu tahu backend-nya adalah llama.cpp agar panel **Manage** model aktif.

1. **Settings (⚙️) → Admin Panel → Connections → OpenAI**.
2. Cek baris koneksi yang otomatis dibuat dari `OPENAI_API_BASE_URL`:
   - URL: `http://llama-server:8080/v1`
   - API Key: isi dari `LLAMA_API_KEY` (sama dengan yang di env).
3. Pada dropdown **Advanced → Provider**, pastikan terpilih **llama.cpp**.
   (Biasanya terdeteksi otomatis; set manual bila tidak.)

### 4.3. Unduh model dari Hugging Face

1. **Settings → Admin Panel → Models → Manage**.
2. Klik **Download**.
3. Masukkan nama repo HF: `ibm-granite/granite-4.2-3b-GGUF`
   (bisa juga pakai `:Q4_K_M` untuk kuantisasi spesifik).
4. Biarkan llama-server mengunduh ke volume `data` (ter-mount di `/models`). Ukuran Q4_K_M ≈ 2,2 GB.

> Cocok utk VPS ini: model **±3B / Q4_K_M** (~2 GB). Jangan pilih Q8/_8B+ tanpa GPU.

### 4.4. Chat pertama

1. Kembali ke halaman utama Open WebUI.
2. Dropdown model (pojok atas) → pilih **granite-4.2-3b** (yang baru diunduh).
3. Model termuat otomatis; tulis pesan, mis. *"Halo, perkenalkan dirimu dalam 1 kalimat."*
4. Karena CPU-only, respons pertama agak lama (loading model), selanjutnya lebih cepat.

### 4.5. Manage: load / unload / delete

Di **Settings → Admin Panel → Models → Manage** (atau dropdown model):
- Model yang termuat ditandai hijau **Loaded**; klik **eject** utk membebaskan RAM tanpa restart.
- **Delete** utk menghapus file dari disk (bebaskan storage).

---

## 5. Verifikasi log & periksa kondisi

### 5.1. Cek log lewat panel Dokploy (tab **Logs**)

Di halaman service, buka tab **Logs** → pilih service per service. Tanda sehat setelah deploy:

- **`llama-server`**: muncul `server is listening on http://0.0.0.0:8080` (router mode aktif).
- **`open-webui`**: muncul `Uvicorn running on http://0.0.0.0:8080`.
- Tidak ada `error`, `Traceback`, `panic`, atau *exit code ≠ 0* di kedua service;
  status kontainer hijau `healthy`/`running` di tab **Monitoring**.

> Setiap deploy baru tercatat di tab **Deployments**; bila deploy gagal, log di situ menunjuk
> penyebab (env belum terisi `LLAMA_API_KEY`, pull image gagal, port bentrok, dll.).

### 5.2. Tes endpoint internal `llama-server`

`llama-server` ada di network internal, jadi tesnya **dari dalam container**. Pakai tab
terminal/exec Dokploy untuk service `llama-server`, atau SSH VPS lalu:

```bash
# di folder project deploy Dokploy

docker compose exec llama-server sh -c \
  "wget -qO- http://localhost:8080/health; echo; \
   wget -qO- http://localhost:8080/v1/models"
```

- `{"status":"ok"}` → server hidup.
- `v1/models` → daftar model yang terdeteksi router (sebelum unduh mungkin kosong ‒ normal).

### 5.3. Verifikasi persistensi volume

Untuk memastikan named volume benar-benar dipakai (tidak hilang saat redeploy):

- Unduh satu model lalu **push perubahan**/redeploy; cek `v1/models` — model harus **masih ada**.
- Cek isi volume dari container: `df -h` → `/models` bertambah sesuai ukuran model
  (jangan sampai sisa disk < 5 GB). Lihat file via terminal service `open-webui`
  (`ls /app/backend/data`) atau halaman project Dokploy → **Volumes** (`data`).
- Cek RAM saat model dipakai: `htop` → pastikan 12 GB tidak menipis habis saat 2 model termuat.

---

## 6. Langkah lanjutan (opsional) — sesuai kebutuhanmu

- **Auto-pull model saat pertama deploy**: tambahkan satu-liner di compose (mis. satu service `bootstrap`
  yang `docker compose run llama-server ... -hf ibm-granite/granite-4.2-3b-GGUF` sekali jalan), supaya
  model sudah tersedia tanpa klik manual.
- **Healthcheck** di `llama-server` (mis. `curl -f http://localhost:8080/health`) agar Dokploy menandai `healthy`.
- **Backup otomatis**: aktifkan **Volume Backups** Dokploy untuk volume `data`
  (mis. ke S3) supaya model + histori chat selalu aman — backup named volume jauh lebih rapi
  daripada menyalin folder `./`.
- **Domain + HTTPS** sudah disediakan Traefik Dokploy — tinggal tempel subdomain di service `open-webui` (tab Domains).

---

## 7. Troubleshooting

| Gejala | Penyebab / solusi |
|--------|-------------------|
| `llama-server` **crash-loop**, log `error: invalid argument: llama-server` | Image `llama.cpp:server` sudah ber-`ENTRYPOINT "/app/llama-server"`, jadi `command` TIDAK boleh diawali binari `llama-server` (komposisi jadi `llama-server llama-server ...`). Hapus item `llama-server` dari `command` lalu redeploy. | Service dibuat dengan **Build Method `Dockerfile`** padahal repo ini pakai image registry. Hapus service, buat baru dengan **Build Method `Docker Compose`**, dan set **Compose Path** ke `compose.yaml`. |
| `timeout` saat list model | Model besar → naikkan `AIOHTTP_CLIENT_TIMEOUT_MODEL_LIST=30` (sudah ada di compose). |
| Panel **Manage** model tidak muncul / tombol eject error | Provider OpenAI belum **llama.cpp** → set manual di Connections → OpenAI. |
| Server mati (OOM) saat 2 model jalan | Turunkan `CTX_SIZE` ke `2048`, atau `--models-max 1`, atau pilih model kuantisasi lebih kecil. |
| Storage penuh | **Delete** model yang tak dipakai di Manage panel; cek `df -h`. |
| Download model gated (butuh login) | Set `HF_TOKEN` di env compose (token HF read) lalu redeploy. |
| Chat lambat di CPU | Normal utk CPU 2-core; pakai model 3B Q4_K_M, naikkan thread bila core > 2. |

---

## 8. Referensi

- [Dokploy — Docker Compose](https://dokploy.com)
- [Open WebUI — mulai dengan llama.cpp](https://docs.openwebui.com/getting-started/quick-start/connect-a-provider/starting-with-llama-cpp/)
- [llama.cpp — model management / router](https://huggingface.co/blog/ggml-org/model-management-in-llamacpp)