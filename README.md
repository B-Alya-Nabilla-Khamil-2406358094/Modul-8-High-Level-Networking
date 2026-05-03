# Rust gRPC Tutorial - Refleksi

## 1. Perbedaan Utama antara Unary, Server Streaming, dan Bi-Directional Streaming

Unary merupakan pola komunikasi paling sederhana, di mana klien mengirimkan satu permintaan dan server merespons dengan satu jawaban. Pola ini cocok untuk operasi singkat seperti pemrosesan pembayaran atau autentikasi pengguna.

Server Streaming adalah pola di mana klien mengirimkan satu permintaan, tetapi server merespons dengan beberapa jawaban secara bertahap. Pola ini sesuai untuk kasus-kasus seperti pengambilan riwayat transaksi yang besar atau pembaruan harga saham secara langsung.

Bi-Directional Streaming memungkinkan baik klien maupun server untuk saling mengirim pesan secara bersamaan tanpa tergantung pada urutan permintaan-respons. Pola ini sangat efektif untuk aplikasi obrolan, kolaborasi dokumen secara real-time, atau sistem notifikasi dua arah.

## 2. Pertimbangan Keamanan dalam Implementasi gRPC di Rust

Aspek pertama yang harus diperhatikan adalah enkripsi data menggunakan TLS atau mTLS. Tanpa enkripsi TLS, semua data yang ditransmisikan termasuk kredensial dan token dapat dibaca oleh pihak yang tidak berhak di jaringan. Tonic sudah mendukung TLS melalui `tonic::transport::ServerTlsConfig`.

Autentikasi juga merupakan aspek kritis. Implementasi dapat menggunakan JWT atau token OAuth2 yang dikirimkan melalui metadata gRPC di setiap permintaan untuk memverifikasi identitas pengguna.

Untuk otorisasi, dapat digunakan interceptor untuk memeriksa izin pengguna sebelum memproses panggilan RPC. Validasi data masukan juga sangat penting untuk mencegah serangan injeksi atau pemrosesan data yang tidak valid yang dapat mengompromikan sistem.

## 3. Tantangan dalam Menangani Bi-Directional Streaming untuk Chat

Salah satu tantangan utama adalah deadlock, yaitu kondisi di mana klien dan server saling menunggu pesan satu sama lain tanpa ada inisiatif pengiriman, sehingga komunikasi dapat berhenti total.

Masalah backpressure juga signifikan, yaitu ketika salah satu pihak mengirim data terlalu cepat sementara pihak lain tidak dapat memproses dengan kecepatan yang sama. Kondisi ini menyebabkan buffer penuh dan potensi kehilangan pesan. Jika koneksi terputus secara tiba-tiba tanpa penanganan error yang memadai, pesan yang belum dikirimkan dapat hilang tanpa notifikasi. Penutupan stream yang benar tanpa mengganggu pesan dalam proses juga merupakan kompleksitas tersendiri dalam lingkungan asinkron Rust.

## 4. Keuntungan dan Kekurangan tokio_stream::wrappers::ReceiverStream

ReceiverStream sangat intuitif karena berfungsi sebagai jembatan antara channel tokio::mpsc dengan stream gRPC. Integrasi yang baik dengan ekosistem async Tokio memungkinkan spawn task terpisah untuk menghasilkan data tanpa memblokir handler gRPC. Backpressure ditangani secara otomatis melalui bounded channel, sehingga jika buffer penuh, pengirim akan menunggu secara otomatis.

Namun demikian, ada beberapa kekurangan. Terdapat satu lapisan channel tambahan yang menambah latency minimal. Jika ukuran buffer channel terlalu kecil dan producer mengirim data terlalu cepat, producer dapat tertahan. Penanganan error juga lebih kompleks karena harus dikirim melalui channel sebagai Result, dan fleksibilitas kontrol aliran data sangat terbatas.

## 5. Cara Menyusun Kode gRPC Rust agar Mudah Dirawat dan Dikembangkan

Struktur kode yang baik memerlukan pemisahan setiap service ke dalam file terpisah, misalnya services/payment.rs dan services/transaction.rs, untuk mencegah penumpukan kode dalam satu file besar. Penggunaan trait untuk mendefinisikan kontrak service memudahkan substitusi dan testing dengan mock object.

Pembuatan modul utilitas bersama untuk logika yang digunakan oleh banyak service, seperti autentikasi atau logging, sangat membantu. Dependency injection melalui penyuntikan koneksi database atau state bersama ke dalam struct service, bukan deklarasi global, juga penting. Penggunaan satu file lib.rs untuk mengekspos semua tipe yang di-generate dari proto mencegah duplikasi import.

## 6. Langkah Tambahan untuk Payment Processing yang Lebih Kompleks

Dalam tutorial sederhana, MyPaymentService hanya mengembalikan success: true. Untuk aplikasi produksi, diperlukan beberapa komponen tambahan. Pertama, koneksi database diperlukan untuk menyimpan data transaksi secara permanen. Idempotency key harus diterapkan untuk mencegah pemrosesan ganda permintaan yang sama dan menghindari charge berlipat.

Integrasi dengan payment gateway seperti Midtrans atau Stripe dengan logika retry untuk kegagalan juga wajib ada. Validasi saldo atau limit transaksi sebelum pemrosesan pembayaran harus diimplementasikan. Error mapping yang tepat ke kode gRPC seperti INVALID_ARGUMENT atau FAILED_PRECONDITION harus ditangani dengan benar. Terakhir, audit log untuk mencatat setiap transaksi diperlukan untuk kepatuhan regulasi dalam konteks produksi.

## 7. Dampak Penggunaan gRPC terhadap Arsitektur Sistem Terdistribusi

gRPC mengharuskan setiap service mendefinisikan kontrak melalui file .proto sebelum implementasi. Hal ini membuat antarmuka antar service menjadi eksplisit dan terdokumentasi secara otomatis, sehingga lebih mudah dipahami oleh tim pengembang.

Dengan gRPC, service yang ditulis dalam Rust dapat berkomunikasi langsung dengan service Java, Python, atau Go menggunakan file .proto yang sama. Komunikasi antar service juga jauh lebih efisien dibandingkan REST/JSON berkat kombinasi HTTP/2 dan Protocol Buffers, terutama dalam arsitektur microservice yang memerlukan banyak interaksi antar service.

Namun gRPC tidak dapat digunakan langsung dari browser tanpa gRPC-Web, sehingga untuk API publik yang menghadap pengguna akhir, REST masih lebih praktis.

## 8. Keuntungan dan Kekurangan HTTP/2 yang Digunakan gRPC dibanding HTTP/1.1

HTTP/2 memiliki fitur multiplexing yang memungkinkan banyak permintaan dikirim secara bersamaan dalam satu koneksi tanpa harus antri. Fitur ini menghilangkan masalah head-of-line blocking yang umum terjadi di HTTP/1.1. HTTP/2 juga menggunakan header compression melalui HPACK, sehingga header yang biasanya besar dapat dikompresi dan menghemat bandwidth. Data dikirim dalam format biner yang lebih efisien dibandingkan teks pada HTTP/1.1, dan server dapat mengirim data ke klien sebelum diminta melalui fitur server push.

Namun HTTP/2 lebih sulit di-debug karena format binernya tidak dapat dibaca langsung seperti teks biasa. Head-of-line blocking masih terjadi di level TCP dan hanya sepenuhnya teratasi di HTTP/3 dengan QUIC. Tidak semua proxy dan tool lama juga mendukung HTTP/2 dengan optimal.

## 9. Perbedaan Model Request-Response REST vs Bi-Directional Streaming gRPC untuk Real-Time

REST menggunakan model request-response yang asimetris. Untuk mendapatkan update real-time, klien harus melakukan polling berulang-ulang, yang menghabiskan bandwidth dan menambah latency yang tidak perlu.

WebSocket dapat memberikan komunikasi dua arah di atas protokol HTTP, namun tidak memiliki standar kontrak service yang jelas seperti gRPC. gRPC bi-directional streaming mempertahankan koneksi HTTP/2 tetap terbuka dan memungkinkan kedua pihak mengirim pesan kapan saja tanpa membuka koneksi baru setiap kali.

Untuk aplikasi obrolan, pesan dapat terkirim segera setelah dikirim tanpa polling, menghasilkan responsivitas yang lebih baik dan efisiensi resource yang jauh lebih tinggi dibandingkan REST tradisional.

## 10. Perbandingan Pendekatan Schema-Based Protocol Buffers vs Schema-Less JSON

Protocol Buffers bersifat type-safe, sehingga kesalahan kontrak antar service dapat terdeteksi saat kompilasi, bukan saat runtime. Ukuran payload Protocol Buffers dapat 3 hingga 10 kali lebih kecil dari JSON karena menggunakan format biner. Parsing juga lebih cepat, dan kode klien serta server di-generate otomatis dari file .proto, menghemat waktu pengembangan.

Namun Protocol Buffers tidak dapat dibaca manusia secara langsung seperti JSON yang dapat dibuka di browser. Perubahan schema memerlukan recompile dan deployment ulang di semua service terkait, sehingga Protocol Buffers kurang fleksibel untuk API yang struktur datanya sering berubah di tahap awal pengembangan.

JSON dan REST lebih cocok untuk API publik yang diakses oleh developer eksternal atau dari browser karena mudah dibaca, di-debug, dan tidak memerlukan tools khusus. Kesimpulannya, Protocol Buffers cocok untuk komunikasi internal antar microservice yang membutuhkan performa tinggi, sedangkan JSON lebih cocok untuk API publik yang mengutamakan aksesibilitas dan kemudahan penggunaan.