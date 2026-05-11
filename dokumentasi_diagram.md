# Dokumentasi Perancangan Sistem Bank Sampah

Dokumen ini berisi berbagai diagram perancangan untuk sistem Bank Sampah berdasarkan struktur model dan controller yang ada di dalam proyek. Diagram menggunakan sintaks Mermaid.

## 1. Entity Relationship Diagram (ERD) & Logical Record Structure (LRS)
ERD dan LRS menggambarkan struktur tabel, *primary key* (PK), *foreign key* (FK), dan relasi antar tabel pada database `bank-sampah`.

```mermaid
erDiagram
    users {
        bigint id PK
        string name
        string email
        string password
        enum role "admin, nasabah"
    }
    nasabah {
        bigint id PK
        bigint user_id FK
        string no_rekening
        string alamat
        string no_hp
        decimal saldo
    }
    kategori_sampah {
        bigint id PK
        string nama_kategori
        string jenis
        decimal harga_per_kg
        string keterangan
    }
    transaksi {
        bigint id PK
        bigint nasabah_id FK
        bigint admin_id FK
        date tanggal
        decimal total_nilai
        string catatan
    }
    detail_transaksi {
        bigint id PK
        bigint transaksi_id FK
        bigint kategori_id FK
        decimal berat_kg
        decimal nilai
    }
    penilaian {
        bigint id PK
        bigint nasabah_id FK
        int bulan
        int tahun
        decimal total_berat
        int jumlah_setor
        decimal total_nilai
        decimal skor
        string predikat
    }

    users ||--o| nasabah : "memiliki profil (jika role=nasabah)"
    users ||--o{ transaksi : "mencatat (sebagai admin)"
    nasabah ||--o{ transaksi : "melakukan"
    nasabah ||--o{ penilaian : "mendapatkan hasil"
    transaksi ||--o{ detail_transaksi : "terdiri dari"
    kategori_sampah ||--o{ detail_transaksi : "termasuk dalam"
```

## 2. Use Case Diagram
Diagram Use Case ini menunjukkan apa saja fitur yang dapat diakses oleh Aktor `Admin` dan `Nasabah` di dalam sistem.

```mermaid
flowchart LR
    Admin([Admin])
    Nasabah([Nasabah])

    subgraph Sistem Bank Sampah
        UC1((Login / Logout))
        UC2((Kelola Kategori Sampah))
        UC3((Kelola Data Nasabah))
        UC4((Kelola & Input Transaksi Setor Sampah))
        UC5((Lakukan Penilaian Nasabah))
        UC6((Lihat Laporan))
        UC7((Lihat Dashboard & Saldo))
        UC8((Lihat Riwayat Transaksi))
    end

    Admin --> UC1
    Admin --> UC2
    Admin --> UC3
    Admin --> UC4
    Admin --> UC5
    Admin --> UC6

    Nasabah --> UC1
    Nasabah --> UC7
    Nasabah --> UC8
```

## 3. Class Diagram
Class diagram berikut menggambarkan struktur representasi *Model* dalam aplikasi Laravel, beserta metode-metode utama (termasuk metode untuk relasi *Eloquent*).

```mermaid
classDiagram
    class User {
        -int id
        -string name
        -string email
        -string password
        -string role
        +isAdmin() bool
        +nasabah()
    }
    class Nasabah {
        -int id
        -int user_id
        -string no_rekening
        -string alamat
        -string no_hp
        -decimal saldo
        +generateNoRekening() string
        +user()
        +transaksi()
        +penilaian()
    }
    class KategoriSampah {
        -int id
        -string nama_kategori
        -string jenis
        -decimal harga_per_kg
        -string keterangan
        +detailTransaksi()
    }
    class Transaksi {
        -int id
        -int nasabah_id
        -int admin_id
        -date tanggal
        -decimal total_nilai
        -string catatan
        +nasabah()
        +admin()
        +detail()
    }
    class DetailTransaksi {
        -int id
        -int transaksi_id
        -int kategori_id
        -decimal berat_kg
        -decimal nilai
        +transaksi()
        +kategori()
    }
    class Penilaian {
        -int id
        -int nasabah_id
        -int bulan
        -int tahun
        -decimal total_berat
        -int jumlah_setor
        -decimal total_nilai
        -decimal skor
        -string predikat
    }

    User "1" -- "1" Nasabah : hasOne
    User "1" -- "*" Transaksi : hasMany (Admin)
    Nasabah "1" -- "*" Transaksi : hasMany
    Nasabah "1" -- "*" Penilaian : hasMany
    Transaksi "1" -- "*" DetailTransaksi : hasMany
    KategoriSampah "1" -- "*" DetailTransaksi : hasMany
```

## 4. Activity Diagram (Proses Setor Sampah)
Activity diagram ini menggambarkan alur kerja langkah demi langkah ketika Admin sedang memproses atau menginput Setor Sampah dari Nasabah.

```mermaid
stateDiagram-v2
    [*] --> LoginAdmin
    LoginAdmin --> BukaDashboardAdmin
    BukaDashboardAdmin --> MasukMenuTransaksi
    MasukMenuTransaksi --> PilihNasabah
    PilihNasabah --> InputDetailSampah : Pilih Kategori & Masukkan Berat
    InputDetailSampah --> SistemHitungTotalNilai : (Berat * Harga Per Kg)
    SistemHitungTotalNilai --> SimpanTransaksi
    SimpanTransaksi --> UpdateSaldoNasabah : Saldo + Total Nilai
    UpdateSaldoNasabah --> Selesai & TampilkanPesanSukses
    Selesai & TampilkanPesanSukses --> [*]
```

## 5. Sequence Diagram (Proses Simpan Transaksi Setor Sampah)
Sequence diagram ini menggambarkan interaksi antar komponen (View, Controller, Model, Database) dalam urutan waktu tertentu saat proses Setor Sampah (Input Transaksi) dijalankan.

```mermaid
sequenceDiagram
    actor Admin
    participant View as Halaman Transaksi
    participant Ctrl as TransaksiController
    participant ModelTrans as Transaksi (Model)
    participant ModelDetail as DetailTransaksi (Model)
    participant ModelNas as Nasabah (Model)
    participant DB as Database

    Admin->>View: Pilih Nasabah & Input Data Sampah (Kategori & Berat)
    View->>Ctrl: Submit Form (nasabah_id, rincian_sampah)
    
    Ctrl->>ModelTrans: create([nasabah_id, admin_id, tanggal, total_nilai])
    ModelTrans->>DB: INSERT INTO transaksi
    DB-->>ModelTrans: Return objek Transaksi (termasuk ID)
    
    loop Untuk Setiap Item Sampah
        Ctrl->>ModelDetail: create([transaksi_id, kategori_id, berat, nilai])
        ModelDetail->>DB: INSERT INTO detail_transaksi
    end
    
    Ctrl->>ModelNas: Cari Nasabah(nasabah_id)
    DB-->>ModelNas: Return Data Nasabah
    Ctrl->>ModelNas: update(saldo = saldo + total_nilai)
    ModelNas->>DB: UPDATE saldo di tabel nasabah
    
    Ctrl-->>View: Redirect() dengan pesan Sukses
    View-->>Admin: Menampilkan alert transaksi berhasil
```
