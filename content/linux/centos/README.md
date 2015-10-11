CentOS installation
-------------------

#### Add rpm library

```
rpm -ivh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
```

#### Security plugin

```
yum install yum-plugin-security
yum --security check-update
yum --security update
```

#### Install useful tools

```
yum install vim rsync zsh
chsh -s /bin/zsh
```

#### Install iptables

```
yum install system-config-firewall-tui
```

#### Install Lamp

```
yum --enablerepo=remi install httpd php php-devel php-gd php-mysql mysql mysql-server phpmyadmin

# Start services
service mysqld start
service httpd start
# Start services at startup
chkconfig mysqld on
chkconfig httpd on
# Set mysql configuration
sudo mysql_secure_installation
```

#### Php configuration

vi /etc/php.ini
modify `short_opentag=Off` to `short_opentag=On`

#### Set permission for WWW

```
chown -R apache:apache /var/www
chmod 775 /var/www
chcon -v -R --type=httpd_sys_content_t /var/www
```

#### Firewall setting

vi /etc/sysconfig/iptables

```
# YOUR_IP_ADDRESS can be x.x.x.x or x.x.0.0/16
-A INPUT -p tcp -s YOUR_IP_ADDRESS -m tcp --dport xx -j ACCEPT
-A INPUT -m state --state NEW -m iprange --src-range YOUR_IP_ADDRESS-YOUR_IP_ADDRESS -p tcp --dport xx -j ACCEPT
```

#### Add user

```
adduser ____
passwd ____
gpasswd -a ____ wheel
"chmod u+w /etc/sudoers"
add `____ ALL=(ALL) ALL` below `root ALL=(ALL) ALL` 
chmod u-w /etc/sudoers
usermod -a -G apache ______
"chmod u+w /etc/sudoers"
```

#### Ssh configuration

add public key into `/etc/ssh/authorized_keys`

```
chmod 400 /etc/ssh/authorized_keys
```

vi /etc/ssh/sshd_config

```
RSAAuthentication yes
PubkeyAuthentication yes
# Make sure that you can login with key before you set the following config.
PermitRootLogin no
PermitEmptyPasswords no
PasswordAuthentication no
```

#### Backup

#### Install crontab 

```
yum install cronie
service crond start
chkconfig crond on
```

#### Assign permission while mounting

```
UUID=____ /media/root/home              ntfs    uid=1001,gid=1001,dmask=022,fmask=133 0       4
```

#### Limit user 

edit /etc/ssh/sshd_config

```
Match User ____
Chrootdirectory /media/root
```

```
ldd /bin/ls
ldd /bin/pwd
ldd /bin/cp
ldd /bin/rsync
cp /bin/ls /media/root/bin/
cp -v /lib/x86_64-linux-gnu/libxxx /media/root/lib/x86_64-linux-gnu/
cp -r .ssh /media/root/home
```

#### Set crontab

crontab -e

```
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
52 1 * * * cd ~&&./shell.sh
```

#### Increased backup

```
#!/bin/bash
#backup.sh
mysqldump -u____ -p_____ --all-databases --skip-lock-tables > $(date +"%m_%d_%y").sql
file=$(date +"%m_%d_%y").sql
if [ -f "$file" ]
then
        echo "$file found.">a
        rsync -arHqP -e "ssh -p ____" ~/$file user@x.x.x.x:/home/backup
        rsync -artlHSqP --delete -e "ssh -p ____" /var/www user@x.x.x.x:/home/backup/inc_backup
        ssh -p 1022 user@x.x.x.x cp -r /home/backup/inc_backup /home/backup/$(date +"%m_%d_%y")
        echo "Finished backup."
else
        echo "$file not found."
fi
```
