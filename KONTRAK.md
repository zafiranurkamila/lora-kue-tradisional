# KONTRAK PROYEK: LoRA Kue Tradisional (SDXL)

> Dokumen ini berisi kesepakatan teknis tim supaya pekerjaan tiga orang bisa digabung tanpa bentrok.
> **Aturan utama:** kalau ada yang mau mengubah isi kontrak ini, diskusikan dulu di grup, lalu update dokumen ini lewat Pull Request. Jangan ubah diam-diam di kode.

## 1. Ringkasan Keputusan

| Hal | Keputusan |
|---|---|
| Backbone | `stabilityai/stable-diffusion-xl-base-1.0` |
| Metode | LoRA (PEFT) pada UNet, text encoder **tidak** dilatih |
| Jalur training | Satu jalur saja: skrip resmi Diffusers `train_text_to_image_lora_sdxl.py` (Kohya_ss tidak dipakai) |
| Jenis kue | 4–5 jenis, disepakati di Fase 0 (kandidat: klepon, serabi, dadar gulung, onde-onde) |
| Format dataset | Folder gambar + `metadata.jsonl` (format `imagefolder` Hugging Face) |
| Resolusi training | 1024 × 1024 |
| Deployment | Gradio + Diffusers (Ollama tidak dipakai untuk diffusion) |
| Komputasi | Semua yang butuh GPU dijalankan di PC Zafira |
| Alur Git | Satu branch per bagian kerja, merge ke `main` lewat Pull Request |

## 2. Pembagian Peran

Panduan tugas mengasumsikan 4 anggota. Kelompok ini bertiga, jadi peran Deployment digabung ke ML/Fine-Tuning, dan hal ini dijelaskan di laporan.

| Peran | Nama | Folder yang dipegang | Branch |
|---|---|---|---|
| Data Engineer + kode UI | Safira | `data/README.md`, `scripts/` (skrip dataset), `app/` | `feat/dataset`, `feat/demo` |
| ML/Fine-Tuning + Deployment | Zafira | `training/`, `requirements.txt`, `notebooks/` | `feat/setup`, `feat/baseline`, `feat/training` |
| QA + Technical Writer | Celsyka | `eval/`, `docs/`, `README.md` | `feat/evaluasi`, `docs/laporan` |

**Review silang:** PR dari Zafira direview oleh Safira atau Celsyka, PR dari Safira dan Celsyka direview oleh Zafira. Dengan begitu semua anggota ikut memahami seluruh alur untuk sesi tanya jawab.

**Tiap orang menulis bagian tutorial untuk tugasnya sendiri** (langkah, screenshot, error yang ditemui), lalu Celsyka menyatukan dan merapikan.

## 3. Pembagian Komputasi

PC Zafira (RTX 5060 Ti 16 GB) adalah satu-satunya GPU, jadi waktunya dipakai hanya untuk pekerjaan yang benar-benar butuh GPU.

| Hanya di PC Zafira (GPU) | Bisa di laptop masing-masing (CPU) |
|---|---|
| Generate baseline SDXL polos | Kurasi foto, crop, resize, caption, `metadata.jsonl` |
| Training LoRA dan eksperimen hyperparameter | Menulis kode Gradio (dites dengan gambar dummy) |
| Generate gambar evaluasi (base vs LoRA) | CLIP score dan analisis dari gambar yang sudah jadi |
| Menjalankan demo akhir | Survei, tabel, grafik, laporan |

Uji coba kecil sebelum memakai waktu PC Zafira boleh dilakukan di Google Colab.

## 4. Dataset

- **Jumlah:** 25–40 foto per jenis kue, total sekitar 100–200 foto. Variasi sudut, latar, dan cahaya.
- **Sumber:** foto sendiri atau berlisensi bebas. **Sumber dan lisensi tiap foto dicatat** di `data/README.md`.
- **Ukuran:** crop persegi dan resize ke 1024 × 1024, format `.jpg`.
- **Penamaan file:** `<kue>_<nomor 2 digit>.jpg`, contoh `klepon_01.jpg`, `dadar_gulung_07.jpg`.
- **Caption:** bahasa Inggris, diawali kalimat tetap `a photo of <kue>`, lalu deskripsi visual. Boleh dibantu BLIP, tapi **wajib dikoreksi manual**.
  ```
  a photo of klepon, green glutinous rice balls coated with grated coconut, on a banana leaf
  ```
- **Format `metadata.jsonl`:** satu baris per gambar.
  ```json
  {"file_name": "klepon_01.jpg", "text": "a photo of klepon, green glutinous rice balls coated with grated coconut, on a banana leaf"}
  ```
- **Struktur folder:**
  ```
  data/kue_v1/
  ├── train/
  │   ├── metadata.jsonl
  │   └── *.jpg
  └── holdout/          ← beberapa foto per kue, TIDAK ikut training, untuk referensi evaluasi
      └── *.jpg
  ```
- **Versi:** tiap kali dataset berubah, naikkan versi (`kue_v1`, `kue_v2`, ...) dan kirim zip baru ke Drive. Jangan menimpa versi lama.

## 5. Training (Zafira)

Titik awal hyperparameter. Perubahan dari nilai ini dicatat sebagai eksperimen.

| Parameter | Nilai awal |
|---|---|
| LoRA rank | 16 (alpha = rank, bawaan skrip) |
| Learning rate | 1e-4, scheduler constant |
| Batch size | 1, gradient accumulation 4 |
| Langkah | 1500–3000 |
| Presisi | bf16 |
| Hemat VRAM | gradient checkpointing, Adam 8-bit |
| VAE | `madebyollin/sdxl-vae-fp16-fix` |
| Seed | 42 |
| Validasi | prompt dan seed tetap, simpan gambar tiap 250–500 langkah |
| Log | TensorBoard |

- **Satu eksperimen = satu perubahan** dari baseline (misalnya rank 8/32, atau lr 5e-5), supaya pengaruhnya jelas.
- Hasil tiap run disimpan di `outputs/<nama_run>/` (tidak di-commit). Yang dibagikan lewat Drive: file LoRA `.safetensors`, kurva loss, galeri sampel per checkpoint, dan waktu training.
- Konfigurasi dan perintah tiap run **di-commit** di `training/` supaya bisa direplikasi.

## 6. Evaluasi (Celsyka, gambar dari Zafira)

- Daftar prompt tetap disusun Celsyka di `eval/prompts.json` (misalnya 4 kue × 3 prompt × 3 seed).
- Zafira generate gambar dengan prompt dan seed yang **sama persis** untuk SDXL polos dan SDXL + LoRA.
- Penamaan file gambar evaluasi: `<model>_<kue>_p<nomor prompt>_s<seed>.png`, contoh `lora_klepon_p1_s42.png`.
- Metrik:
  - Perbandingan visual berdampingan (base vs LoRA).
  - Survei Google Form (10–20 responden), kemiripan dengan kue asli skala 1–5.
  - CLIP score sebagai angka pendukung (opsional).
- Kekurangan ditulis dengan jujur, misalnya kue yang masih salah bentuk.

## 7. Deployment (Safira menulis kode, Zafira menjalankan)

- Load SDXL Base, lalu `pipe.load_lora_weights(...)`.
- Fitur UI: input prompt, dropdown jenis kue, slider steps dan guidance, input seed, dan **tampilan berdampingan base vs LoRA**.
- Path model dan LoRA dibaca dari argumen atau variabel di bagian atas file, **jangan di-hardcode** ke folder pribadi.
- Siapkan gambar cadangan untuk presentasi kalau demo lambat.

## 8. Lingkungan (PC Zafira)

1. **Python 3.11 atau 3.12** di virtual environment khusus proyek. Python 3.14 belum didukung sebagian library training.
2. **PyTorch dengan CUDA 12.8 (cu128)**, karena RTX seri 50 membutuhkannya. Perintah instal diambil dari [pytorch.org/get-started/locally](https://pytorch.org/get-started/locally/).
3. Versi `diffusers` harus cocok dengan skrip training yang dipakai. Skrip diambil dari tag rilis Diffusers yang sama dengan versi yang terinstal.
4. Semua versi paket yang benar-benar dipakai dicatat di `requirements.txt` setelah instalasi berhasil.

## 9. Serah-Terima File

File besar **tidak masuk Git**. Dibagikan lewat folder Google Drive kelompok:

| Dari → Ke | Isi |
|---|---|
| Safira → Zafira | `kue_vN.zip` (folder `train/` + `holdout/`) |
| Zafira → semua | LoRA `.safetensors`, galeri sampel, kurva loss, screenshot training |
| Zafira → Celsyka | Folder gambar evaluasi (base dan LoRA) |

Link folder Drive ditulis di `README.md`.

## 10. Struktur Repo

```
lora-kue-tradisional/
├── README.md
├── KONTRAK.md
├── requirements.txt        ← Zafira
├── data/                   ← isi tidak di-commit, hanya README.md (Safira)
├── scripts/                ← skrip bantu dataset (Safira)
├── training/               ← konfigurasi & perintah training (Zafira)
├── app/                    ← UI Gradio (Safira)
├── eval/                   ← prompt evaluasi, CLIP score, survei (Celsyka)
├── notebooks/              ← eksplorasi & visualisasi, output di-clear sebelum commit
├── outputs/                ← tidak di-commit
└── docs/
    ├── laporan/            ← laporan tutorial akhir (Celsyka)
    └── screenshot/         ← screenshot tiap langkah, dikumpulkan semua anggota
```

## 11. Alur Git

- Jangan push langsung ke `main`. Semua masuk lewat **Pull Request**, minimal dilihat satu anggota lain.
- Tiap orang commit dengan akun GitHub-nya sendiri supaya kontribusi terlihat.
- Urutan merge yang disarankan: `feat/setup` dan `feat/dataset` lebih dulu (bisa paralel), lalu `feat/baseline`, `feat/training`, kemudian `feat/demo` dan `feat/evaluasi`, terakhir `docs/laporan`.
- Format pesan commit:
  ```
  feat(dataset): tambah caption klepon dan serabi
  feat(training): tambah konfigurasi run baseline rank 16
  exp(training): eksperimen rank 32
  fix(app): perbaiki load LoRA di Gradio
  docs(laporan): tambah bab setup lingkungan
  ```
