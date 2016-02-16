Install laravel 4 in Ubuntu 12.04
--------------------------------------

#### Upgrade php 5.3 to 5.4

```
sudo apt-get install python-software-properties
sudo add-apt-repository ppa:ondrej/php5-oldstable
sudo apt-get update
sudo apt-cache policy php5
```

#### Install dependances

```
sudo apt-get install apache2
sudo apt-get install php5
sudo apt-get install mysql-server
sudo apt-get install php5-mysql

sudo apt-get install unzip
sudo apt-get install curl
sudo apt-get install openssl
sudo apt-get install php5-mcrypt
```

#### Install Composer

```
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

#### Add a virtualhost to Apache(/etc/apache2/sites-available)

```
<VirtualHost *:80>
   ServerName laravel.neothread.com
   # !!! Be sure to point DocumentRoot to 'public'!
   DocumentRoot path/to/laravel/public
   <Directory path/to/laravel/public>
      # This relaxes Apache security settings.
      AllowOverride all
      # MultiViews must be turned off.
      Options -MultiViews
      # Uncomment this if you're on Apache >= 2.4:
      #Require all granted
      Order deny,allow
      deny from all
      allow from all
   </Directory>
</VirtualHost>
```

```
sudo a2ensite YourDomainName

sudo service apache2 restart
```

### Clone laravel

```
git clone https://github.com/laravel/laravel.git
git checkout v4.2.11
```

### Change permission

```
cd laravel
chown -R :www-data app/storage
chmod -R 775 app/storage
```
### Install with Composer

```
composer install
```
