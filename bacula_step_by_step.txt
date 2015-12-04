# Bacula Step by Step

## Installazione

yum install yum install mysql mysql-devel mysql-server mysql-libs perl-DBD-MySQL perl-DBI

yum install php-*

wget https://repos.fedorapeople.org/repos/slaanesh/bacula7/epel-bacula7.repo
cp epel-bacula7.repo /etc/yum-repos.d/
yum update

yum install mt* 

yum install bacula*

yum install epel-release

## Configurazione

cd /usr/libexec/bacula/

./create_mysql_database
./make_mysql_tables
./grant_mysql_privileges

Collego bacula a mysql
alternatives --config libbaccats.so

### Impostazione per la libreria automatizzata (Sun StorageTek L40 Tape Library)

vim /usr/libexec/bacula/mtx-changer.conf
offline=1
offline_sleep=60
load_sleep=60
inventory=0
vxa_packetloader=0
debug_log=1


#### Se la libreria conteneva dati e si vuole una situazione pulita

prima di fare RELABEL

##### Script per riavvolgere i nastri e svuotarli
for i in {1..40}
do
echo $i
/usr/libexec/bacula/mtx-changer /dev/sg3 load $i /dev/st0 0 && mt -f /dev/st0 rewind && mt -f /dev/st0 weof && mt -f /dev/st0 rewind && /usr/libexec/bacula/mtx-changer /dev/sg3 unload $i /dev/st0 0
done


#### Label dei volumi (nastri) con Barcodes 

bconsole 
label barcodes

##### Note
If your autochanger has barcode labels, you can label all the Volumes in your autochanger one after another by using the label barcodes command. For each tape in the changer containing a barcode, Bacula will mount the tape and then label it with the same name as the barcode. An appropriate Media record will also be created in the catalog. Any barcode that begins with the same characters as specified on the "CleaningPrefix=xxx" command, will be treated as a cleaning tape, and will not be labeled. For example with:

Please note that Volumes must be pre-labeled to be automatically used in the autochanger during a backup. If you do not have a barcode reader, this is done manually (or via a script).

Pool {
  Name ...
  Cleaning Prefix = "CLN"
}
Any slot containing a barcode of CLNxxxx will be treated as a cleaning tape and will not be mounted.


#### Label del volume per il backup del Catalog
bconsole
label
Automatically selected Catalog: MyCatalog
Using Catalog "MyCatalog"
The defined Storage resources are:
     1: File1
     2: File2
     3: STK
Select Storage resource (1-3): 1
Enter new Volume name: Vol-File-Bkp
Defined Pools:
     1: Default
     2: Nastri-Win
     3: Nastri-LNX
     4: File
     5: Scratch
Select the Pool (1-5): 1

Catalog record for Volume "Vol-File-Bkp", Slot 0  successfully created.

*quit


#### ATTENZIONE In caso di errori (catalog not found)

check the permissions of your bacula-dir.conf.  Your bacula-dir runs as user bacula and MUST have
enough permissions to read its bacula-dir.conf (and also the query.sql).

chmod -R 777 /etc/bacula



## Installazione di Webacula
(http://webacula.sourceforge.net/)

download and unpack webacula7

mv webacula-7.0.0 /usr/share/webacula
cp /usr/share/webacula/install/apache/webacula.conf /etc/httpd/conf.d/

### Installo i componenti necessari a webacula

yum install php-ZendFramework-full.noarch php-ZendFramework-Auth-Adapter-Ldap.noarch php-ZendFramework-Db-Adapter-Mysqli.noarch php-ZendFramework-Db-Adapter-Pdo-Mysql.noarch

### Imposto i permessi per webacula tramite sudo

visudo:
......
Defaults    requiretty

### Per webacula
Defaults:apache    !requiretty
apache ALL = NOPASSWD: /usr/sbin/bconsole, /sbin/stop

### Imposto la password di webacula
./password-to-hash.php ##PASSWORD##

paste it in db.conf

### Ulteriori configurazioni di webacula
cd /usr/share/webacula/
vim application/config.ini

cd /usr/share/webacula/install
./check_system_requirements.php

cd MySql/
./10_make_tables.sh
./20_acl_make_tables.sh

#### In caso estremo (per risolvere i problemi di login):
mysql
use bacula
update webacula_users set pwd='$P$BMAiISUFah71ZDpzy1Vx1emAZU5Rli1' where id = 1000;


## Installazione di Bacula Web 
(http://www.bacula-web.org/)
download bacula-web 7
The latest version Bacula-Web is available through the project site download page
http://www.bacula-web.org/download.html

Go to your Apache root's folder
cd /var/www/html
mkdir -v bacula-web
tar -xzvf bacula-web.tar.gz -C /var/www/html/bacula-web
chown -Rv apache: /var/www/html/bacula-web
chmod -Rv ug+w /var/www/html/bacula-web/application/view/cache

From the installation folder, go to the folder mentioned below
 application/config/

 - Open the file config.php.sample and modify the settings regarding your installation
 - Save this file as config.php in the same folder
 
### Test

Open your web browser and go to the address below

 http://youserver/bacula-web/test.php
 


## RESTART BACULA FULL
for i in `ls /etc/init.d/bacula-*`; do $i $1; done

