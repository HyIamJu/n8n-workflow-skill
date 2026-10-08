# 06 — Protokol MODIFY, AUDIT & Checklist Mekanis

## PATH MODIFY — PROTOKOL M (tukang bedah, bukan kontraktor renovasi)

Prinsip: JSON milik user = KEBENARAN TUNGGAL. Kamu menyalin, bukan mencipta ulang.

### M1 — Kontrak Scope (tulis DI AWAL jawaban)

```
MODE: MODIFY
PERMINTAAN: <satu kalimat, kata user dipersingkat>
TOUCH: [nama node persis yang akan disentuh]
FROZEN: semua node lain — nama, posisi, parameter, credentials TIDAK BERUBAH
TEMUAN LUAR SCOPE: dikumpulkan ke REPORT (M5), TIDAK DIEKSEKUSI
```

### M2 — FREEZE (larangan mutlak)

- DILARANG rename node frozen (name = kunci connections; rename = putus kabel).
- DILARANG rapikan posisi / re-layout node frozen.
- DILARANG sekalian upgrade tipe node, ubah auth, ubah pola response, merge, hapus node — kecuali permintaan eksplisit user.
- DILARANG regenerate JSON dari ingatan — baca ulang JSON input, salin apa adanya.
- Saran perbaikan di luar TOUCH = REPORT. Tangan tetap di saku.

### M3 — Bangun PATCH (INI format output-nya, BUKAN full JSON)

```
[ADD] <nama node baru> — JSON node utuh
      (nama unik; posisi dekat node jangkar, offset x+220)
[MODIFY] <nama node> — field persis: OLD -> NEW (tampilkan keduanya)
[WIRE] <from node>[output index] -> <to node>   (index 1 = output error!)
[UNWIRE] <koneksi yang lepas karena patch>
[VERIFY-BY] cara user mengecek hasil di UI n8n (1-2 baris)
```

Node/Code baru TETAP wajib ikut standar inti (TEMPLATE X, KONTRAK DATA, binding, onError+wiring).
Full JSON hanya jika user minta eksplisit — salin byte-identical semua node frozen, lalu jalankan M4 dan M6.

### M4 — Cek Keseimbangan (hanya zona sentuh, jawab eksplisit)

- Apakah node setelah zona sentuh membaca field dari node yang kamu ubah?
- Apakah field yang kamu hapus masih direferensikan ekspresi di tempat lain?
- Apakah koneksi baru memakai output-index yang benar (0 = sukses, 1 = error)?
- Patch ini tidak merusak aturan "onError + main[1] wired" di node sekitar?
- Bahaya → perbaiki PATCH-nya, bukan node lain.

### M5 — REPORT Rekomendasi (temuan luar scope — DILARANG dieksekusi)

Urut dari paling berat, satu baris per temuan:

```
[P1|P2|P3] <lokasi node> | <temuan> | <kenapa jadi masalah> | <usulan>
```

- P1 = kritis (bug/keamanan/behavior salah di produksi)
- P2 = pelanggaran standar skill ini (sebaiknya diperbaiki)
- P3 = opsional kerapian

Tutup dengan kalimat fix:
"Poin di atas TIDAK saya eksekusi. Sebut nomornya ('eksekusi P2') jika ingin diterapkan — dikerjakan sebagai MODIFY baru dengan kontrak scope sendiri."

### M6 — Self-Check (tulis hasil eksplisit sebelum kirim)

- Node yang disentuh = persis TOUCH list? (hitung, bandingkan)
- Ada nama node frozen yang berubah? (harus: tidak)
- Ada perubahan di luar TOUCH? (harus: tidak; kalau iya → pindahkan ke REPORT, hapus dari patch)

## PATH AUDIT — PROTOKOL A (nol perubahan JSON)

### A1 — Peta struktur

Baca seluruh JSON. Tulis: jumlah node, tipe, koneksi (termasuk mana yang main[1]), zona, sticky, credentials yang dipakai.

### A2 — Nilai dengan 12 HARD RULES + checklist mekanis (bawah)

Cari juga: respond node ganda ATAU tanpa responseCode eksplisit; query tanpa binding; node tanpa try/catch; catch-all jadi 422; onError tanpa kabel error; SELECT-list tanpa alwaysOutputData; idempotency absen/tidak scoped user; auth absen di endpoint non-publik; user_id dibaca dari body; internal_detail bocor ke client; race condition check-then-act; PII di log; secret hardcode; node default name ("Code1").

### A3 — Output

REPORT format M5 (P1/P2/P3 + alasan + usulan) + usulan urutan eksekusi.
NOL perubahan JSON. Eksekusi hanya jika user bilang "eksekusi P\<n\>".

## CHECKLIST AUDIT MEKANIS (STEP 8 / AUDIT — tulis hasil eksplisit, hitung/cari string)

| # | Cek | Cara | Lolos bila |
|---|---|---|---|
| 1 | Satu respond node | hitung `"type": "n8n-nodes-base.respondToWebhook"` | tepat 1 |
| 2 | responseCode eksplisit | cari `responseCode` di node Respond | ada & expression dari code |
| 3 | Anti interpolasi SQL | tiap node Postgres: ada `queryReplacement` & string SQL tanpa `{{` / `${` | semua lolos |
| 4 | try/catch | cari `try {` per node Code | ada di semua |
| 5 | Error wiring | tiap node fallible: `onError: "continueErrorOutput"` DAN `connections.<node>.main[1]` berisi target | dua-duanya |
| 6 | Auth sebelum logic | urutan Verify Auth → Run Logic; `user_id` dari auth, bukan body | ya (non-publik) |
| 7 | internal_detail aman | di Format Response/Respond hanya sebagai komentar | tidak ikut output |
| 8 | List kosong aman | node SELECT-list: `alwaysOutputData: true` + Map Result tangani kosong | dua-duanya |
| 9 | Status code vs tabel | tiap kemungkinan code dibanding TABEL STATUS (references/03) | semua cocok |
| 10 | Sticky >= 6 | hitung `stickyNote` | >= 6 |
| 11 | Node naming | cari nama default `Code\d|IF\d|Set\d|Postgres\d` | tidak ada |
| 12 | Secret hygiene | cari token/password/nilai rahasia di parameter/sticky | tidak ada (hanya credential ID) |

Satu saja gagal → perbaiki → ulangi. Untuk AUDIT mode, gagalan masuk REPORT, bukan diubah.

## Verifikasi pasca-deploy (saat bekerja via MCP)

1. `validate_workflow` — tangkap error struktur.
2. Tarik ulang workflow (`n8n_get_workflow`) dan BACA connections langsung — validasi tidak menangkap wiring valid-tapi-salah (error output nyangkut nol, fallback Switch kosong).
3. `n8n_test_workflow` + baca eksekusi — konfirmasi bentuk output & status HTTP. EFEK SAMPING NYATA ikut jalan saat test: konfirmasi user dulu bila ada write.
4. Aktifkan hanya setelah 1-3 lolos. (Siklus lengkap: skill n8n-workflow-patterns.)
