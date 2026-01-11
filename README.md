# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [X]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [X]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [ ]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [ ]  LRU / CLOCK gibi algoritmaları
- [X]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [X]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo

| Kavram      | Bellek                          | Disk / DB(SQLite) |
| ----------- | ---------------                 | -------------- |
| Adresleme   | Pointer(doğrudan hafıza adresi) | Page + Offset  |
| Hız         | O(1) - doğrudan erişim          | Page IO        |
| PK          | Yok                             | Index anahtarı |
| Veri yapısı | Array / Linked List/ Pointer    | B+Tree         |
| Cache       | CPU cache                       | Buffer Pool   |

---

# Video [Linki](https://www.youtube.com/watch?v=Nw1OvCtKPII&t=2635s) 
Ekran kaydı. 2-3 dk. açık kaynak V.T. kodu üzerinde konunun gösterimi. Video kendini tanıtma ile başlamalıdır (Numara, İsim, Soyisim, Teknik İlgi Alanları). 

---

# Açıklama (Ort. 600 kelime)

## Blok Bazlı Disk Erişimi (block_id + offset)

Sistem programlama açısından diskler blok adreslenebilir cihazlardır ve veritabanları bu gerçeği yansıtacak şekilde tasarlanır. SQLite'da `src/pager.c` dosyası bu blok bazlı erişimi yönetir. Disk üzerindeki veriler 4KB veya 8KB'lik sayfalara (bloklara) bölünür; her sayfa bir `block_id` ile tanımlanır ve sayfa içindeki belirli bir kayda erişmek için `offset` değeri kullanılır. Bu yaklaşım, rasgele erişim maliyetini sabit O(1) seviyesinde tutar. `sqlite3PagerGet()` fonksiyonu, bir sayfa numarası (block_id) verildiğinde ilgili sayfayı bellek önbelleğine getirir, kullanıcılar ise sayfa içindeki offset değerleriyle kayıtlara erişir.

## Satır vs Sayfa Okuması: Veritabanlarının Tercihi

SQLite ve diğer modern veritabanı sistemleri, disk I/O verimliliği için veriyi diskten sayfa bazında okur. Bir kullanıcı sadece bir satır istese bile, veritabanı bu satırın bulunduğu tüm sayfayı belleğe yükler. `btreeGetPage()` gibi fonksiyonlar diskten sayfa okurken, `btreeParseCell()` gibi fonksiyonlar belleğe alınmış bir sayfa içindeki hücreleri ayrıştırarak belirli bir kayda erişim sağlar. Bu strateji, yerellik (locality) prensibine dayanır: bir satıra erişen kullanıcının büyük olasılıkla aynı sayfadaki diğer satırlara da erişeceği varsayılır. Sayfa bazlı okuma, tek tek satır okuma işlemlerine göre daha az disk başvurusu gerektirdiğinden önemli performans kazancı sağlar.

## Disk I/O'nun Minimizasyonu

Disk erişimleri, veritabanı performansındaki en kritik darboğazdır. SQLite, I/O işlemlerini azaltmak için iki ana mekanizma kullanır: **sayfa önbelleği (page cache)** ve **Write-Ahead Logging (WAL)**. Sayfa önbelleği, sık erişilen veri sayfalarını bellekte tutarak tekrarlanan disk okumalarını önler. `src/pager.c` dosyasında önbellek politikaları ve dirty page listesi yönetimi uygulanır; dirty page'ler (bellekte değiştirilmiş ancak diske yazılmamış sayfalar) toplu halde yazılarak yazma işlemlerinin sayısı azaltılır. WAL mekanizması ise `src/wal.c` dosyasında gerçekleştirilir. Geleneksel "rollback journal" yönteminin aksine, WAL'de değişiklikler önce ayrı bir WAL dosyasına eklenir. Veri dosyasına yazma işlemleri daha sonra, checkpoint adı verilen aralıklarla toplu halde gerçekleştirilir. Bu yaklaşım, özellikle çoklu işlem ortamlarında okuma ve yazma işlemlerinin paralelleşmesini sağlayarak performansı önemli ölçüde artırır.

## B+ Tree Veri Yapısının Kullanımı

SQLite, tablo ve indeks verilerini düzenlemek için B-tree (B+ ağacı özelliklerine sahip) veri yapısını kullanır. Bu yapının tüm uygulaması `btree.c` dosyasında bulunur. B-tree'ler, dengeli ağaç yapısı sayesinde arama, ekleme ve silme işlemlerini O(log n) zaman karmaşıklığında gerçekleştirir. Bu ağaç yapısının önemli özellikleri şunlardır: yaprak düğümler birbirine bağlı olduğundan aralık sorguları (range queries) çok verimlidir; her düğüm çok sayıda alt düğüm işaretçisi içerebilir (yüksek fan-out), bu da ağacın derinliğini azaltır; tablo iç düğümleri sadece anahtar ve işaretçi içerir ve veri sadece yapraklarda tutulur, oysa indeks iç düğümleri anahtar ile birlikte veri taşıyabilir. BtCursor yapısı ağaç üzerinde gezinmeyi sağlarken, `balance()` fonksiyonu ağaç dengesini korumak için üç stratejiden birini kullanır: `balance_quick()` hızlı dengeleme için yeni kardeş node oluşturur, `balance_deeper()` ağaca yeni seviye ekler, `balance_nonroot()` komşu node'lar arasında hücreleri yeniden dağıtarak sayfa bölme ve birleştirme işlemlerini gerçekleştirir.

## WAL (Write Ahead Log) İlkesi

WAL, ACID uyumluluğunu sağlarken performansı korumak için kullanılan temel bir veritabanı ilkesidir. SQLite'ın `src/wal.c` dosyası bu mekanizmanın tam uygulamasını içerir. WAL ilkesi, "önce log yaz" prensibine dayanır: bir işlem veriyi kalıcı depolamaya yazmadan önce, yapacağı değişiklikleri log kaydı olarak yazmalıdır. WAL'in çalışma prensibi şöyledir: tüm değişiklikler önce WAL dosyasına eklenir (append-only); veri dosyası değiştirilmeden önce log kayıtları kalıcı hale getirilir; checkpoint işlemi periyodik olarak WAL'deki değişiklikleri ana veri dosyasına uygular; okuma işlemleri, hem veri dosyasını hem de WAL'deki ilgili kayıtları okuyabilir. Bu yaklaşımın avantajları şunlardır: yazarlar okuyucuları engellemez (okuma-yazma paralelliği); değişiklikler sıralı olarak WAL dosyasına eklendiğinden rasgele erişimden daha hızlıdır (daha az disk yazma işlemi); commit sadece WAL'e bir kayıt eklemeyi gerektirir (daha hızlı commit işlemi). SQLite'da `sqlite3WalFrames()` fonksiyonu değişiklik çerçevelerini WAL dosyasına yazarken, `sqlite3WalCheckpoint()` fonksiyonu ihtiyaca göre WAL içeriğini ana veritabanı dosyasına aktarır.

## VT Üzerinde Gösterilen Kaynak Kodları

Açıklama [Linki](https://github.com/sqlite/sqlite/blob/master/src/pager.c) \
Açıklama [Linki](https://github.com/sqlite/sqlite/blob/master/src/btree.c) \
Açıklama [Linki](https://github.com/sqlite/sqlite/blob/master/src/wal.c) \
