# python-crud-api

REST API Python sederhana dengan pipeline CI/CD otomatis menggunakan Jenkins, GitHub, Docker, dan ngrok. Dibuat untuk praktikum CI/CD.

## Alur CI/CD

```
git push -> GitHub -> webhook HTTPS -> ngrok -> Jenkins :8080 -> Docker -> API :5000 -> named volume
```

Setiap push ke branch `main` memicu Jenkins untuk checkout kode, memeriksa sintaks, membangun image Docker, menjalankan ulang container, lalu menguji `/health`.

## Struktur proyek

```
python-crud-api/
  app.py                  kode REST API
  Dockerfile              image aplikasi (Python 3.11)
  .dockerignore           file yang tidak ikut ke image
  .gitattributes          menjaga akhir baris LF
  Jenkinsfile             definisi pipeline
  README.md               dokumentasi ini
  jenkins-docker/
    Dockerfile            image Jenkins yang berisi Docker CLI
```

## Endpoint

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/health` | Cek status layanan |
| GET | `/messages` | Daftar semua pesan |
| GET | `/messages/{id}` | Satu pesan berdasarkan nomor urut (mulai dari 0) |
| POST | `/messages` | Tambah pesan, body JSON `{"message": "teks"}` |
| DELETE | `/messages?index=0` | Hapus pesan berdasarkan nomor urut |

Kode status: 200 (berhasil), 201 (dibuat), 400 (input tidak valid), 404 (tidak ditemukan).

## Menjalankan secara lokal

```
docker build -t python-crud-api:manual .
docker volume create production-api-data
docker run -d --name production-api-container -p 5000:5000 -v production-api-data:/var/lib/backend_data python-crud-api:manual
curl http://localhost:5000/health
```

Di Windows PowerShell, gunakan `curl.exe` dan bukan `curl`.

Contoh memakai PowerShell:

```
Invoke-RestMethod -Method Post -Uri http://localhost:5000/messages -ContentType "application/json" -Body '{"message":"halo"}'
Invoke-RestMethod http://localhost:5000/messages
Invoke-RestMethod http://localhost:5000/messages/0
Invoke-RestMethod -Method Delete -Uri "http://localhost:5000/messages?index=0"
```

## Pipeline Jenkins

`Jenkinsfile` memiliki lima stage:

1. **Checkout Code**: mengambil kode dari GitHub.
2. **Code Syntax Verification**: memeriksa sintaks `app.py` di container Python sementara.
3. **Build Docker Image**: membangun image dengan tag nomor build dan `latest`.
4. **Automated Deployment**: menghentikan container lama, lalu menjalankan container baru dengan volume yang sama.
5. **Health Check**: memanggil `/health` lewat jaringan `jenkins-net`.

## Menyiapkan Jenkins

1. Buat network dan volume: `docker network create jenkins-net` dan `docker volume create jenkins-data`.
2. Bangun image Jenkins: `docker build -t jenkins-docker ./jenkins-docker`.
3. Jalankan Jenkins di port 8080 dengan Docker socket terpasang.
4. Simpan GitHub Personal Access Token (Contents: Read-only) sebagai credential Jenkins dengan ID `github-pat-readonly`.
5. Buat job Pipeline dengan Pipeline script from SCM, branch `*/main`, dan Script Path `Jenkinsfile`.
6. Jalankan `ngrok http 8080`, lalu pasang webhook GitHub dengan Payload URL `https://<alamat-ngrok>/github-webhook/` untuk event push.

## Persistensi data

Pesan disimpan di `/var/lib/backend_data/messages.json`, yang dipasang ke Docker named volume `production-api-data`. Container boleh dihapus dan dibuat ulang oleh pipeline tanpa menghilangkan data.

## Fitur tambahan

- Endpoint `GET /messages/{id}`.
- Docker `HEALTHCHECK` yang memeriksa `/health` tiap 10 detik, sehingga `docker ps` menampilkan status `healthy`.

## Catatan keamanan

- PAT dan authtoken ngrok disimpan di luar repository (Jenkins Credentials dan konfigurasi lokal), dan tidak pernah di-commit.
- Memasang Docker socket ke container Jenkins memberi akses besar ke Docker host. Konfigurasi ini hanya untuk lab, bukan produksi.
- ngrok membuka Jenkins ke internet selama tunnel aktif.
