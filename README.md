## Semua CLIENT harus menggunakan konfigurasi dari DHCP Server (1).

### Siapkan DHCP Server di Himmel

### Network Config Client
```
auto eth0
iface eth0 inet dhcp
```

![](/SS/1a.JPG "")

### DHCP Relay (Di router)

#### isc-dhcp-relay

Diatur untuk melayani semua interface (minimal interface subnet client + subnet DHCP Server)

```
# What servers should the DHCP relay forward requests to?
SERVERS="192.189.1.1"

# On what interfaces should the DHCP relay (dhrelay) serve DHCP requests?
INTERFACES="eth1 eth2 eth3 eth4"

# Additional options that are passed to the DHCP relay daemon?
OPTIONS=""
```
![](/SS/1b.JPG "")

#### sysctl.conf

Aktifkan ip forwarding.

```
net.ipv4.ip_forward=1
```

![](/SS/1c.JPG "")

## Client yang melalui Switch3 mendapatkan range IP dari [prefix IP].3.16 - [prefix IP].3.32 dan [prefix IP].3.64 - [prefix IP].3.80 (2)
## Client yang melalui Switch4 mendapatkan range IP dari [prefix IP].4.12 - [prefix IP].4.20 dan [prefix IP].4.160 - [prefix IP].4.168 (3)

### DHCP Server

#### dhcpd.conf

DHCP Server melayani langsung subnet 192.189.1.0 (eth0), maka harus didefinisikan subnetnya. Akan tetapi, yang perlu diberi pengaturan hanya permintaan dari subnet 192.189.3.0 dan 192.189.4.0 yang akan dilayani oleh DHCP Relay.

```
subnet 192.189.1.0 netmask 255.255.255.0 {
}
subnet 192.189.3.0 netmask 255.255.255.0 {
    range 192.189.3.16 192.189.3.32;
    range 192.189.3.64 192.189.3.80;
    option routers 192.189.3.0;
    option broadcast-address 192.189.3.255;
    option domain-name-servers 192.189.1.2;
    default-lease-time 180;
    max-lease-time 5760;
}
subnet 192.189.4.0 netmask 255.255.255.0 {
    range 192.189.4.12 192.189.4.20;
    range 192.189.4.160 192.189.4.168;
    option routers 192.189.4.0;
    option broadcast-address 192.189.4.255;
    option domain-name-servers 192.189.1.2;
    default-lease-time 720;
    max-lease-time 5760;
}
```

`MASALAH : ERROR Code dari DHCP Server hanya muncul di log linux dan harus mencari artinya di internet.`

![](/SS/3a.JPG "")

## Client mendapatkan DNS dari Heiter dan dapat terhubung dengan internet melalui DNS tersebut (4)

**Dalam balasan DHCP Server, diberi alamat IP dari DNS server Heiter.**

#### DNS Forward ke router. 

Pengaturan named.conf.options

```

options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        forwarders {
              192.168.122.1;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        //dnssec-validation auto;
        
        
        // ENABLE FOR SUBDOMAINS
        allow-query{any;}; 


        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
};

```

`MASALAH : Karena DHCP server butuh waktu untuk setup, client harus direstart setelah DHCP Server selesai untuk mendapatkan IP.`



## Lama waktu DHCP server meminjamkan alamat IP kepada Client yang melalui Switch3 selama 3 menit sedangkan pada client yang melalui Switch4 selama 12 menit. Dengan waktu maksimal yang dialokasikan untuk peminjaman alamat IP selama 96 menit (5)

Berikut pengaturan di DHCP Server. Nilai variabel adalah dalam satuan detik.

#### Switch 3

```
    default-lease-time 180;
    max-lease-time 5760;
```

#### Switch 4

```
    default-lease-time 720;
    max-lease-time 5760;
```

![](/SS/4a.JPG "")

## Pada masing-masing worker PHP, lakukan konfigurasi virtual host untuk website berikut dengan menggunakan php 7.3. (6)

### DNS Server

Pertama siapkan untuk domain granz.channel.b22.com

```
mkdir /etc/bind/channel
cp /root/granz.channel.b22.com /etc/bind/channel/granz.channel.b22.com
cp /root/2.189.192.in-addr.arpa /etc/bind/channel/2.189.192.in-addr.arpa
```

Berikut isi file confignya

##### named.conf.local

```
 //
 // Do any local configuration here
 //

 // Consider adding the 1918 zones here, if they are not used in your
 // organization
 //include "/etc/bind/zones.rfc1918";


 zone "granz.channel.b22.com" {
 		type master;
 		file "/etc/bind/granz.channel.b22/granz.channel.b22.com";
 };

 zone "2.189.192.in-addr.arpa" {
 	type master;
 	file "/etc/bind/granz.channel.b22/2.189.192.in-addr.arpa";
 };
```


##### granz.channel.b22.com

```
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     granz.channel.b22.com. root.granz.channel.b22.com. (
                              2023111501         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      granz.channel.b22.com.
@       IN      A       192.189.2.2 ; IP Eisen
@       IN      AAAA    ::1
```

##### 2.189.192.in-addr.arpa

```
 ;
 ; BIND data file for local loopback interface
 ;
 $TTL    604800
 @       IN      SOA     granz.channel.b22.com. root.granz.channel.b22.com. (
                                2023111501         ; Serial
                                604800    ; Refresh
                                86400     ; Retry
                                2419200   ; Expire
                                604800 )  ; Negative Cache TTL
 ;
 2.189.192.in-addr.arpa.         IN      NS      granz.channel.b22.com.
 1                               IN      PTR     granz.channel.b22.com.
```

`MASALAH : Jika panjang serial terlalu panjang (di atas 10) maka terjadi error.`

### WORKER PHP

#### Download dan extract website
```
apt-get update
apt-get install nginx php php-fpm -y
apt-get install wget unzip -y
wget --no-check-certificate 'https://docs.google.com/uc?export=download&id=1ViSkRq7SmwZgdK64eRbr5Fm1EGCTPrU1' -O granz.zip
mkdir /var/www
unzip -j granz.zip -d /var/www/granz.channel.b22.com
```

#### Setup nginx config

Atur port sesuai worker (8001-8003)

```
 server {

      listen 8001; 

      root /var/www/granz.channel.b22.com;

      index index.php index.html index.htm;
      server_name _;

      location / {      
		try_files $uri $uri/ /index.php?$query_string;
 	}

 	# pass PHP scripts to FastCGI server
 	location ~ \.php$ {
 	include snippets/fastcgi-php.conf;
 	fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
 	}

 location ~ /\.ht {
 			deny all;
 	}

 	error_log /var/log/nginx/granz_error.log;
 	access_log /var/log/nginx/granz_access.log;
 }
```

#### Script.sh
```
echo nameserver 192.168.122.1 > /etc/resolv.conf
apt-get update
apt-get install nginx php php-fpm -y

#Isi file granz.channel.b22.com
...
...
#End

cp -r /root/modul-3 /var/www/granz.channel.b22.com
cp /root/granz.channel.b22.com  /etc/nginx/sites-available/granz.channel.b22.com
ln -s /etc/nginx/sites-available/granz.channel.b22.com /etc/nginx/sites-enabled

rm -rf /etc/nginx/sites-enabled/default
service php7.3-fpm start
service php7.3-fpm restart
service nginx restart
```

### LOAD BALANCER

#### Nginx config
```
 # Default menggunakan Round Robin
 upstream myweb  {
 	server 192.189.3.1:6001; 
 	server 192.189.3.2:6002; 
      server 192.189.3.3:6003; 

 }

 # START : Pengaturan password + whitelist ip (no 10 & 12)
 ...
satisfy all;

allow 192.189.3.69;
allow 192.189.3.70;
allow 192.189.4.167;
allow 192.189.4.168;
deny  all;

        auth_basic           "Protected Area";
        auth_basic_user_file /etc/nginx/rahasisakita/.htpasswd;
# END
 server {
 	listen 80;
 	server_name granz.channel.b22.com;

        auth_basic           "Protected Area";
        auth_basic_user_file /etc/nginx/rahasisakita/.htpasswd;

 	location / {
 	proxy_pass http://myweb;
                  proxy_set_header    X-Real-IP $remote_addr;
                proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header    Host $http_host;
 	}
      # START : Proxy pass untuk no 11
      location ~ /its {
            proxy_pass https://www.its.ac.id;
      }
      # END
 }
```

#### Scripts.sh
```

echo nameserver 192.168.122.1 > /etc/resolv.conf
apt-get update && apt-get install nginx php php-fpm -y

cp /root/lb-eisen /etc/nginx/sites-available/lb-eisen
      unlink /etc/nginx/sites-enabled/default
 ln -s /etc/nginx/sites-available/lb-eisen /etc/nginx/sites-enabled
service nginx restart
```

## ...aturlah agar Eisen dapat bekerja dengan maksimal, lalu lakukan testing dengan 1000 request dan 100 request/second. (7)

### Setup tools
- Di client = `apt-get install apache2-utils -y`
- Di worker = `apt-get install htop -y`

- TESTING
```
ab -n 1000 -c 100 http://granz.channel.b22.com/
```

## Karena diminta untuk menuliskan grimoire, buatlah analisis hasil testing dengan 200 request dan 10 request/second masing-masing algoritma Load Balancer dengan ketentuan sebagai berikut:
- Nama Algoritma Load Balancer
- Report hasil testing pada Apache Benchmark
- Grafik request per second untuk masing masing algoritma. 
- Analisis (8)

##### Command ab

```
ab -n 200 -c 10 http://granz.channel.b22.com/
```

##### Tambahkan di `upstream` untuk kumpulan IP worker di config load balancer sesuai jenis algoritma (Tidak perlu tambah untuk Round Robin) 
- least_conn; = Least connection
- ip_hash; = IP Hash
- hash $request_uri consistent; = Generic Hash

**NOTE : Weighted Round robin tidak dicoba karena menurut htop, kemampuan hardware setiap worker SAMA**

## Dengan menggunakan algoritma Round Robin, lakukan testing dengan menggunakan 3 worker, 2 worker, dan 1 worker sebanyak 100 request dengan 10 request/second, kemudian tambahkan grafiknya pada grimoire. (9)

Command yang digunakan adalah :

```
ab -n 100 -c 10 http://granz.channel.b22.com/
```

Setiap pengujian, salah satu worker dimatikan.

## Selanjutnya coba tambahkan konfigurasi autentikasi di LB dengan dengan kombinasi username: “netics” dan password: “ajkyyy”, dengan yyy merupakan kode kelompok. Terakhir simpan file “htpasswd” nya di /etc/nginx/rahasisakita/ (10)

### Install htpasswd + Membuat file htpasswd

```
 apt install apache2-utils
 mkdir /etc/nginx/rahasisakita
 htpasswd -c -b /etc/nginx/rahasisakita/.htpasswd netics ajkb22
```

### Tambah di file config LB file password yang digunakan.

Tambahkan di **scope server**.

```
auth_basic_user_file /etc/nginx/rahasisakita/.htpasswd;
```

## Lalu buat untuk setiap request yang mengandung /its akan di proxy passing menuju halaman https://www.its.ac.id. (11) hint: (proxy_pass)

### Tambah di file config LB

~ = Matching dengan substring '/its' pertama yang ditemukan. Posisi substring bisa dimanapun dalam link.

```
      location ~ /its {
            proxy_pass https://www.its.ac.id;
      }
```

## Selanjutnya LB ini hanya boleh diakses oleh client dengan IP [Prefix IP].3.69, [Prefix IP].3.70, [Prefix IP].4.167, dan [Prefix IP].4.168. (12)

### Tambah di file config LB

- Satify all = Agar harus memenuhi syarat password DAN IP.
- Allow = IP yang diwhitelist
- deny all = Blok semua IP selain yang diwhitelist

**NOTE : Perintah dieksekusi dari atas ke bawah. Jika *deny all* diletakkan pertama, semua perintah *allow* di bawahnya tidak akan dieksekusi.**

```
      satisfy all;
      ....
    allow 192.189.3.69;
    allow 192.189.3.70;
    allow 192.189.4.167;
    allow 192.189.4.168;
    deny  all;
```

Untuk mengakses, salah satu client harus diganti ipnya menjadi sesuai dengan soal.

## `SS Hasil`

- ##### IP Salah

![](/SS/12-ip.JPG "")

- ##### Password

![](/SS/12-pass.JPG "")


- ##### http://granz.channel.b22.com

![](/SS/12-done.JPG "")


- ##### http://granz.channel.b22.com dengan /its

![](/SS/12-its-1.JPG "")

-----


## Tambahan Setup Laravel

### Tambah user di database

#### Script tambah_user.sql
```
CREATE USER 'kelompokb22'@'%' IDENTIFIED BY 'ajkb22';
CREATE USER 'kelompokb22'@'localhost' IDENTIFIED BY 'ajkb22';
CREATE DATABASE dbkelompokb22;
GRANT ALL PRIVILEGES ON *.* TO 'kelompokb22'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'kelompokb22'@'localhost';
FLUSH PRIVILEGES;
```
#### Script bash tambah_user.sh
```
mysql -uroot -D mysql -e "source ./tambah_user.sql"
```

----

## Semua data yang diperlukan, diatur pada Denken dan harus dapat diakses oleh Frieren, Flamme, dan Fern. (13)

### Script.sh di root digunakan untuk menginstall mariadb-server dan memulai service mysql
```
apt-get update

apt-get install mariadb-server -y

service mysql start

cp /root/my.cnf /etc/mysql/

cp /root/50-server.cnf /etc/mysql/mariadb.conf.d/
```

Pada script di atas juga diberikan perintah untuk memasukkan file konfigurasi mysql, meliputi my.cnf dan 50-server.cnf

Berikut isi dari file tersebut
### my.cnf
```
# The MariaDB configuration file
#
# The MariaDB/MySQL tools read configuration files in the following order:
# 1. "/etc/mysql/mariadb.cnf" (this file) to set global defaults,
# 2. "/etc/mysql/conf.d/*.cnf" to set global options.
# 3. "/etc/mysql/mariadb.conf.d/*.cnf" to set MariaDB-only options.
# 4. "~/.my.cnf" to set user-specific options.
#
# If the same option is defined multiple times, the last one will apply.
#
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.

#
# This group is read both both by the client and the server
# use it for options that affect everything
#
[client-server]

# Import all .cnf files from configuration directory
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/

[mysqld]
skip-networking=0
skip-bind-address
```

### 50-server.cnf
```
#
# These groups are read by MariaDB server.
# Use it for options that only the server (but not clients) should see
#
# See the examples of server my.cnf files in /usr/share/mysql

# this is read by the standalone daemon and embedded servers
[server]

# this is only for the mysqld standalone daemon
[mysqld]

#
# * Basic Settings
#
user                    = mysql
pid-file                = /run/mysqld/mysqld.pid
socket                  = /run/mysqld/mysqld.sock
#port                   = 3306
basedir                 = /usr
datadir                 = /var/lib/mysql
tmpdir                  = /tmp
lc-messages-dir         = /usr/share/mysql
#skip-external-locking

# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
bind-address            = 0.0.0.0

#
# * Fine Tuning
#
#key_buffer_size        = 16M
#max_allowed_packet     = 16M
#thread_stack           = 192K
#thread_cache_size      = 8
# This replaces the startup script and checks MyISAM tables if needed
# the first time they are touched
#myisam_recover_options = BACKUP
#max_connections        = 100
#table_cache            = 64
#thread_concurrency     = 10

#
# * Query Cache Configuration
#
#query_cache_limit      = 1M
query_cache_size        = 16M

#
# * Logging and Replication
#
# Both location gets rotated by the cronjob.
# Be aware that this log type is a performance killer.
# As of 5.1 you can enable the log at runtime!
#general_log_file       = /var/log/mysql/mysql.log
#general_log            = 1
#
# Error log - should be very few entries.
#
log_error = /var/log/mysql/error.log
#
# Enable the slow query log to see queries with especially long duration
#slow_query_log_file    = /var/log/mysql/mariadb-slow.log
#long_query_time        = 10
#log_slow_rate_limit    = 1000
#log_slow_verbosity     = query_plan
#log-queries-not-using-indexes
#
# The following can be used as easy to replay backup logs or for replication.
# note: if you are setting up a replication slave, see README.Debian about
#       other settings you may need to change.
#server-id              = 1
#log_bin                = /var/log/mysql/mysql-bin.log
expire_logs_days        = 10
#max_binlog_size        = 100M
#binlog_do_db           = include_database_name
#binlog_ignore_db       = exclude_database_name

#
# * Security Features
#
# Read the manual, too, if you want chroot!
#chroot = /var/lib/mysql/
#
# For generating SSL certificates you can use for example the GUI tool "tinyca".
#
#ssl-ca = /etc/mysql/cacert.pem
#ssl-cert = /etc/mysql/server-cert.pem
#ssl-key = /etc/mysql/server-key.pem
#
# Accept only connections using the latest and most secure TLS protocol version.
# ..when MariaDB is compiled with OpenSSL:
#ssl-cipher = TLSv1.2
# ..when MariaDB is compiled with YaSSL (default in Debian):
#ssl = on

#
# * Character sets
#
# MySQL/MariaDB default is Latin1, but in Debian we rather default to the full
# utf8 4-byte character set. See also client.cnf
#
character-set-server  = utf8mb4
collation-server      = utf8mb4_general_ci

#
# * InnoDB
#
# InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.
# Read the manual for more InnoDB related options. There are many!

#
# * Unix socket authentication plugin is built-in since 10.0.22-6
#
# Needed so the root database user can authenticate without a password but
# only when running as the unix root user.
#
# Also available for other users if required.
# See https://mariadb.com/kb/en/unix_socket-authentication-plugin/

# this is only for embedded server
[embedded]

# This group is only read by MariaDB servers, not by MySQL.
# If you use the same .cnf file for MySQL and MariaDB,
# you can put MariaDB-only options here
[mariadb]

# This group is only read by MariaDB-10.3 servers.
# If you use the same .cnf file for MariaDB of different versions,
# use this group for options that older servers don't understand
[mariadb-10.3]
```

Kemudian pada masing-masing Frieren, Flamme, dan Fern diinstal mariadb-client
```
apt-get update

apt-get install mariadb-client -y
```

Dan dapat dicek menggunakan perintah berikut:
```
mariadb --host=(IP Database) --port=3306 --user=xxx --password=xxx
```

## Frieren, Flamme, dan Fern memiliki Riegel Channel sesuai dengan quest guide berikut. Jangan lupa melakukan instalasi PHP8.0 dan Composer (14)

### script.sh berisi konfigurasi untuk laravel yang meliputi clone dari git dan konfigurasi .env
```
echo nameserver 192.168.122.1 > /etc/resolv.conf

apt-get update && apt-get install git -y
apt-get install mariadb-client -y

apt-get install -y lsb-release ca-certificates apt-transport-https software-properties-common gnupg2
curl -sSLo /usr/share/keyrings/deb.sury.org-php.gpg https://packages.sury.org/php/apt.gpg
sh -c 'echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'

apt-get update
apt-get install php8.0-mbstring php8.0-xml php8.0-cli php8.0-common php8.0-intl php8.0-opcache php8.0-readline php8.0-mysql php8.0-fpm php8.0-curl unzip wget -y
apt-get install nginx -y

wget https://getcomposer.org/download/2.0.13/composer.phar
chmod +x composer.phar
mv composer.phar /usr/bin/composer

service php8.0-fpm start

cd /var/www
git clone https://github.com/martuafernando/laravel-praktikum-jarkom.git

cd /var/www/laravel-praktikum-jarkom 
composer update 
composer install

cp ~/.env /var/www/laravel-praktikum-jarkom
php artisan migrate:fresh
php artisan db:seed --class=AiringsTableSeeder
php artisan jwt:secret

chown -R www-data:www-data /var/www

service nginx restart

php artisan key:generate
echo 'server {

        listen 8004;

        root /var/www/laravel-praktikum-jarkom/public;

        index index.php index.html index.htm;
        server_name _;

        location / {
                        try_files $uri $uri/ /index.php?$query_string;
        }

        # pass PHP scripts to FastCGI server
        location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
        }

 location ~ /\.ht {
                        deny all;
        }

        error_log /var/log/nginx/laravelb22_error.log;
        access_log /var/log/nginx/laravelb22_access.log;
 }' > /etc/nginx/sites-available/laravelb22

ln -s /etc/nginx/sites-available/laravelb22 /etc/nginx/sites-enabled/

chown -R www-data.www-data /var/www/laravel-praktikum-jarkom/storage

service php8.0-fpm restart
service nginx restart
```

## Riegel Channel memiliki beberapa endpoint yang harus ditesting sebanyak 100 request dengan 10 request/second. Tambahkan response dan hasil testing pada grimoire.

```
ab -n 100 -c 10 http://riegel.canyon.b22.com:8004/
```

REFERENSI : **https://github.com/jefersonralmeida/laravel-benchmarks**

- ### POST /auth/register (15)

```
ab -n 100 -c 10 -p register.json -T application/json http://riegel.canyon.b22.com:8004/api/auth/register
```


- ### POST /auth/login (16)

```
ab -n 100 -c 10 -p login.json -T application/json http://riegel.canyon.b22.com:8004/api/auth/login
```


- ### GET /me (17)

```
ab -n 100 -c 10 http://riegel.canyon.b22.com/api/me
```


## Untuk memastikan ketiganya bekerja sama secara adil untuk mengatur Riegel Channel maka implementasikan Proxy Bind pada Eisen untuk mengaitkan IP dari Frieren, Flamme, dan Fern. (18)

### Karena ketiganya dikaitkan ke domain yang sama :

File lb-riegel
```
upstream myweb  {
        server 192.189.4.1:8004;
        server 192.189.4.2:8005;
        server 192.189.4.3:8006;
 }

 server {
        listen 80;
        server_name riegel.canyon.b22.com;

        location / {
        proxy_bind 192.189.2.2;
        proxy_pass http://myweb;
        }
 }

```


## Untuk meningkatkan performa dari Worker, coba implementasikan PHP-FPM pada Frieren, Flamme, dan Fern. Untuk testing kinerja naikkan 
- pm.max_children
- pm.start_servers
- pm.min_spare_servers
- pm.max_spare_servers
## sebanyak tiga percobaan dan lakukan testing sebanyak 100 request dengan 10 request/second kemudian berikan hasil analisisnya pada Grimoire.(19)

### Ubah isi www.conf. Untuk kemudahan dalam mengubah variabel, hapus semua komentar.

```
[www]
user = www-data
group = www-data
listen = /run/php/php8.0-fpm.sock
listen.owner = www-data
listen.group = www-data
php_admin_value[disable_functions] = exec,passthru,shell_exec,system
php_admin_flag[allow_url_fopen] = off

; Choose how the process manager will control the number of child processes.

pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

### Ubah isi /etc/php/8.0/fpm/pool.d/www.conf sesuai poin yang diminta.

Aturan testing :

**Setiap pengujian, semua variabel yang diminta soal dikali 2.**

- Test 1
```
pm.max_children = 10
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 6
``` 
- Test 2
```
pm.max_children = 20
pm.start_servers = 8
pm.min_spare_servers = 4
pm.max_spare_servers = 12
```
- Test 3
```
pm.max_children = 40
pm.start_servers = 16
pm.min_spare_servers = 8
pm.max_spare_servers = 24
```

### Ulangi load test di nomor 16.

## Nampaknya hanya menggunakan PHP-FPM tidak cukup untuk meningkatkan performa dari worker maka implementasikan Least-Conn pada Eisen. Untuk testing kinerja dari worker tersebut dilakukan sebanyak 100 request dengan 10 request/second. (20)

### Kembalikan setup www.conf

```
cp /root/www.conf /etc/php/8.0/fpm/pool.d/www.conf 
```

### Ubah config lb agar menggunakan least-conn

```
least_conn;
```

### Ulangi load test di nomor 16.
