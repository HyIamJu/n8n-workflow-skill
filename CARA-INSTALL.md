# n8n-api-architect — Skill ZCode

Standar engineering backend untuk workflow n8n webhook/API + PostgreSQL: scoping, pemilihan node yang tepat, keamanan (binding, auth, secret hygiene), kontrak response tunggal, layout canvas & dokumentasi, dan audit mekanis. Berbahasa Indonesia, format skill folder standar (SKILL.md + references/).

## Cara pasang di ZCode

1. Ekstrak zip ini.
2. Salin folder `n8n-api-architect/` ke SALAH SATU lokasi:
   - Global (tersedia di semua project): `~/.zcode/skills/n8n-api-architect/`
   - Per-project: `<repo>/.agents/skills/n8n-api-architect/`
3. Restart ZCode (skill baru terdeteksi saat session baru).
4. Cek: ketik `/` di input box → grup **Skills** → `n8n-api-architect` harus muncul. Bisa juga dipaksakan load via `/skill n8n-api-architect <pertanyaan>`.
5. Trigger otomatis: cukup bilang "bikin workflow n8n / bikin endpoint / audit workflow ini".

Struktur wajib yang harus tetap utuh: `n8n-api-architect/SKILL.md` (nama folder = nama skill).

## Kompatibilitas lain

Format folder SKILL.md + frontmatter name/description adalah format skill generik — kompatibel dengan tool lain yang membaca direktori `.agents/skills/` (mis. Claude Code: taruh di `~/.claude/skills/` atau `<repo>/.claude/skills/`). Mekanisme discovery bisa sedikit beda per tool; isi skillnya tetap markdown biasa.

## Catatan

- Skill ini berdiri sendiri. Tabel routing di dalamnya merujuk skill pendamping opsional (`n8n-agents`, `n8n-error-handling`, `n8n-workflow-patterns`, dll.) — kalau belum terpasang, bagian routing diabaikan saja; standar inti (12 hard rules, kontrak response, skeleton, audit) tetap lengkap di sini.
- Versi: 2026-10-08 (pengganti draft "N8N ENTERPRISE v6" yang tidak pernah terpasang).
