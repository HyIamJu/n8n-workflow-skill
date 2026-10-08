---
name: n8n-api-architect
description: Standar engineering backend untuk workflow n8n webhook/API + PostgreSQL — scoping, pemilihan node yang tepat, keamanan (binding, auth, secret hygiene), kontrak response tunggal, layout canvas & dokumentasi, dan audit mekanis. Gunakan saat membuat, mengubah, atau mengaudit workflow n8n yang melayani HTTP API, menyentuh database, atau butuh standarisasi response/dokumentasi — termasuk saat user hanya bilang "bikin workflow n8n" atau "bikin endpoint". Skill ini orchestrator — detail teknis dirujuk ke skill n8n-* spesialis via tabel routing.
---

# n8n API Architect

n8n = ORCHESTRATOR (routing, validasi ringan, format, respond). PostgreSQL = integritas data & validasi state.
Skill ini = lapisan standar (scoping → SQL → workflow → canvas → docs → audit) + ROUTER ke skill n8n-* spesialis. Jangan duplikasi isi skill spesialis — rujuk lewat tabel routing di bawah.

## KONTRAK EKSEKUSI

- Tulis MODE di baris pertama jawaban (deteksi di STEP 0).
- Kerjakan SATU STEP pada satu waktu; output artifact step itu dulu, baru lanjut.
- Semua kode/node = SALIN TEMPLATE dari `references/` lalu isi `{{blank}}`. Dilarang mencipta ulang bentuk response/tipe node dari ingatan.
- Ragu memilih → buka TABEL DEFAULTS di `references/01-decision-tables.md`. Jangan berunding lama.
- Request tidak jelas APA yang diubah (bukan bagaimana) → boleh TANYA SATU pertanyaan.
- Template tidak punya slot untuk sesuatu → itu artinya dilarang.

## STEP 0 — DETECT MODE

| Kondisi input | MODE | Jalur |
|---|---|---|
| User minta workflow/endpoint baru | CREATE | STEP 1-8 |
| JSON terlampir + review/audit saja | AUDIT | PROTOKOL A (`references/06-modify-audit.md`) |
| JSON terlampir + minta tambah/ubah/fix | MODIFY | PROTOKOL M (`references/06-modify-audit.md`) |
| User bilang "eksekusi P\<n\>" dari REPORT sebelumnya | MODIFY | scope = poin itu saja |
| Ambigu (ada JSON tapi tidak jelas) | tanya 1 pertanyaan → tetap tak jelas = MODIFY | |

## 12 HARD RULES

Kedua-duanya diverifikasi di GERBANG AKHIR sebelum jawaban dikirim.

1. Tepat SATU node Respond to Webhook per workflow API, dengan `responseCode` EKPLISIT (`={{ $json.code }}`). Default node ini adalah 200 — tanpa ini, error 401/404/409/422 terkirim sebagai HTTP 200 dan error handling client tidak pernah menyala.
2. Semua query berparameter wajib binding `$1,$2,...` via Query Parameters (`queryReplacement`). Dilarang interpolasi string SQL. Query statis tanpa parameter boleh polos.
3. Semua node Code dibungkus try/catch (TEMPLATE X). Validasi yang gagal = return EKSPLISIT 400/422 + `details` manusiawi; exception tak terduga = 500. Bukan semua-tangkap-jadi-422.
4. Setiap node yang bisa gagal (Postgres/HTTP/Execute Workflow): `onError: "continueErrorOutput"` DAN output error-nya (`main[1]`) DIWIRE ke Format Response. Satu saja dari dua ini = error hilang diam-diam, eksekusi tampak sukses.
5. SELECT yang mungkin kosong: `alwaysOutputData: true` + penanganan item kosong (list kosong = 200 dengan `items: []`, bukan error).
6. Validasi yang butuh baca state DB = tugas SQL (constraint/CTE/function) — bukan check-then-act di n8n (race condition).
7. `user_id` HANYA dari sumber terverifikasi (klaim JWT terverifikasi node Webhook, atau output node Verify Auth). Dilarang dari body/query param/path. Endpoint publik harus dinyatakan eksplisit di STEP 1 — default: butuh auth.
8. Efek samping non-krusial (log, notifikasi, IoT) diletakkan SETELAH node Respond, `waitForSubWorkflow: false`.
9. Error ke client = pesan manusiawi — tanpa stack trace/SQL/nama tabel/upstream body. `internal_detail` tidak pernah keluar ke client, hanya untuk api_logs.
10. Secret hanya di credentials n8n — dilarang hardcode di Code node, sticky note, atau URL. Webhook masuk pakai auth bawaan node Webhook (Basic/Header/JWT) bila memungkinkan.
11. MODIFY = output PATCH, bukan full JSON (kecuali diminta eksplisit). Node FROZEN tidak disentuh; temuan di luar scope = REPORT + alasan, dilarang dieksekusi diam-diam.
12. Setiap workflow: sticky 📘 Docs (color 2) berisi cara pakai API + JSON diekspor & di-commit ke repo + nama `[API][vX] - <Domain> - <Aksi>`.

## KONTRAK DATA — satu amplop untuk SEMUA node & function

```json
{ "status": "success"|"error"|"queued", "code": <int>, "message": "...", "data": {...},
  "details": "<opsional, HANYA 400/422, teks validasi manusia>",
  "internal_detail": "<opsional, diagnostik teknis — HANYA untuk api_logs>" }
```

- Map Result = PASS-THROUGH amplop function apa adanya; dilarang mengubah shape (normalisasi error/kosong diperbolehkan, lihat TEMPLATE).
- `status:"queued"` hanya diproduksi endpoint ASYNC202, code 202, `data` berisi `job_id`.
- Detail lengkap + tabel status/error: `references/03-response-contract.md`.

## STEP 1 — SCOPING GATE

Jawab satu baris per pertanyaan (panduan + interpretasi: `references/01-decision-tables.md`):

- Q0: Endpoint publik atau butuh auth? (default: auth)
- Q1: Proses bisa > 10 detik / berat (file, bulk, chain approval)?
- Q2: Logic dipakai ulang di >= 2 endpoint berbeda?
- Q3: Ada efek samping non-krusial (log DB, notifikasi, IoT)?
- Q4: Environment (uat/prod) + apakah datanya sensitif (PII)?

Tulis hasil: `AUTH = ...` | `PATTERN = SYNC (default) | ASYNC202 (Q1=ya) | LIBRARY (Q2=ya)` | `SIDEEFFECTS = ...` | `ENV = ...`.

## STEP 2 — SQL: pilih tingkat akses DB

Function SQL itu untuk LOGIKA, bukan untuk CRUD polos. CRUD 1 tabel = statement langsung (satu statement di PostgreSQL sudah atomic).

| Kondisi | Media |
|---|---|
| Baca/list/agregat | SELECT langsung + binding (TEMPLATE L) |
| Tulis 1 tabel, constraint DB cukup jadi penjaga | INSERT/UPDATE/DELETE langsung + RETURNING + binding |
| 2-3 operasi tulis terkait yang harus atomik | CTE satu statement |
| Kompleks / dipakai ulang / race-prone / butuh error map per kasus | SQL function `fn_*(p_payload jsonb)` (TEMPLATE A1/A2) |
| Claim job antar-worker | `UPDATE ... FOR UPDATE SKIP LOCKED` |

Semua template SQL: `references/02-sql-patterns.md`.

## STEP 3-7 (ringkas; detail per step ada di references)

- STEP 3 — API DOCS: salin TEMPLATE B (`references/05-canvas-and-docs.md`).
- STEP 4 — PARENT: salin SKELETON P (`references/04-skeletons.md`). Urutan & nama node tetap: Webhook → Gen Context → [Verify Auth] → Run Logic → Format Response → Respond → [Fire Forget].
- STEP 5 — CHILD: salin SKELETON C: Validate → DB Action → Map Result.
- STEP 6 — ASYNC ZONE (hanya jika Q1=ya atau Q3 ada): 202 + job_id + worker SKELETON W.
- STEP 7 — CANVAS: 6 zona sticky warna tetap + penamaan (`references/05-canvas-and-docs.md`).

## STEP 8 — AUDIT MEKANIS (wajib tulis hasil eksplisit)

1. Hitung `respondToWebhook` → tepat 1. Cek ada `responseCode` eksplisit.
2. Setiap node berparameter DB: ada `queryReplacement` DAN string SQL tidak mengandung `{{` atau `${`.
3. Setiap node fallible: `onError: "continueErrorOutput"` **dan** kabel `main[1]` tersambung ke Format Response/Map Result.
4. Cari `try {` → ada di setiap node Code.
5. AUTH≠PUBLIK: Verify Auth ada SEBELUM Run Logic; `user_id` tidak dibaca dari body.
6. `internal_detail` di Format Response hanya sebagai komentar, tidak ikut ke output.
7. Node SELECT-list: `alwaysOutputData: true`.
8. Cek tiap status code keluaran vs TABEL STATUS (`references/03-response-contract.md`).
9. Hitung sticky note >= 6.

Satu saja gagal → perbaiki → ulangi STEP 8. Tanpa pengecualian.

## ROUTING KE SKILL SPESIALIS

Muat skill tujuan saat task-nya muncul; jangan duplikasi isinya di sini.

| Task | Skill |
|---|---|
| Node AI/agent (`@n8n/n8n-nodes-langchain.*`) | n8n-agents |
| JavaScript di Code node | n8n-code-javascript |
| Ekspresi `{{ }}`, `$json`, `$node` | n8n-expression-syntax |
| Bentuk/arsitektur workflow, pola webhook/DB/batch | n8n-workflow-patterns |
| Sub-workflow reuse, Execute Workflow | n8n-subworkflows |
| Detail wiring error, error workflow, retry | n8n-error-handling |
| File/binary/base64 | n8n-binary-and-data |
| Validasi workflow (validate_workflow) | n8n-validation-expert |
| Operasi instance n8n via MCP | using-n8n-mcp-skills, n8n-mcp-tools-expert |
| Lebih dari satu instance/env | n8n-multi-instance |
| Deploy/self-host n8n | n8n-self-hosting |

## FORMAT JAWABAN AKHIR

- CREATE: (1) hasil STEP 1 (2) SQL (3) API Docs (4) JSON workflow utuh (5) curl sukses + 1 error (6) hasil STEP 8 eksplisit.
- MODIFY: (1) Kontrak Scope M1 (2) PATCH M3 + cara apply (3) REPORT M5 (4) Self-Check M6.
- AUDIT: (1) peta struktur (2) REPORT P1/P2/P3 (3) usulan urutan eksekusi.

## GERBANG AKHIR

- 12 hard rules terpenuhi? Audit mekanis (STEP 8 / M6) sudah ditulis hasilnya?
- MODIFY: sentuhan = persis TOUCH list? Temuan lain hanya REPORT?
- Jika satu saja jawabannya tidak: hentikan, perbaiki, ulangi pengecekan.
