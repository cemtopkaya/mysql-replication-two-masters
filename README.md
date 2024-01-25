# mysql-replication-two-masters

## Adım 1 – Yapılandırmaları/veri klasörlerini hazırlayın 

Bu docker mysql görüntüsünün en iyi yanı, gereksinimlerinize göre kendi Verilerinizi, Günlüğünüzü, yapılandırmanızı ve şifrelerinizi ayarlayabilmenizdir. 
Yani öncelikle döndürmek istediğimiz her düğüm için aşağıdaki gibi dizin yapısını oluşturacağız.

**`~/server#/conf.d`** – özel konfigürasyon dosyalarının montajı.
**`~/server#/backup`** - Herhangi bir
**`~/server#/data`** – Verilerin bağlanacağı/oluşturulacağı yerden başlangıç ​​noktası olacaktır. yine, ana bilgisayara monte edilmiş bir birim olacağından veriler yeniden başlatma sırasında kalıcı olacaktır.
**`~/server#/log`** – Günlük dosyalarını depolamak ve kalıcı kılmak için

Yukarıdaki klasörlerin/dosyaların sahibinin `999:999` olarak ayarlandığından emin olun. 
Her iki düğüm için de 2 yapılandırma dosyası oluşturalım. İçerik aşağıdaki gibi olacak


### `~/sunucu1/conf.d/sunucu1.cnf`
```text
[mysqld]
 server-id = 101
 log_bin = /var/log/mysql/mysql-bin.log
 binlog_do_db = mydata
 bind-address = 0.0.0.0 # make sure to bind it to all IPs, else mysql listens on 127.0.0.1
 character_set_server = utf8
 collation_server = utf8_general_ci
 
[mysql]
 default_character_set = utf8
```

### `~/sunucu1/backup/initdb.sql`
```text
use mysql;
create user 'replicator'@'%' identified by 'repl1234or';
grant replication slave on *.* to 'replicator'@'%';
# do note that the replicator permission cannot be granted on single database.
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
SHOW VARIABLES LIKE 'server_id';
```

### `~/sunucu1/conf.d/sunucu2.cnf`
```text
[mysqld]
server-id = 102 # Remember this is only Integer per official documentation
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = mydata
bind-address = 0.0.0.0 # make sure to bind it to all IPs, else mysql listens on 127.0.0.1
character_set_server = utf8
collation_server = utf8_general_ci
[mysql]
default_character_set = utf8
```

### `~/sunucu2/backup/initdb.sql`
```text
use mysql;
create user 'replicator'@'%' identified by 'repl1234or';
grant replication slave on *.* to 'replicator'@'%';
# do note that the replicator permission cannot be granted on single database.
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
SHOW VARIABLES LIKE 'server_id';
```


## Adım 2 – Yapılandırmalarla Düğümleri başlatın
Yukarıdaki dosyalar oluşturulduktan sonra, yukarıdaki konfigürasyonlara/Veri klasörlerine sahip Konteynerleri oluşturmaya hazırız.

**Düğüm1'i başlat**

```shell
docker run --name mysql1 \
           -e MYSQL_ROOT_PASSWORD=mysql1pass \
           -e MYSQL_DATABASE=mydata -dit \
           -p 33061:3306 \
           -v /opt2/mysql/server1/conf.d:/etc/mysql/mysql.conf.d/ \
           -v /opt2/mysql/server1/data:/var/lib/mysql \
           -v /opt2/mysql/server1/log:/var/log/mysql \
           -v /opt2/mysql/server1/backup:/backup \
           -h  mysql1 \
           mysql
```

**Düğüm2'yi başlat**

```shell
docker run --name mysql2 \
           --link mysql1 \
           -e MYSQL_ROOT_PASSWORD=mysql2pass \
           -e MYSQL_DATABASE=mydata \
           -dit \
           -p 33062:3306 \
           -v /opt2/mysql/server2/conf.d:/etc/mysql/mysql.conf.d/ \
           -v /opt2/mysql/server2/data:/var/lib/mysql \
           -v /opt2/mysql/server2/log:/var/log/mysql \
           -v /opt2/mysql/server2/backup:/backup \
           -h  mysql2 \
           mysql
```

Düğümlere önyükleme yapmaları ve hizmetleri kullanılabilir hale getirmeleri için biraz zaman tanıyın. Ayrıca, "docker çalıştırması" sırasında mysql2 düğümünü mysql1 düğümüne bağladığımızı da unutmayın.

## Adım 3 – Düğüm1'i düğüm2'ye bağlayın (resmi olmayan yol)
Bazı makalelerde/stackoverflow'da okuduğum kadarıyla bağlantının başka bir yolu resmi olarak mümkün değil, ancak mysql1'i mysql2 ile docker0 arayüzü içinde bağlamak için bir geçici çözüm buldum . Önemli olan, docker'ın yalnızca bağlı konteynere ana bilgisayar girişi oluşturmasıdır ve bu, çalışan kapsayıcı içindeki ana bilgisayar dosyasını değiştirirsek başarılabilir. Konteyneriniz yeniden başlatılırsa bu IP'nin liman işçisi tarafından değiştirilebileceğini unutmayın.

Böylece, mysql2 düğümünün çalışma zamanı IP'sini buluyoruz ve ardından mysql1 düğümü içinde, mysql2'nin doğru IP'sini işaret edecek bir ana bilgisayar girişi oluşturuyoruz. 
İşte adımlar:

```shell
# mysql2'nin IP Adresini öğrenin
mysql2ip=$(docker inspect --format '{{ .NetworkSettings.IPAddress }}' mysql2)
 
# Yeni IP'yi mysql1'in ana bilgisayar dosyasına yeni ana bilgisayar girişi olarak ekleyin.
docker exec -i mysql1 sh -c "echo '$mysql2ip mysql2 mysql2' >> /etc/hosts"
 
# Yukarıdaki komutun işe yarayıp yaramadığını kontrol edin
docker exec -i mysql1 sh -c "cat /etc/hosts"

docker exec -ti mysql2 sh -c "ping mysql1"
docker exec -ti mysql1 sh -c "ping mysql2"
```

## Adım 4 – Eşleştirme kullanıcıları oluşturmak, Ana Günlüğü / konumu kontrol etmek ve sunucu_id'sini doğrulamak için Düğümleri başlatın

### Düğüm 1 
Düğüm1'e bağlanın ve `/backup/initdb.sql` dosyasını çalıştırın

```shell
/opt2/mysql$ docker exec -ti mysql1 sh -c "mysql -uroot -p"
Enter password:
mysql> source /backup/initdb.sql
Database changed
Query OK, 0 rows affected (0.00 sec)
Query OK, 0 rows affected (0.00 sec)
Query OK, 0 rows affected (0.00<span                 data-mce-type="bookmark"                id="mce_SELREST_start"              data-mce-style="overflow:hidden;line-height:0"              style="overflow:hidden;line-height:0"           >﻿</span> sec)
+------------------+----------+--------------+------------------+-------------------+
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 | 154 | mydata | | |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id | 101 |
+---------------+-------+
1 row in set (0.01 sec)
```

### Düğüm 2
Düğüm2'ye bağlanın ve `/backup/initdb.sql` dosyasını çalıştırın

```shell
/opt2/mysql$ docker exec -ti mysql2 sh -c "mysql -uroot -p"
Enter password:
mysql> source /backup/initdb.sql
Database changed
Query OK, 0 rows affected (0.00 sec)
Query OK, 0 rows affected (0.00 sec)
Query OK, 0 rows affected (0.00 sec)
+------------------+----------+--------------+------------------+-------------------+
| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 | 154 | mydata | | |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id | 102 |
+---------------+-------+
1 row in set (0.01 sec)
```

Artık her iki Düğüm de Dosya adı ve konumu açısından çok benzer görünüyor. Ayrıca görüntülenen sunucu kimliğinin benzersiz olması gerektiğini unutmayın; bu nedenle `sunucu1.cnf` ve `sunucu2.cnf`'nin farklı sunucu kimliği değişkenleri vardı.


## Adım 5 – Her iki düğüm için Eşleştirmenin kaynağını ayarlayın 
### Düğüm 1 

```shell
/opt2/mysql$ docker exec -ti mysql1 sh -c "mysql -uroot -p"
Enter password:
mysql> stop slave;
mysql> CHANGE MASTER TO MASTER_HOST = 'mysql2', MASTER_USER = 'replicator',
    -> MASTER_PASSWORD = 'repl1234or', MASTER_LOG_FILE = 'mysql-bin.000003',
    -> MASTER_LOG_POS = 154;
mysql> start slave;
mysql> show slave status\g
```

### Düğüm 2

```shell
/opt2/mysql$ docker exec -ti mysql2 sh -c "mysql -uroot -p"
Enter password:
mysql> stop slave;
mysql> CHANGE MASTER TO MASTER_HOST = 'mysql1', MASTER_USER = 'replicator',
    -> MASTER_PASSWORD = 'repl1234or', MASTER_LOG_FILE = 'mysql-bin.000003',
    -> MASTER_LOG_POS = 154;
mysql> start slave;
mysql> show slave status\g
```


## Adım 6 – Master - Master Eşleştirmenin Test Edilmesi
Çoğaltmayı test edeceğiz. Bunu yapmak için, Düğüm 1'deki mydata  veritabanımızda bir tablo oluşturacağız ve Düğüm 2'yi yansıtıp yansıtmadığını kontrol edeceğiz. Daha sonra tabloyu Düğüm2'den kaldıracağız ve ideal olarak Düğüm1 artık Düğüm1'de görünmemelidir.

Bir tablo oluşturalım

```sql
use mydata;
create table students ('id' int,  'name' varchar(20));
```

Şimdi tablomuzun var olup olmadığını görmek için Düğüm2'yi kontrol edeceğiz.

```sql
show tables in mydata;
```

Aşağıdakine benzer bir çıktı görmeliyiz:

```sql
+-------------------+
| Tables_in_mydata |
+-------------------+
| students |
+-------------------+
1 row in set (0.00 sec)
```

Yapılacak son test tablomuzu node2'den silmek. Ayrıca Düğüm1'den de silinmelidir. Bunu node2 mysql istemine aşağıdakini girerek yapabiliriz:

```sql
DROP TABLE students;
```

Bunu doğrulamak için düğüm1'de "tabloları göster" komutunu çalıştırmak hiçbir tablo göstermeyecektir:

```sql
Empty set (0.00 sec)
```
Bu kadar!
