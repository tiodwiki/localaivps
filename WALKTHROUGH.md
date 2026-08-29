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
3. Di project tersebut, **+ New Service** → pilih **Docker Compose** (jangan pilih upload-folder).
4. Set **source**: sambungkan ke **Git Provider** (GitHub) → pilih repo `openwebui-llamacpp`.

### 3.2. Config deploy (Compose)

Dokploy membaca `compose.yaml` di repo. Di pengaturan service:

- **Git Branch**: `main`.
- **Build / Command**: biarkan default (compose langsung jalan; tidak ada Dockerfile tambahan).
- **Environment / Advanced variables**: isi dari `.env.example`. Wajib:
  ```
  LLAMA_API_KEY=isi_luapan_openssl_rand_hex_--_jangan_tebak_tebak_kata_kunci
  ```
  Opsional: `PORT=3000`, `CTX_SIZE=4096`, `THREADS=2`.

> Di compose, prefix `./models` dan `./data` berarti lokasi deploy di server. Dokploy biasanya
> memakai folder projectnya; pastikan storage-nya **persisten** (lihat 3.4).

### 3.3. Public access (domain / port)

Dokploy menjalankan Traefik sebagai reverse proxy.

- **Domain / Traefik**: assign ke service `open-webui` → subdomain mis. `chat.vpsmu.com`
  + centang HTTPS. Akses jadi `https://chat.vpsmu.com`.
- **ATAU** pakai `Port = 3000` pada service, akses lewat `http://<IP_VPS>:3000`
  (karena `PORT` di env dipetakan ke `8080` container Open WebUI).

> `llama-server` **tidak** perlu dipublic — dia hanya diakses oleh `open-webui`
> via network internal `http://llama-server:8080`.

### 3.4. (Sesuai keinginanmu) Simpan storage lewat volume mount

Repo sudah meng-`bind mount`:

- `./models:/models` → tempat semua GGUF + cache HF (`LLAMA_CACHE=/models/cache`).
- `./data:/app/backend/data` → database & chat Open WebUI.

Pastikan di pengaturan service, `models/` dan `data/` mengarah ke path yang **persisten**
(bukan ephemeral build folder). Di Dokploy kamu bisa atur lewat konfigurasi volume / store bila
ingin pindah ke path penyimpanan terpisah — ubah bagian `volumes:` di compose sesuai kebutuhan.

### 3.5. Deploy

Klik **Deploy** → tunggu log. Tanda sehat:
- `llama-server` log berisi `server is listening on http://0.0.0.0:8080` (router mode).
- `open-webui` log berisi `Uvicorn running on http://0.0.0.0:8080`.

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
4. Biarkan llama-server mengunduh ke `./models/`. Ukuran Q4_K_M ≈ 2,2 GB.

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

## 5. Verifikasi & periksa kondisi

`llama-server` berada di network internal, jadi tesnya dari dalam container:

```bash
# Masuk ke folder project deploy di Dokploy (atau pakai tombol logs), lalu:
docker compose exec llama-server sh -c \
  "wget -qO- http://localhost:8080/health; echo; \
   wget -qO- http://localhost:8080/v1/models"
```

- `{"status":"ok"}` → server hidup.
- `v1/models` → daftar model yang terdeteksi router.
- Cek disk: `df -h` → `/models` bertambah sesuai ukuran model (jangan sampai < 5 GB sisa).
- Cek RAM saat model dipakai: `htop` → pastikan 12 GB tidak menipis habis saat 2 model termuat.

---

## 6. Langkah lanjutan (opsional) — sesuai kebutuhanmu

- **Auto-pull model saat pertama deploy**: tambahkan satu-liner di compose (mis. satu service `bootstrap`
  yang `docker compose run llama-server ... -hf ibm-granite/granite-4.2-3b-GGUF` sekali jalan), supaya
  model sudah tersedia tanpa klik manual.
- **Healthcheck** di `llama-server` (mis. `curl -f http://localhost:8080/health`) agar Dokploy tahu `healthy`.
- **Backup**: tar/rsync `models/` + `data/` secara berkala (model & histori chat) ke S3 / disk lain.
- **Domain + HTTPS** sudah disediakan Traefik Dokploy — tinggal tempel subdomain di service `open-webui`.

---

## 7. Troubleshooting

| Gejala | Penyebab / solusi |
|--------|-------------------|
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