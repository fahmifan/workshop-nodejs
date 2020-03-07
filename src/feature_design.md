# Features & Design

## Features
1. Sebagai pengunjung saya ingin dapat mendaftarkan diri ke platform, menggunakan username dan password
2. Sebagai pemilik platform berita hanya dapat ditulis/diubah oleh pengguna yg terotorisasi
3. Sebagai pengunjung saya ingin melihat daftar berita yang berisi judul, penulis, tanggal terbit dan isi berita


## Design
```
User {
    id: Int
    username: String
    password: String
    name: String
}

Content {
    id: Int
    author_id: Int
    title: String
    body: String
    slug: String
}
```
