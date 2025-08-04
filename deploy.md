# Proses Melakukan Deploy Laravel dengan GITLAB CI CD ke VPS
## 1. Enable SSH Public Key

Login lewat open console  
```
nano /etc/ssh/sshd_config
```

Cari baris terakhir, ubah isian di bagian <code>PasswordAuthentication </code> dari yang sebelumnya berisi <b>no</b> ke <b>yes</b>  

```
PasswordAuthentication yes
```

Lakukan reload 
```
sudo service sshd reload
```

## 2. Install Docker

Lakukan beberapa perintah berikut ini:

```
sudo apt update
```

```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
```

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```

```
apt-cache policy docker-ce
```

```
sudo apt install docker-ce
```

```
sudo systemctl status docker
```

```
sudo usermod -aG docker ${USER}
```

```
su - ${USER}
```

Cek dengan perintah
```
groups
```
## 3. Create SSH key Di Server untuk Deploy
Jalankan perintah berikut ini untuk:
```
sudo adduser deployer
#silakan install acl jika belum punya
sudo apt install acl

sudo setfacl -R -m u:deployer:rwx /home/aplikasi

# set permission yang perlu di folder tujuan, saat sudah ada projectnya
chmod 777 -R /home/aplikasi/storage
chmod 777 -R /home/aplikasi/public
```
login ke aplikasi sebagai deployer:
```
sudo deployer
```
Jalankan perintah untuk membuat pasangan private key dan public key
```
ssh-keygen -t rsa
```
copy isi dari public key ke authorized key:
```
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
## 4. Install PHP MySQL Apache dengan Docker & Install Lets Encrypt
```
# masuk folder, buka terminal di /home/aplikasi folder
git clone https://github.com/dirumahrafif/docker-apache-php8-mysql.git .
docker-compose up -d
```
Proses install letsencrypt
```
# masuk ke container webserver
docker exec -it webserver bash
certbot --apache -d resumerafif.com -m dirumahrafif@gmail.com
certbot --apache -d resumerafif.com -d www.resumerafif.com -m dirumahrafif@gmail.com
```
## 5. Buka project yang ada di GITLAB
### Menambahkan Variabel
Buka bagian settings > CI/CD > Variables kemudian tambahkan variabel, misalnya diberi nama SSH_PRIVATE_KEY, dan isi didapatkan dari 
```
# pastikan sudah login sebagai user yang diberi akses ke folder aplikasi
cat ~/.ssh/id_rsa
```

File .htaccess [contoh file htaccess]
<IfModule mod_rewrite.c>
RewriteEngine On
# ditulis $$1, bukan $1 => karena saat deploy tanda $ terhapus
RewriteRule ^(.*)$ public/$$1 [L]
</IfModule>

![Tambahkan variabel](https://raw.githubusercontent.com/dirumahrafif/devlogs/main/DEVOPS/images/1.png)
## 6. Buat Runner
```
docker run -d --name gitlab-runner --restart always -v /srv/gitlab-runner/config:/etc/gitlab-runner -v /var/run/docker.sock:/var/run/docker.sock gitlab/gitlab-runner:latest
```
Daftarkan runner
```
docker exec -it gitlab-runner gitlab-runner register
```

```
sudo usermod -aG docker deployer
```
## 7. Buka project Laravel
### Tambahkan file .gitlab-ci.yml
File .gitlab-ci.yml, [file berikut ini]

stages:
  - deploy

Deploy:
  stage: deploy
  variables:
    VAR_DIREKTORI: "/home/rafifresume/APLIKASI/www"
    VAR_GIT_URL_TANPA_HTTP: "gitlab.com/dirumahrafif/namauser.git"
    VAR_CLONE_KEY: "xxx"
    VAR_USER: "xxx"
    VAR_IP: "xxx"
    VAR_FILE_ENV: $FILE_ENV
    VAR_FILE_HTACCESS: $FILE_HTACCESS

  before_script:
    - "which ssh-agent || ( apt-get install openssh-client )"
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan $VAR_IP >> ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
    - echo "$VAR_FILE_HTACCESS"

  script:
    - ssh $VAR_USER@$VAR_IP "git config --global safe.directory '*'"
    - ssh $VAR_USER@$VAR_IP "if [ ! -d $VAR_DIREKTORI/.git ]; then echo 'Project belum ditemukan di direktori $VAR_DIREKTORI' && cd $VAR_DIREKTORI && git clone https://oauth2:$VAR_CLONE_KEY@$VAR_GIT_URL_TANPA_HTTP .; fi"
    - ssh $VAR_USER@$VAR_IP "cd $VAR_DIREKTORI && git pull origin main && exit"
    - ssh $VAR_USER@$VAR_IP "if [ -d $VAR_DIREKTORI/.env ]; then rm .env; fi"
    - ssh $VAR_USER@$VAR_IP "cd $VAR_DIREKTORI && echo '$VAR_FILE_ENV' >> .env"
    - ssh $VAR_USER@$VAR_IP "if [ -d $VAR_DIREKTORI/.htaccess ]; then rm .htaccess; fi"
    - ssh $VAR_USER@$VAR_IP "cd $VAR_DIREKTORI && echo '$VAR_FILE_HTACCESS' >> .htaccess"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver composer install"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver composer update"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver php artisan migrate"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver php artisan db:seed"
    - ssh $VAR_USER@$VAR_IP "docker exec webserver php artisan key:generate"
    - echo "A!"
```
VAR_DIREKTORI: "/home/rafifresume/APLIKASI/www"
VAR_GIT_URL_TANPA_HTTP: "gitlab.com/dirumahrafif/deployer.git"
VAR_CLONE_KEY: "xxx" # diambil dari halaman profile (lihat di bawah)
VAR_USER: "deployer" #user yang sudah diberi akses
VAR_IP: "xxx" #ip server
VAR_FILE_ENV: $FILE_ENV #dari point 5 di atas
VAR_FILE_HTACCESS: $FILE_HTACCESS #dari point 5 di atas
```

### Cara mendapatkan Token User
Buka halaman profile
![gambar2](https://raw.githubusercontent.com/dirumahrafif/devlogs/main/DEVOPS/images/2.png)
Masuk ke menu access token:
- masukkan <code>token name</code>
- <code>expiration date</code> dikosongkan saja
- <code>select scopes</code> saya checklist semua
- Kemudian klik tombol **Create personal access token**

![gambar3](https://raw.githubusercontent.com/dirumahrafif/devlogs/main/DEVOPS/images/3.png)
