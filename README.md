#!/bin/bash

# Stoping firewall
/sbin/service iptables stop
/sbin/service ip6tables stop

# Stoping firewall to start as daemon
/sbin/chkconfig iptables off
/sbin/chkconfig ip6tables off

#Installing Apache2
usermod -U apache
chkconfig httpd on

# Set your timezone in php.ini to get rid of an ugly warinng in the future.
# find date.timezone at the [date] section, uncomment and set it. for me it's Europe/Bucharest
# date.timezone = Europe/Bucharest
# find yours here: http://www.vmware.com/support/developer/vc-sdk/visdk400pubs/ReferenceGuide/timezone.html
vim /etc/php.ini

# Add port 80 to Firewall
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT 
sudo service iptables save

# creating nagios log folder
mkdir -p /var/log/nagios

# add the line rocommunity public 127.0.0.1/32 to snmpd.conf
sed -i '$ a\rocommunity public 127.0.0.1/32' /etc/snmp/snmpd.conf
service snmpd start
chkconfig snmpd on
# enabling NTP couldn't hurt
chkconfig ntpd on 
ntpdate pool.ntp.org 
service ntpd start
# 

# Creating the nagios user
/usr/sbin/useradd -m nagios
/usr/sbin/usermod -L nagios

# Creating a group to be able to use external commands
/usr/sbin/groupadd nagcmd
/usr/sbin/usermod -G nagios,nagcmd nagios
/usr/sbin/usermod -G nagios,nagcmd apache

# Download & Compile
cd /usr/local/src/
wget -N http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-3.5.0.tar.gz
tar -xzf nagios-3.5.0.tar.gz
cd nagios

./configure --prefix=/usr/local/nagios --with-command-group=nagcmd --enable-nanosleep --enable-event-broker
make all
make install
make install-init
make install-commandmode
make install-config
chkconfig nagios on
service nagios start
#

#
cd /usr/local/src
wget -N http://sourceforge.net/projects/nagiosplug/files/nagiosplug/1.4.16/nagios-plugins-1.4.16.tar.gz

wget -N http://search.cpan.org/CPAN/authors/id/M/MS/MSCHWERN/ExtUtils-MakeMaker-6.64.tar.gz
tar -xzf ExtUtils-MakeMaker-6.64.tar.gz
cd ExtUtils-MakeMaker-6.64
perl Makefile.PL
make
make install

cd /usr/local/src
tar -xzf nagios-plugins-1.4.16.tar.gz
cd nagios-plugins-1.4.16
./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl=/usr/bin/openssl --enable-perl-modules
make

# Test-Simple-0.70 fails to compile otherwise so have to do it manually first
cd ./perlmods/Test-Simple-0.70
perl Makefile.PL
make
make install

# go back to nagios-plugins to make install
cd ../..
make install
# 

#
cd /usr/local/src
wget -N http://sourceforge.net/projects/nagios/files/ndoutils-1.x/ndoutils-1.5.2/ndoutils-1.5.2.tar.gz
tar -zxvf ndoutils-1.5.2.tar.gz
cd ndoutils-1.5.2

# get the patch and apply it; wget to same folder as ndoutils
wget -N http://svn.centreon.com/trunk/ndoutils-patch/ndoutils1.5.2_light.patch
patch -p1 -N < ndoutils1.5.2_light.patch
# continue installation
./configure --prefix=/usr/local/nagios/ --enable-mysql --with-ndo2db-user=nagios --with-ndo2db-group=nagios
make

mkdir -p /usr/local/nagios/bin/
mkdir -p /usr/local/nagios/etc/ 

# After creating the binaries and libraries they have to be copied
cp -f ./src/ndomod-3x.o /usr/local/nagios/bin/ndomod.o
cp -f ./src/ndo2db-3x /usr/local/nagios/bin/ndo2db
cp -f ./config/ndo2db.cfg-sample /usr/local/nagios/etc/ndo2db.cfg
cp -f ./config/ndomod.cfg-sample /usr/local/nagios/etc/ndomod.cfg
sudo chmod 774 /usr/local/nagios/bin/ndo*
sudo chown nagios:nagios /usr/local/nagios/bin/ndo*

# make ndo2db daemon autorun
cp -f ./daemon-init /etc/init.d/ndo2db
chmod +x /etc/init.d/ndo2db
chkconfig ndo2db on
#

## Add innodb_file_per_table=1 under the [mysqld] section in my.cnf
vim /etc/my.cnf

## then
service mysqld start
chkconfig mysqld on
# 

#
service httpd start
groupadd centreon
useradd -g centreon centreon
cd /usr/local/src/
wget http://download.centreon.com/index.php?id=4290

tar -zxf centreon-2.4.3.tar.gz
cd centreon-2.4.3
export PATH="$PATH:/usr/local/nagios/bin/"

# create an answer file with the following contents:
vim ./answer
#
./install.sh -f ./answer

# After installation configure SELinux
#yum -y install policycoreutils-python
#semanage fcontext -a -t httpd_sys_rw_content_t "/usr/local/centreon(/.*)?"
#restorecon -R /usr/local/centreon/
#semanage fcontext -a -t httpd_sys_rw_content_t "/etc/centreon(/.*)?"
#restorecon -R /etc/centreon
#semanage fcontext -a -t httpd_sys_rw_content_t "/usr/local/nagios/var/spool(/.*)?"
#semanage fcontext -a -t httpd_sys_content_t "/usr/local/nagios/share(/.*)?"
#restorecon -R /usr/local/nagios/
#semanage fcontext -a -t httpd_sys_content_t "/usr/share/php(/.*)?"
#restorecon -R /usr/share/php
#semanage fcontext -a -t httpd_sys_content_t "/usr/share/pear(/.*)?"
#restorecon -R /usr/share/pear

# give the correct permission
chown apache:apache /usr/local/centreon/GPL_LIB/SmartyCache/ -R

# disable selinux
setenforce 0

# Restart some services
service httpd restart
service ndo2db restart
service nagios restart
#
