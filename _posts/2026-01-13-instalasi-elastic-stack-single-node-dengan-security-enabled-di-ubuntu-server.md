---
title: Instalasi Elastic Stack (Single Node) dengan Security Enabled di Ubuntu Server
date: 2026-01-13 08:00:00 +0700
categories: [Blue Team, Linux, SOC]
tags: [SOC]
author: luminis
published: true
media_subpath: /assets/img/posts/1-install-elk-stack
---

Elasticsearch Stack (Elastic Stack) merupakan kumpulan tools powerfull yang banyak digunakan untuk kebutuhan log monitoring, observability, analisis data, dan keamanan sistem (SIEM). Elastic Stack terdiri dari beberapa komponen utama seperti `Elasticsearch`, `Kibana`, `Logstash`, `Beats`, serta integrasi `Elasticsearch Hadoop`.

Pada artikel ini, saya akan melakukan instalasi lengkap Elastic Stack dalam satu server (single-node) dengan TLS dan security enabled, cocok untuk Home Lab, Lab SOC, Pembelajaran ELK Stack, dan simulasi monitoring & security

Prasyarat sistem
- OS: Ubuntu Server (22.04/24.04 direkomendasikan)
- IP Server: `192.168.17.151`
- RAM minimal: 4GB (disarankan 8GB)

## Task 1 - Persiapan Sistem & Java
- Update repository
```
sudo apt-get update && sudo apt-get upgrade -y
```
- Install Java Runtime Environment
```
sudo apt-get install default-jre -y
```
- Verifikasi Java
```
java -version
```
![Java Version](01-java%20-version.png)
## Task 2 - Menambahkan Repository Elasticsearch
### Step 1: Install dependency & import GPG Key
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```
![Dependency](02%20-%20dependency.png)
### Step 2: Install `apt-transport-https`
```
sudo apt-get install apt-transport-https
```

### Step 3: Tambahkan repository Elastic

```
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-9.x.list
```
![Repository](03%20-%20repository.png)

## Task 3 - Instalasi Elasticsearch
### Step 1: Install Elasticsearch
```
sudo apt-get update && sudo apt-get install elasticsearch
```

### Step 2: Konfigurasi Elasticsearch
Edit file 
```
sudo nano /etc/elasticsearch/elasticsearch.yml
```

contoh Konfigurasi :
![Konfigurasi Elasticsearch](04-konfigurasi-elasticsearch.png)
```
# ---------------------------------- Cluster -----------------------------------  
cluster.name: Luminis-home-lab  
# ----------------------------------- Paths ------------------------------------  node.name: luminis-home

# ----------------------------------- Paths ------------------------------------  
path.data: /var/lib/elasticsearch  
path.logs: /var/log/elasticsearch  
# ---------------------------------- Network -----------------------------------  
network.host: 192.168.17.151
http.port: 9200  
# --------------------------------- Discovery ----------------------------------  
discovery.type: single-node  
#cluster.initial_master_nodes: ["node-1", "node-2"]

#----------------------- BEGIN SECURITY AUTO CONFIGURATION -----------------------  
  
xpack.security.enabled: true  
  
xpack.security.enrollment.enabled: true  
  
xpack.security.http.ssl:  
  enabled: true  
  keystore.path: certs/http.p12  
  
xpack.security.transport.ssl:  
  enabled: true  
  verification_mode: certificate  
  keystore.path: certs/transport.p12  
  truststore.path: certs/transport.p12  
  
http.host: 0.0.0.0  
  
transport.host: 0.0.0.0
```

Secara default, Elasticsearch sudah melakukan auto JVM heap sizing berdasarkan  role node dan total RAM. untuk sebagian besar kasus, pengaturan default sudah optimal. Namun, pada kondisi tertentu seperti Home Lab atau server dengan resource terbatas, kita dapat melakukan penyesuaian JVM heap secara manual.

Untuk mengkonfigurasi opsi JVM di file `jvm.options`, buka file konfigurasi di teks editor kalian 
```
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
```

isi dengan :

```
-Xms2g
-Xmx2g
```
contoh diatas cocok untuk server RAM 4GB.
lalu restart Elasticsearch:
```
sudo systemctl restart elasticsearch
```

verifikasi Heap size dengan menggunakan API Elasticsearch:
```
curl -u elastic --cacert /etc/elasticsearch/certs/http_ca.crt \
https://localhost:9200/_nodes/_all/jvm?pretty
```

### Step 3: Generate Self-Signed Certificate

```
/usr/share/elasticsearch/bin/elasticsearch-certutil http
```

 kemudia akan menampilkan beberapa ikuti seperti gambar di bawah ini: 
 
 option untuk Generate certificate signing request (CSR)? pilih type [y] dan press [enter]
![option 1](05-option1.png)
 selanjutnya pilih  [N] dan tekan [enter]
![option 2](../images/posts/02-option-2.png)
 selanjutnya masukkan hostname server Elasticsearch, lalu pilih  [Y] dan tekan [enter]
![option 3](06-option2.png)
 selanjutnya masukkan IP server Elasticsearch, lalu pilih  [Y] dan tekan [enter]
![option 4](07-option3.png)
 selanjutnya pilih  [N] dan tekan [enter]
![option 5](08-option-4.png)
lalu masukkan password dan tekan [enter]
![option 6](09-option5.png)

### Step 4: Ekstrak Sertifikat ke Direktori Elasticsearch

Setelah proses pembuatan sertifikat selesai, file sertifikat akan  otomatis tersimpan dalam bentuk file ZIP di direktori `/usr/share/elasticsearch/`. Masuk ke direktori tersebut dan pastikan file ZIP sertifikat sudah tersedia.

Install package `zip` dan `unzip` jika belum tersedia, kemudian ekstrak file sertifikat ke direktori `/etc/elasticsearch/certs`.

```
sudo apt-get install zip unzip -y // jika belum tersedia

sudo unzip filename.zip -d /etc/elasticsearch/certs
```

>Pastikan nama file ZIP di sesuaikan dengan nama file yang dihasilkan oleh sebelumnya

kemudian Verifikasi Sertifikat
```
ls -l /etc/elasticsearch/certs
```

![list certificate](10-list-cert.png)


### Step 6: Menjalankan Elasticsearch

```
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
sudo systemctl status elasticsearch
```
### Step 7: Reset Passowrd User Elastic

Pada tahap ini adalah mengatur ulang password untuk user bawaan Elasticsearch(`elastic`).
```
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic --interactive
```
![Reset passowrd Elastic](11-reset-password.png)



### Step 8: Verifikasi Elasticsearch
```
curl --cacert /etc/elasticsearch/certs/http_ca.crt https://192.168.17.151:9200 -u elastic
```
jika berhasil, Elasticsearch siap digunakan.
![Verifikasi Elasticsearch](12-verifikasi-elasticsearch.png)

## Task 4: Instalasi Kibana

### Step 1: Install Kibana
```
sudo apt install kibana -y
```

### Step 2: Konfigurasi Sertifikat untuk Kibana
```
mkdir -p /etc/kibana/certs
cp /etc/elasticsearch/certs/http_ca.crt /etc/kibana/certs/
```

### Step 3: Reset Password kibana_system
```
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -i 
```
User ini di gunakan Kibana untuk berkomunikasi dengan Elasticsearch. opsi `-i` memungkinkan kita untuk mengatur password khusus. jika anda tidak ingin menggunakan password khusus dan hanya ingin mengatur ulang password, gunakan `-a` sebagai pengganti  `-i`
![reset password kibana](14-reset-passoword-kibana.png)

### Step 4: Konfigurasi Kibana

edit file `/etc/kibana/kibana.yml`:

Pastikan konfigurasi berikut aktif:
```
server.port: 5601
server.host: "192.168.17.151"

elasticsearch.hosts: ["https://192.168.17.151:9200"]

elasticsearch.username: "kibana_system"
elasticsearch.password: "admin123"

elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/http_ca.crt"]
```

### Step 5: Menjalankan Kibana 
```
sudo systemctl start kibana
sudo systemctl enable kibana
sudo systemctl status kibana
```

jika proses instalasi berhasil buka browser dan akses:
```
http://192.168.17.151:5601
```

login menggunakan:
- Username: elastic
- Password: Pasword yang sudah dibuat
when you navigate to `[security]->[Dashboard]` you will get an error something like this

![dashboard elastic](15-dashboard-elastic.png)
to solve this error generate `kibana-encription-keys`
```
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
```


ini akan menhasilkan output seperti  ini:
```
xpack.encryptedSavedObjects.encryptionKey: <key>
xpack.reporting.encryptionKey: <key>
xpack.security.encryptionKey: <key>
```

copy dan paste di  file `/etc/kibana/kibana.yml` dan restart kibana lagi
```
sudo systemctl restart kibana
```
maka error pun akan hilang
![error elastic](16-dashboard-elastic-fix.png)

## Task 5: Instalasi Logstash
Logstash adalah pipeline pemrosesan data sisi server  opensource yang mengumpulkan data dari berbagai sumber secara bersamaan. Kibana memproses informasi ini, dan Elasticsearch menyimpanya. untuk menginstall Logstash terdapat beberapa step seperti berikut:
### Step 1: install logstash
```
sudo apt install logstash 
```

### Step 2: Verifikasi Instalasi
untuk cek apakah logstash berhasil di install, gunakan perintah 
```
sudo /usr/share/logstash/bin/logstash --version
```
![version logstash](17-version-logstash.png)
### Step 3: Menjalankan service Logstash
setelah berhasil install, mulai logstash service dengan menggunakan `systemctl`
```
sudo systemctl start logstash
sudo systemctl enable logstash
```

cek jika service dapat berjalan atau tidak:
```
sudo systemctl status logstash
```
![status logstash](18-status-logstash.png)
### Step 4: Konfigurasi input Logstash (Beats)
setelah Logstash berhasil diaktifkan, sekarang kita dapat mengkonfigurasi untuk menyiapkan `INPUT`, `FILTERS`, dan `OUTPUT`pipelines berdasarkan kebutuhan. file konfigurasi Logstash custom berada di `/etc/logstash/conf.d/`. berikut konfigurasi nya:

membuat file konfigurasi dengan nama 02-beats-input-conf:
```
sudo nano /etc/logstash/conf.d/02-beats-input.conf
```

```
input {
  beats {
    port => 5044
  }
}
```

### Step 5: Menyiapkan Sertifikat CA Elasticsearch
salin sertifikat CA agar dapat diakses oleh Logstash:
```
sudo mkdir -p /etc/logstash/certs
sudo cp /etc/elasticsearch/certs/http_ca.crt /etc/logstash/certs/
sudo chown root:logstash /etc/logstash/certs/http_ca.crt
sudo chmod 640 /etc/logstash/certs/http_ca.crt
```
Buat file konfigurasi lagi dengan nama 30-elasticsearch-output.conf:
```
output {
  if [@metadata][pipeline] {
    elasticsearch {
      hosts => ["https://192.168.17.151:9200"]
      user => "elastic"
      password => "password_elastic"
      ssl_enabled => true
      ssl_certificate_authorities => ["/etc/logstash/certs/http_ca.crt"]
      index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
      pipeline => "%{[@metadata][pipeline]}"
    }
  } else {
    elasticsearch {
      hosts => ["https://192.168.17.151:9200"]
      user => "elastic"
      password => "password_elastic"
      ssl_enabled => true
      ssl_certificate_authorities => ["/etc/logstash/certs/http_ca.crt"]
      index => "logstash-%{+YYYY.MM.dd}"
    }
  }
}


```
### Step 6: Validasi & Restart Logstash
```
sudo -u logstash /usr/share/logstash/bin/logstash \
  --path.settings /etc/logstash \
  -t

```
kemudian restart service 
```
sudo systemctl restart logstash
```

## Task 6: Instalasi Filebeat
Filebeat digunakan untuk mengumpulkan dan mengirim file log ke Logstash atau Elasticsearch. ini adalah modul Beats yang paling populer. Fitur utama Filebeat adalah kemampuanya untuk mengurangi kecepatan pemrosesan jika Logstash menerima sejumlah besar data. Untuk menginstall Filebeat terdapat beberapa step seperti berikut:
### Step 1: install Filebeat
```
sudo apt-get install filebeat
```
>Catatan: Pastikan service Kibana aktif dan berjalan saat anda menginstall dan mengkonfigurasinya

### Step 2: Konfigurasi Filebeat
secara default, Filebeat mengirim data ke Elasticsearch. Namun, kita dapat mengaturnya agar mengirim data peristiwa ke Logstash sebagai gantinya. Untuk melakukan ini, ubah file konfigurasi filebeat menggunakan teks editor di direktori `/etc/filebeat/filebeat.yml` seperti berikut

```   
filebeat.inputs:
- type: filestream
  id: system-logs-luminis
  enabled: true
  paths:
  - /var/log/*.log
- type: journald
  id: my-journald-id
  enabled: true
  seek: head
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: true
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
  preset: balanced
output.logstash:
  hosts: ["192.168.17.151:5044"]
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0640
```


Selanjutnya, aktifkan modul sistem Filebeat untuk menganalisis log sitem lokal:
```
sudo filebeat modules enable system
```

cek list enabled & disabled modules:
```
sudo filebeat modules list
```
Pastikan fileset aktif:

```
sudo nano /etc/filebeat/modules.d/system.yml
```

```
- module: system
  # Syslog
  syslog:
    enabled: true // jika false ubah ke true

  # Authorization logs
  auth:
    enabled: true // jika false ubah ke true
```
![Konfigurasi filebeat](19-konfigurasi-filebeat.png)

Set up Filebeat
```
sudo filebeat setup \
  -E output.logstash.enabled=false \
  -E output.elasticsearch.hosts=["https://192.168.17.151:9200"] \
  -E output.elasticsearch.ssl.certificate_authorities=["/etc/elasticsearch/certs/http_ca.crt"] \
  -E output.elasticsearch.username=elastic \
  -E output.elasticsearch.password='admin123' \
  -E setup.kibana.host=http://192.168.17.151:5601


```


start FIlebeat:
```
sudo systemctl start filebeat
sudo systemctl enable filebeat
```

Untuk memverifikasi bahwa Elasticsearch memang menerima data ini, lakukan query index filebeat dengan perintah ini:
```
curl --cacert /etc/elasticsearch/certs/http_ca.crt \
		-u elastic \
		https://localhost:9200/filebeat-*/_search?pretty

```

output:
![verifikasi filebeat](20-verifikasi-filebeat.png)

## Penutup
Dengan mengikuti panduan ini, Elastic Stack single-node dengan TLS dan security enabled telah berhasil di implementasikan. Lingkungan ini sangat cocok untuk:
- Home Lab
- SOC / Blue Team Lab
- Pembelajaran ELK Stack
- Simulasi SIEM dan monitoring keamanan

Semoga bermanfaat  dan happy hacking :)

