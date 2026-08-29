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
  dan `HF_HOME=/models/hf-cache` memastikan semua unduhan masuk `./models/`, bukan cache tersembunyi.
- Tuned untuk **VPS ARM 2-core / 12GB RAM**: `--threads 2`, `--ctx-size 4096`, `--flash-attn`,
  `--models-max 2` (maksimal 2 model termuat bersamaan).

---

## Deploy di Dokploy

1. Buat **project** baru di Dokploy → pilih jenis **Docker Compose**.
2. Set koneksi repo: tempel URL GitHub repo ini.
3. Di pengaturan project, tambah **env** dari `.env.example`. Isi sekurangnya:
   - `LLAMA_API_KEY` — token apa pun (mis. hasil `openssl rand -hex 32`). Wajib.
   - `PORT` (default `3000`), `CTX_SIZE` (default `4096`), `THREADS` (default `2`) — opsional.
4. Deploy. Dokploy akan membuat volume maupun bind mount `models/` dan `data/` sesuai store yang kamu set.
   > Semua model GGUF tersimpan di `models/` dan data Open WebUI di `data/` — **persist lewat volume mount**.

---

## Penggunaan: pull & kelola model via GUI

1. Buka Open WebUI → `http://<VPS_IP>:3000`.
2. Buat akun admin pertama kali, lalu buka **Settings (⚙️) → Admin Panel → Models → Manage**.
3. Di panel itu kamu bisa:
   - **Download**: isi nama repo HF (contoh `ibm-granite/granite-4.2-3b-GGUF`)
     → llama-server mengunduh GGUF ke volume `./models/`.
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