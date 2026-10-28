---
name: sc-phase3-verify
description: Fase 3 Audit Lintas-Rantai - Verifikasi hipotesis melalui pengujian unit/invariant di sandbox lokal, eliminasi false-positive, dan penyusunan laporan akhir beserta patch remediasi
version: 3.0.0
metadata:
  hermes:
    tags: [security, web3, audit, verification, testing, foundry, phase3]
    category: security
    requires_toolsets: [terminal]
---

# Phase 3: Multi-Chain Sandbox Verification & Structured Reporting

## When to Use
Gunakan skill ini setelah Fase 2 menghasilkan satu atau lebih hipotesis berstatus [FLAGGED] di dalam /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/.audit/04-hypotheses.md. Skill ini menguji kebenaran hipotesis secara objektif di lingkungan pengujian lokal dan menyusun laporan akhir.

## Pitfalls (Larangan Keras di Fase 3)
- HANYA PENGUJIAN LOKAL TERISOLASI: Seluruh pengujian WAJIB dijalankan menggunakan framework pengujian lokal di dalam repositori proyek. Dilarang berinteraksi dengan jaringan produksi/publik.
- JANGAN MEMAKSAKAN TEMUAN PALSU: Jika hasil pengujian membuktikan bahwa pengaman protokol (require, assert!, atau handler bounced) berhasil menggagalkan skenario anomali, terima fakta tersebut dan pindahkan hipotesis ke .audit/05-dead-ends.md.
- JAGA KEBERSIHAN REPOSITORI TARGET: Buat file pengujian verifikasi dengan nama yang jelas (misal VerifyHypothesis.t.sol atau modul test terpisah) dan jangan merusak kode sumber asli kontrak.
- DILARANG MEMANGGIL `skill_manage` ATAU `memory`.

---

## Procedure (4 Langkah Verifikasi & Pelaporan)

Untuk setiap hipotesis di /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/.audit/04-hypotheses.md, jalankan langkah berikut:

### Langkah 1: Pemilihan Adapter Pengujian Sesuai Rantai Proyek
Baca .audit/00-project-map.md untuk melihat toolchain pengujian proyek, lalu siapkan lingkungan test yang sesuai:
1. EVM / Tron (Foundry / Hardhat):
   - Periksa keberadaan foundry.toml atau hardhat.config.*.
   - Gunakan forge test --match-test test_Verify_<NamaHipotesis> -vvvv untuk melihat jejak eksekusi state dan perubahan saldo secara presisi.
2. Solana (Rust / Anchor):
   - Periksa Cargo.toml dan Anchor.toml.
   - Gunakan unit test Rust (cargo test <nama_test> -- --nocapture) atau cargo test-sbf / LiteSVM / ProgramTest lokal untuk memverifikasi mutasi akun dan validasi remaining_accounts.
3. Sui / Aptos (Move):
   - Gunakan modul #[test] dan test_scenario bawaan Move, lalu jalankan sui move test <nama_test> atau aptos move test untuk menguji interaksi objek (UID), Capability, atau *Hot Potato*.
4. TON (FunC / Tact / Tolk):
   - Gunakan @ton/sandbox (Jest / Blueprint) dan jalankan npx blueprint test atau npx jest untuk memeriksa pengiriman pesan asinkron, perilaku bounced 256-bit, dan perubahan storage kontrak.

### Langkah 2: Penulisan & Eksekusi Test Pembuktian Invarian
1. Pelajari satu file test yang sudah ada di folder test/ atau tests/ proyek (baca maksimal 100 baris dengan read_file) agar kamu mengikuti cara inisialisasi (*setup*) resmi proyek tersebut.
2. Buat skenario pengujian yang membandingkan Invarian Seharusnya (dari .audit/00-project-map.md) dengan State Aktual setelah fungsi target dipanggil:
   - Nyatakan kondisi awal (*initial state*: saldo awal, total shares, indeks waktu, atau status objek).
   - Jalankan urutan pemanggilan fungsi sesuai jejak Radius 0 -> Radius 1 -> Radius 2 di .audit/04-hypotheses.md.
   - Gunakan asersi (assertEq, assertGt, assert!, expect) untuk memeriksa apakah invarian protokol benar-benar patah (misal: totalShares tidak seimbang dengan sum(userShares), arah pembulatan merugikan pool, atau state gagal pulih saat pesan bounced).
3. Jalankan perintah test di terminal dan analisis output jejaknya (*trace*).

### Langkah 3: Klasifikasi Hasil (Valid vs False Positive)
1. Jika Pengujian Gagal Membuktikan Anomali (False Positive):
   - Analisis mengapa skenario tersebut tertahan oleh penjaga protokol.
   - Hapus hipotesis tersebut dari daftar temuan aktif, catat alasan kegagalannya ke dalam `.audit/05-dead-ends.md`, dan ubah status fungsinya di .audit/03-micro-queue.md menjadi [DONE - AMAN (Refuted in Phase 3)].
2. Jika Pengujian Berhasil Membuktikan Patahnya Invarian (Verified Finding):
   - Tandai hipotesis sebagai [VERIFIED] dan lanjutkan ke Langkah 4.
3. *(Catatan Fallback Kompilasi):* Jika repositori target memiliki dependensi eksternal yang rusak sehingga kompilasi lokal tidak memungkinkan, dokumentasikan bukti jejak matematis baris demi baris secara eksak beserta perhitungan angkanya sebagai [STATICALLY VERIFIED - ENV BLOCKED].

### Langkah 4: Penyusunan Laporan Akhir & Rekomendasi Remediasi (.audit/FINAL-REPORT.md)
Untuk setiap temuan berstatus [VERIFIED], tuliskan laporan terstruktur ke dalam `.audit/FINAL-REPORT.md` dengan format berikut:
1. Judul & Klasifikasi Keparahan: (Critical / High / Medium) beserta ringkasan 2 kalimat mengenai akar masalah dan dampaknya pada protokol.
2. Lokasi Kode Presisi & Invarian yang Dilanggar: Path file, rentang nomor baris (NamaFile:BarisAwal-BarisAkhir), dan invarian dari 00-project-map.md yang dilanggar.
3. Akar Masalah Mekanis (Root Cause & Radius Trace): Penjelasan teknis bagaimana mutasi state di Radius 0 berinteraksi dengan Radius 1/2 dan mengapa pengaman yang ada gagal menghentikannya.
4. Bukti Verifikasi Pengujian (Test Trace): Cuplikan kode unit/invariant test yang dijalankan beserta hasil output terminalnya.
5. Rekomendasi Perbaikan Kode (Remediation Patch `diff`): Berikan blok diff perbaikan kode yang presisi dan minimal untuk menutup celah tersebut tanpa merusak fungsionalitas utama.

---

## Verification
Pastikan setiap hipotesis di .audit/04-hypotheses.md telah diproses hingga tuntas (masuk ke .audit/FINAL-REPORT.md jika terbukti valid, atau masuk ke .audit/05-dead-ends.md jika gugur).
