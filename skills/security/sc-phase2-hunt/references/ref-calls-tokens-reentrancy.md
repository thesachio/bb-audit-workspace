# Checklist Universal: Interaksi Eksternal, Reentrancy, & Perilaku Token/Aset

Periksa setiap transfer aset, panggilan keluar, dan perubahan state terhadap 6 vektor berikut:

1. Cross-Function & Cross-Contract Reentrancy:
   - Jika fungsi ini melakukan panggilan eksternal (pengiriman native coin, transfer token dengan hook/callback, atau CPI/panggilan kontrak luar) SEBELUM seluruh mutasi state selesai:
   - Meskipun fungsi ini memakai nonReentrant, periksa di Radius 1 & 2: apakah ada fungsi lain di kontrak ini atau di kontrak pasangan yang TIDAK memakai pengunci yang sama, namun membaca atau memodifikasi variabel state yang sedang belum sinkron tersebut?

2. Read-Only Reentrancy (Manipulasi Fungsi View saat Callback):
   - Pada saat eksekusi keluar terjadi di tengah fungsi (misal ketika saldo aset sudah berubah tetapi jumlah totalShares belum diperbarui, atau sebaliknya), apakah ada fungsi pembaca harga/rasio (getRate, getVirtualPrice, totalAssets) yang dapat dibaca oleh kontrak terintegrasi dengan nilai yang sedang miring?

3. Selisih Saldo Masuk (Fee-on-Transfer / Deflationary / Rebasing Assets):
   - Apakah fungsi langsung menambahkan parameter input ke dalam pencatatan state (state += amount) setelah menarik token dari pengguna?
   - Jika protokol tidak membatasi token secara ketat, token dengan potongan biaya transfer akan membuat catatan internal lebih besar daripada saldo riil di brankas kontrak. Wajib ukur selisih balanceAfter - balanceBefore.

4. Ketidaksesuaian Skala Desimal Aset (Decimals Mismatch):
   - Jangan pernah berasumsi semua aset berdesimal 18! (USDC/USDT/TRX = 6 desimal, WBTC/APT = 8 desimal, SOL/SUI = 9 desimal).
   - Periksa setiap rumus yang menggabungkan dua aset berbeda atau mengalikan dengan konstanta tetap (1e18): apakah sudah dinormalisasi terhadap desimal masing-masing aset?

5. Penolakan Layanan via Pengiriman Paksa (Push-Payment / Blacklist DoS):
   - Jika fungsi mengirim dana secara langsung (*push*) ke alamat pengguna lama (misal saat pengembalian tawaran lelang, likuidasi, atau distribusi hasil dalam loop):
   - Apa yang terjadi jika penerima adalah kontrak yang menolak pembayaran native coin, atau akun yang masuk daftar hitam (*blacklist* seperti pada USDC/USDT), atau akun yang telah ditutup? Jika satu pengiriman gagal menyebabkan seluruh fungsi revert, fitur kritis protokol akan macet permanen (DoS).

6. Perilaku Return Value Non-Standar & Side-Effect Callback:
   - Pada EVM, apakah transfer ERC-20 sudah menggunakan SafeERC20 untuk menangani token yang tidak mengembalikan boolean? Apakah ada risiko callback dari ERC-777, ERC-721 (safeTransferFrom/safeMint), atau ERC-1155?
