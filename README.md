````md
# Laravel Deployment on Kubernetes (Minikube)

Dokumentasi ini menjelaskan langkah lengkap untuk melakukan deployment aplikasi Laravel ke Kubernetes menggunakan Minikube dan Docker Desktop pada Windows.

---

# Teknologi yang Digunakan

- PHP 8.3
- Laravel 13
- Docker
- Kubernetes
- Minikube
- MySQL
- Apache
- Kubectl

---

# Prasyarat

Pastikan software berikut sudah terinstal:

- PHP 8.3+
- Composer
- Docker Desktop
- Minikube
- Kubectl
- Git
- Laragon *(opsional untuk testing lokal)*

---

# 1. Membuat Project Laravel

```bash
cd C:\Users\aksal\Documents

mkdir kubernetes-laravel
cd kubernetes-laravel

composer create-project laravel/laravel laravel-minikube

cd laravel-minikube
````

Generate application key:

```bash
copy .env.example .env
php artisan key:generate
```

---

# 2. Konfigurasi Database Laravel

Edit file `.env`:

```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel_k8s
DB_USERNAME=root
DB_PASSWORD=root
```

## Penjelasan

`DB_HOST=mysql` digunakan agar container Laravel dapat terhubung ke service MySQL di dalam Kubernetes.

---

# 3. Pengujian Lokal (Opsional)

Sebelum menggunakan Docker dan Kubernetes, pastikan aplikasi Laravel dapat berjalan secara lokal.

## Menjalankan Laragon

* Buka Laragon
* Klik `Start All`

## Jalankan Migrasi

```bash
php artisan migrate
```

## Jalankan Laravel

```bash
php artisan serve
```

Akses:

```text
http://127.0.0.1:8000
```

---

# 4. Membuat Dockerfile

Buat file `Dockerfile` di root project.

## Struktur Project

```text
laravel-minikube/
├── app/
├── bootstrap/
├── public/
├── routes/
├── storage/
├── Dockerfile
└── .env
```

## Isi Dockerfile

```dockerfile
FROM php:8.3-apache

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    zip \
    curl

RUN docker-php-ext-install pdo pdo_mysql

RUN a2enmod rewrite

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

ENV APACHE_DOCUMENT_ROOT=/var/www/html/public

RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
    /etc/apache2/sites-available/000-default.conf

RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
    /etc/apache2/apache2.conf

WORKDIR /var/www/html

COPY . .

RUN composer install

RUN chown -R www-data:www-data /var/www/html

RUN chmod -R 775 storage bootstrap/cache

EXPOSE 80
```

---

# 5. Menjalankan Minikube

Pastikan Docker Desktop sudah aktif.

## Start Minikube

```bash
minikube start --driver=docker
```

## Verifikasi Node Kubernetes

```bash
kubectl get nodes
```

Jika berhasil:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...
```

---

# 6. Membuat Kubernetes Manifest

Buat folder:

```bash
mkdir k8s
cd k8s
```

Buat file berikut:

```text
laravel.yaml
mysql.yaml
service.yaml
```

---

# 7. Konfigurasi Laravel Deployment

## laravel.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel

spec:
  replicas: 3

  selector:
    matchLabels:
      app: laravel

  template:
    metadata:
      labels:
        app: laravel

    spec:
      containers:
        - name: laravel
          image: laravel-app
          imagePullPolicy: Never

          ports:
            - containerPort: 80

          env:
            - name: DB_HOST
              value: mysql

            - name: DB_PORT
              value: "3306"

            - name: DB_DATABASE
              value: laravel_k8s

            - name: DB_USERNAME
              value: root

            - name: DB_PASSWORD
              value: root
```

---

# 8. Konfigurasi MySQL Deployment

## mysql.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql

spec:
  replicas: 1

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
        - name: mysql
          image: mysql:8

          ports:
            - containerPort: 3306

          env:
            - name: MYSQL_ROOT_PASSWORD
              value: root

            - name: MYSQL_DATABASE
              value: laravel_k8s
```

---

# 9. Konfigurasi Service Kubernetes

## service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-nodeport

spec:
  type: NodePort

  selector:
    app: laravel

  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080

---
apiVersion: v1
kind: Service
metadata:
  name: mysql

spec:
  selector:
    app: mysql

  ports:
    - port: 3306
      targetPort: 3306
```

---

# 10. Build Docker Image

Kembali ke root project:

```bash
cd ..
```

Build image Laravel:

```bash
docker build -t laravel-app .
```

Load image ke Minikube:

```bash
minikube image load laravel-app
```

---

# 11. Deploy ke Kubernetes

```bash
kubectl apply -f k8s
```

Cek deployment:

```bash
kubectl get pods
kubectl get svc
```

---

# 12. Migrasi Database

Masuk ke pod Laravel:

```bash
kubectl get pods
```

```bash
kubectl exec -it <nama-pod> -- bash
```

Jalankan migrasi:

```bash
php artisan migrate
```

Jika menggunakan database session:

```bash
php artisan session:table
php artisan migrate
```

---

# 13. Menjalankan Aplikasi

```bash
minikube service laravel-nodeport
```

Kubernetes akan membuka browser otomatis.

---

# 14. Scaling Deployment

## Scale Up

```bash
kubectl scale deployment laravel --replicas=5
```

## Scale Down

```bash
kubectl scale deployment laravel --replicas=2
```

## Monitoring Pods

```bash
kubectl get pods -w
```

---

# 15. Arsitektur Sistem

```text
User
  ↓
Kubernetes Service (NodePort)
  ↓
Laravel Pods (Replicas)
  ↓
MySQL Pod
```

## Kubernetes Menangani

* Load balancing
* Service discovery
* Horizontal scaling
* Self healing container

---

# 16. Update Docker Image

Jika terdapat perubahan source code:

```bash
docker build -t laravel-app .

minikube image load laravel-app

kubectl rollout restart deployment laravel
```

---

# 17. Troubleshooting

## Hapus Deployment

```bash
kubectl delete deployment laravel
```

## Cek Pods

```bash
kubectl get pods
```

## Melihat Log Pod

```bash
kubectl logs <nama-pod>
```

## Masuk ke Container

```bash
kubectl exec -it <nama-pod> -- bash
```

## Restart Minikube

```bash
minikube delete --all --purge
docker rm -f minikube
```

---

# 18. Reset Total Environment

```bash
kubectl delete all --all

minikube stop

minikube delete --all --purge

docker system prune -a --volumes

docker builder prune -a
```

Hapus cache Minikube:

```bash
rmdir /s /q C:\Users\aksal\.minikube\cache
```

---

# 19. Membersihkan Temporary File Windows

Buka:

```text
Win + R → %temp%
```

Hapus seluruh temporary file.

---

# Hasil Akhir

Dengan konfigurasi ini:

* Laravel berjalan di dalam Docker container
* Laravel dideploy ke Kubernetes menggunakan Minikube
* Kubernetes melakukan load balancing otomatis
* Deployment mendukung horizontal scaling
* Environment lokal menyerupai production environment
* Service antar container berjalan menggunakan Kubernetes networking
