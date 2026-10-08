# 02 — Pola SQL (akses database)

Aturan dasar: **function SQL untuk LOGIKA, bukan untuk CRUD polos.** Satu statement PostgreSQL sudah atomic — INSERT polos dengan binding tidak butuh dibungkus function hanya demi "aturan".

## Tingkat akses — cara memilih

| # | Kondisi | Media | Kenapa |
|---|---|---|---|
| 1 | Baca/list/agregat | SELECT langsung + binding (TEMPLATE L) | Read-only, tanpa state |
| 2 | Tulis 1 tabel; constraint (NOT NULL/CHECK/UNIQUE/FK) cukup jadi penjaga | statement langsung + `RETURNING *` + binding | Single statement = atomic di PG; unique_violation/check_violation ditangkap error-output → dipetakan amplop |
| 3 | 2-3 operasi tulis terkait harus atomik | CTE satu statement | Semua atau tidak sama sekali, tanpa function |
| 4 | Kompleks (>3 langkah), dipakai ulang >= 2 endpoint, race-prone (check-then-act), atau butuh error map per kasus (404/409/422 berbeda) | SQL function `fn_*(p_payload jsonb)` RETURNS jsonb (TEMPLATE A1/A2) | Transaksi + percabangan error dalam satu panggilan; menghilangkan race condition |
| 5 | Claim job antar-worker | `UPDATE ... FOR UPDATE SKIP LOCKED` | Claim atomik tanpa dua worker mengambil job sama |

Naik tingkat hanya saat kondisinya terpenuhi — jangan langsung function untuk semuanya.

## TEMPLATE D — DDL tabel utama

```sql
CREATE TABLE IF NOT EXISTS {{tabel}} (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL,
  idempotency_key text,
  version int NOT NULL DEFAULT 1,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  {{kolom_domain + tipe + constraint}}
);
-- idempotency scoped per user (endpoint tulis saja):
CREATE UNIQUE INDEX IF NOT EXISTS uq_{{tabel}}_idem
  ON {{tabel}} (user_id, idempotency_key) WHERE idempotency_key IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_{{tabel}}_user ON {{tabel}} (user_id);
CREATE INDEX IF NOT EXISTS idx_{{tabel}}_created ON {{tabel}} (created_at DESC);
```

Aturan idempotency: endpoint tulis MENERIMA `Idempotency-Key` header; kalau header absen → tolak 400 (jangan diam-diam lanjut tanpa proteksi), KECUALI user eksplisit bilang endpoint ini boleh non-idempotent.

## TEMPLATE A0 — fn_verify_token (WAJIB-SQL saja; buat SEKALI per project)

```sql
CREATE TABLE IF NOT EXISTS auth_tokens (
  token text PRIMARY KEY,
  user_id uuid NOT NULL,
  expires_at timestamptz NOT NULL,
  revoked_at timestamptz
);

CREATE OR REPLACE FUNCTION fn_verify_token(p_token text)
RETURNS jsonb LANGUAGE plpgsql AS $$
DECLARE v_user uuid;
BEGIN
  SELECT user_id INTO v_user FROM auth_tokens
   WHERE token = p_token AND expires_at > now() AND revoked_at IS NULL;
  IF NOT FOUND THEN
    RETURN jsonb_build_object('status','error','code',401,
      'message','Token invalid atau kedaluwarsa','data',null);
  END IF;
  RETURN jsonb_build_object('status','success','code',200,'message','OK',
    'data', jsonb_build_object('user_id', v_user));
END $$;
```

Untuk WAJIB-JWT: verifikasi signature dilakukan node Webhook (credential JWT Auth) — TIDAK perlu tabel ini. Bila user_id perlu di-resolve dari klaim/sub ke tabel users, buat `fn_resolve_user(p_sub text)` dengan bentuk return yang sama.

## TEMPLATE A1 — SQL function CREATE (idempotency)

```sql
CREATE OR REPLACE FUNCTION fn_{{action}}(
  p_payload jsonb, p_user_id uuid, p_idem_key text
) RETURNS jsonb LANGUAGE plpgsql AS $$
DECLARE
  v_existing {{tabel}}%rowtype; v_row {{tabel}}%rowtype;
BEGIN
  -- 1. IDEMPOTENCY (scoped per user — wajib untuk write)
  IF p_idem_key IS NOT NULL THEN
    SELECT * INTO v_existing FROM {{tabel}}
     WHERE idempotency_key = p_idem_key AND user_id = p_user_id;
    IF FOUND THEN
      RETURN jsonb_build_object('status','error','code',409,
        'message','Duplicate request','data',null);
    END IF;
  END IF;

  -- 2. AKSI UTAMA (satu transaksi function)
  INSERT INTO {{tabel}} (user_id, idempotency_key, {{kolom}})
  VALUES (p_user_id, p_idem_key, {{nilai dari p_payload->>'...'}})
  RETURNING * INTO v_row;

  RETURN jsonb_build_object('status','success','code',201,
    'message','{{pesan sukses}}','data', row_to_json(v_row));
EXCEPTION
  WHEN unique_violation THEN
    RETURN jsonb_build_object('status','error','code',409,
      'message','Duplicate request','data',null);
  WHEN check_violation THEN
    RETURN jsonb_build_object('status','error','code',422,
      'message','Data tidak memenuhi aturan','data',null);
  WHEN OTHERS THEN
    -- SQLSTATE/SQLERRM hanya untuk api_logs (INSERT ke api_logs via autonomous channel
    -- atau cukup kembali sebagai internal_detail yang TIDAK diteruskan ke client)
    RETURN jsonb_build_object('status','error','code',500,
      'message','Internal error','data',null,
      'internal_detail', SQLSTATE || ': ' || SQLERRM);
END $$;
```

## TEMPLATE A2 — SQL function UPDATE berversi (optimistic locking + ownership)

```sql
CREATE OR REPLACE FUNCTION fn_{{action}}(
  p_payload jsonb, p_user_id uuid, p_idem_key text
) RETURNS jsonb LANGUAGE plpgsql AS $$
DECLARE v_row {{tabel}}%rowtype;
BEGIN
  SELECT * INTO v_row FROM {{tabel}} WHERE id = (p_payload->>'id')::uuid;
  IF NOT FOUND THEN
    RETURN jsonb_build_object('status','error','code',404,'message','Not found','data',null);
  END IF;

  -- OWNERSHIP: resource orang lain = 404 (jangan bocorkan keberadaan)
  IF v_row.user_id <> p_user_id THEN
    RETURN jsonb_build_object('status','error','code',404,'message','Not found','data',null);
  END IF;

  UPDATE {{tabel}}
     SET {{kolom = nilai}}, version = version + 1, updated_at = now()
   WHERE id = v_row.id AND version = (p_payload->>'version')::int
  RETURNING * INTO v_row;
  IF NOT FOUND THEN
    RETURN jsonb_build_object('status','error','code',409,
      'message','Data changed by another process, please reload','data',null);
  END IF;

  RETURN jsonb_build_object('status','success','code',200,
    'message','{{pesan sukses}}','data', row_to_json(v_row));
EXCEPTION
  WHEN unique_violation THEN RETURN jsonb_build_object('status','error','code',409,
    'message','Duplicate request','data',null);
  WHEN check_violation THEN RETURN jsonb_build_object('status','error','code',422,
    'message','Data tidak memenuhi aturan','data',null);
  WHEN OTHERS THEN RETURN jsonb_build_object('status','error','code',500,
    'message','Internal error','data',null,
    'internal_detail', SQLSTATE || ': ' || SQLERRM);
END $$;
```

## TEMPLATE C — CTE atomik (tingkat 3)

```sql
-- Contoh: buat order + tulis audit_log atomik, satu statement, semua binding.
WITH ins AS (
  INSERT INTO {{tabel}} (user_id, {{kolom}})
  VALUES ($1, $2)
  RETURNING id, user_id, {{kolom}}
), audit AS (
  INSERT INTO audit_log (table_name, row_id, payload, created_at)
  SELECT '{{tabel}}', ins.id, to_jsonb(ins), now() FROM ins
)
SELECT * FROM ins;
```

## TEMPLATE L — endpoint BACA/LIST (query langsung, tanpa function)

```sql
-- node "DB Read" (Postgres, Execute Query):
SELECT {{kolom}} FROM {{tabel}}
 WHERE user_id = $1 {{AND filter tambahan — SEMUA via $n}}
 ORDER BY {{kolom_urut}} DESC
 LIMIT $2 OFFSET $3;
```

- `queryReplacement`: `[user_id, limit (maks 100), offset]`.
- **`alwaysOutputData: true`** wajib di node ini — SELECT kosong menghasilkan NOL item dan tanpa ini node downstream tidak jalan / `$input.first()` melempar.
- List kosong = sukses: Map Result membungkus jadi `{status:'success', code:200, data:{items:[], limit, offset}}`.
- Baca tunggal milik orang lain → 404 (jangan bocorkan keberadaan).
- Pagination: validasi `limit` di node Validate (clamp 1-100), jangan percaya client.

## Worker — claim job atomik (SKELETON W memakai ini)

```sql
-- Reset job macet (query statis, boleh polos):
UPDATE async_jobs SET status='queued', locked_at=NULL
 WHERE status='processing' AND locked_at < now() - interval '10 minutes'
   AND attempts < 3;

-- Claim atomik (ambil + kunci dalam 1 statement):
UPDATE async_jobs SET status='processing', locked_at=now(), attempts=attempts+1
 WHERE id IN (SELECT id FROM async_jobs WHERE status='queued'
              ORDER BY created_at FOR UPDATE SKIP LOCKED LIMIT 5)
RETURNING *;
```

## Optimisasi query — checklist

- WHERE selalu pakai kolom ber-index (idx user_id, created_at dari TEMPLATE D).
- `SELECT` sebut kolom eksplisit untuk tabel lebar — hindari `SELECT *` di endpoint list (kecuali RETURNING * internal).
- LIMIT wajib di endpoint list (maks 100); offset untuk paging; pertimbangkan keyset pagination (`WHERE (created_at, id) < ($ts, $id)`) untuk tabel besar.
- Hindari query N+1: kalau butuh data tabel lain per-row, gabung dengan JOIN/CTE sekali panggil.
- Fungsi berat (agregasi besar) → pertimbangkan materialized view / kolom pre-computed, bukan hitung ulang tiap request.

## PII & logging

- api_logs boleh menyimpan: request_id, user_id, endpoint, status code, durasi, internal_detail.
- TIDAK boleh menyimpan: password, token mentah, payload PII penuh (redact/trim).
- Log percobaan auth gagal (401) — sinyal brute force.
