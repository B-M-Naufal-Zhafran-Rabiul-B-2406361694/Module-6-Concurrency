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