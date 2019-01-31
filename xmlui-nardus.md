## Instalacija XMLUI i preuzimanje podataka sa NaRDuS-a
---

* Potrebno je instalirati **xmlui** veb aplikaciju kako bi bio omogućen **OAI-PMH**
* **NaRDuS** predstavlja Nacionalni repozitorijum doktorskih disertacija

### Instalacija XMLUI

Prvi problem koji se pojavljuje kod `xmlui` aplikacije je taj što nedostaje `hibernate dependency`. Potrebno je izmeniti fajl `/opt/crisinstallation/dspace-parent/dspace-xmlui/pom.xml`, u bloku `dependencies` dodati sledeći kod.
```
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.13.Final</version>
</dependency>
```
Drugi mogući problem predstavlja nedostatak `dspace-resourcesync` modula. Rešenje tog problema se nalazi na sledećem [linku](https://github.com/DSpace/DSpaceResourceSync).

Koraci u rešavanju ovog problema:
1) Instalacija `ResourceSyncJava` modula
```
git clone https://github.com/4Science/ResourceSyncJava
cd ResourceSyncJava
mvn clean package install
```
2) Instalacija `DSpaceResourceSync` modula
```
git clone https://github.com/DSpace/DSpaceResourceSync
cd DSpaceResourceSync
```
Potrebno je saznati tačnu verziju prethodno instaliranog `DSpace-CRIS` repozitorijuma
```
/dspace/bin/dspace version
```
U fajlu `pom.xml` red `<dspace.version>5.9-SNAPSHOT</dspace.version>` treba zameniti `5.9-SNAPSHOT` sa verzijom instaliranog `DSpace-CRIS`.

Takođe je potrebno izvršiti izmenu u 78. lini fajla `src/main/java/org/dspace/resourcesync/DSpaceResourceList.java`. Liniju koda
```
BrowseEngine be = new BrowseEngine(context);
```
Potrebno je zameniti sledećom linijom
```
BrowseEngine be = new BrowseEngine(context,null);
```
Nakon toga je moguća instalacija ovog modula.
```
mvn clean package install
```
Dodavanje ovog modula u `dependencies` blok  `/opt/crisinstallation/dspace-parent/dspace-xmlui/pom.xml` fajla.
```
<dependency>
    <groupId>org.dspace</groupId>
    <artifactId>dspace-resourcesync</artifactId>
    <version>1.1-SNAPSHOT</version>
    <type>jar</type>
    <classifier>classes</classifier>
</dependency>
```
3) Build i instalacija **xmlui**
```
cd /opt/crisinstallation/dspace-parent
mvn clean package
cd dspace/target/dspace-installer
ant fresh_install
cd /dspace/webapps/
cp -R /dspace/webapps/xmlui/ /opt/tomcat/webapps/
systemctl restart tomcat
```
Nakon ovog koraka trebalo bi da može da se pristupi **xmlui** veb aplikaciji na sledećem linku `ip_addr_vm:8080/xmlui` (zameiniti `ip_addr_vm` sa ip adresom virtualne mašine).

### Harvesting podataka sa NaRDuSa 

Koraci prilikom prikupljanja podataka sa `NaRDuS-a`:
* Ulogovati se kao admin korisnik
* Kreiranje `Community` i `Collection`
* Na kreiranoj kolekciji se ode na opciju `Edit Collection`
* Nakon toga na tab `Content Source`, klikom na dugme `Change Settings` moguće je uneti podešavanja za `NaRDus`.
1) `Content source` izabrati opciju `This collection harvests its content from an external source` 
2) Za `OAI Provider` staviti `http://nardus.mpn.gov.rs/oai/request`
3) Moguće je podesiti i da se prilikom preuzimanja preuzmu samo određeni podaci popunjavanjem `OAI Set id` opcije. Sve `Set id` opcije moguće je naći na sledećem [linku](http://nardus.mpn.gov.rs/oai/request?verb=ListSets). `Set id` za Univerzitet u Kragujevcu je `com_123456789_14`.
4) Poslednja opcija koju je moguće podesiti je `Content being harvested`, tj. mogućnost biranja koji će sadržaj biti pokupljen, samo metapodaci ili fajlovi priloženi za svaku disertaciju.
* Kad se snime podešavanja preuzimanje podataka se pokreće klikom na dugme `Import now`.

### Prikazivanje prikupljnih podataka iz NaRDuS-a 

* Problem sa prikazivanjem datuma i autora kod publikacija na `jspui`. Da bi se ovaj problem rešio potrebno je u fajlu `/dspace/config/dspace.cfg` izmeniti red
```
webui.itemlist.columns = dc.date.issued(date),dc.title(title),dc.contributor.author(itemcrisref)
```
sa redom u kojem se podaci o datumu i autorima čitaju iz polja (`dc.date` i `dc.creator`)
```
webui.itemlist.columns = dc.date(date),dc.title(title),dc.creator(itemcrisref)
```

* Prilikom povlačenja svih disertacija moguć je problem sa fajlovima koji u svom nazivu imaju sledeća slova (š, đ, č, ž, ć). Ovaj problem se rešava dodavanjem sledeće linije koda u fajlu `/opt/crisinstallation/dspace-parent/dspace-api/src/main/java/org/dspace/content/crosswalk/OREIngestionCrosswalk.java`, neposredno pre  `ARurl = new URL(processedURL);` linije.
```
processedURL = processedURL.replaceAll("(%C4%87|%C4%8|%C5%A1|%C5%BD|%c5%a1|%c5%bd|%c4%87|%c4%8)","+");
```
Nakon ovog koraka potrebno je ponovo pokrenuti build proces `dspace` i ponovo prekopirati sve veb aplikacije (`solr`,`jspui` i `xmlui`) u `tomcat` folder za veb aplikacije.

* Ukoliko se prilikom preuzimanja disertacija u logovima pojavi neka od sledećih grešaka: `java.lang.OutOfMemoryError: Java heap space` ili `java.lang.OutOfMemoryError: GC overhead limit exceeded`, potrebno je izmeniti fajl `/etc/systemd/system/tomcat.service`. Potrebno je izmeniti vrednosti parametara `JAVA_OPTS` i `CATALINA_OPTS`.
```
Environment="JAVA_OPTS=-Xms2048M -Xmx2048M -Djava.security.egd=file:///dev/urandom -XX:-UseGCOverheadLimit"
Environment="CATALINA_OPTS=-Xms2048M -Xmx2048M -server -XX:+UseParallelGC"
```
Nakon izmena na fajlu `tomcat.service` potrebno je restartovati `tomcat`.
```
systemctl daemon-reload
service tomcat restart
```
