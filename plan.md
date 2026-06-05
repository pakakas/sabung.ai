# sabung.ai — Plan

## Konsep

Dua AI agent saling berdebat atas suatu topik yang diinput oleh manusia.
Debat berjalan real-time — output ditampilkan di samping (Web UI) seiring proses berjalan.

```
Human → [topik] → Web UI / CLI / Agent Tool
                      ↓
                  Asisten (orchestrator)
                      ↓
              spawn Lover + Hater
                      ↓
              Lover → argumen pro → [output]
                      ↓
              Hater → kritik keras → [output]
                      ↓
              Lover → bantahan → [output]
                      ↓
              [ulang] ... sampai ada yang "capek"

[output] = Web UI (real-time) / sosmed / file / stdout / audio / visual
```

**Live stream**: Setiap ronde langsung dikirim ke output saat agent menghasilkan respons — tidak perlu tunggu selesai.

### Ekstensi Audio & Visual (Opsional)

| Format | Keterangan | Tool |
| :--- | :--- | :--- |
| **Audio** | Debate jadi file audio — dua suara berbeda, natural flow, ada jeda & intonasi | ElevenLabs / OpenAI TTS / Google TTS |
| **Video** | Debate jadi video — avatar bicara, subtitle sync | HeyGen / Synthesia / Kling |
| **Visual** | Infographic debate, card per ronde | Canvas / HTML render |

- **Audio & video sepenuhnya opsional** — diaktifkan via config, butuh API key pihak ketiga
- Audio: bukan sekedar TTS per paragraf — harus terdengar seperti dua orang lagi debat di podcast, ada jeda, intonasi, emosi
- Bisa tambah intro/outro, background music, sound effect ringan
- Video: pakai AI video generation (HeyGen/Synthesia) — langsung jadi, tidak perlu compose manual. Avatar + suara + subtitle sekaligus
- Format video: landscape 16:9 (YouTube) atau portrait 9:16 (TikTok/Reels)
- Visual: render card gambar per ronde (quote + avatar)
- Semua output tambahan, bukan pengganti text

---

## Arsitektur

```
Web UI / CLI / Agent Tool
    ↓
Asisten (orchestrator)
├── spawn → Subagent: Lover
├── spawn → Subagent: Hater
└── output → Web UI (real-time) / sosmed (opsional) / file / stdout
```

- **Asisten** adalah orchestrator yang menerima topik dari Web UI, CLI, atau Agent Tool, lalu men-spawn dua subagent
- **Input** bisa datang dari tiga sumber (semuanya masuk ke Asisten yang sama):
  - **Web UI** → human input topik langsung via browser
  - **CLI** → memanggil `sabung.ai "<topik>"` via terminal
  - **Agent Tool** → AI Asisten lain memanggil sabung.ai sebagai tool
- **Lover** dan **Hater** adalah subagent independen dengan kepribadian/system prompt masing-masing
- Keduanya berkomunikasi **melalui orchestrator** — tidak ada komunikasi langsung antar subagent
- **Sosmed opsional** — posting ke sosmed hanya aktif jika dikonfigurasi

---

## Aktor / Subagent

| Aktor           | Peran                                          |
| :-------------- | :--------------------------------------------- |
| **Lover** 🩷    | Menerima topik dari orchestrator, menyusun argumen pro |
| **Hater** 🖤    | Menerima argumen dari orchestrator, menyusun kritik keras |

---

## Alur Kerja

### Langkah 0 — Setup
- Konfigurasi karakter Lover dan Hater (system prompt, nama)
- **Validasi karakter Hater:** agent memvalidasi konfigurasi karakter Hater sebelum debat dimulai — memastikan karakter yang didefinisikan user benar-benar masuk kriteria "hater". Jika tidak valid, debat tidak dimulai
- (Opsional) Konfigurasi akun sosmed jika ingin post ke platform publik

### Langkah 1 — Input Topik

Topik masuk ke Asisten dari salah satu sumber:

**Via Web UI:**
- Human membuka browser, mengetik topik di form input, klik **"Mulai Duel"**

**Via CLI:**
```
sabung.ai "TypeScript lebih baik dari JavaScript"
```

**Via Agent Tool:**
- AI Asisten lain memanggil sabung.ai sebagai tool dengan parameter topik

Semua masuk ke Asisten yang sama → spawn Lover + Hater.

### Langkah 2 — Lover Argumen
- Orchestrator mengirim topik ke Lover
- Lover menyusun argumen **pro/pendukung** menggunakan LLM
- Output langsung dikirim ke channel (Web UI real-time / sosmed / file)

### Langkah 3 — Hater Merespons
- Orchestrator mengirim argumen Lover ke Hater
- Hater menyusun **kritik keras dan tajam** sesuai karakter yang dikonfigurasi
- Output langsung dikirim ke channel

### Langkah 4 — Lover Membantah
- Orchestrator mengirim kritik Hater ke Lover
- Lover menyusun **bantahan** yang argumentatif menggunakan LLM
- Output langsung dikirim ke channel

### Langkah 5 — Pengulangan
- Langkah 3 dan 4 diulang secara bergantian (loop)
- Setiap ronde langsung di-output real-time
- Loop berhenti ketika:
  - Salah satu agent menghasilkan post yang mengandung sinyal "kesimpulan" / "mengalah"
  - Atau setelah mencapai batas maksimum putaran (misalnya 5 ronde)

---

## Kondisi Berhenti ("Yang Capek")

Agent dianggap "capek" (mengakhiri debat) apabila LLM-nya menghasilkan respons yang:
- Mengakui kekalahan ("Kamu benar, aku salah tentang X...")
- Menarik kesimpulan yang merangkum perdebatan
- Mengindikasikan tidak ada argumen baru yang bisa disampaikan

Alternatif: Setelah N putaran, sistem otomatis meminta LLM untuk membuat **summary akhir** dari perdebatan.

---

## Platform Sosmed (Opsional)

Jika `post_mode=sosmed`, debat bisa dipublikasikan ke platform berikut:

| Opsi          | Kelebihan                                 | Kekurangan                     |
| :------------ | :---------------------------------------- | :----------------------------- |
| **Bluesky**   | API terbuka (AT Protocol), gratis, modern | Belum sepopuler X              |
| **Mastodon**  | Open source, API mudah, self-host option  | Komunitas lebih niche          |
| **X/Twitter** | Platform terbesar                         | API mahal dan terbatas         |

**Dikonfigurasi via `.env`:** `SOCIAL_PLATFORM=bluesky` / `mastodon` / `x`

---

## Stack Teknis

- **Input dari:** Web UI / CLI / Agent Tool — semua masuk ke Asisten
- **Runtime:** Bun (TypeScript)
- **LLM Provider:** Configurable via `.env` (OpenAI, xAI Grok, Google Gemini, Anthropic Claude)
- **Social Media SDK:** Configurable — `@atproto/api` (Bluesky), Mastodon REST API, atau X API v2
- **Mode Output:** JSON by default, `--ascii` untuk output human-readable di terminal
- **Entry Point CLI:** `sabung.ai "<topik>"`
- **Entry Point Web:** HTTP server yang menerima POST `{ topic: string }`

### Config: Mode Post

Dikonfigurasi via `.env` atau flag CLI:

| Mode | Keterangan |
| :--- | :--- |
| `post_mode=sosmed` | Post ke akun sosmed (Bluesky/Mastodon/X) |
| `post_mode=local` | Tulis ke file lokal (`output/debate-<timestamp>.jsonl`) — tanpa akun sosmed |
| `post_mode=off` | Jalankan debate saja, output ke stdout — tanpa post ke mana pun |

Mode `local` berguna untuk development/testing. Mode `off` berguna untuk sekedar lihat hasil debate tanpa simpan.

**Sosmed sepenuhnya opsional** — sabung.ai bisa jalan tanpa akun sosmed sama sekali. Fitur sosmed (post, baca-timeline, mention) bisa diaktif/dinaktifkan via config.

### Config: Mode Duel

| Mode | Keterangan |
| :--- | :--- |
| `duel_mode=live` | Debate real-time di sosmed — setiap post langsung publik, penonton bisa nimbrung |
| `duel_mode=batch` | Generate semua ronde dulu di lokal, baru post sekaligus / dijadwal |

---

## Dashboard & Settings (Web UI)

### Dashboard

Halaman utama user setelah login:

| Section | Fungsi |
| :--- | :--- |
| **Saldo** | Sisa saldo, riwayat top up |
| **Duel History** | Daftar duel sebelumnya — topic, winner, jumlah ronde, link post sosmed |
| **Mulai Duel** | Form input topik, pilih karakter, pilih provider, mulai |
| **Live Duel** | View duel yang sedang berjalan — stream real-time via WebSocket |

### Settings

| Setting | Keterangan |
| :--- | :--- |
| **Profil** | Nama, email, avatar |
| **LLM Provider** | Default provider (Gemini/Grok/Claude/GPT-4), auto-pilih termurah |
| **Social Media** | Connect/disconnect akun sosmed (Bluesky/Mastodon/X), toggle on/off |
| **Duel Mode** | live / batch |
| **Karakter Custom** | Buat/edit karakter fighter — system prompt, nama, avatar (upload / AI-generated / preset) |
| **Top Up** | Tambah saldo (payment gateway integration) |
| **API Keys** | Generate/manage API key untuk akses programmatic |
| **Webhook** | URL callback untuk notifikasi duel selesai |

---

## Struktur Direktori

```
pakakas/sabung.ai/
├── plan.md               ← dokumen ini
├── index.ts              ← entry point CLI
├── server.ts             ← entry point Web (HTTP server + WebSocket)
├── web/                  ← Web UI (dashboard, live duel view, settings)
├── agents/
│   ├── lover.ts          ← subagent Lover
│   └── hater.ts          ← subagent Hater
├── output/               ← debate output (post_mode=local)
└── package.json
```

---

## Prompts

Prompt yang dikirim ke masing-masing subagent per ronde sangat minimal:

**Hater** (menerima post dari Lover):
```
tuh, debatin

<isi post Lover>
```

**Lover** (menerima kritik dari Hater):
```
ada yg debatin nih

<isi post Hater>
```

Kepribadian masing-masing agent ditentukan oleh **system prompt** mereka (`prompts/lover-system.txt` dan `prompts/hater-system.txt`), bukan dari user prompt per ronde.

---


## Contoh Output CLI (`--ascii`)

**Tanpa sosmed (default):**
```
sabung.ai ⚔️
Topic    : "TypeScript lebih baik dari JavaScript"
Mode     : local
Rounds   : max 5

─────────────────────────────────────────────
🩷 [Lover] Round 1
  "TypeScript memberikan type safety yang membuat kode lebih..."

🖤 [Hater] Round 1
  "Hahaha, type safety? Itu hanya mempersulit hidup developer..."

🩷 [Lover] Round 2
  "Justru type safety mengurangi bug di production sebesar..."

🖤 [Hater] Round 2 ← MENYERAH
  "Oke, kamu punya poin soal bug di production. Tapi..."

─────────────────────────────────────────────
🏁 SELESAI — Hater menyerah setelah 2 ronde.
→ Saved: output/debate-20260606.jsonl
```

**Dengan sosmed (`post_mode=sosmed`):**
```
🩷 [Lover] Round 1
  "TypeScript memberikan type safety..."
  → Posted: https://bsky.app/profile/hyuze-lover/post/xyz

🖤 [Hater] Round 1
  "Hahaha, type safety? Itu hanya mempersulit..."
  → Posted: https://bsky.app/profile/hyuze-hater/post/abc
```

---

## Risiko

### TOS / Platform Risk
- Sebagian besar platform sosmed melarang akun bot yang tidak diberi label jelas sebagai bot
- Akun yang terdeteksi sebagai bot otomatis bisa di-suspend, menghapus seluruh riwayat debat
- Beberapa platform membatasi frekuensi posting otomatis (rate limit)
- Perlu dikaji lebih lanjut: apakah platform yang dipilih mengizinkan bot, dan seperti apa syaratnya

### Ethical / Social Risk
- Pengguna lain di timeline tidak tahu bahwa akun tersebut adalah bot — berpotensi mislead
- Debat artifisial bisa memancing reaksi emosional dari manusia yang tidak sadar itu AI
- Konten yang dihasilkan Hater (kritik keras) berpotensi dianggap sebagai ujaran yang menyinggung

> **Catatan:** Risiko-risiko ini belum diselesaikan. Plan ini dipublikasikan sebagai ide — tanpa rencana implementasi untuk saat ini.

---

## Business Plan

### Cost Structure

Setiap duel mengonsumsi token LLM di kedua sisi (Lover + Hater). Estimasi per duel:

| Komponen | Estimasi Token/Ronde | x 5 Ronde |
| :--- | :--- | :--- |
| System prompt (per agent) | ~500 | sekali |
| User prompt + context | ~200 | x5 |
| LLM response | ~300 | x5 |
| **Total per agent per duel** | | **~3,000** |
| **Total duel (2 agent)** | | **~6,000** |

Biaya tergantung provider:
- Grok: ~$0.30-0.60/duel (tergantung model)
- Gemini Flash: ~$0.05-0.10/duel
- Claude/GPT-4: ~$0.30-1.00/duel

### Monetisasi

| Model | Harga | Target |
| :--- | :--- | :--- |
| **Free tier** | 1 duel/hari, 3 ronde max, post ke local only | Casual users, demo |
| **Saldo/token** | Top up mulai Rp 50.000, potong per duel sesuai provider | Fleksibel, pay-as-you-go |
| **Pay per duel** | Rp 5.000 - 15.000 / duel | One-time users, trial |
| **Subscription** | Rp 99.000/bulan (20 duel) | Content creators, enthusiast |
| **API access** | Rp 500.000/bulan (100 duel + webhook) | Developer, integrator |
| **Custom characters** | Rp 25.000/karakter | Personalisasi fighter |

### Token/Saldo System

User top up saldo, sistem potong per duel berdasarkan biaya aktual provider + margin:

| Provider | Biaya Duel | Harga Jual | Margin |
| :--- | :--- | :--- | :--- |
| Gemini Flash | ~Rp 1.100 | Rp 5.000 | ~4.5x |
| Grok | ~Rp 5.000 | Rp 10.000 | ~2x |
| Claude/GPT-4 | ~Rp 8.000 | Rp 15.000 | ~1.9x |

- User pilih provider saat mulai duel (atau sistem otomatis pilih termurah)
- Saldo tidak hangus, berlaku selamanya
- Minimum top up Rp 50.000 (~10 duel termurah)

### Target Market

| Segment | Kebutuhan | Volume |
| :--- | :--- | :--- |
| **Content creator** | Konten viral, engagement, hiburan | Tinggi |
| **Social media manager** | Auto-generate debate content untuk brand | Sedang |
| **Developer/hacker** | API untuk integrasi, experiment | Sedang |
| **Pengguna casual** | Hiburan, iseng adu argumen | Tinggi |

### Value Proposition

- **Untuk creator**: Konten debate otomatis tanpa perlu mikir ide — tinggal kasih topik, biarkan AI berdebat, post ke sosmed
- **Untuk developer**: API sederhana untuk spawn debate AI yang bisa diintegrasikan ke app lain
- **Untuk hiburan**: Tontonan seru AI saling adu argumen di timeline publik

### Growth Strategy

1. **Phase 1 — Local mode**: Build & test tanpa sosmed, pastikan kualitas debate menarik
2. **Phase 2 — Bot di Bluesky/Mastodon**: Launch akun bot, post debate publik, kumpul audience organik
3. **Phase 3 — Web UI + monetisasi**: Buka akses publik via web, terapkan free tier + pay per duel
4. **Phase 4 — API + custom characters**: Buka API untuk developer, jual custom fighter

### Break-even Target

Asumsi provider termurah (Gemini Flash, ~$0.07/duel):
- 1,000 duel/bulan = ~$70 biaya LLM
- Butuh ~50 subscriber @ Rp 99rb atau ~1,400 pay-per-duel @ Rp 5rb untuk BEP di bulan pertama
