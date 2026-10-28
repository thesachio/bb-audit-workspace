# WORKSPACE CONSTITUTION & MULTI-CHAIN SMART CONTRACT AUDIT RULES
Workspace Root (CWD): /home/ubuntu/.hermes/profiles/bb/workspace
Lokasi Proyek Target: /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>/

Aturan di dalam file ini adalah Konstitusi Operasional Keras yang mengikat Parent Agent maupun seluruh Subagent (`delegate_task`).

## 1. DISIPLIN LINGKUNGAN & PROTOKOL DISPATCHER PROYEK
1. Resolusi Direktori Proyek & Larangan `AGENTS.md` Lokal:
   - Seluruh repositori proyek berada di dalam ./projects/<nama-proyek>/.
   - DILARANG melakukan git init di level workspace/ dan DILARANG membuat file AGENTS.md di dalam folder ./projects/<nama-proyek>/ agar tidak bentrok dengan repo target.
   - Seluruh peta arsitektur proyek, kamus storage/objek, antrean fungsi, dan catatan temuan WAJIB disimpan di dalam folder ./projects/<nama-proyek>/.audit/.
2. Langkah Pertama Mutlak Saat Menyentuh Proyek:
   - Karena sesi berjalan di workspace/, file di bawah ./projects/<nama-proyek>/ tidak dimuat otomatis. Setiap kali pengguna menyebut nama proyek atau meminta analisis:
   - Periksa keberadaan ./projects/<nama-proyek>/.audit/00-project-map.md. Jika SUDAH ADA, wajib baca file tersebut terlebih dahulu menggunakan read_file. Jika BELUM ADA, jalankan prosedur Fase 1 (sc-phase1-recon) terlebih dahulu.
3. Larangan Mutasi Global:
   - DILARANG KERAS memanggil tool skill_manage atau memory untuk menyimpan informasi spesifik proyek yang sedang diaudit.

## 2. HUKUM KOGNITIF INSPEKSI LINTAS-RANTAI (WAJIB BAGI PARENT & SUBAGENT)
1. KENALI 3 PARADIGMA EKSEKUSI RANTAI SEJAK AWAL:
   Jangan pernah memaksakan asumsi satu rantai ke rantai lain. Sesuaikan analisis dengan paradigma proyek:
   - Paradigma A — Atomic Account/Storage (EVM, Tron/TVM, Solana):
     Transaksi bersifat atomik (revert membatalkan seluruh state). Lacak mutasi pada storage/mapping atau akun PDA (+=, -=, delete). Waspadai perbedaan unit dasar (misal Tron 1 TRX = 10^6 SUN vs EVM 10^18 wei), *reentrancy* lintas fungsi/read-only, serta validasi akun dinamis (remaining_accounts di Solana).
   - Paradigma B — Object/Capability & PTB (Sui, Aptos / Move):
     Data disimpan di dalam Object (`UID`), bukan mapping global. Fokus pada validasi ID lintas objek (assert!(obj.pool_id == object::id(pool))), kebocoran Capability, celah *struct* Hot Potato di dalam *Programmable Transaction Blocks (PTB)*, serta *overflow* pada operasi geser bit (<<, >>).
   - Paradigma C — Asynchronous Actor & Message Passing (TON / FunC / Tact / Tolk):
     TIDAK ADA ROLLBACK OTOMATIS LINTAS KONTRAK! Jika Kontrak A mengubah state lalu mengirim pesan ke Kontrak B dan gagal di Kontrak B, state Kontrak A TIDAK batal otomatis. Wajib periksa: (a) keberadaan & kelengkapan handler bounced (ingat batas payload bounce 256-bit!), (b) kecukupan *forward gas* (msg_value) di sepanjang rantai pesan, (c) validasi pengirim Jetton Wallet, dan (d) *race condition* antar-pesan asinkron.
2. DETEKSI BERBASIS MUTASI STATE & SINTAKS (BUKAN NAMA FUNGSI):
   - Dilarang menebak peran fungsi dari namanya (deposit, withdraw, liquidate, frob).
   - Kenali peran fungsi murni dari variabel storage / objek apa yang diubah (+=, -=, delete, mutasi field objek) dan kata kunci level bahasa (.call, delegatecall, msg.value, block.timestamp, remaining_accounts, send_raw_message, transfer::public_transfer).
3. MIKRO-KE-MAKRO & DATA-FIRST (ANTI AWANG-AWANG):
   - Dilarang membaca seluruh repositori sekaligus. Gunakan search_files (rg -n) dan batasi read_file maksimal 150–200 baris per panggilan.
   - Abaikan folder lib/, node_modules/, mock/, mocks/, script/, dan test/ saat memetakan logika inti.
   - Pahami struktur data (storage, struct, Object, constants) terlebih dahulu sebelum membaca logika fungsi.
   - Saat membedah fungsi (Fase 2), mulai dari Radius 0 (1 fungsi target), panggil skill_view pada modul references/ yang relevan di skill sc-phase2-hunt, lalu perluas ke Radius 1 (fungsi pasangan/pembaca state yang sama) dan Radius 2 (lintas kontrak/pesan) hanya jika ada pemicu anomali.
4. DEVIL'S ADVOCATE & ZERO-NOISE:
   - Abaikan isu kualitas kode dangkal (*floating pragma*, *gas optimization*, *naming*, *centralization* standar).
   - Sebelum mengklaim bug, wajib cari apakah ada modifier, require(), assert!(), handler bounced, atau batasan sistem yang menggagalkan skenario serangan tersebut.

## 3. PROTOKOL WAJIB DELEGASI SUBAGENT (delegate_task)
Karena Subagent tidak mewarisi riwayat obrolan Parent, melewati SOUL.md (skip_soul=True), dan cwd awalnya berada di workspace/, setiap kali Parent memanggil delegate_task, parameter context WAJIB menyertakan secara eksplisit:
1. Path Absolut Proyek: /home/ubuntu/.hermes/profiles/bb/workspace/projects/<nama-proyek>
2. Paradigma & Rantai Proyek: (Misal: EVM, Solana, Sui Move, atau TON Async).
3. File Konteks `.audit/` & Target Spesifik: Path ke file di .audit/ (seperti 00-project-map.md dan 02-storage-map.md) beserta nama file dan rentang baris fungsi yang harus diperiksa.
4. Instruksi Pemuatan Referensi: Perintahkan Subagent secara eksplisit memanggil skill_view("sc-phase2-hunt", "references/...") sesuai tag [LOAD: ...] pada fungsi target.
