# Checklist Paradigma Rantai: EVM (Ethereum/L2) & Tron (TVM)

Gunakan daftar periksa ini saat mengaudit kontrak Solidity/Vyper di jaringan EVM maupun Tron TVM:

1. Penyalahgunaan `msg.value` di Dalam Loop, Multicall, atau Delegatecall:
   - Sangat Kritis: Di dalam EVM/TVM, nilai msg.value bersifat tetap sepanjang konteks panggilan dan TIDAK berkurang setelah dibaca satu kali.
   - Jika sebuah fungsi menggunakan msg.value di dalam perulangan (for loop), atau jika kontrak mewarisi Multicall (yang mengeksekusi banyak panggilan via delegatecall), pemanggil dapat mengirim 1 unit koin satu kali dan memakainya berulang kali di setiap iterasi untuk mencetak/membeli aset berkali-kali lipat!

2. Celah Proxy, Upgradeability, & Uninitialized Implementation:
   - Apakah kontrak implementasi memanggil _disableInitializers() di dalam constructor()? Tanpa itu, siapa pun dapat memanggil initialize() langsung pada kontrak implementasi.
   - Apakah fungsi initialize() dapat dipanggil lebih dari sekali (lupa memasang modifier initializer), atau apakah terdapat tabrakan slot penyimpanan (*storage collision*) antara Proxy dan Implementation?

3. Kerentanan Kriptografi, EIP-712, Permit, & Merkle Proof:
   - Signature Replay: Apakah hash tanda tangan mengikat nonce (yang langsung dinaikkan di storage), block.chainid, dan address(this)?
   - Zero-Address `ecrecover`: Apakah hasil ecrecover diverifikasi signer != address(0)?
   - Permit Front-Running DoS: Jika fungsi wajib mengeksekusi token.permit(...) tanpa blok try/catch, penyerang dapat mengambil tanda tangan dari mempool dan memanggil permit lebih dulu agar transaksi utama pengguna gagal (revert) karena nonce sudah naik.
   - Hash Collision: Jangan pernah gunakan abi.encodePacked pada lebih dari satu tipe data dinamis (string, bytes, array).

4. Celah Jembatan Lintas Rantai (Bridge / Relayer Message Spoofing):
   - Jika kontrak *Messenger/Relayer* mengeksekusi panggilan dinamis _to.call(_data), periksa apakah SELURUH kontrak internal berhak istimewa yang mempercayai msg.sender == Messenger sudah masuk ke daftar hitam (*blacklist*) _to.

5. Jebakan EVM Tingkat Rendah (EIP-1153 Transient Storage & Return Bomb):
   - Jika menggunakan *Transient Storage* (TSTORE/TLOAD), pastikan slot dibersihkan di akhir panggilan agar tidak bocor ke panggilan berikutnya di dalam transaksi yang sama.
   - Pada panggilan .call() ke alamat yang tidak dipercaya, waspadai *Return Bomb* (penyalinan memori balik berukuran besar yang menghabiskan gas pemanggil).

6. KHUSUS TRON (TVM) — Perangkap Unit `SUN` vs `Wei` & TRC-10:
   - Di Tron TVM, 1 TRX = 10^6 SUN (hanya 6 desimal, BUKAN 18 desimal seperti 1 ether = 10^18 wei). Periksa semua kode warisan EVM yang masih mengalikan/membagi msg.value dengan 1e18!
   - Waspadai pemisahan antara pembayaran native TRX (msg.value) dan token bawaan TRC-10 (msg.tokenvalue & msg.tokenid), serta keunikan penanganan *Energy* pada panggilan eksternal TVM.
