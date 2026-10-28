# Checklist Paradigma Rantai: Sui & Aptos (Move — Object, Capability, & PTB)

Gunakan daftar periksa ini saat mengaudit modul Move di Sui atau Aptos:

1. Kebingungan ID Lintas Objek (Cross-Object ID Confusion / Spoofing):
   - Di Move, data disimpan di dalam objek mandiri (has key). Ketika sebuah fungsi menerima dua atau lebih objek (misal pool: &mut Pool<A, B> dan position: &mut Position atau receipt: &Receipt), kepemilikan tipe Move tidak menjamin bahwa `position` tersebut berasal dari `pool` yang sama!
   - Wajib periksa apakah ada asersi eksplisit: assert!(position.pool_id == object::id(pool), EWrongPool). Tanpa pemeriksaan ID ini, penyerang dapat membuat Pool baru dengan koin tidak berharga, mencetak Position besar di sana, lalu menggunakannya untuk menguras dana dari Pool utama!

2. Celah Pola "Hot Potato" & Manipulasi Programmable Transaction Blocks (PTB):
   - Di Sui, pemanggil dapat merangkai puluhan panggilan fungsi publik berbeda dalam satu transaksi atomik melalui PTB tanpa menulis kontrak baru.
   - Untuk alur *Flash Loan* atau eksekusi bertahap, periksa struct Hot Potato (struct tanpa kemampuan drop, store, atau key):
     *(a)* Apakah struct tersebut tidak sengaja diberi kemampuan drop atau store (sehingga pinjaman tidak perlu dikembalikan)?
     *(b)* Apakah struct tersebut menyimpan dan memverifikasi pool_id serta jumlah bayar kembali secara tepat?
     *(c)* Apakah ada fungsi publik lain di modul tersebut yang dapat mengonsumsi/menghancurkan *Hot Potato* tersebut dengan syarat yang lebih ringan?

3. Overflow pada Operasi Geser Bit & Matematika Kustom (Kasus Cetus $223M):
   - Meskipun VM Move memeriksa *overflow* secara otomatis pada +, -, dan *, operasi geser bit kiri (<<) pada u64/u128/u256 di Move TIDAK memeriksa apakah bit signifikan yang bergeser keluar telah hilang (terpotong), atau akan *abort* jika jumlah geseran melebihi lebar bit!
   - Periksa seluruh fungsi matematika kustom (seperti checked_shlw, mul_shr, perhitungan sqrt_price atau likuiditas CLMM): apakah batas *masking* pemeriksaan overflow sudah benar? Kesalahan satu bit pada batas mask memungkinkan penyerang memasukkan angka besar yang terpotong menjadi 1 saat dihitung sebagai kewajiban setor, namun mencetak likuiditas penuh!

4. Kebocoran Hak `Capability` & Pembekuan Shared Object (DoS):
   - Periksa fungsi yang menerima atau mengembalikan objek kemampuan (AdminCap, TreasuryCap, UpgradeCap). Pastikan tidak ada fungsi publik yang mengizinkan peminjaman mutlak atau pembungkusan (*wrapping*) Capability oleh pihak tak berhak.
   - Pastikan objek bersama (*Shared Object* yang diakses semua pengguna, seperti Pool atau GlobalState) tidak pernah bisa dikirim ke alamat pribadi menggunakan transfer::public_transfer (yang akan mengubahnya menjadi *Owned Object* dan membekukan seluruh protokol selamanya).

5. Substitusi Tipe Generik (`Phantom Type <T>` Spoofing):
   - Pada fungsi yang menggunakan parameter tipe generik public fun execute<CoinA, CoinB>(...), pastikan kode memverifikasi bahwa tipe <CoinA, CoinB> yang dimasukkan pemanggil benar-benar sesuai dengan tipe koin yang terdaftar di dalam konfigurasi Vault/Pool target.
