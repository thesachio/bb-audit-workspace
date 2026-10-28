---
name: sc-phase2-hunt
description: Fase 2 Audit Lintas-Rantai - Inspeksi Mikro-ke-Makro (Radius 0-2) berbasis mutasi state dengan pemuatan dinamis modul referensi ekonomi dan rantai
version: 3.0.0
metadata:
  hermes:
    tags: [security, web3, audit, vulnerability-hunting, multichain, phase2]
    category: security
    requires_toolsets: [terminal]
---

# Phase 2: Multi-Chain Micro-to-Macro Vulnerability Hunting (Radius 0 -> 2)

## When to Use
Gunakan skill ini setelah Fase 1 selesai (setelah .audit/00-project-map.md, .audit/02-storage-map.md, dan .audit/03-micro-queue.md tersedia di folder proyek). Skill ini membedah fungsi satu per satu dari cakupan terkecil (Radius 0) dan hanya melebar ke Radius 1 & 2 jika ditemukan bukti anomali state atau batas kepercayaan.

## Pitfalls (Larangan Keras di Fase 2)
- DILARANG MEMERIKSA BANYAK FUNGSI SEKALIGUS DALAM SATU PIKIRAN: Jika menjalankan audit penuh atas seluruh antrean di .audit/03-micro-queue.md, Parent WAJIB menggunakan delegate_task agar setiap Subagent hanya membedah 1 fungsi (atau 1 pasang fungsi siklus hidup yang memodifikasi state yang sama) dengan konteks bersih.
- WAJIB MEMUAT MODUL REFERENSI (`skill_view`): Jangan mengandalkan ingatan umum. Panggil skill_view("sc-phase2-hunt", "references/<nama-file>.md") sesuai tag [LOAD: ...] yang tertera pada fungsi tersebut di .audit/03-micro-queue.md.
- DILARANG BERGANTUNG PADA NAMA FUNGSI: Analisis murni berdasarkan variabel/objek yang dimutasi (+=, -=, delete) dan sintaks level bahasa.
- DILARANG MEMANGGIL `skill_manage` ATAU `memory`.

---

## Procedure (Eksekusi Ketat Per Fungsi / Pasangan Fungsi)

Untuk fungsi target yang diambil dari /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/.audit/03-micro-queue.md, jalankan 4 langkah berikut:

### Langkah 1: Persiapan Konteks Mikro (Hemat Token)
1. Baca .audit/00-project-map.md (jika belum dibaca di sesi ini) untuk mengetahui Paradigma Rantai (A/B/C) dan Core Invariants proyek.
2. Baca baris kode fungsi target menggunakan read_file dengan batas baris presisi (150–200 baris).
3. Gunakan search_files (rg) pada .audit/02-storage-map.md dan .audit/01-intake-and-env.md untuk mengambil HANYA definisi variabel storage/objek dan klaim dokumentasi yang disentuh oleh fungsi ini.

### Langkah 2: Bedah RADIUS 0 — 5 Cek Universal + Pemuatan Referensi Dinamis
Periksa baris demi baris fungsi target menggunakan 5 Pemeriksaan Logika Universal berikut, LALU panggil skill_view sesuai tag [LOAD: ...]:
#### A. 5 Pemeriksaan Logika Universal (Wajib di Setiap Fungsi):
1. Kepatuhan Klaim vs Implementasi (Doc-to-Code): Apakah rumus matematika, urutan operasi, atau syarat validasi di fungsi ini menyimpang dari janji di .audit/01-intake-and-env.md atau komentar di atasnya?
2. Otorisasi & Kepemilikan Parameter: Apakah titik masuk ini lupa memverifikasi otoritas pemanggil (msg.sender, is_signer, Capability, atau pengirim pesan Jetton Wallet), atau mengizinkan pemanggil menyuplai alamat/ID objek milik pengguna lain tanpa verifikasi kepemilikan?
3. Simetri Indeks & Ikatan State (Read vs Write Symmetry): Saat membaca suatu mapping, array, atau field objek lalu menulisnya kembali, apakah kunci indeks atau ID objek yang dibaca 100% identik dengan yang ditulis?
4. Input Ekstrem & Referensi Diri (Boundary & Self-Reference):
   - Apa yang terjadi jika argumen nominal bernilai 0, 1 unit terkecil (wei/sun/lamport/mist), atau type(uint).max?
   - Apa yang terjadi jika dua argumen alamat/akun/objek diisi entitas yang sama (from == to, tokenA == tokenB, liquidator == borrower, atau akun duplikat di dalam array [X, X])?
5. Pemanggilan Berulang (Non-Idempotency / Double Call): Jika fungsi ini dipanggil dua kali berturut-turut oleh pengguna (misal mengubah konfigurasi, mengaktifkan bit, bermigrasi, atau mengeklaim hak), apakah state lama dibersihkan sepenuhnya atau justru tertinggal dan memicu *double-counting*?

#### B. Pemuatan Kamus Referensi via skill_view (Wajib Sesuai Tag Fungsi):
Lihat tag [LOAD: ...] pada fungsi ini di .audit/03-micro-queue.md (atau deteksi langsung komponen di dalam kode fungsi), lalu panggil skill_view("sc-phase2-hunt", "references/<nama-file>.md"):
- Modul Spesifik Rantai (Wajib Muat 1 Sesuai Rantai Proyek):
  * references/ref-chain-evm-tron.md (Untuk Solidity/Vyper di EVM & Tron TVM)
  * references/ref-chain-solana.md (Untuk Rust Anchor/Native di Solana)
  * references/ref-chain-sui-move.md (Untuk Move di Sui/Aptos)
  * references/ref-chain-ton-async.md (Untuk FunC/Tact/Tolk di TON)
- Modul Ekonomi & Logika Universal (Muat Sesuai Bahan Baku Fungsi):
  * references/ref-math-and-vaults.md (Jika ada bagi/kali, *shares/LP*, unchecked, *downcasting*, atau operasi bit/tick)
  * references/ref-calls-tokens-reentrancy.md (Jika ada transfer aset/token, pengiriman nilai native, *callback*, atau panggilan keluar)
  * references/ref-defi-oracles-liquidation.md (Jika ada pembacaan harga Oracle, *swap* AMM, akrual bunga/indeks waktu, atau penyitaan/likuidasi)

### Langkah 3: Ekspansi RADIUS 1 & RADIUS 2 (Berbasis Bukti Mutasi State)
Jika ditemukan kejanggalan atau asumsi yang belum tervalidasi di Langkah 2, gunakan search_files (rg -n) untuk memperluas pemeriksaan secara terukur:
1. RADIUS 1 (Intra-Kontrak / Pasangan Siklus Hidup & Pembaca State):
   - Cek Pasangan Mutasi (Mirror State Check): Buka fungsi yang melakukan operasi kebalikan (-=, delete, *burn*, atau *on_bounce*) pada variabel storage / objek yang sama. Apakah seluruh state yang ditambah saat masuk juga dikurangi secara simetris saat keluar, saat penarikan darurat, atau saat likuidasi?
   - Cek Konsumen Variabel (Consumer Check): Cari semua fungsi lain yang MEMBACA variabel storage / objek tersebut untuk menghitung nilai uang, kesehatan pinjaman, atau hak klaim.
2. RADIUS 2 (Lintas Kontrak / Lintas Pesan Asinkron):
   - Lacak ke mana panggilan eksternal atau pesan asinkron mengarah, dan kontrak internal mana saja di repositori yang mempercayai panggilan dari kontrak ini.
   - Khusus Paradigma C (TON Async): Lacak seluruh rantai pesan (Kontrak A -> Kontrak B -> Kontrak C). Periksa apa yang terjadi pada state di Kontrak A jika eksekusi gagal di Kontrak B atau C!

### Langkah 4: Saringan Skeptis (Devil's Advocate) & Pencatatan Hasil
Sebelum menyimpulkan adanya kerentanan, wajib lakukan penyangkalan diri (*Self-Refutation*):
1. Cari bukti yang mematahkan hipotesismu: Apakah ada modifier (nonReentrant, onlyOwner), pengecekan require()/assert!() di fungsi pemanggil/hilir, normalisasi desimal di helper, atau pemulihan state di handler bounced yang membuat serangan ini mustahil menguntungkan penyerang atau merusak protokol?
2. Keputusan Akhir untuk Fungsi Ini:
   - Jika GUGUR / AMAN: Perbarui status fungsi di .audit/03-micro-queue.md menjadi [DONE - AMAN] beserta 1 kalimat alasan teknis mengapa aman.
   - Jika LOLOS SEMUA PENJAGA (POTENSIAL VALID): Perbarui statusnya menjadi [FLAGGED] di .audit/03-micro-queue.md, lalu tulis rincian hipotesisnya (Lokasi File:Baris, Invarian yang Dilanggar, Jejak Mekanis Radius 0 -> 1 -> 2, dan Mengapa Guard Gagal) ke dalam `.audit/04-hypotheses.md` untuk dibuktikan dengan pengujian di Fase 3.
