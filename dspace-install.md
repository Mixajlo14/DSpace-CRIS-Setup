## Instalacija DSpace-CRIS (verzije 5 i 6) na CentOS 7 virtualnoj mašini.
---

* Potrebno je kreirati virtualnu mašinu sa instaliranim CentOS 7 operativnim sistemom. Osnovni sistemski zahtevi za instalaciju DSpace-a se mogu pronaći na sledećem [linku](https://wiki.duraspace.org/display/DSPACE/User+FAQ#UserFAQ-WhatsortofhardwaredoesDSpacerequire?Whataboutsizingtheserver?HowmuchdiskspacedoIneed?).
* Nakon pokretanja virtualne mašine potrebno je izvršini sledeće korake.

### Instalacija osnovnih alata


```
sudo yum install -y wget git
```

### Instalacija Java JDK

```
sudo yum -y install java-1.8.0-openjdk-devel
```

### Instalacija Maven

```
cd /usr/local
wget http://www-eu.apache.org/dist/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.tar.gz
tar xzf apache-maven-3.6.0-bin.tar.gz
ln -s apache-maven-3.6.0 maven 
```
Kreirati fajl `/etc/profile.d/maven.sh` sa sledećim sadržajem
```
#!/bin/bash
export M2_HOME=/usr/local/maven
export PATH=${M2_HOME}/bin:${PATH}
```
Nakon toga je potrebno pokrenuti sledeće komande
```
chmod +x /etc/profile.d/maven.sh
source /etc/profile.d/maven.sh
```

### Instalacija Ant

```
cd /usr/local
wget http://www-eu.apache.org/dist//ant/binaries/apache-ant-1.10.5-bin.tar.gz
tar xzf apache-ant-1.10.5-bin.tar.gz
ln -s apache-ant-1.10.5 ant
```
Kreirati fajl `/etc/profile.d/ant.sh` sa sledećim sadržajem
```
#!/bin/bash
export ANT_HOME=/usr/local/ant
export PATH=$ANT_HOME/bin:$PATH
```
Nakon toga je potrebno pokrenuti sledeće komande
```
chmod +x /etc/profile.d/ant.sh
source /etc/profile.d/ant.sh
```

### Instalacija Tomcat
```
cd /opt
wget https://www-eu.apache.org/dist/tomcat/tomcat-8/v8.5.35/bin/apache-tomcat-8.5.35.tar.gz
tar xzf apache-tomcat-8.5.35.tar.gz
ln -s apache-tomcat-8.5.35 tomcat
sudo chmod +x /opt/tomcat/bin/*.sh
```
Kreirati fajl `/etc/systemd/system/tomcat.service`. 
U fajlu izmeniti `User` i `Group` parametre da odgovaraju parametrima za Tomcat.
```
[Unit]
Description=Tomcat 8.5 servlet container
After=network.target

[Service]
Type=forking

User=root
Group=root

Environment="JAVA_HOME=/usr/lib/jvm/jre"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"

Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```
U fajlu `/opt/tomcat/conf/server.xml` izmeniti podešavanja za `Connector` na portu `8080`.
```
<Connector port="8080" protocol="HTTP/1.1"
           maxThreads="150"
           minSpareThreads="25"
           maxSpareThreads="75"
           enableLookups="false"
           acceptCount="100"
           disableUploadTimeout="true"
           URIEncoding="UTF-8"
           connectionTimeout="20000"
           redirectPort="8443" />
```
Nakon toga izvršiti sledeće komande
```
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl status tomcat
sudo systemctl enable tomcat
```
### Instalacija PostgreSQL
```
sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
sudo yum install -y postgresql11-server postgresql11-contrib
/usr/pgsql-11/bin/postgresql-11-setup initdb
systemctl enable postgresql-11
systemctl start postgresql-11
```
Podešavanje `postgres` i `dspace` korisnika. Za `postgres` korisnika je potrebno izmeniti šifru u komandi (staviti šifru umesto `new_password`), za `dspace` korisnika treba uneti šifru `dspace`.
```
su - postgres
psql -d template1 -c "ALTER USER postgres WITH PASSWORD 'new_password';"
createuser -U postgres -d -A -P dspace
exit
```
U fajlu `/var/lib/pgsql/11/data/pg_hba.conf` je potrebno izmeniti sledeće redove.
```
local  all   all                             md5
host   all   all     127.0.0.1/32            md5
```
Nakon izmene `pg_hba.conf` fajla, potrebno je restartovati `postgresql`.
```
service postgresql-11 restart
```
Sledeći korak je  kreiranje `dspace` baze i `pgcrypto` ekstenzije.
```
su - postgres
createdb -U dspace -E UNICODE dspace
psql --username=postgres dspace -c "CREATE EXTENSION pgcrypto;"
exit
```
Na kraju je potrebno restartovati `postgresql` i proveriti status.
```
service postgresql-11 restart
service postgresql-11 status
```
### Instalacija httpd i podešavanje Firewall
```
sudo yum install -y httpd
systemctl start httpd
systemctl enable httpd
systemctl status httpd
firewall-cmd --zone=public --permanent --add-service=http
firewall-cmd --zone=public --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```
### Instalacija DSpace-CRIS
```
mkdir /opt/crisinstallation
cd /opt/crisinstallation
```
Sledeći korak zavisi od verzije `DSpace-CRIS`.
* **Verzija 5**
```
git clone https://github.com/4Science/DSpace.git --branch dspace-5_x_x-cris dspace-parent/
cd dspace-parent
```
* **Verzija 6RC2**
```
wget https://github.com/4Science/DSpace/archive/dspace-cris-6RC2.tar.gz
tar xzf dspace-cris-6RC2.tar.gz
cd DSpace-dspace-cris-6RC2/
cp dspace/config/local.cfg.EXAMPLE local.cfg
```
Sve potrebne promene u podešavanjima `DSpace` se izvršavaju nakon ovog koraka u fajlu `build.properties` za verziju 5 ili u fajlu `local.cfg` za verziju 6RC2.

Nakon toga se pokreće kreiranje paketa i proces instalacije.
```
mkdir /dspace
mvn package
cd dspace/target/dspace-installer
ant fresh_install
systemctl stop tomcat
cd /dspace/webapps/
cp -R jspui/ solr/ /opt/tomcat/webapps
/dspace/bin/dspace create-administrator
systemctl start tomcat
```
Usled mogućih problema sa kreiranjem paketa, pokretati komandu `mvn package` sve dok se ne dobije poruka `BUILD SUCCESSFUL`. 

Moguće je prekopirati i ostale veb aplikacije iz foldera `/dspace/webapps/`, a ne samo `jspui` i `solr` ali to može dovesti do preopterećenja slabijih mašina.

Na kraju se `DSpace-CRIS` repozitorijumu pristupa iz internet pretraživača odlaskom na sledeći link `ip_addr_vm:8080/jspui` (zameiniti `ip_addr_vm` sa ip adresom virtualne mašine).



