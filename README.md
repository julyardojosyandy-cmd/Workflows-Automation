# 🧭 1. Gambaran Singkat

Bayangkan workflow ini seperti **editor majalah digital**:

1. **Seseorang menulis artikel** (AI atau kamu sendiri) → ini *Generate Konten*
2. **Dikirim ke pemimpin redaksi** via email untuk review → ini *Email Approval*
3. **Pemimpin redaksi klik tombol**: "Setuju" atau "Revisi" → ini *Wait + IF node*
4. Kalau **Revisi** → konten diperbaiki, lalu dikirim ulang untuk review
5. Kalau **Setuju** → artikel **otomatis naik ke semua platform** sekaligus
6. Setelah publish, **laporan dikirim** dan **data disimpan** di spreadsheet

Tidak ada yang perlu login ke Instagram/LinkedIn secara manual. Semua otomatis.

---

# 🛠️ 2. Step-by-Step Implementasi

---

## STEP 1 — Trigger (Pemicu Workflow)

**Node:** `Schedule Trigger` atau `Manual Trigger`
**Fungsi:** Memulai workflow — bisa terjadwal otomatis atau dipicu manual

**Cara setup (Schedule Trigger):**
- Di n8n, klik `+` → cari `Schedule Trigger`
- Field `Trigger Interval`: pilih `Days`
- Field `Days Between Triggers`: isi `1`
- Field `Trigger at Hour`: isi `9` (jam 9 pagi)
- Klik **Test step** untuk verifikasi

**Contoh input:** Tidak ada (trigger tidak butuh input, hanya waktu)

**Output:**
```json
{
  "executionId": "exec_001",
  "timestamp": "2025-01-15T09:00:00.000Z"
}
```

**Koneksi:** → STEP 2 (Set Variables)

---

## STEP 2 — Set Variables (Data Konten)

**Node:** `Set`
**Fungsi:** Mendefinisikan topik, nada, dan platform target sebelum generate

**Cara setup:**
- Tambah node `Set`
- Klik `Add Value` → pilih tipe `String`
- Tambah field-field berikut:

| Name | Value |
|---|---|
| `topic` | `Tips produktivitas kerja dari rumah` |
| `tone` | `informatif dan sedikit humoris` |
| `platforms` | `instagram,linkedin,facebook` |
| `content_type` | `AI` (atau `manual` untuk input manual) |
| `approval_id` | `={{ $now.toISO() }}_{{ $execution.id }}` |

**Contoh input:** (dari Step 1) timestamp trigger
**Output:** Semua variabel di atas tersedia untuk step berikutnya
**Koneksi:** → STEP 3 (IF: AI atau Manual?)

---

## STEP 3 — IF: AI atau Manual?

**Node:** `IF`
**Fungsi:** Memilih jalur generate konten — otomatis AI atau input manual

**Cara setup:**
- Tambah node `IF`
- Conditions: `{{ $json.content_type }}` **equals** `AI`
- Output TRUE → lanjut ke OpenAI
- Output FALSE → lanjut ke node tunggu input manual

**Contoh input:** `{ "content_type": "AI" }`
**Output:** Dua jalur: TRUE (AI) dan FALSE (manual)
**Koneksi:** TRUE → STEP 4A | FALSE → STEP 4B

---

## STEP 4A — Generate Konten via OpenAI

**Node:** `OpenAI` (atau `HTTP Request` ke OpenAI API)
**Fungsi:** Membuat draft konten social media menggunakan GPT

**Cara setup (gunakan node OpenAI native):**
- Tambah node `OpenAI`
- **Credential:** tambahkan API key OpenAI kamu
- **Resource:** `Text` → `Message a Model`
- **Model:** `gpt-4o` (atau `gpt-4o-mini` untuk hemat biaya)
- **Messages - Role:** `user`
- **Messages - Content:** (isi prompt — lihat Section 4 untuk prompt lengkap)

**Contoh prompt di field Content:**
```
Buat konten social media tentang: {{ $json.topic }}

Nada: {{ $json.tone }}

Output HARUS dalam format JSON seperti ini:
{
  "instagram": "Caption IG (max 2200 karakter, gunakan emoji, 5-10 hashtag)",
  "linkedin": "Post LI (profesional, max 1300 karakter, tanpa hashtag berlebihan)",
  "facebook": "Post FB (conversational, max 500 karakter)",
  "image_prompt": "Deskripsi visual untuk gambar pendukung"
}

Hanya output JSON saja, tanpa teks lain.
```

**Contoh output:**
```json
{
  "message": {
    "content": "{\"instagram\": \"Pernah ngerasa...\", \"linkedin\": \"5 tahun WFH...\", ...}"
  }
}
```

**Koneksi:** → STEP 5 (Parse JSON Output)

---

## STEP 4B — Input Manual (Jika Bukan AI)

**Node:** `Wait`
**Fungsi:** Menunggu konten diinput manual via form atau webhook

**Cara setup:**
- Tambah node `Wait`
- **Resume:** `On Webhook Call`
- Salin **Webhook URL** yang muncul
- Buat form sederhana (Google Form / Typeform) yang POST ke URL ini
- Field form: `instagram_text`, `linkedin_text`, `facebook_text`

**Koneksi:** → STEP 5 (Parse & Merge)

---

## STEP 5 — Parse & Standardisasi Konten

**Node:** `Code`
**Fungsi:** Mengubah output OpenAI (string JSON) menjadi object yang bisa dipakai

**Cara setup:**
- Tambah node `Code`
- Language: `JavaScript`
- Isi code:

```javascript
// Ambil output dari OpenAI
const rawContent = $input.first().json.message.content;

// Parse JSON dari string
let content;
try {
  content = JSON.parse(rawContent);
} catch(e) {
  // Fallback jika OpenAI menambah teks sebelum JSON
  const jsonMatch = rawContent.match(/\{[\s\S]*\}/);
  content = JSON.parse(jsonMatch[0]);
}

return [{
  json: {
    instagram_text: content.instagram,
    linkedin_text: content.linkedin,
    facebook_text: content.facebook,
    image_prompt: content.image_prompt,
    topic: $('Set Variables').first().json.topic,
    approval_id: $('Set Variables').first().json.approval_id,
    generated_at: new Date().toISOString(),
    status: 'pending'
  }
}];
```

**Contoh output:**
```json
{
  "instagram_text": "Kerja dari rumah? Ini 5 tips yang...",
  "linkedin_text": "Setelah 5 tahun WFH, saya belajar bahwa...",
  "facebook_text": "WFH bisa produktif asal kamu tahu caranya!",
  "approval_id": "2025-01-15T09:00:00_exec_001",
  "status": "pending"
}
```

**Koneksi:** → STEP 6 (Email Approval)

---

## STEP 6 — Kirim Email Approval

**Node:** `Gmail` (atau `Send Email` untuk SMTP)
**Fungsi:** Mengirim email berisi preview konten + tombol Approve/Revisi

**Cara setup (Gmail node):**
- Tambah node `Gmail`
- **Credential:** hubungkan akun Gmail kamu (OAuth2)
- **Resource:** `Message` → `Send`
- **To:** `approver@perusahaan.com`
- **Subject:** `[REVIEW] Konten Social Media - {{ $json.topic }}`
- **Message:** pilih `HTML`
- **HTML Body:** (lihat template lengkap di Section 5)

**Contoh HTML Body:**
```html
<h2>Review Konten Social Media</h2>
<p><strong>Topik:</strong> {{ $json.topic }}</p>
<p><strong>Generated:</strong> {{ $json.generated_at }}</p>

<hr>
<h3>Instagram:</h3>
<p>{{ $json.instagram_text }}</p>

<h3>LinkedIn:</h3>
<p>{{ $json.linkedin_text }}</p>

<h3>Facebook:</h3>
<p>{{ $json.facebook_text }}</p>
<hr>

<p>
  <a href="{{ $resumeWebhookUrl }}?status=approved" 
     style="background:#22c55e;color:white;padding:12px 24px;
            text-decoration:none;border-radius:6px;margin-right:10px">
    ✅ SETUJU - Publish Sekarang
  </a>
  
  <a href="{{ $resumeWebhookUrl }}?status=revise&note=TULIS_CATATAN_DI_SINI"
     style="background:#ef4444;color:white;padding:12px 24px;
            text-decoration:none;border-radius:6px">
    ✏️ PERLU REVISI
  </a>
</p>
```

> ⚠️ **Penting:** `$resumeWebhookUrl` adalah URL otomatis dari n8n. Harus ada node `Wait` **SETELAH** node Gmail ini — n8n baru generate URL ini kalau ada Wait node berikutnya.

**Output:** Email terkirim, workflow lanjut ke STEP 7
**Koneksi:** → STEP 7 (Wait node)

---

## STEP 7 — Tunggu Respon Approver

**Node:** `Wait`
**Fungsi:** Mem-pause workflow sampai approver klik link di email

**Cara setup:**
- Tambah node `Wait`
- **Resume:** `On Webhook Call`
- **Limit Wait Time:** aktifkan, set `3 Days` (agar tidak nunggu selamanya)
- **On Timeout:** lanjut ke node notifikasi timeout

**Contoh input:** (klik dari email) `?status=approved`
**Output:**
```json
{
  "query": {
    "status": "approved"
  }
}
```

**Koneksi:** → STEP 8 (IF Approved?)

---

## STEP 8 — IF: Approved atau Revisi?

**Node:** `IF`
**Fungsi:** Percabangan utama — menentukan apakah konten dipublish atau direvisi

**Cara setup:**
- Tambah node `IF`
- **Condition:**
  - Value 1: `{{ $json.query.status }}`
  - Operation: `equals`
  - Value 2: `approved`

**Output TRUE:** status = approved → lanjut publish
**Output FALSE:** status = revise → lanjut revisi
**Koneksi:** TRUE → STEP 9 (Publish) | FALSE → STEP 10 (Revisi)

---

## STEP 9 — Publish ke Instagram

**Node:** `HTTP Request`
**Fungsi:** Upload dan publish konten ke Instagram Business via Facebook Graph API

**Cara setup:**
- Tambah node `HTTP Request`
- **Method:** `POST`
- **URL:** `https://graph.facebook.com/v21.0/{{ $env.IG_USER_ID }}/media`
- **Authentication:** `Generic Credential Type` → Bearer Token (Facebook Access Token)
- **Body Type:** `JSON`
- **Body:**
```json
{
  "caption": "{{ $('Parse Konten').first().json.instagram_text }}",
  "image_url": "{{ $json.image_url }}",
  "access_token": "{{ $env.FB_ACCESS_TOKEN }}"
}
```

Setelah dapat `container_id` dari response, buat node kedua untuk publish:
- **URL:** `https://graph.facebook.com/v21.0/{{ $env.IG_USER_ID }}/media_publish`
- **Body:** `{ "creation_id": "{{ $json.id }}", "access_token": "..." }`

**Output:**
```json
{ "id": "17841234567890" }
```

**Koneksi:** → STEP 10 (Publish LinkedIn)

---

## STEP 10 — Publish ke LinkedIn

**Node:** `LinkedIn` (native n8n node)
**Fungsi:** Post konten ke LinkedIn personal atau company page

**Cara setup:**
- Tambah node `LinkedIn`
- **Credential:** Login via OAuth2
- **Resource:** `Post`
- **Person:** pilih akun/company kamu
- **Text:** `{{ $('Parse Konten').first().json.linkedin_text }}`
- **Visibility:** `PUBLIC`

**Output:**
```json
{ "id": "urn:li:share:123456789" }
```

**Koneksi:** → STEP 11 (Publish Facebook)

---

## STEP 11 — Publish ke Facebook

**Node:** `Facebook Graph API` atau `HTTP Request`
**Fungsi:** Post ke Facebook Page

**Cara setup:**
- Tambah node `HTTP Request`
- **Method:** `POST`
- **URL:** `https://graph.facebook.com/v21.0/{{ $env.FB_PAGE_ID }}/feed`
- **Body:**
```json
{
  "message": "{{ $('Parse Konten').first().json.facebook_text }}",
  "access_token": "{{ $env.FB_PAGE_ACCESS_TOKEN }}"
}
```

**Output:**
```json
{ "id": "123456789_987654321" }
```

**Koneksi:** → STEP 12 (Threads/Alternatif)

---

## STEP 12 — Publish ke Threads (via Ayrshare)

**Node:** `HTTP Request`
**Fungsi:** Karena Threads belum punya API publik yang stabil, gunakan **Ayrshare** (free tier: 3 platform, unlimited posts)

**Cara setup:**
- Daftar di [ayrshare.com](https://ayrshare.com) (free)
- Hubungkan akun Threads di dashboard Ayrshare
- Di n8n tambah `HTTP Request`:
- **Method:** `POST`
- **URL:** `https://app.ayrshare.com/api/post`
- **Headers:** `Authorization: Bearer YOUR_AYRSHARE_API_KEY`
- **Body:**
```json
{
  "post": "{{ $('Parse Konten').first().json.instagram_text }}",
  "platforms": ["threads"],
  "scheduleDate": ""
}
```

**Koneksi:** → STEP 13 (Agregat Hasil)

---

## STEP 13 — Kumpulkan Hasil Publish

**Node:** `Merge`
**Fungsi:** Menggabungkan result dari semua platform menjadi satu object

**Cara setup:**
- Tambah node `Merge`
- **Mode:** `Combine` → `Merge By Position`
- Hubungkan output dari IG, LinkedIn, FB, Threads ke node ini

**Output:**
```json
{
  "instagram_post_id": "17841234567890",
  "linkedin_post_id": "urn:li:share:123456789",
  "facebook_post_id": "123456789_987654321",
  "threads_post_id": "abc123",
  "published_at": "2025-01-15T09:05:00Z"
}
```

**Koneksi:** → STEP 14A (Email Report) dan STEP 14B (Google Sheets) — jalankan paralel

---

## STEP 14A — Kirim Email Report

**Node:** `Gmail`
**Fungsi:** Mengirim ringkasan hasil publish ke tim

**Cara setup:**
- **To:** `tim@perusahaan.com`
- **Subject:** `✅ Konten Berhasil Dipublish - {{ $now.format('dd MMM yyyy') }}`
- **HTML Body:**
```html
<h2>Laporan Publish Konten</h2>
<p>Konten tentang <strong>{{ $('Set Variables').first().json.topic }}</strong> 
   berhasil dipublish pada {{ $now.toISO() }}</p>

<h3>Detail per Platform:</h3>
<table border="1" cellpadding="8" style="border-collapse:collapse">
  <tr><th>Platform</th><th>Status</th><th>Post ID</th></tr>
  <tr><td>Instagram</td><td>✅ Published</td><td>{{ $json.instagram_post_id }}</td></tr>
  <tr><td>LinkedIn</td><td>✅ Published</td><td>{{ $json.linkedin_post_id }}</td></tr>
  <tr><td>Facebook</td><td>✅ Published</td><td>{{ $json.facebook_post_id }}</td></tr>
  <tr><td>Threads</td><td>✅ Published</td><td>{{ $json.threads_post_id }}</td></tr>
</table>
```

**Koneksi:** → End

---

## STEP 14B — Log ke Google Sheets

**Node:** `Google Sheets`
**Fungsi:** Menyimpan semua data untuk audit, analytics, dan reporting

**Cara setup:**
- Tambah node `Google Sheets`
- **Credential:** OAuth2 Google
- **Resource:** `Sheet` → `Append Row`
- **Document ID:** ID spreadsheet kamu (dari URL)
- **Sheet Name:** `Content Log`
- **Columns - Mapping Column Mode:** `Map Each Column Manually`

| Column | Value |
|---|---|
| Date | `={{ $now.format('yyyy-MM-dd') }}` |
| Topic | `={{ $('Set Variables').first().json.topic }}` |
| IG Post ID | `={{ $json.instagram_post_id }}` |
| LI Post ID | `={{ $json.linkedin_post_id }}` |
| FB Post ID | `={{ $json.facebook_post_id }}` |
| Status | `published` |
| Approval ID | `={{ $('Set Variables').first().json.approval_id }}` |

**Koneksi:** → End

---

## STEP 10 (REVISI PATH) — Revisi Konten

**Node:** `IF` (cek apakah revisi via AI atau manual)
**Fungsi:** Menentukan cara revisi

Setelah IF Approved? output FALSE, tambah node:

**IF Revisi via AI:**
```
Kondisi: $json.query.revision_mode equals 'ai'
```

Jika AI → tambah node `OpenAI` lagi dengan prompt:
```
Revisi konten berikut berdasarkan catatan ini:

Konten sebelumnya:
Instagram: {{ $('Parse Konten').first().json.instagram_text }}

Catatan revisi: {{ $json.query.note }}

Output format JSON yang sama seperti sebelumnya.
```

Jika Manual → kembali ke `Wait` node untuk input manual.

Setelah revisi selesai → sambungkan kembali ke **STEP 6 (Email Approval)** untuk dikirim ulang ke approver.

---

# 🔀 3. Logika Workflow

Ada dua percabangan utama:

**Percabangan 1: AI vs Manual**

```
content_type == "AI"
├── TRUE → OpenAI generate otomatis
└── FALSE → Wait untuk input manual via form
```

Contoh kasus: Kalau hari Senin kamu sudah tahu topiknya, set `content_type = AI` dan biarkan berjalan sendiri jam 9 pagi. Kalau ada event spesial yang butuh tone khusus, set `content_type = manual` — workflow akan pause dan tunggu kamu submit konten via form.

**Percabangan 2: Approve vs Revisi**

```
query.status == "approved"
├── TRUE → Publish ke semua platform
└── FALSE → Cek revision_mode
    ├── revision_mode == "ai" → OpenAI revisi otomatis
    └── revision_mode == "manual" → Wait input manual
        └── Keduanya → Loop kembali ke Email Approval
```

Contoh kasus nyata: Kamu dapat email approval jam 10 pagi. Kamu review konten Instagram-nya dan rasa kurang menarik. Kamu klik "PERLU REVISI" dengan catatan `"Opening kurang menarik, tambahkan statistik"`. n8n otomatis revisi via AI dengan catatan itu, lalu kirim ulang email approval untuk review kedua.

---

# 🤖 4. AI Generation (Praktis)

## Prompt Siap Pakai

**Untuk konten umum:**
```
Kamu adalah social media copywriter profesional untuk brand Indonesia.

Buat konten untuk topik: {{ $json.topic }}
Nada: {{ $json.tone }}
Target audiens: profesional muda 25-35 tahun

Aturan:
- Instagram: maks 2200 karakter, gunakan 3-5 emoji relevan, 8-10 hashtag campuran besar/kecil
- LinkedIn: maks 1300 karakter, profesional, awali dengan hook kuat, tambahkan 1-2 insight
- Facebook: maks 500 karakter, conversational, tanya pertanyaan di akhir untuk engagement

Output HANYA JSON tanpa teks lain:
{
  "instagram": "...",
  "linkedin": "...", 
  "facebook": "...",
  "image_prompt": "deskripsi visual untuk Canva/Midjourney"
}
```

## Setup di n8n

**Opsi 1 — Native OpenAI Node (paling mudah):**
- Cari `OpenAI` di node list
- Masukkan API key
- Model: `gpt-4o-mini` (hemat, cukup untuk copywriting)
- Prompt masuk di field `User Message`

**Opsi 2 — HTTP Request (lebih fleksibel, bisa ganti model):**
```
Method: POST
URL: https://api.openai.com/v1/chat/completions
Headers:
  Authorization: Bearer {{ $env.OPENAI_API_KEY }}
  Content-Type: application/json
Body:
{
  "model": "gpt-4o-mini",
  "messages": [
    {"role": "system", "content": "Kamu adalah social media copywriter profesional."},
    {"role": "user", "content": "{{ $json.prompt }}"}
  ],
  "temperature": 0.7,
  "max_tokens": 1500
}
```

**Opsi 3 — Claude via Anthropic API:**
```
URL: https://api.anthropic.com/v1/messages
Headers:
  x-api-key: {{ $env.ANTHROPIC_API_KEY }}
  anthropic-version: 2023-06-01
Body:
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 1500,
  "messages": [{"role": "user", "content": "{{ $json.prompt }}"}]
}
```

---

# 📧 5. Email Approval System

## Cara Kerja Sistem Approval n8n

Kunci sistemnya adalah **`$resumeWebhookUrl`** — URL unik yang di-generate n8n untuk setiap eksekusi. Cara kerjanya:

1. n8n generate URL unik, misal: `https://your-n8n.app.n8n.cloud/webhook/abc123/resume`
2. URL ini disisipkan di email sebagai tombol
3. Approver klik tombol → GET request ke URL tersebut
4. n8n menerima request → workflow lanjut dari titik Wait
5. Data dari URL (query params) tersedia sebagai `$json.query`

## Template Email Approval Lengkap

```html
<!DOCTYPE html>
<html>
<body style="font-family:Arial,sans-serif;max-width:600px;margin:0 auto">

<div style="background:#1e40af;color:white;padding:20px;border-radius:8px 8px 0 0">
  <h2 style="margin:0">📋 Review Konten Social Media</h2>
  <p style="margin:4px 0 0">Menunggu persetujuan Anda</p>
</div>

<div style="background:#f8fafc;padding:20px">
  <table style="width:100%;border-collapse:collapse">
    <tr><td style="padding:4px"><strong>Topik:</strong></td>
        <td>{{ $('Set Variables').first().json.topic }}</td></tr>
    <tr><td style="padding:4px"><strong>Dibuat:</strong></td>
        <td>{{ $now.toISO() }}</td></tr>
    <tr><td style="padding:4px"><strong>ID:</strong></td>
        <td>{{ $json.approval_id }}</td></tr>
  </table>
</div>

<div style="padding:20px">
  <h3 style="color:#1e40af">Instagram</h3>
  <div style="background:#fafafa;border:1px solid #e2e8f0;border-radius:6px;padding:12px">
    <p>{{ $json.instagram_text }}</p>
  </div>

  <h3 style="color:#0077b5">LinkedIn</h3>
  <div style="background:#fafafa;border:1px solid #e2e8f0;border-radius:6px;padding:12px">
    <p>{{ $json.linkedin_text }}</p>
  </div>

  <h3 style="color:#1877f2">Facebook</h3>
  <div style="background:#fafafa;border:1px solid #e2e8f0;border-radius:6px;padding:12px">
    <p>{{ $json.facebook_text }}</p>
  </div>
</div>

<div style="padding:20px;text-align:center;background:#f1f5f9;border-top:1px solid #e2e8f0">
  <p style="margin:0 0 16px">Pilih tindakan:</p>
  
  <a href="{{ $resumeWebhookUrl }}?status=approved"
     style="display:inline-block;background:#16a34a;color:white;padding:14px 28px;
            text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;
            margin:0 8px">
    ✅ SETUJU & PUBLISH
  </a>
  
  <a href="{{ $resumeWebhookUrl }}?status=revise&revision_mode=ai&note=Perlu+diperbaiki"
     style="display:inline-block;background:#dc2626;color:white;padding:14px 28px;
            text-decoration:none;border-radius:8px;font-weight:bold;font-size:16px;
            margin:0 8px">
    ✏️ PERLU REVISI
  </a>
</div>

</body>
</html>
```

> **Tips:** Untuk catatan revisi yang spesifik, approver bisa edit URL manual sebelum klik — ubah bagian `note=Perlu+diperbaiki` dengan catatan yang diinginkan. Atau tambahkan form revisi sederhana (lihat alternatif di bawah).

## Alternatif: Form Revisi via Typeform/Tally

Jika ingin approver bisa tulis catatan bebas:
1. Buat form gratis di [tally.so](https://tally.so) dengan field: `Note`, `Revision Mode` (AI/Manual)
2. Form submission di-POST ke webhook n8n
3. Di email, link "Revisi" → form Tally (bukan langsung ke n8n)
4. Setelah submit → Tally forward data ke n8n webhook

---

# 📤 6. Publishing per Platform

## Instagram (via Facebook Graph API)

Butuh: Instagram Business Account + Facebook Page

**Cara paling realistis:** Dua step HTTP Request

```
Step 1 — Buat container:
POST https://graph.facebook.com/v21.0/{ig-user-id}/media

Body:
{
  "caption": "{{ $json.instagram_text }}",
  "image_url": "URL gambar yang sudah diupload ke server/S3",
  "access_token": "{{ $env.FB_TOKEN }}"
}
→ Dapat: { "id": "container_id" }

Step 2 — Publish container:
POST https://graph.facebook.com/v21.0/{ig-user-id}/media_publish

Body:
{
  "creation_id": "{{ $json.id }}",
  "access_token": "{{ $env.FB_TOKEN }}"
}
```

**Alternatif lebih mudah:** Gunakan **Ayrshare** (mendukung IG, gratis 3 platform) — satu HTTP Request saja, tidak perlu urus container.

## LinkedIn

n8n punya native LinkedIn node. Setup:
- Credential: OAuth2 (login di n8n)
- Resource: `Post` → `Create`
- Text: isi konten LinkedIn kamu
- Visibility: `PUBLIC`

Jika ingin posting ke Company Page, tambahkan `organizationId` di settings.

## Facebook Page

```
POST https://graph.facebook.com/v21.0/{page-id}/feed

Body:
{
  "message": "{{ $json.facebook_text }}",
  "access_token": "{{ $env.FB_PAGE_TOKEN }}"
}
```

**Cara dapat Page Access Token:** Facebook Developer → App → Tools → Graph API Explorer → pilih Page kamu → generate token (set expiry long-lived via token exchange).

## Threads (Alternatif)

| Opsi | Cara | Biaya |
|---|---|---|
| **Ayrshare** | HTTP Request ke API-nya | Free (3 platform) / $29/bln |
| **Buffer** | Schedule via Buffer API | Free (3 channel) |
| **Later** | Auto-publish via Later | Free tier ada |
| **Manual notif** | n8n kirim WhatsApp/Telegram reminder untuk post manual | Free |

**Rekomendasi:** Gunakan Ayrshare untuk semua platform (IG, LinkedIn, FB, Threads) dalam satu HTTP Request — jauh lebih sederhana dari mengurus API masing-masing platform.

---

# 📊 7. Logging ke Google Sheets

## Struktur Kolom yang Direkomendasikan

| Kolom | Tipe | Isi |
|---|---|---|
| `Date` | Date | Tanggal publish |
| `Approval_ID` | Text | ID unik per konten |
| `Topic` | Text | Topik konten |
| `Content_Type` | Text | AI / Manual |
| `IG_Text_Preview` | Text | 100 karakter pertama |
| `LI_Text_Preview` | Text | 100 karakter pertama |
| `FB_Text_Preview` | Text | 100 karakter pertama |
| `IG_Post_ID` | Text | ID dari Instagram |
| `LI_Post_ID` | Text | ID dari LinkedIn |
| `FB_Post_ID` | Text | ID dari Facebook |
| `Publish_Status` | Text | published / failed |
| `Approval_Time` | DateTime | Kapan di-approve |
| `Revision_Count` | Number | Berapa kali direvisi |
| `Error_Notes` | Text | Catatan error jika ada |

## Setup di n8n

```
Node: Google Sheets
Resource: Sheet In Spreadsheet → Append Row
Document: [pilih spreadsheet kamu]
Sheet: Content Log
Column Mapping:
  Date         = {{ $now.format('yyyy-MM-dd HH:mm') }}
  Approval_ID  = {{ $('Set Variables').first().json.approval_id }}
  Topic        = {{ $('Set Variables').first().json.topic }}
  IG_Post_ID   = {{ $('Publish Instagram').first().json.id }}
  LI_Post_ID   = {{ $('Publish LinkedIn').first().json.id }}
  Status       = published
```

---

# ⚠️ 8. Error Handling

## Error Umum dan Cara Handle

**Error 1: OpenAI timeout atau rate limit**
- Penyebab: request terlalu banyak atau API lambat
- Solusi: Di node OpenAI → tab `Settings` → aktifkan `Retry On Fail` → set `Max Tries: 3`, `Wait Between Tries: 5000ms`

**Error 2: Facebook token expired**
- Penyebab: Access token FB expired (biasanya 60 hari)
- Solusi: Buat token long-lived via Facebook API → simpan di n8n Credentials/Environment Variables → set reminder calendar untuk renew

**Error 3: Instagram media publish gagal**
- Penyebab: URL gambar tidak bisa diakses FB, atau format salah
- Solusi: Tambah node `IF` setelah Step 1 Instagram (buat container). Cek apakah response punya `id`. Jika tidak → kirim Slack/email notifikasi error

**Error 4: Wait node timeout (approver tidak klik)**
- Penyebab: Approver lupa/tidak sempat review
- Solusi: Di Wait node → aktifkan `Limit Wait Time: 3 days` → sambungkan output timeout ke Gmail node untuk kirim pengingat

**Error 5: Google Sheets auth gagal**
- Penyebab: OAuth token refresh fail
- Solusi: Gunakan Service Account (bukan OAuth2) untuk koneksi Sheets yang lebih stabil — download JSON key dari Google Cloud Console

## Pattern Error Handling Global

Tambah ini di setiap node kritis (IG, LinkedIn, FB):
- Di node settings → `Error Output` → aktifkan
- Sambungkan error output ke node `Set` yang set `status = "failed"` dan `platform = "instagram"`
- Lalu sambungkan ke Google Sheets untuk log error
- Dan ke Gmail untuk notifikasi: `"Publish ke {{ $json.platform }} GAGAL: {{ $json.error.message }}"`

---

# 💡 9. Kenapa Workflow Dibuat Seperti Ini?

**Analogi sederhana:** Bayangkan kamu punya **mesin cetak otomatis di pabrik**, tapi dengan tombol QC (quality control) manual di tengahnya.

Mesin bisa cetak sendiri (AI), tapi sebelum barang keluar ke toko (publish), ada inspektur (approver) yang harus tekan tombol "OK". Kalau ada cacat, barang dikembalikan ke mesin untuk diperbaiki — bukan dibuang, bukan dimulai dari nol.

Kenapa dibuat begini? Karena tiga alasan:

Pertama, **keamanan konten.** AI bagus tapi tidak sempurna. Satu konten yang salah di LinkedIn bisa merusak reputasi dalam hitungan menit. Human-in-the-loop (approver) adalah net safety yang tidak mahal.

Kedua, **modularitas.** Setiap step bisa diswap tanpa ngaruhin step lain. Mau ganti dari OpenAI ke Claude? Hanya ubah satu node. Mau tambah platform TikTok? Tambah satu HTTP Request node, tidak perlu rebuild dari nol.

Ketiga, **audit trail.** Google Sheets sebagai log memungkinkan kamu melihat: konten apa yang paling sering direvisi? Platform mana yang paling sering error? Approval rata-rata butuh berapa jam? Data ini mahal harganya untuk keputusan bisnis ke depan.

---

# ✅ 10. Checklist Implementasi

### Fase 0 — Persiapan Akun dan Credential
- [ ] Buat akun n8n Cloud (gratis 14 hari) atau self-host via Docker
- [ ] Siapkan OpenAI API key (simpan di n8n → Settings → Environment Variables sebagai `OPENAI_API_KEY`)
- [ ] Hubungkan akun Gmail di n8n Credentials
- [ ] Buat Facebook Developer App → dapatkan Page Access Token
- [ ] Pastikan Instagram Business Account terhubung ke Facebook Page
- [ ] Hubungkan Google account untuk Google Sheets
- [ ] (Opsional) Daftar Ayrshare jika ingin support Threads

### Fase 1 — Build Core Workflow
- [ ] Buat workflow baru di n8n
- [ ] Tambah `Schedule Trigger` → test berfungsi
- [ ] Tambah `Set Variables` → test output muncul benar
- [ ] Tambah `IF` (AI vs Manual) → test kedua jalur
- [ ] Tambah `OpenAI` node → test prompt → parse JSON output
- [ ] Tambah `Code` node untuk parse JSON → verifikasi semua field ada

### Fase 2 — Approval System
- [ ] Tambah `Gmail` node → test email terkirim
- [ ] Tambah `Wait` node → verify `$resumeWebhookUrl` muncul di email
- [ ] Klik link Approve dari email → verify workflow resume
- [ ] Klik link Revisi → verify jalur revisi aktif
- [ ] Test loop: revisi → email ulang → approve

### Fase 3 — Publishing
- [ ] Test publish ke Facebook (paling mudah) → verify post muncul
- [ ] Test publish ke LinkedIn → verify post muncul
- [ ] Test publish ke Instagram (Step 1: container, Step 2: publish)
- [ ] Setup Ayrshare untuk Threads (opsional)
- [ ] Tambah error handling di setiap publish node

### Fase 4 — Reporting & Logging
- [ ] Buat Google Sheets dengan kolom sesuai Section 7
- [ ] Test `Google Sheets` node → verify row ditambahkan
- [ ] Test email report → verify format dan isi benar
- [ ] Jalankan full workflow end-to-end sekali → cek semua step

### Fase 5 — Production Hardening
- [ ] Aktifkan `Retry on Fail` di semua HTTP Request node
- [ ] Tambah `Error Output` di semua node kritis → log ke Sheets
- [ ] Set `Wait` timeout 3 hari + pengingat email
- [ ] Test timeout scenario
- [ ] Aktifkan workflow (toggle `Active` di n8n)
