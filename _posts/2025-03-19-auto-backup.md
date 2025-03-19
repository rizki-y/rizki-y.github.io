---
title: "Backup PostgreSQL Otomatis ke Google Drive dengan Rclone"
categories: 
  - Database
tags:
  - PostgreSQL
  - Rclone
  - Backup
  - Google Drive
  - Cron
---

## 1. Instal rclone di Server dan PC Lokal
Pastikan versi rclone di server dan PC lokal sama agar tidak ada masalah kompatibilitas.

### Ubuntu (Server)
```sh
curl https://rclone.org/install.sh | sudo bash
rclone version
```

### Windows (PC Lokal)
1. Download rclone dari [rclone.org/downloads](https://rclone.org/downloads/).
2. Ekstrak dan jalankan:
```sh
rclone.exe version
```

## 2. Konfigurasi rclone dan Generate Token OAuth

### a. Jalankan konfigurasi di server:
```sh
rclone config
```
Ikuti langkah-langkah berikut:
- Pilih "n" untuk membuat remote baru.
- Masukkan nama remote (misalnya: `my_remote`).
- Pilih Google Drive sebagai penyimpanan.
- Pilih client_id dan client_secret default (langsung tekan Enter).
- Pilih scope dengan opsi `1` (Full access).
- Pilih konfigurasi untuk shared drive (jika ada).
- Saat diminta login OAuth, catat link yang diberikan (jika server tidak memiliki browser, pilih opsi manual).

### b. Jalankan rclone di PC Lokal untuk login OAuth:
```sh
rclone authorize "drive"
```
- Login ke akun Google Drive.
- Jika sukses, akan muncul pesan "Success! All done."
- Salin token OAuth ke server.

### c. Simpan konfigurasi di server
Setelah token dimasukkan di server, ketik `q` untuk keluar.

### d. Uji koneksi ke Google Drive dari server:
```sh
rclone lsd my_remote:
```
Pastikan folder Google Drive bisa ditampilkan.

## 3. Buat Folder Backup di Google Drive
```sh
rclone mkdir my_remote:"Database Backup"
```
Cek apakah folder sudah ada:
```sh
rclone lsf my_remote:"Database Backup"
```

## 4. Buat File .pgpass untuk Otentikasi PostgreSQL
Buat file `.pgpass` di home directory:
```sh
nano ~/.pgpass_mydb
```
Isi dengan format:
```sh
localhost:5432:mydb:myuser:mypassword
```
Simpan (`Ctrl + X`, `Y`, `Enter`).

Ubah permission agar hanya bisa dibaca oleh user:
```sh
chmod 600 ~/.pgpass_mydb
```

## 5. Buat Script Backup Otomatis
Buat file `backup_mydb.sh`:
```sh
nano ~/backup_mydb.sh
```
Isi dengan:
```bash
#!/bin/bash

set -e  # Keluar otomatis jika ada error

# Konfigurasi
DB_NAME="database_name"
DB_USER="database_user"
BACKUP_DIR="~/database_backup"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M")  # Format: YYYY-MM-DD_HH-MM
BACKUP_FILE="database_backup_${TIMESTAMP}.sql"
REMOTE_FOLDER="Database Backup" # Folder di Google Drive
PGPASSFILE="~/.pgpass"
LOG_FILE="~/backup.log"

# Redirect output ke log file
exec >> "$LOG_FILE" 2>&1

echo "===== Backup dimulai: $(date) ====="

# Pastikan direktori backup ada
mkdir -p "$BACKUP_DIR"

# Backup database menggunakan pg_dump
if PGPASSFILE="$PGPASSFILE" pg_dump -U "$DB_USER" -h localhost "$DB_NAME" > "$BACKUP_DIR/$BACKUP_FILE"; then
    echo "Database berhasil di-backup: $BACKUP_FILE"
else
    echo "Gagal melakukan backup database" >&2
    exit 1
fi

# Kompres file backup jika belum dikompresi
if [ ! -f "$BACKUP_DIR/$BACKUP_FILE.gz" ]; then
    gzip "$BACKUP_DIR/$BACKUP_FILE"
    echo "File backup dikompresi: $BACKUP_FILE.gz"
else
    echo "File backup sudah dikompresi sebelumnya"
fi

# Upload ke Google Drive menggunakan rclone
if rclone copy "$BACKUP_DIR/$BACKUP_FILE.gz" backup_remote:"$REMOTE_FOLDER"; then
    echo "Backup berhasil diunggah ke Google Drive: $BACKUP_FILE.gz"
    
    # Hapus backup lokal setelah diunggah
    rm -f "$BACKUP_DIR/$BACKUP_FILE.gz"
    rm -f "$BACKUP_DIR/$BACKUP_FILE"
    echo "File backup lokal dihapus: $BACKUP_FILE.gz"
    echo "File backup lokal dihapus: $BACKUP_FILE"
else
    echo "Gagal mengunggah backup ke Google Drive" >&2
    exit 1
fi

echo "===== Backup selesai: $(date) ====="
```
Simpan (`Ctrl + X`, `Y`, `Enter`).

Beri izin eksekusi:
```sh
chmod +x ~/backup_mydb.sh
```

## 6. Jalankan Backup Secara Manual (Tes)
```sh
~/backup_mydb.sh
```
Jika sukses, file backup akan muncul di Google Drive.

## 7. Jadwalkan Backup Otomatis dengan Cronjob
Edit cronjob:
```sh
crontab -e
```
Tambahkan baris berikut untuk backup setiap hari pukul 23:50:
```sh
50 23 * * * ~/backup_mydb.sh >> ~/backup.log 2>&1
```
Simpan (`Ctrl + X`, `Y`, `Enter`).

Cek apakah cron aktif:
```sh
crontab -l
```

## 8. Verifikasi File Backup di Google Drive
Pastikan file muncul dengan:
```sh
rclone lsf my_remote:"Database Backup"
```

Selesai! 🎉
Sekarang backup database PostgreSQL berjalan otomatis ke Google Drive setiap hari pukul 23:50.

Jika ada error, cek:
```sh
cat ~/backup.log
```

Selamat mencoba! 🚀

