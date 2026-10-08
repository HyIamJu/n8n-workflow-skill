# 01 — Tabel Keputusan (SCOPING GATE + DEFAULTS)

Buka file ini saat STEP 1 (scoping) atau saat ragu memilih sesuatu.

## STEP 1 — SCOPING GATE

Jawab satu baris per pertanyaan:

- **Q0: Endpoint publik atau butuh auth?** Default: **butuh auth**. Publik harus ada alasan eksplisit dari user (mis. health check, webhook dari gateway terpercaya yang sudah punya auth sendiri).
- **Q1: Proses bisa > 10 detik / berat?** (file besar, bulk import, chain approval, panggil LLM). Ya → ASYNC202.
- **Q2: Logic dipakai ulang di >= 2 endpoint berbeda?** Ya → LIBRARY (jadikan sub-workflow / SQL function dipanggil beberapa endpoint).
- **Q3: Ada efek samping non-krusial?** (log DB, notifikasi, IoT). Daftar → Fire Forget SETELAH Respond.
- **Q4: Environment + sensitivitas?** uat/prod (resource di-tag env), dan apakah payload menyimpan PII (menentukan apa yang boleh masuk api_logs).

Hasil ditulis:

```
AUTH = WAJIB-JWT | WAJIB-SQL | WAJIB-HEADER | PUBLIK
PATTERN = SYNC | ASYNC202 | LIBRARY
SIDEEFFECTS = <daftar> | tidak ada
ENV = uat | prod | keduanya
```

## TABEL AUTH — pilih mode verifikasi

Webhook node n8n punya auth bawaan: **Basic auth, Header auth, JWT auth, None** (diverifikasi SEBELUM workflow jalan).

| Mode | Cara | Kapan | Catatan |
|---|---|---|---|
| WAJIB-JWT | Webhook node → Authentication: JWT Auth (credential) | Token diterbitkan gateway/IAM; signature diverifikasi di edge, nol query DB | Penolakan terjadi sebelum workflow → bentuk 401 bawaan n8n, BUKAN amplop kita. Kalau kontrak amplop wajib konsisten sampai 401, pakai WAJIB-SQL. |
| WAJIB-HEADER | Webhook node → Header Auth credential | API key statis gateway→n8n (internal service) | Sama catatan di atas. |
| WAJIB-SQL | Node Verify Auth → `SELECT * FROM fn_verify_token($1)` (TEMPLATE A0) | Sesi token di tabel PG; butuh kontrol revoke/expiry; atau butuh response 401 berbentuk amplop | Token dihapus/di-revoke terlihat real-time; tambah 1 round-trip DB. |
| PUBLIK | Tanpa auth | Wajib alasan eksplisit user di STEP 1 | Rate limit & proteksi jadi tanggung jawab gateway. |

Untuk semua mode berauth: `user_id` hanya boleh berasal dari sumber terverifikasi (klaim JWT yang sudah lolos verifikasi node Webhook, atau output Verify Auth) — tidak pernah dari body/query/path.

## TABEL PATTERN

| Pattern | Cirinya | Bentuk |
|---|---|---|
| SYNC (default) | Selesai < 10s, ringan | Parent+Child, respond langsung |
| ASYNC202 | Berat/lama, client cukup tahu "diterima" | INSERT job (queued) → 202 + job_id; worker SKELETON W mengolah; endpoint GET status |
| LIBRARY | Logic dipakai >= 2 endpoint | Sub-workflow dipanggil Execute Workflow (lihat skill n8n-subworkflows) |

## TABEL RETRY (node-level)

`retryOnFail: true, maxTries: 3, waitBetweenTries: 2000` untuk semua node keluaran jaringan (Postgres, HTTP Request, Execute Workflow ke sistem lain).
Batas engine: `maxTries` maks 5, `waitBetweenTries` maks 5000 ms. Retry terjadi untuk error APAPUN (tidak ada filter per status) — makanya 4xx bisnis jangan lewat node yang di-retry berkali-kali tanpa alasan.

| Kategori | Kode | Retry? |
|---|---|---|
| FATAL-VALIDATION | 400, 413, 422 | Tidak |
| FATAL-AUTH | 401, 403 | Tidak (tapi LOG percobaan — indikasi serangan) |
| FATAL-CONFLICT | 404, 409 | Tidak |
| RETRIABLE | 500, 502, 503, 504, timeout, network, db connection | Ya (maxTries 3) |

## TABEL PEMILIHAN NODE (ringkas — detail: skill n8n-workflow-patterns & n8n-node-configuration)

| Butuh | Pakai | Bukan |
|---|---|---|
| Set/map field, validasi skema ringan | Set node | Code node |
| Logic kompleks / transform > 1 field bergantung | Code node (Run Once for All Items) | rantai Set panjang |
| Cabang kondisional 2 arah | IF | Switch |
| Cabang >= 3 arah nilai diskrit | Switch | IF berantai |
| Gabung stream | Merge | Code manual |
| Kirim HTTP keluar | HTTP Request (+ timeout eksplisit di options) | Code fetch |
| Query SQL | Postgres node (Execute Query + Query Parameters) | Code node SQL string |
| Loop besar | SplitInBatches (batchSize sebesar constraint izinkan) | Each-Item Code |

Timeout: setiap HTTP Request di jalur API wajib `options.timeout` eksplisit (default 30000; sesuaikan). Workflow settings: execution timeout di-set untuk ASYNC worker.

## TABEL DEFAULTS (kalau ragu, jangan berpikir lama — pakai ini)

| Ragu soal | Default |
|---|---|
| sync atau async | sync |
| publik atau auth | auth |
| 400 atau 422 | 400 (json rusak/field hilang), 422 (validasi semantik gagal) |
| retry atau tidak | tidak (kecuali jelas timeout/5xx/network) |
| function SQL atau query langsung | query langsung; naik ke CTE/function hanya bila tabel tingkat akses di 02 bilang begitu |
| taruh efek samping | setelah Respond |
| validasi di n8n atau SQL | n8n untuk bentuk/ukuran field; SQL begitu butuh baca state DB |
| sumber user_id | Verify Auth / klaim JWT terverifikasi |
| response shape | persis KONTRAK DATA + TEMPLATE R (references/03) |
| ada JSON user, mode apa | MODIFY |
| sticky color | 6 Context, 3 Validation, 4 Database, 5 Response, 7 Async, 2 📘 Docs (paling kanan) |
