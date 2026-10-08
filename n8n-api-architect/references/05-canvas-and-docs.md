# 05 — Standar Canvas & Dokumentasi

Tujuan: workflow yang dibaca orang lain (atau diri sendiri 3 bulan kemudian) langsung paham. Canvas yang rapi bukan kosmetik — ia adalah dokumentasi.

## Zona sticky note (warna & urutan tetap, progresif kiri → kanan)

| Zona | Warna sticky (index n8n) | Isi |
|---|---|---|
| Context | 6 (biru) | Webhook, Gen Context, Verify Auth |
| Validation | 3 (kuning) | IF Auth OK, Validate |
| Database | 4 (merah) | DB Action / DB Read, Ambil Job, Update Status |
| Response | 5 (hijau) | Map Result, Format Response, Respond |
| Async | 7 (ungu) | worker, Fire Forget |
| 📘 Documentation | 2 (abu) | TEMPLATE B — SELALU paling kanan |

Minimal 6 sticky per workflow API (STEP 8 menghitung ini).

## Layout

- Alur kiri → kanan mengikuti urutan zona; error branch boleh turun ke bawah lalu kembali ke kanan.
- Jarak antar node minimum **x+220** (lebar standar node) — jangan menumpuk.
- Node dalam satu zona sejajar Y (± 0). Fork/join boleh offset Y ± 200.
- Posisi sticky mencakup (cover) seluruh zona node-nya; ukuran disesuaikan, jangan ada node di luar sticky-nya.
- Node diberi nama deskriptif — DILARANG nama default ("Code1", "IF2", "Postgres3").

## Penamaan

- Workflow: `[API][vX] - <Domain> - <Aksi>` — contoh: `[API][v2] - Notification - Send Push`.
- Node: `Zone Verb Object` — "Verify Auth", "DB Action", "Map Result", "Format Response", "Respond", "Fire Forget Audit".
- Child workflow: sama dengan aksi parent + suffix ` (child)`.
- Worker: `[WORKER][vX] - <Domain> - <Aksi>`.

## Versioning & repo

- Perubahan KONTRAK (path, method, status code, bentuk response) → bump `vX+1` + catat di sticky 📘 apa yang berubah.
- Perbaikan internal (tanpa perubahan kontrak) → versi tetap, commit message menjelaskan.
- Setiap workflow DIEKSPOR (Download JSON) dan di-commit ke repo — satu file per workflow, path mengikuti pola repo (mis. `push-service/n8n/<nama>.json`). JSON di repo = source of truth untuk review; instance produksi di-update dari JSON ini.
- Credential ID di JSON boleh ter-commit; NILAI secret tidak pernah.

## TEMPLATE B — API Docs (isi sticky 📘 Documentation + section dokumentasi jawaban)

```
📘 API — {{nama endpoint}}
ENDPOINT  : {{METHOD}} {{/api/v1/path}}
AUTH      : Authorization: Bearer <token>
            Idempotency-Key: <uuid> (WAJIB untuk write)
REQUEST   : { ...contoh json valid... }
RESPONSES : (semua status yang mungkin, satu contoh per baris)
  200 { "status":"success", "request_id":"...", "data":{...} }
  401 { "status":"error", "error_code":401, "request_id":"..." }
  422 { "status":"error", "error_code":422, "request_id":"...", "details":"..." }
ASYNC     : (bila 202) GET /status/{{resource}}/{job_id} → 200 done | 202 queued
CATATAN   : {{batasan, rate limit, env (uat/prod), changelog vX}}
TEST      : curl siap copy-paste — versi sukses + minimal 1 versi error
```

curl wajib bisa langsung dijalankan (ganti variabel saja):

```bash
curl -X POST {{base}}/api/v1/{{path}} \
  -H "Authorization: Bearer $TOKEN" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{ ...contoh request... }'
```

## Environment (uat/prod)

- Resource yang menyentuh environment (topic FCM, path webhook, tabel) WAJIB di-tag env — contoh pola yang sudah berjalan: topic `all_uat` vs `all_prod`.
- Satu workflow untuk dua env hanya boleh bila perbedaannya MURNI via environment variable/credential; kalau beda logic → dua workflow dengan suffix env di nama.
- Credential per env (jangan sharing), lihat skill n8n-multi-instance.
