# Dokumentasi Troubleshooting Error Backend & MySQL Gadget Gallery

## 🎯 Tujuan
Dokumentasi ini mencatat berbagai error yang ditemui selama proses setup dan deployment backend Gadget Gallery yang berjalan di Docker, terutama yang berkaitan dengan interaksi Node.js (Sequelize) dengan database MySQL, serta solusi yang diterapkan.

---

### Error: `Dialect needs to be explicitly supplied as of v4.0.0`

*   **Kapan Terjadi:** Saat aplikasi Node.js backend pertama kali mencoba menginisialisasi koneksi Sequelize.
*   **Penyebab:** Sejak Sequelize v4, parameter `dialect` (misalnya `'mysql'`) wajib disertakan dalam konfigurasi saat membuat instance `new Sequelize()`. Ini tidak secara otomatis terdeteksi.
*   **Solusi yang Diterapkan:** Menambahkan opsi `dialect: 'mysql'` ke dalam objek konfigurasi yang dilewatkan ke konstruktor `Sequelize`. Contoh:
    ```javascript
    // Di file src/models/index.js atau file konfigurasi DB
    const sequelize = new Sequelize(dbName, dbUser, dbPassword, {
      host: dbHost,
      dialect: 'mysql', // <--- Ditambahkan
      // opsi lain...
    });
    ```

---

### Error: `.env` Tidak Terbaca di Docker & Variabel Environment Hilang

*   **Kapan Terjadi:** Aplikasi gagal mendapatkan konfigurasi penting (seperti kredensial DB, host Redis, dll.) saat berjalan di dalam container Docker, meskipun file `.env` ada di direktori proyek lokal.
*   **Penyebab:** Container Docker berjalan dalam lingkungan terisolasi dan tidak secara otomatis membaca file `.env` dari host. Variabel environment perlu didefinisikan secara eksplisit untuk container.
*   **Solusi yang Diterapkan:** Menggunakan direktif `env_file` di `docker-compose.yaml` untuk service `backend`. Ini menginstruksikan Docker Compose untuk membaca file `.env` dari path yang ditentukan dan memuat variabelnya ke dalam environment container.
    ```yaml
    # Di docker-compose.yaml, service backend:
    services:
      backend:
        # ...
        env_file:
          - ./gadget-gallery-BE/.env # <-- Path ke file .env backend
        # ...
    ```

---

### Error: Koneksi Redis Gagal (`ECONNREFUSED` / Log: `Nyalain Redis oi`)

*   **Kapan Terjadi:** Aplikasi backend gagal terhubung ke service Redis saat berjalan di Docker.
*   **Penyebab:** Kode aplikasi mencoba terhubung ke Redis menggunakan `localhost:6379`. Di dalam container `backend`, `localhost` mengacu pada container itu sendiri, bukan container `redis`. Nama service (`redis` sesuai definisi di `docker-compose.yaml`) harus digunakan sebagai hostname di dalam jaringan Docker.
*   **Solusi yang Diterapkan:** Menggunakan variabel environment `REDIS_URL` yang benar di file `.env` dan dibaca oleh aplikasi.
    ```dotenv
    # Di file .env backend
    REDIS_URL=redis://redis:6379 # 'redis' adalah nama service di docker-compose
    ```
    Kode Node.js kemudian dikonfigurasi untuk membaca `process.env.REDIS_URL`:
    ```javascript
    const client = createClient({
      url: process.env.REDIS_URL || 'redis://localhost:6379', // Menggunakan REDIS_URL dari env
      legacyMode: true,
    });
    ```

---

### Error: `Incorrect string value: '\xCB\x9A...'` saat Seeding (Masalah Karakter/UTF-8)

*   **Kapan Terjadi:** Saat menjalankan `sequelize-cli db:seed` untuk data yang mengandung karakter non-ASCII atau simbol kompleks (misalnya, dari deskripsi produk HTML).
*   **Penyebab:** Kolom di tabel MySQL (misalnya `description` di tabel `Products`) menggunakan character set dan collation default (seringkali `latin1` atau `utf8` di MySQL 5.7) yang tidak sepenuhnya mendukung UTF-8 (khususnya karakter 4-byte). Data yang di-seed menggunakan encoding `utf8mb4` (UTF-8 penuh).
*   **Solusi yang Diterapkan:**
    1.  **Mengubah Default Server MySQL:** Menambahkan `command` di `docker-compose.yaml` untuk service `mysql` agar menggunakan `utf8mb4` secara default untuk server dan koneksi baru.
        ```yaml
        services:
          mysql:
            image: mysql:5.7
            command:
              - --character-set-server=utf8mb4
              - --collation-server=utf8mb4_unicode_ci
            # ...
        ```
    2.  **Memperbaiki Kolom yang Sudah Ada:** Membuat migrasi Sequelize baru (`npx sequelize-cli migration:generate ...`) untuk mengubah character set dan collation kolom yang bermasalah secara eksplisit menggunakan `ALTER TABLE ... MODIFY COLUMN ...`.
        ```javascript
        // Di dalam file migrasi baru (up function):
        await queryInterface.sequelize.query(
          'ALTER TABLE `Products` MODIFY `description` TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
        );
        ```
    3.  Menjalankan migrasi baru tersebut (`npx sequelize-cli db:migrate`).
    4.  Menjalankan kembali proses seeding.

---

### Error: `write EPIPE` saat Seeding Gambar (Masalah `max_allowed_packet`)

*   **Kapan Terjadi:** Saat menjalankan `sequelize-cli db:seed` untuk data dalam jumlah besar, terutama yang melibatkan data BLOB/Buffer seperti gambar.
*   **Penyebab:** Ukuran total paket data SQL yang dikirim oleh `bulkInsert` melebihi batas `max_allowed_packet` yang dikonfigurasi di server MySQL. Server menutup koneksi, menyebabkan error `EPIPE` (Broken Pipe) di sisi klien (Node.js).
*   **Solusi yang Diterapkan:** Meningkatkan nilai `max_allowed_packet` di server MySQL melalui `command` di `docker-compose.yaml`.
    ```yaml
    services:
      mysql:
        image: mysql:5.7
        command:
          - --character-set-server=utf8mb4
          - --collation-server=utf8mb4_unicode_ci
          - --max_allowed_packet=128M # <-- Tingkatkan nilai (misal 128M)
        # ...
    ```
    Setelah mengubah `docker-compose.yaml`, container `mysql` perlu di-restart (`docker-compose stop mysql && docker-compose up -d --no-deps mysql` atau `docker-compose down && docker-compose up -d`).

---

### Error: `Expression #... is not in GROUP BY clause... incompatible with sql_mode=only_full_group_by`

*   **Kapan Terjadi:** Saat menjalankan query SQL (melalui Sequelize atau langsung) yang menggunakan `GROUP BY` pada aplikasi yang berjalan.
*   **Penyebab:** MySQL (terutama 5.7.5+ dan 8.x) secara default mengaktifkan mode `ONLY_FULL_GROUP_BY`. Mode ini mengharuskan setiap kolom dalam `SELECT list` yang tidak diagregasi (bukan `COUNT`, `SUM`, `MIN`, dll.) harus ada dalam klausa `GROUP BY`. Query di backend melanggar aturan ini.
*   **Solusi yang Diterapkan (Perbaikan Cepat):** Mengubah `sql_mode` server MySQL untuk menghapus `ONLY_FULL_GROUP_BY` melalui `command` di `docker-compose.yaml`.
    ```yaml
    services:
      mysql:
        image: mysql:5.7
        command:
          - --character-set-server=utf8mb4
          - --collation-server=utf8mb4_unicode_ci
          - --max_allowed_packet=128M
          # Hapus ONLY_FULL_GROUP_BY dari daftar mode
          - --sql-mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
        # ...
    ```
    Container `mysql` perlu di-restart setelah perubahan ini. *(Solusi alternatif yang lebih 'benar' tetapi lebih sulit adalah mengidentifikasi dan memperbaiki query SQL yang bermasalah di kode backend agar sesuai dengan aturan `ONLY_FULL_GROUP_BY`)*.

---

### Error: `ENOENT: no such file or directory, open '/usr/src/app/package.json'` (Setelah Beralih ke Bind Mount)

*   **Kapan Terjadi:** Tepat setelah mengubah konfigurasi volume backend di `docker-compose.yaml` dari named volume ke bind mount (`./gadget-gallery-BE:/usr/src/app`).
*   **Penyebab:** Bind mount menimpa seluruh isi direktori `/usr/src/app` (yang dibuat saat build image) dengan isi direktori lokal. Jika ada ketidaksesuaian path antara `build.context` dan path `volumes` di `docker-compose.yaml` (misalnya, `gadget-gallery_BE` vs `gadget-gallery-BE`), direktori yang salah/kosong akan dimount, sehingga `package.json` tidak ditemukan saat container mencoba menjalankan `npm run dev`.
*   **Solusi yang Diterapkan:** Memastikan path direktori backend **konsisten** antara `build.context:` dan `volumes:` di `docker-compose.yaml`, sesuai dengan nama direktori yang sebenarnya ada di host machine.
    ```yaml
    # Contoh jika nama direktori lokal adalah gadget-gallery_BE
    services:
      backend:
        build:
          context: ./gadget-gallery_BE # <-- Pastikan konsisten
        volumes:
          - ./gadget-gallery_BE:/usr/src/app # <-- Pastikan konsisten
          - /usr/src/app/node_modules # <-- Tetap diperlukan
        # ...
    ```

---

Dokumentasi ini membantu melacak masalah umum dan solusi yang telah berhasil diterapkan, mempercepat proses debugging di masa mendatang.
