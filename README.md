## Commit 1 Reflection notes

Di dalam fungsi `handle_connection`, terdapat beberapa poin penting mengenai bagaimana Rust menangani request HTTP yang masuk:

* **`BufReader`**: Digunakan untuk membungkus `TcpStream` (mutable). `BufReader` menambahkan *buffering* sehingga pembacaan data dari *socket* menjadi lebih efisien dibandingkan membaca byte demi byte secara langsung dari *stream*.
* **`.lines()`**: Metode ini memecah aliran data dari *stream* berdasarkan karakter baris baru (`\n`). Karena HTTP *request* bersifat teks berbasis baris, ini mempermudah kita membaca pesan dari browser secara terstruktur.
* **`.map(|result| result.unwrap())`**: Karena setiap baris dalam `lines()` dibungkus dalam `Result` (untuk menangani potensi error I/O), kita perlu melakukan *unwrap* untuk mendapatkan `String` aslinya.
* **`.take_while(|line| !line.is_empty())`**: Protokol HTTP menyatakan bahwa akhir dari *header* ditandai dengan baris kosong (*blank line*). Fungsi ini memastikan kita berhenti membaca saat bertemu baris kosong tersebut, sehingga program tidak menunggu data selamanya (*hanging*) karena kita hanya ingin mengambil bagian *header* saja untuk saat ini.
* **`.collect()`**: Mengumpulkan hasil iterasi baris-baris tersebut ke dalam sebuah koleksi, dalam hal ini adalah `Vec<String>`.


## Commit 2 Reflection notes

Pada Milestone ini, fungsi `handle_connection` dimodifikasi untuk mengirimkan respons HTTP yang valid kembali ke browser:

1. **`status_line`**: Ini adalah baris pertama respons HTTP yang memberi tahu browser bahwa permintaan berhasil (`200 OK`).
2. **`fs::read_to_string("hello.html")`**: Digunakan untuk membaca isi file HTML secara utuh ke dalam sebuah `String`.
3. **`Content-Length`**: Header ini sangat penting karena memberi tahu browser seberapa besar data yang akan dikirim (dalam *bytes*), sehingga browser tahu kapan transmisi selesai.
4. **`format!`**: Digunakan untuk menyusun respons HTTP sesuai protokol. Terdapat dua kali `\r\n` (baris kosong) antara header dan isi (*body*) file HTML, sesuai aturan protokol HTTP.
5. **`stream.write_all`**: Mengonversi string respons menjadi *bytes* dan mengirimkannya melalui *socket* ke browser.

![Commit 2 screen capture](commit2.png)


## Commit 3 Reflection notes

Pada Milestone ini, kode telah direfaktor untuk memisahkan antara logika pengecekan request dan pembuatan respons. 

1. **Refactoring**: Alih-alih menggunakan blok `if/else` yang mengulang kode `fs::read_to_string` dan `stream.write_all`, saya menggunakan `if/else` sebagai *expression* untuk menentukan `status_line` dan `filename`. Hal ini membuat kode lebih *DRY (Don't Repeat Yourself)*, lebih bersih, dan lebih mudah dipelihara.
2. **Validasi Request**: Server sekarang hanya akan memberikan halaman utama jika `request_line` bernilai tepat `"GET / HTTP/1.1"`. Jika user meminta path lain, server secara eksplisit mengirimkan status `404 NOT FOUND`.
3. **Pemisahan Respons**: Dengan cara ini, struktur respons (format HTTP) tetap konsisten, hanya konten dan statusnya saja yang berubah secara dinamis berdasarkan input user.

![Commit 3 screen capture](commit3.png)


## Commit 4 Reflection notes

Pada Milestone ini, saya mensimulasikan respons lambat untuk memahami keterbatasan server *single-threaded*:

1. **Simulasi `/sleep`**: Dengan menggunakan `thread::sleep(Duration::from_secs(10))`, server dipaksa berhenti bekerja selama 10 detik sebelum memproses respons.
2. **Masalah Blocking**: Karena server ini berjalan pada satu *thread* tunggal, ia hanya bisa memproses satu permintaan dalam satu waktu. Ketika permintaan `/sleep` masuk, seluruh proses server tertahan (*blocked*). Akibatnya, permintaan lain yang masuk (bahkan yang seharusnya cepat seperti `/`) harus mengantri sampai permintaan sebelumnya selesai diproses.
3. **Dampak Real-world**: Dalam kondisi nyata, jika satu pengguna memicu proses yang berat (seperti pengolahan data besar), pengguna lain tidak akan bisa mengakses website sama sekali. Hal ini menunjukkan perlunya mekanisme *multi-threading* atau *thread pool* untuk menangani banyak permintaan secara paralel.


## Commit 5 Reflection notes

Pada Milestone ini, saya mengimplementasikan `ThreadPool` untuk meningkatkan *throughput* server:

1. **Ide Utama**: Dibandingkan membuat thread baru tanpa batas untuk setiap koneksi (yang berisiko membanjiri sumber daya sistem), `ThreadPool` mengelola sejumlah thread yang sudah ditentukan sejak awal (dalam hal ini 4).
2. **Penggunaan `Arc<Mutex<T>>`**: Karena ujung penerima (*Receiver*) pada channel Rust bersifat *Single Consumer*, saya menggunakan `Arc` (Atomic Reference Counting) agar *receiver* dapat dimiliki oleh banyak `Worker`, dan `Mutex` untuk memastikan hanya satu `Worker` yang mengambil satu pekerjaan dalam satu waktu agar tidak terjadi *race condition*.
3. **Mekanisme Kerja**: Saat ada koneksi baru, `ThreadPool` akan mengirimkan tugas melalui *channel*. Salah satu `Worker` yang sedang menganggur akan mengambil tugas tersebut, menjalankannya, dan kembali menunggu tugas baru setelah selesai. Hal ini memungkinkan akses ke `/` tetap cepat meskipun ada user lain yang sedang mengakses `/sleep`.


## Commit Bonus Reflection notes

Saya melakukan perbaikan pada cara pembuatan `ThreadPool` dengan menambahkan fungsi `build` sebagai alternatif dari `new`:

1. **Perbedaan `new` vs `build`**: Fungsi `new` biasanya digunakan oleh konvensi Rust untuk inisialisasi yang diasumsikan tidak akan gagal. Namun, jika ada input yang tidak valid (seperti `size = 0`), `new` akan memicu `panic!`. Sebaliknya, fungsi `build` mengembalikan tipe `Result<ThreadPool, PoolCreationError>`.
2. **Keuntungan `Result`**: Dengan mengembalikan `Result`, kita memberikan kendali penuh kepada pemanggil fungsi (*caller*) untuk menangani error secara elegan (misal: memberikan pesan error yang ramah user) daripada membiarkan seluruh program berhenti mendadak.
3. **Robustness**: Pola ini membuat kode lebih tangguh dan sesuai dengan filosofi penanganan error di Rust yang eksplisit.