---
name: sc-phase1-recon
description: Fase 0 & 1 Audit Lintas-Rantai - Intake adaptif, deteksi paradigma rantai, pemetaan storage/objek (Data-First), dan penyusunan antrean fungsi berbasis mutasi state
version: 3.0.0
metadata:
  hermes:
    tags: [security, web3, recon, intake, smart-contract, multichain, phase1]
    category: security
    requires_toolsets: [terminal]
---

# Phase 0 & 1: Multi-Chain Adaptive Intake & Data-First Reconnaissance

## When to Use
Gunakan skill ini setiap kali memulai proyek baru di /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/ (atau ketika .audit/00-project-map.md belum tersedia). Skill ini memetakan fondasi proyek dari titik nol tanpa membaca isi logika fungsi secara penuh agar konteks tetap bersih.

## Pitfalls (Larangan Keras di Fase 1)
- DILARANG MEMBUAT `AGENTS.md` DI DALAM FOLDER PROYEK: Seluruh output fase ini WAJIB disimpan di dalam /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/.audit/.
- DILARANG MEMBACA ISI LOGIKA FUNGSI SECARA PENUH: Di fase ini kamu hanya memetakan konfigurasi, deklarasi penyimpanan (contoh: storage, mapping, struct, Object, akun PDA), dan tanda tangan fungsi (*function signatures*) beserta jejak variabel yang diubahnya.
- DILARANG MENGANDALKAN NAMA FUNGSI: Jangan mencari fungsi berdasarkan nama kaku (seperti deposit atau liquidate). Kelompokkan fungsi murni dari mutasi state (+=, -=, delete), kemampuan objek, dan sintaks panggilan keluar.
- ADAPTIF TERHADAP KETIADAAN DOKUMENTASI: Jika dokumentasi eksternal atau halaman bug bounty tidak ada, jangan berhenti. Tandai sebagai KOSONG / N/A dan ekstrak klaim dari README.md serta komentar kode (@notice, @dev, ///).
- DILARANG MEMANGGIL `skill_manage` ATAU `memory`.

---

## Procedure (5 Langkah Berurutan)

Pastikan kamu bekerja pada target /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/. Buat folder .audit/ terlebih dahulu (mkdir -p /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/.audit), lalu jalankan 5 langkah berikut:

### Langkah 1: Fase 0 Intake — Resolusi Scope, Git Diff, & Klaim Protokol
1. Pemangkasan Scope (Scope Pruning):
   - Cari seluruh file kode sumber utama di dalam folder proyek (.sol, .vy, .rs, .move, .fc, .tact, .tolk).
   - Abaikan folder lib/, node_modules/, test/, tests/, mock/, mocks/, dan script/.
   - Hitung jumlah baris kode (wc -l) tiap file untuk membedakan Kontrak/Modul Inti (Core) dari Periphery/Router/Helper.
2. Jejak Perubahan Kode Terbaru (Git History):
   - Jika folder proyek memiliki .git, jalankan git log -n 10 --oneline dan git diff --stat pada commit terakhir untuk menandai file yang baru diubah atau di-upgrade.
3. Ekstraksi Klaim & Janji Ekonomi:
   - Baca README.md atau folder docs/ (jika ada). Jika tidak ada, gunakan rg -n "@notice|@dev|/// " pada file inti.
   - Catat janji eksplisit pengembang mengenai: rumus pembagian saham/bunga, syarat penarikan/likuidasi, batasan otorisasi, dan asumsi keamanan eksternal.

### Langkah 2: Deteksi Paradigma Rantai, Compiler, & Matriks Integrasi
Periksa file konfigurasi (foundry.toml, hardhat.config.*, Cargo.toml, Anchor.toml, Move.toml, blueprint.config.*) dan gunakan rg untuk memetakan 6 Matriks Lingkungan:
1. Identifikasi Paradigma Rantai & Tag Modul Rantai Wajib:
   - EVM / Tron (TVM) (.sol, .vy): Paradigma A (*Atomic Storage*). Cek apakah berjalan di L1, L2 (Arbitrum/Optimism/Base/Scroll -> cek L2 Sequencer feed), atau Tron TVM (waspadai unit 1 TRX = 10^6 SUN vs 10^18 wei). -> Tag Rantai: [LOAD: ref-chain-evm-tron.md]
   - Solana (SVM) (.rs dengan Anchor/Solana): Paradigma A (*Atomic Account/PDA*). Cek Cargo.toml apakah overflow-checks = true aktif di [profile.release]. -> Tag Rantai: [LOAD: ref-chain-solana.md]
   - Sui / Aptos (Move) (.move): Paradigma B (*Object/Capability & PTB*). Identifikasi penggunaan *Shared Object* vs *Owned Object*, Capability, dan *struct* tanpa drop/store (*Hot Potato*). -> Tag Rantai: [LOAD: ref-chain-sui-move.md]
   - TON (.fc, .tact, .tolk): Paradigma C (*Asynchronous Actor*). Tidak ada rollback otomatis lintas kontrak! Identifikasi alur pengiriman pesan (contoh : send_raw_message), keberadaan handler (contoh :bounced), dan arsitektur (contoh:Jetton Wallet). -> Tag Rantai: [LOAD: ref-chain-ton-async.md]
2. Versi Compiler & Perilaku Aritmatika:
   - Catat versi compiler, penggunaan blok unchecked, operasi *casting* penyempitan tipe (uint128, u64), atau operasi geser bit (<<, >>).
3. Pola Proxy, Upgradeability, & Multicall:
   - Gunakan rg -n "Initializable|initialize|UUPS|delegatecall|Multicall|upgrade" untuk mendeteksi risiko *uninitialized implementation*, *storage collision*, atau kombinasi mematikan (contoh:multicall + payable (msg.value)).
4. Matriks Aset & Token (Token Quirks):
   - Identifikasi standar aset yang masuk/keluar: ERC-20/721/1155, TRC-20/TRC-10, SPL Token/Token-2022, Sui Coin<T>, atau TON Jetton.
   - Tandai risiko desimal non-standar (6 atau 8 desimal), *fee-on-transfer*, *rebasing*, *blacklistable/pausable* (USDC/USDT), dan *callback hooks*.
5. Matriks Oracle & AMM:
   - Gunakan rg -n "latestRoundData|getPrice|slot0|getReserves|observe|twap|pyth" untuk mendeteksi sumber harga dan interaksi likuiditas.
6. Matriks Kriptografi & Lintas-Rantai (Bridge/Relayer):
   - Gunakan rg -n "ecrecover|ECDSA|EIP712|permit|MerkleProof|nonces|relay|lzReceive|ccip" untuk mendeteksi verifikasi tanda tangan dan pesan lintas rantai.

Simpan seluruh hasil Langkah 1 dan Langkah 2 ke dalam `.audit/01-intake-and-env.md`.

### Langkah 3: Pemetaan Data & Penyimpanan Menyeluruh (Data-First Mapping)
Baca deklarasi penyimpanan di setiap Kontrak/Modul Inti (tanpa membaca isi fungsi penuh). Kelompokkan seluruh storage, mapping, struct, bitmap, Object (has key), akun PDA, dan constants ke dalam 5 Kategori State di dalam `.audit/02-storage-map.md`:
1. State Akuntansi, Saldo, & Objek Nilai: Variabel/objek yang menyimpan saldo, *shares*, likuiditas, utang, agunan, serta skala desimalnya. Catat invarian keseimbangan wajib (misal: sum(userShares) == totalShares, atau ikatan position.pool_id == object::id(pool)).
2. State Waktu & Indeks Akrual: Variabel pencatat waktu, seperti (lastUpdateTime, rewardIndex, borrowIndex, Clock) yang wajib diperbarui sebelum saldo berubah.
3. State Keamanan, Nonce, & Anti-Replay: Penyimpan, seperti nonces, usedHashes, seqno, merkleRoot.
4. State Kontrol Akses & Otoritas: Penyimpan, seperti owner, admin, Capability (AdminCap), whitelist, atau alamat Jetton Master / Messenger.
5. State Pengunci Siklus & Status Asinkron: Variabel , seperti paused, reentrancyStatus, *Hot Potato* receipt, atau flag pengunci status sementara saat menunggu balasan pesan asinkron di TON.

### Langkah 4: Penyusunan Antrean Mikro Berbasis Mutasi State (.audit/03-micro-queue.md)
Cari seluruh titik masuk (contoh: external, public, pub fn, public entry fun, recv_internal) di kontrak inti. Gunakan rg untuk melihat variabel/objek apa yang disentuh di dalam tiap fungsi, lalu susun daftar antrean di `.audit/03-micro-queue.md` dengan aturan berikut:
1. Klasifikasi & Pasangan Siklus Hidup Berbasis Perilaku (Bukan Nama):
   - Sandingkan Fungsi Penambah Hak/State (+=, *mint/create object*, terima aset) dengan Fungsi Pengurang/Penghapus Hak/State (-=, delete, *burn/destroy object*, tarik aset/likuidasi) yang memodifikasi variabel/objek yang sama.
   - Kelompokkan pula Fungsi Transisi/Konfigurasi dan Fungsi Penerus Eksternal/Pesan (contoh: .call, delegatecall, CPI, send_raw_message).
2. Penempelan Tag `[LOAD: ...]` Wajib pada Setiap Fungsi Antrean:
   Setiap fungsi di .audit/03-micro-queue.md WAJIB memiliki:
   - Satu Tag Modul Rantai sesuai Langkah 2.1 ([LOAD: ref-chain-evm-tron.md], [LOAD: ref-chain-solana.md], [LOAD: ref-chain-sui-move.md], atau [LOAD: ref-chain-ton-async.md]).
   - Satu atau Lebih Tag Modul Ekonomi Universal sesuai bahan baku di dalam fungsi tersebut:
     * [LOAD: ref-math-and-vaults.md] -> Jika fungsi memiliki operasi bagi/kali, perhitungan *shares/LP/vault*, blok unchecked, *downcasting*, atau manipulasi bit/tick.
     * [LOAD: ref-calls-tokens-reentrancy.md] -> Jika fungsi mentransfer token/koin, mengirim/menerima native value, memicu *callback*, atau melakukan panggilan eksternal lintas kontrak.
     * [LOAD: ref-defi-oracles-liquidation.md] -> Jika fungsi membaca harga Oracle, melakukan *swap* AMM, menghitung bunga/kesehatan agunan, atau mengeksekusi penyitaan/likuidasi.
3. Urutan Prioritas: Tempatkan fungsi yang baru diubah di git diff, fungsi penerus pesan lintas kontrak/rantai, dan fungsi mutasi akuntansi utama di urutan paling atas.

### Langkah 5: Pembuatan Peta Utama Proyek (.audit/00-project-map.md)
Sebagai pengganti AGENTS.md lokal, tulis ringkasan eksekutif proyek (maksimal 90 baris) ke dalam `.audit/00-project-map.md` yang berisi:
1. Identitas & Paradigma Proyek: Nama proyek, path absolut, rantai/bahasa, Paradigma Eksekusi (A/B/C), dan perintah kompilasi/test lokal (forge test, cargo test, sui move test, atau npx blueprint test).
2. Daftar Kontrak/Modul In-Scope: Path file inti beserta perannya secara singkat.
3. Core Invariants (Hukum Mutlak Proyek): 5–8 invarian ekonomi dan state dari Langkah 1–3 yang tidak boleh dilanggar dalam kondisi apa pun.
4. Indeks Memori Audit: Tautan ke .audit/01-intake-and-env.md, .audit/02-storage-map.md, dan .audit/03-micro-queue.md.

---

## Verification
Pastikan keempat file di dalam .audit/ (00-project-map.md, 01-intake-and-env.md, 02-storage-map.md, dan 03-micro-queue.md) telah terbentuk dengan lengkap, tidak ada file AGENTS.md yang dibuat di dalam folder proyek, dan setiap fungsi di 03-micro-queue.md sudah memiliki tag [LOAD: ref-...].
