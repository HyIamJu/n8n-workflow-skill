# 03 — Kontrak Response (amplop, kode, template Code)

## KONTRAK DATA — satu amplop untuk SEMUA node Code, SQL function, dan sub-workflow

```json
{ "status": "success"|"error"|"queued",
  "code": <int sesuai TABEL STATUS>,
  "message": "<manusiawi>",
  "data": {...},
  "details": "<opsional, HANYA untuk 400/422, pesan validasi manusiawi>",
  "internal_detail": "<opsional, diagnostik teknis — HANYA untuk api_logs>" }
```

Aturan:

- Map Result = PASS-THROUGH amplop function apa adanya (normalisasi bentuk tak dikenal diperbolehkan, mengubah amplop valid dilarang).
- `details` boleh terisi hanya jika code 400/422 dan isinya ditulis manusia (mis. "field email wajib diisi") — bukan `e.message`.
- `internal_detail` tidak pernah diteruskan Format Response ke client.
- `status:"queued"` hanya dari endpoint ASYNC202 (code 202, `data.job_id`).

## Output HTTP ke client (bentuk akhir setelah Format Response)

```json
// sukses / queued:
{ "status": "success", "message": "OK", "request_id": "...", "data": {...} }
// error:
{ "status": "error", "error_code": 422, "message": "...", "request_id": "...", "details": "..." }
```

`request_id` WAJIB ada di semua response — client memakainya saat komplain/trace.

## TABEL STATUS

| Code | Arti | Kapan |
|---|---|---|
| 200 | baca/ubah sukses | GET/PUT/DELETE sukses |
| 201 | buat baru | POST sukses |
| 202 | diterima-async | ASYNC202 (+ `data.job_id`) |
| 400 | json rusak / field hilang / Idempotency-Key absen di write | validasi bentuk |
| 401 | token invalid/kedaluwarsa | auth gagal |
| 403 | role kurang | auth ok, izin tidak |
| 404 | tak ditemukan — DIPAKAI JUGA untuk resource milik orang lain | jangan bocorkan keberadaan |
| 409 | duplikat / konflik versi (optimistic locking) | idempotency hit, version mismatch |
| 413 | body > 1MB | batas gateway/n8n |
| 422 | validasi semantik gagal | nilai ada tapi melanggar aturan |
| 429 | rate limit | di-set gateway/reverse proxy DI LUAR n8n — n8n hanya meneruskan |
| 500 | crash internal | exception tak terduga |
| 502 | upstream gagal | HTTP keluar / sub-workflow gagal |
| 503 | service unavailable | downstream down, hint retry |
| 504 | upstream timeout | timeout HTTP keluar |

Prinsip 4xx vs 5xx: **4xx = salah client, 5xx = salah kita.** Monitoring client alert di 5xx; retry policy butuh perbedaan ini. Karena itu: validasi (4xx) diputuskan UPSTREAM via IF/branch, kegagalan tak terduga (5xx) keluar dari error output node.

## TEMPLATE X — wrapper WAJIB semua node Code

```javascript
try {
  const items = $input.all();
  const body = items.length ? items[0].json : {};
  // ... logic di sini, hasil ke `result` ...
  const result = body;
  return [{ json: { status: 'success', code: 200, message: 'OK', data: result } }];
} catch (e) {
  return [{ json: { status: 'error', code: 500, message: 'Unexpected error',
    internal_detail: String(e?.message ?? e).slice(0, 200) } }];
}
```

PENTING — bedakan dua jenis gagal:

- **Validasi yang GAGAL = return eksplisit 400/422**, bukan throw:

```javascript
if (!body.title) {
  return [{ json: { status: 'error', code: 422, message: 'Validation failed',
    details: 'field title wajib diisi' } }];
}
```

- **Exception tak terduga (TypeError, null access, dsb.) = 500** lewat catch.
  Semua-tangkap-jadi-422 adalah BUG: client & monitoring akan mengira salahnya user padahal crash kita.

Catatan mode: pakai "Run Once for All Items" (default) — jangan "Each Item" kecuali memang per-item (lihat skill n8n-code-javascript).

## TEMPLATE R — node "Format Response" (satu-satunya pembentuk output client)

Menerima 3 kemungkinan bentuk input: (a) amplop dari child/function, (b) item error-output node `{error: {...}}`, (c) item kosong/tak dikenal.

```javascript
const r = $input.first().json;
const rid = $('Gen Context').first().json.request_id;
let env;
if (r && r.status) {                        // (a) amplop
  env = r;
} else if (r && r.error) {                  // (b) error output node (main[1])
  env = { status: 'error', code: 500, message: 'Upstream node failed',
    internal_detail: String(r.error.message || r.error).slice(0, 200) };
} else {                                    // (c) kosong / tak dikenal
  env = { status: 'success', code: 200, message: 'OK', data: r ?? {} };
}
let out;
if (env.status === 'error') {
  const code = env.code || 500;
  out = { status: 'error', error_code: code,
          message: env.message || 'Internal error', request_id: rid };
  if (code === 400 || code === 422) out.details = env.details || '';
  // env.internal_detail sengaja TIDAK diteruskan — hanya untuk api_logs
} else if (env.status === 'queued') {
  out = { status: 'queued', message: env.message || 'Accepted',
          request_id: rid, data: env.data || {} };   // data wajib berisi job_id
} else {
  out = { status: 'success', message: env.message || 'OK',
          request_id: rid, data: env.data ?? {} };
}
return [{ json: out }];
```

## Node Respond — konfigurasi WAJIB

```
Respond to Webhook:
  respondWith : firstIncomingItem        (kirim item pertama apa adanya)
  responseCode: ={{ $json.code }}        EKSPLISIT — default node ini 200!
  responseHeaders:
    x-request-id : ={{ $('Gen Context').first().json.request_id }}
```

Tanpa `responseCode` eksplisit, error 404/409/422 terkirim sebagai **HTTP 200** — client HTTP (Dio/fetch/axios) tidak akan menganggapnya error. Ini bug paling serius pada pola response-node dan tidak terlihat di UI.

## request_id

- Sumber: `{{ $json.headers['x-request-id'] || $execution.id }}` (di Gen Context).
- Echo header `x-request-id` di semua response; client boleh kirim sendiri untuk trace end-to-end.
