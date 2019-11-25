# Tugas 3 Basis Data Terdistribusi

- [Diagram Arsitektur](#diagram-arsitektur)
- [Buat 3 Node Redis Server (1 Master dan 2 Slave)](#3-node)
- [Buat 2 Node Web Server (Di dalamnya ada Wordpress dan MySQL)](#2-node)
- [Pengujian performa kedua web server](#tes-performa)
- [Pengujian fail over](#tes-failover)
- [Pengujian JMeter](#j-meter)

## Diagram Arsitektur

![arsitektur](/img/arsitektur.jpg)

Spesifikasi
- Redis Master
  - IP : 15.16.17.16
  - OS : bento/ubuntu-18.04
  - Memory : 512 mb
- Redis Slave 1
  - IP : 15.16.17.17
  - OS : bento/ubuntu-18.04
  - Memory : 512 mb
- Redis Slave 1
  - IP : 15.16.17.18
  - OS : bento/ubuntu-18.04
  - Memory : 512 mb
- Wordpress Redis
  - IP : 15.16.17.19
  - OS : bento/ubuntu-18.04
  - Memory : 512 mb
- Wordpress Non
  - IP : 15.16.17.20
  - OS : bento/ubuntu-18.04
  - Memory : 512 mb

Vagrant File

```ruby
Vagrant.configure("2") do |config|

  config.vm.define "redismaster" do |node|
    node.vm.hostname = "redismaster"
    node.vm.box = "bento/ubuntu-18.04"
    node.vm.network "private_network", ip: "15.16.17.16"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "redismaster"
      vb.gui = false
      vb.memory = "512"
    end

    node.vm.provision "shell", path: "provision/redismaster.sh", privileged: false
  end

  (1..2).each do |i|
    config.vm.define "redisslave#{i}" do |node|
      node.vm.hostname = "redisslave#{i}"
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.network "private_network", ip: "15.16.17.#{16+i}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "redisslave#{i}"
        vb.gui = false
        vb.memory = "512"
      end

      node.vm.provision "shell", path: "provision/redisslave#{i}.sh", privileged: false
    end
  end

  (1..2).each do |i|
    config.vm.define "wordpress#{i}" do |node|
      node.vm.hostname = "wordpress#{i}"
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.network "private_network", ip: "15.16.17.#{18+i}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "wordpress#{i}"
        vb.gui = false
        vb.memory = "512"
      end

      node.vm.provision "shell", path: "provision/wordpress.sh", privileged: false
    end
  end

end
```

## Buat 3 Node Redis Server (1 Master dan 2 Slave)

### Vagrant File untuk membuat 3 Redis Server

```ruby
...
  config.vm.define "redismaster" do |node|
    node.vm.hostname = "redismaster"
    node.vm.box = "bento/ubuntu-18.04"
    node.vm.network "private_network", ip: "15.16.17.16"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "redismaster"
      vb.gui = false
      vb.memory = "512"
    end

    node.vm.provision "shell", path: "provision/redismaster.sh", privileged: false
  end

  (1..2).each do |i|
    config.vm.define "redisslave#{i}" do |node|
      node.vm.hostname = "redisslave#{i}"
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.network "private_network", ip: "15.16.17.#{16+i}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "redisslave#{i}"
        vb.gui = false
        vb.memory = "512"
      end

      node.vm.provision "shell", path: "provision/redisslave#{i}.sh", privileged: false
    end
  end
...
```

### File Provision

redis-master

```bash
sudo cp /vagrant/sources/hosts /etc/hosts
sudo cp '/vagrant/sources/sources.list' '/etc/apt/'

sudo apt update -y

sudo apt-get install redis -y

sudo cp /vagrant/config/redis-master.conf /etc/redis/redis.conf
sudo cp /vagrant/config/sentinel-master.conf /etc/redis-sentinel.conf

```

redis-slave-1

```bash
sudo cp /vagrant/sources/hosts /etc/hosts
sudo cp '/vagrant/sources/sources.list' '/etc/apt/'

sudo apt update -y

sudo apt-get install redis -y

sudo cp /vagrant/config/redis-slave-1.conf /etc/redis/redis.conf
sudo cp /vagrant/config/sentinel-slave-1.conf /etc/redis-sentinel.conf

```

redis-slave-2

```sh
sudo cp /vagrant/sources/hosts /etc/hosts
sudo cp '/vagrant/sources/sources.list' '/etc/apt/'

sudo apt update -y

sudo apt-get install redis -y

sudo cp /vagrant/config/redis-slave-2.conf /etc/redis/redis.conf
sudo cp /vagrant/config/sentinel-slave-2.conf /etc/redis-sentinel.conf

```

### File Config

redis-master

```bash
bind 15.16.17.16
port 6379

dir "/etc/redis"
```

sentinel-master

```bash
bind 15.16.17.16
port 26379

sentinel monitor redis-cluster 15.16.17.16 6379 2
sentinel down-after-milliseconds redis-cluster 5000
sentinel parallel-syncs redis-cluster 1
sentinel failover-timeout redis-cluster 10000
```

redis-slave-1

```bash
bind 15.16.17.17
port 6379

dir "/etc/redis"

slaveof 15.16.17.16 6379
```

sentinel-slave-1

```bash
bind 15.16.17.17
port 26379

sentinel monitor redis-cluster 15.16.17.16 6379 2
sentinel down-after-milliseconds redis-cluster 5000
sentinel parallel-syncs redis-cluster 1
sentinel failover-timeout redis-cluster 10000
```

redis-slave-2

```bash
bind 15.16.17.18
port 6379

dir "/etc/redis"

slaveof 15.16.17.16 6379
```

sentinel-slave-2

```bash
bind 15.16.17.18
port 26379

sentinel monitor redis-cluster 15.16.17.16 6379 2
sentinel down-after-milliseconds redis-cluster 5000
sentinel parallel-syncs redis-cluster 1
sentinel failover-timeout redis-cluster 10000
```

## Buat 2 Node Web Server (Di dalamnya ada Wordpress dan MySQL)

### Vagrant File untuk membuat 2 Web Server

```ruby
...
  (1..2).each do |i|
    config.vm.define "wordpress#{i}" do |node|
      node.vm.hostname = "wordpress#{i}"
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.network "private_network", ip: "15.16.17.#{18+i}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "wordpress#{i}"
        vb.gui = false
        vb.memory = "512"
      end

      node.vm.provision "shell", path: "provision/wordpress.sh", privileged: false
    end
  end
...
```

### File Provision

```bash
sudo cp /vagrant/sources/hosts /etc/hosts
sudo cp '/config/sources/sources.list' '/etc/apt/'

sudo apt update -y

# Instalasi Web Server apache2
sudo apt install apache2 -y
sudo ufw allow in "Apache Full"

# Instalasi PHP
sudo apt install php libapache2-mod-php php-mysql php-pear php-dev -y
sudo a2enmod mpm_prefork && sudo a2enmod php7.0
sudo pecl install redis
sudo echo 'extension=redis.so' >> /etc/php/7.2/apache2/php.ini

# Instalasi Database MySQL
sudo debconf-set-selections <<< 'mysql-server mysql-server/root_password password admin'
sudo debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password admin'
sudo apt install mysql-server -y
sudo mysql_secure_installation -y
sudo ufw allow 3306

# Konfigurasi user untuk mysql
sudo mysql -u root -padmin < /vagrant/config/wordpress.sql

# Instalasi Wordpress
cd /tmp
wget -c http://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
sudo mkdir -p /var/www/html
sudo mv wordpress/* /var/www/html
sudo cp /vagrant/config/wp-config.php /var/www/html/
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
sudo systemctl restart apache2
```

### File Config

wordpress sql
```sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;

CREATE USER 'user'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES on wordpress.* to 'user'@'%';
FLUSH PRIVILEGES;
```

wordpress config
```php
<?php
 /**
  * The base configuration for WordPress
  *
  * The wp-config.php creation script uses this file during the
  * installation. You don't have to use the web site, you can
  * copy this file to "wp-config.php" and fill in the values.
  *
  * This file contains the following configurations:
  *
  * * MySQL settings
  * * Secret keys
  * * Database table prefix
  * * ABSPATH
  *
  * @link https://codex.wordpress.org/Editing_wp-config.php
  *
  * @package WordPress
  */
 // ** MySQL settings - You can get this info from your web host ** //
 /** The name of the database for WordPress */
 define('DB_NAME', 'wordpress');
 /** MySQL database username */
 define('DB_USER', 'user');
 /** MySQL database password */
 define('DB_PASSWORD', 'password');
 /** MySQL hostname */
 define('DB_HOST', 'localhost:3306');
 /** Database Charset to use in creating database tables. */
 define('DB_CHARSET', 'utf8');
 /** The Database Collate type. Don't change this if in doubt. */
 define('DB_COLLATE', '');
 /**#@+
  * Authentication Unique Keys and Salts.
  *
  * Change these to different unique phrases!
  * You can generate these using the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}
  * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.
  *
  * @since 2.6.0
  */
 define('AUTH_KEY',         'bdt redis');
 define('SECURE_AUTH_KEY',  'bdt redis');
 define('LOGGED_IN_KEY',    'bdt redis');
 define('NONCE_KEY',        'bdt redis');
 define('AUTH_SALT',        'bdt redis');
 define('SECURE_AUTH_SALT', 'bdt redis');
 define('LOGGED_IN_SALT',   'bdt redis');
 define('NONCE_SALT',       'bdt redis');
 /**#@-*/
 /**
  * WordPress Database Table prefix.
  *
  * You can have multiple installations in one database if you give each
  * a unique prefix. Only numbers, letters, and underscores please!
  */
 $table_prefix  = 'wp_';
 /**
  * For developers: WordPress debugging mode.
  *
  * Change this to true to enable the display of notices during development.
  * It is strongly recommended that plugin and theme developers use WP_DEBUG
  * in their development environments.
  *
  * For information on other constants that can be used for debugging,
  * visit the Codex.
  *
  * @link https://codex.wordpress.org/Debugging_in_WordPress
  */
 define('WP_DEBUG', false);
 /* That's all, stop editing! Happy blogging. */
 /** Absolute path to the WordPress directory. */
 if ( !defined('ABSPATH') )
     define('ABSPATH', dirname(__FILE__) . '/');
 /** Sets up WordPress vars and included files. */
 require_once(ABSPATH . 'wp-settings.php');
```

## Validasi Konfigurasi

Jalankan
```
vagrant up
```

Masuk ke Redis Server
```
vagrant ssh redismaster
```
```
vagrant ssh redisslave1
```
```
vagrant ssh redisslave2
```

![login](/img/login.png)

Pada masing-masing redis server jalankan redis-server dan sentinel nya.  
Jalankan command di bawah secara berurutan
```
cd /
```
(Menggunakan '&' agar command berjalan terus.)  
```
sudo redis-server etc/redis/redis.conf &
```
```
sudo redis-server etc/redis-sentinel.conf --sentinel &
```

Untuk menghentikan proses
```
sudo pkill redis-server
```
atau
```
sudo service redis-server stop
```
Untuk mengecek apakah proses berjalan
```
ps -ef | grep redis
```
![cluster](/img/cluster.png)
Bisa dilihat bahwa replikasi telah berjalan

## Pengujian performa kedua web server

### Instalasi Wordpress pada salah satu web server
Langkah - langkah  
Agar lebih mudah maka halaman awal default apache dinonaktifkan terlebih dahulu
![change_index](/img/change_index.png)
1. Buka alamat IP wordpress pada browser (Alamat_IP/index.php)
![index](/img/index.png)
2. Ikuti langkah-langkah instalasinya sampai selesai
![dashboard](/img/dashboard.png)
3. Tambah Plugin Redis Object Cache dengan cara pilih Menu Plugins --> Add New --> Search Plugins --> Install Now
![redis_object](/img/redis_object.png)
4. Ubah ServerInfo.php dengan cara
    ```
    vagrant ssh wordpress1
    ```
    ```
    cd /
    ```
    ```
    sudo nano /var/www/html/wp-content/plugins/redis-cache/includes/predis/src/Command/ServerInfo.php
    ```
    Ubah fungsi getId() menjadi
    ```php
    public function getId()
    {
        return 'info';
    }
    ```
5. Tambah konfigurasi pada wp-config.php dengan cara
    ```
    sudo nano /var/www/html/wp-config.php 
    ```
    Tambah konfigurasi
    ```
    define('FS_METHOD', 'direct');
    define('WP_REDIS_CLIENT', 'predis');
    define('WP_REDIS_SENTINEL', 'redis-cluster');
    define('WP_REDIS_SERVERS', ['tcp://15.16.17.16:26379', 'tcp://15.16.17.17:26379', 'tcp://15.16.17.18:26379']);
    ```
    ![wp_config](/img/wp_config.png)
6. Aktifkan Redis Object Cache
7. Cek Diagnostics
![diagnostics](/img/diagnostics.png)
8. Tampilan Wordpress
![index_awal](/img/index_awal.png)
## Pengujian fail over

1. Dengan cara mematikan redis server master, dalam kasus ini mematikan redisslave2
![redisslave_master](/img/redisslave_master.png)
![redisslave_death](/img/redisslave_death.png)
2. Melakukan pengecekan siapa yang menggantikan redisslave2
![new_master](/img/new_master.png)
Dari hasil keluaran bisa dilihat bahwa sekarang master pindah ke redisslave1
3. Memastikan dengan login ke redisslave2
![new_master2](/img/new_master2.png)
4. Proses fail over berhasil ditangani

## Pengujian JMeter

1. 50 Koneksi
![jmeter-50](/img/jmeter-50.png)
2. 100 + 16(NRP)
![jmeter-116](/img/jmeter-116.png)
2. 200 + 16(NRP)
![jmeter-216](/img/jmeter-216.png)

    Dari pengujian menggunakan JMeter Wordpress yang menggunakan redis sebagai cache nya melakukan load data relatif lambat daripada yang tidak menggunakan redis. Padahal seharusnya Wordpress yang menggunakan redis bisa melakukan load data lebih cepat.  
    Asumsi saya pribadi hal ini terjadi karena memori yang relatif kecil pada masing-masing redis server.