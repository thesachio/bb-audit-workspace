# Checklist Paradigma Rantai: TON (FunC / Tact / Tolk — Asynchronous Actor Model)

Gunakan daftar periksa ini saat mengaudit smart contract di jaringan TON. INGAT HUKUM UTAMA TON: Tidak ada transaksi atomik lintas kontrak! Kegagalan di kontrak hilir TIDAK membatalkan perubahan state di kontrak hulu secara otomatis!

1. Ketiadaan atau Cacat pada Handler Pesan `bounced` (Batas 256-Bit):
   - Setiap kali sebuah fungsi mengubah variabel penyimpanan (misal mengurangi saldo, menandai pesanan, atau mencetak klaim) lalu mengirim pesan keluar (send_raw_message), lacak apa yang terjadi jika kontrak tujuan gagal (*throw/revert*)!
   - Cek Wajib di Radius 1:
     *(a)* Apakah pesan dikirim dengan flag *bounceable* (0x18) dan apakah kontrak pengirim memiliki cabang penanganan bounced di dalam recv_internal?
     *(b)* Jebakan Batas 256-Bit: Pesan bounced di TON dipotong oleh jaringan dan HANYA mengembalikan 256 bit pertama dari isi (*body*) pesan asli (dikurangi 32 bit prefix 0xffffffff, sehingga hanya tersisa 224 bit data!). Jika informasi jumlah dana atau alamat pengguna yang harus dipulihkan ditaruh setelah bit ke-224 atau di dalam referensi *cell* (ref), handler bounced tidak akan bisa membacanya dan state/dana akan rusak permanen!

2. Putusnya Rantai Pesan Akibat Kekurangan Gas (`msg_value` Exhaustion):
   - Dalam alur multi-hop (misal User -> Kontrak A -> Kontrak B -> Kontrak C), jika Kontrak A sudah mengubah state sebelum memastikan bahwa msg_value yang dikirim pengguna cukup untuk membayar biaya komputasi (compute_fee) dan biaya penerusan (fwd_fee) untuk seluruh rantai pesan hingga Kontrak C, maka pesan dapat kehabisan gas di Kontrak B.
   - Kegagalan akibat kehabisan gas di tengah jalan atau pada fase *action* sering kali tidak memicu pesan bounced yang utuh kembali ke kontrak pertama, menyebabkan *state desynchronization* permanen.

3. Pemalsuan Notifikasi Token (`Fake Jetton Wallet Spoofing`):
   - Di TON, token (Jetton) menggunakan arsitektur terdistribusi di mana setiap pemilik memiliki kontrak Jetton Wallet tersendiri.
   - Saat kontrak protokol menerima pesan transfer_notification (op 0x7362d09c) atau internal_transfer, DILARANG KERAS langsung mempercayai isi jumlah token di pesan tersebut! Kontrak WAJIB memverifikasi bahwa sender_address pesan tersebut adalah alamat Jetton Wallet resmi milik kontrak protokol sendiri (atau memvalidasi alamatnya menggunakan perhitungan state_init dari Jetton Master). Tanpa ini, penyerang dapat membuat Jetton palsu dan mengirim pesan transfer_notification bohong untuk menguras brankas protokol!

4. Kondisi Balapan Antar-Pesan Asinkron (Async Race Condition / Partial State):
   - Karena pengiriman pesan dari Kontrak A ke Kontrak B memakan waktu antar-blok, selama Kontrak A sedang menunggu pesan balasan (*callback*) dari Kontrak B, pengguna yang sama (atau penyerang) dapat menyisipkan pesan baru ke Kontrak A!
   - Periksa apakah Kontrak A mengunci status pengguna/posisi (is_locked / pending_query_id) selama proses asinkron berlangsung agar pengguna tidak bisa menarik agunan atau memicu dua penarikan ganda sebelum balasan pertama tiba.

5. Penyalahgunaan Mode Pengiriman (`send_raw_message` Mode 128 vs 64) & Storage Fee Drain:
   - Mode 128 mengirimkan seluruh saldo TON milik kontrak, sedangkan Mode 64 hanya mengembalikan sisa msg_value dari pesan masuk. Pastikan tidak ada jalur yang memungkinkan pengguna memicu Mode 128 atau menguras saldo TON kontrak di bawah batas minimum cadangan biaya sewa penyimpanan (raw_reserve), yang dapat menyebabkan kontrak dibekukan oleh jaringan.
