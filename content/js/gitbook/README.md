Gitbook
-------

Gitbook is an application written in nodejs.
You can write some articles in markdown and can also publish your work as
a website or ebook. Enjoy writting with Gitbook!

More information about [`gitbook`](http://cowmanchiang.me/gitbook/gitbook/)
 and [`official site`](http://help.gitbook.com/)

Note
----

####Environment
* Operation System: Ubuntu 12.04 / 64 bit
* Ruby: 2.2.2
* Gitbook: 2.1.0
* Passenger: 5.0.13

####To customize my gitbook
I try to install some plugins, such as `plantuml`, `mathjax`,
`expandable-chapters`, and do some fixes in those.

* `gitbook-plugin-plantuml/index.js`: replaced line 142 with `assetPath`.
* `gitbook-plugin-expandable-chapters/book/expandable-chapters.js`: comment line 23, 24.

####To deploy the application on Apache
First, you need to install `Passenger` and `Apache Passenger Module`.

 ```
$ gem install passenger
```

```
$ echo 'Installing the dependances.'
$ apt-get install -y libcurl4-openssl-dev apache2-threaded-dev libapr1-dev libaprutil1-dev
$ passenger-install-apache2-module
```

If you see the message which is similer to the following while installing, save as a file
 named and located at `/etc/apache2/conf.d/passenger.conf`

```
LoadModule passenger_module .../apache2/mod_passenger.so
<IfModule mod_passenger.c>
  PassengerRoot ...passenger-...
  PassengerDefaultRuby ...wrappers/ruby
</IfModule>
```

Second, you need to create virtual host in Apache.

It is not difficult. You just create a file `/etc/apache2/sites-available/YourDomainName`,
 and write this content in it.

```
<VirtualHost *:80>
  ServerName YourDomainName
  # !!! Be sure to point DocumentRoot to 'public'!
  DocumentRoot .../app/public
  <Directory ../app/public>
     # This relaxes Apache security settings.
     AllowOverride all
     # MultiViews must be turned off.
     Options -MultiViews
     # Uncomment this if you're on Apache >= 2.4:
     #Require all granted
  </Directory>
</VirtualHost>
```

After doing that, try to enable configure and restart apache.
You will see your gitbook on your domain name.

`sudo a2ensite YourDomainName`

`sudo service apache2 restart`
