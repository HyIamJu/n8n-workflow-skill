# 04 — Skeleton Workflow (Parent / Child / Worker)

Semua node bernama tetap. Koneksi termasuk **output error (`main[1]`)** — setengah wiring (onError tanpa kabel) = error hilang diam-diam dan eksekusi tampak sukses. Di JSON, kabel error = `sourceIndex: 1`.

## SKELETON P — Parent (workflow API)

```
[Webhook POST {{path}}]
   │  responseMode: "responseNode"
   │  Authentication: sesuai AUTH — JWT/Header (credential bawaan node) atau None + Verify Auth di bawah
   ▼
[Gen Context] (Set)
   │  request_id = {{ $json.headers['x-request-id'] || $execution.id }}
   │  idem_key   = {{ $json.headers['idempotency-key'] || '' }}
   │  start_time = {{ $now.toISO() }}
   ▼
[Verify Auth] (Postgres — HANYA bila AUTH=WAJIB-SQL; dilewati bila JWT/Header bawaan webhook)
   │  query: SELECT * FROM fn_verify_token($1)
   │  queryReplacement: [{{ ($json.headers.authorization || '').replace(/^Bearer\s+/i, '') }}]
   │  onError: "continueErrorOutput", retryOnFail: true, maxTries: 3, waitBetweenTries: 2000
   │  main[0] ──────────────────────────────► [Run Logic]
   │  main[1] (error) ──────────────────────► [Format Response]
   ▼
[Run Logic] (Execute Workflow → {{child}})
   │  waitForSubWorkflow: true
   │  onError: "continueErrorOutput", retryOnFail: true, maxTries: 3, waitBetweenTries: 2000
   │  kirim: payload + user_id (dari Verify Auth/klaim JWT) + idem_key
   │  main[0] ──────────────────────────────► [Format Response]
   │  main[1] (error) ──────────────────────► [Format Response]
   ▼
[Format Response] (Code — TEMPLATE R, references/03)
   ▼
[Respond] (Respond to Webhook)
   │  respondWith: firstIncomingItem
   │  responseCode: ={{ $json.code }}          ← WAJIB eksplisit
   │  responseHeader x-request-id = ={{ $('Gen Context').first().json.request_id }}
   ▼
[Fire Forget {{tugas}}] (Execute Workflow, waitForSubWorkflow: false) — HANYA bila SIDEEFFECTS
```

Catatan wiring:

- Verify Auth sukses tetap menghasilkan amplop `{status:'error',code:401}` saat token invalid — amplop itu mengalir lewat **main[0]** Run Logic → child TIDAK jalan untuk 401. Maka: cek hasil auth SEBELUM Run Logic — tambahkan [IF Auth OK] (condition `{{ $json.status }}` === 'success') antara Verify Auth dan Run Logic; branch false langsung ke Format Response. (Bentuk ringkas: Verify Auth → IF Auth OK → Run Logic / Format Response.)
- user_id untuk child: `{{ $('Verify Auth').first().json.data.user_id }}` (WAJIB-SQL) atau klaim JWT terverifikasi (WAJIB-JWT).

## SKELETON C — Child (logic endpoint)

```
[When Executed by Another Workflow]
   ▼
[Validate] (Code — TEMPLATE X)
   │  parse body, cek field wajib/tipe/ukuran, clamp limit/offset (1-100)
   │  gagal → return eksplisit {status:'error', code:422, message, details}
   ▼
[DB Action] (Postgres — Execute Query)
   │  tingkat akses sesuai references/02:
   │    - function: SELECT * FROM fn_{{action}}($1,$2,$3)
   │      queryReplacement: [payloadJson, user_id, idem_key]
   │    - statement langsung/CTE: query + binding $1..$n
   │  onError: "continueErrorOutput"  → main[1] ke [Map Result]
   ▼ main[0]
[Map Result] (Code — pass-through + normalisasi)
   ▼
(output: SELALU amplop KONTRAK DATA)
```

Map Result — template:

```javascript
try {
  const items = $input.all();
  if (!items.length) {
    // SELECT kosong (alwaysOutputData menghasilkan 0/1 item kosong)
    return [{ json: { status: 'success', code: 200, message: 'OK',
      data: { items: [] } } }];
  }
  const r = items[0].json;
  if (r && r.status) return items;                 // amplop function → apa adanya
  if (r && r.error) {                              // item error-output node
    return [{ json: { status: 'error', code: 500, message: 'Database node failed',
      internal_detail: String(r.error.message || r.error).slice(0, 200) } }];
  }
  // rows polos dari SELECT langsung → bungkus list
  return [{ json: { status: 'success', code: 200, message: 'OK',
    data: { items: items.filter(i => Object.keys(i.json || {}).length).map(i => i.json) } } }];
} catch (e) {
  return [{ json: { status: 'error', code: 500, message: 'Unexpected error',
    internal_detail: String(e?.message ?? e).slice(0, 200) } }];
}
```

## SKELETON W — Worker async (PATTERN=ASYNC202)

```
[Schedule Trigger] (interval 15-30 detik)
   ▼
[Reset Job Macet] (Postgres, query statis)
   │  UPDATE async_jobs SET status='queued', locked_at=NULL
   │   WHERE status='processing' AND locked_at < now() - interval '10 minutes'
   │     AND attempts < 3;
   ▼
[Ambil Job] (Postgres — claim atomik FOR UPDATE SKIP LOCKED, lihat references/02)
   │  alwaysOutputData: true, onError: "continueErrorOutput"
   ▼
[IF Ada Job] — kosong → berhenti
   ▼
[Proses Job] (Execute Workflow ke processor, atau HTTP Request options.timeout: 30000)
   │  retryOnFail: true, maxTries: 3, waitBetweenTries: 2000
   ▼
[Update Status] (Postgres, binding)
   │  sukses: status='done' + result_json
   │  gagal : attempts < 3 → status='queued' (retry siklus berikutnya)
   │          attempts >= 3 → status='failed' (+ trigger notifikasi)
```

Endpoint ASYNC202 (di parent/child biasa): aksi utama hanya INSERT ke async_jobs (status='queued'), lalu balas amplop `{status:'queued', code:202, data:{job_id}}`. Endpoint pendamping `GET /status/{{resource}}/:job_id` = endpoint baca biasa (TEMPLATE L atas tabel async_jobs, ownership: job milik user lain → 404).

Worker WAJIB punya error workflow (Error Trigger → notifikasi kanal BERBEDA dari kanal yang dimonitor + fallback tabel) — assignment-nya setting UI n8n (Workflow Settings → Error Workflow), tidak bisa via MCP. Ingatkan user.

## Contoh JSON connections (bentuk data wiring error)

```json
"Run Logic": {
  "main": [
    [ { "node": "Format Response", "type": "main", "index": 0 } ],
    [ { "node": "Format Response", "type": "main", "index": 0 } ]
  ]
}
```

`main[0]` = jalur sukses, `main[1]` = jalur error → dua-duanya mendarat di Format Response (satu titik fan-in; TEMPLATE R menormalkan keduanya).
