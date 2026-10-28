# Checklist Universal: Oracle Harga, AMM/DEX, Akrual Waktu, & Mekanika Likuidasi

Periksa setiap pembacaan harga, perhitungan bunga, dan penyitaan agunan terhadap 6 vektor berikut:

1. Manipulasi Harga Spot dalam Satu Transaksi (Flash Loan Spot Manipulation):
   - Apakah protokol menghitung nilai agunan, harga aset, atau rasio pencetakan menggunakan saldo/cadangan instan dari pool AMM saat ini (getReserves, slot0, atau rasio saldo pool) tanpa rata-rata waktu (TWAP)?
   - Semua harga *spot* on-chain dapat dimanipulasi secara ekstrem dalam satu transaksi atomik menggunakan *Flash Loan*.

2. Validasi Oracle Eksternal yang Cacat (Chainlink / Pyth / TWAP):
   - Jika membaca data Oracle (seperti latestRoundData atau Pyth price feed), pastikan kode memvalidasi:
     *(a)* Harga > 0 dan interval kepercayaan (*confidence interval* pada Pyth) tidak terlalu lebar.
     *(b)* Kesegaran data: updatedAt >= currentTime - MAX_STALENESS (pastikan nilai *heartbeat* sesuai dengan feed asetnya).
     *(c)* Jika di jaringan Layer-2 EVM: apakah memeriksa status L2 Sequencer Uptime Feed beserta masa tenggang (*grace period*) setelah sequencer pulih?

3. Ketiadaan Slippage & Deadline yang Efektif (MEV Sandwich Attack):
   - Jika kontrak mengeksekusi pertukaran (*swap*) atau menambah/menarik likuiditas di DEX secara otomatis:
   - Apakah batas minimum penerima (amountOutMin) disetel ke 0 atau dihitung secara on-chain dari harga spot yang sudah dimanipulasi?
   - Apakah batas waktu (deadline) disetel ke block.timestamp (yang berarti transaksi dapat ditahan di mempool tanpa kedaluwarsa)?

4. Perubahan State Sebelum Akrual Waktu (Missing Interest / Reward Accrual):
   - Pada sistem pinjaman, staking, atau pembagian hasil: apakah indeks akrual waktu (accrueInterest / updatePool) WAJIB dijalankan paling awal sebelum saldo pengguna atau total pasokan diubah?
   - Jika ada satu jalur fungsi (misal fungsi transfer internal, klaim, atau penarikan darurat) yang mengubah saldo tanpa memperbarui indeks waktu terlebih dahulu, pengguna dapat mencuri porsi hasil masa lalu atau menghindari beban bunga.

5. Kegagalan Mekanika Likuidasi & Penanganan Bad Debt:
   - Insolvency / Bad Debt: Jika nilai agunan jatuh di bawah nilai utang, apakah likuidator masih memiliki insentif ekonomi untuk melikuidasi, dan apakah sisa utang tanpa agunan (*bad debt*) dibersihkan/disosialisasikan agar tidak merusak akuntansi protokol?
   - Posisi Debu (Dust Positions): Apakah ada batas pinjaman minimum agar biaya gas likuidasi tidak lebih mahal daripada nilai agunan yang disita?
   - Self-Liquidation & Pause Freeze: Apakah peminjam bisa melikuidasi dirinya sendiri dengan keuntungan tidak sah? Apakah status paused memblokir fungsi pelunasan (repay) atau likuidasi sehingga memicu gagal bayar massal?
