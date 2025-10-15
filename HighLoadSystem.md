# High Load System

Çok fazla kullanıcıdan gelen istekleri hızlı, güvenilir ve bozulmadan karşılayabilen sistemlerdir.

1. High Load Sistemlerin Ana Hedefleri

- Scalability (Ölçeklenebilirlik): Trafik arttıkça sistemin çökmesi yerine büyüyerek cevap verebilmesidir. (örn: Cloud provider)

- Availability (Erişilebilirlik): Sistem çalışmaya devam etmeli, kesinti olmamalı.

- Reliability (Güvenilirlik): Veriler kaybolmamalı.

## Realistic Senario:

Diyelim ki kendi URL Shortener sistemimizi yapıyoruz. (bit.ly gibi)

- Günlük 10 milyar yeni link oluşturuyor.
- Günlük 2 milyar tıklama geliyor.

Bu ne demek ? 

```
Günde 2.000.000.000 tıklama
-> Saniye yaklaşık 23.148 istek (avarage)
-> Pik zamanlarda bu 5x' e çıkar. ~115.000 istek/saniye
```

👉 Bu yükü tek bir sunucu kaldıramaz. İşte bu yüzden High Load mimarisi gerekir.

**High Load Sistemlerin olayı sadece güçlü donanım değil, trafiği doğru yönetebilmektir.**

---

# Scaling Yöntemleri

Yüksek trafikli sistemler kurgularken ilk büyük karar şudur: 

**Gücü artırarak mı ölçeceğim, yoksa sayıyı artırarak mı ?**

1. Vertical Scaling (Scale Up) - "Sunucuyu Büyüt"

- Var olan sunucuya daha fazla RAM/CPU/Disk ekliyorsun.
- Kolaydır ama sınırı vardır.

Avantajları:
- Yönetmesi kolaydır. 
- Kodda değişiklik gerekmez.

Dezavantajları: 

- Bir noktadan sonra en büyük sunucuyu bile satın alamazsın.
- Tek bir failure point -> Sunucu çökerse tüm sistem komple gider.

2. Horizontal Scaling (Scale Out) - "Sunucuyu Çoğalt"

- Tek sunucuyu büyütmek yerine aynı sunucudan 10,100,100 tane koyulması.
- İsteklerin Load Balancer ile  dağıtılması.

### Load Balancer (Yük Dengeleyici)

Horizontal Scaling' in kalbi. Müşteriden gelen istekleri alır, sunucuları eşit şekilde dağıtır.

```
Client-> Load Balancer -> Server1 / Server2 / Server3

```

**Popüler Load Balancer' lar :**

| `Yazılım Bazlı` | `Cloud Managed` |
| --- | --- | 
| nginx,HAProxy |  AWS ELB/ ALB, Google LB, Azure LB  |

> Vertical Scaling neden uzun vadede sürdürülebilir değildir?

```
Veritical scaling uzun vadeli sürdürülebilir olmamasının sebebi her sunucu bilgisayarının büyütülmesi için bir limit olmasıdır.
```

> Horizontal Scaling kullanırken tek bir kritik bileşen vardır — o nedir?

```
Horizontal Scaling kullanırken kritik bileşen Load Balancer' dır. 
```

> Load Balancer'dan sonra sistemde stateful olmanın neden sorun olabileceğini tahmin edebiliyor musun?

```
Çünkü istekler her seferinde farklı sunucuya gidebilir ve sunucu içinde tutulan session / kullanıcı bilgisi kaybolur.
```

---

# Stateless vs Stateful Sistemler 

Load Balancer ile sunucular çoğaltıldığında şöyle bir durum oluşur.

```
Client -> Load Balancer -> Server A
Client -> Load Balancer -> Server B
Client -> Load Balancer -> Server C

```

Eğer bir kullanıcıyla ilgili oturum (login bilgisi, sepet, geçici işlem verisi) sunucunun içinde saklanmışsa buna : 

**Stateful Server** (durum tutan sunucu) denir.

Problem şudur:

* Kullanıcı ilk istekte Server A' ya düşer i login olur (session Server A' da tutulur.)
* Sonraki istek Server B' ye düşerse -> kullanıcı yeniden login olmak zorunda kalır.

**İşte bu yüzden High Load sistemlerde 'State' sunucuda saklanmaz. Ortak bir yere alınır.**

Bu sisteme **Stateless Architecture** denir.


> Load Balancer'dan sonra sistemde stateful olmanın neden sorun olabileceğini tahmin edebiliyor musun?

**Çözüm:**

| `State Saklama Yeri` | `Açıklama`                                                                 |
|  ---                 |  ---                                                                       |
| Redis                | En çok kullanılan shared session store                                     |
| Database             | Kullanıcı bilgisi DB' de tutulabilir                                       |
| JWT/ Token           | Durumu client tarafına encode edip geri gönderirsin. (sunucu state tutmaz) |

### 🔁 Bonus: Sticky Session Nedir?

Eğer stateless yapı kurmak istemiyorsan Load Balancer' a "Bu kullanıcı hep aynı sunucuya düşsün." diyebilirsin. Buna **sticky session/session affinity** denir.

**Ama High Load sistemlerde tavsiye edilmez, çünkü sunucu ölürse session' da gider.**

Stateful ve Stateless sorununun somut bir örneği

## Örnek E-ticaret Sitesinde Sepet Sistemi:

Sen bir e-ticaret sitesi yapıyorsun (HepsiBurada/ Trendyol gibi). Kullanıcı ürüne tıklayıp **sepete ekle** diyor. 

- 🧨 Hatalı Stateful Mimarisi:

```
Client -> Load Balancer -> Server A (Sepeti RAM' de tutuyor.)
```

Kullanıcı ikinci ürünü eklediğinde: 

```
Client -> Load Balancer -> Server B (Bu sunucuda sepet boş!)
```

> Kullanıcı "Sepetim kayboldu" diye çıldırır. Çünkü state(durum) sunucuya kilitli kalmıştır.

- ✅ Doğru (Stateless) Mimarisi:

```
Client -> Load Balancer -> Server X -> Sepeti Redis/ DB/ Token' a kayıt eder.
Client -> Load Balancer -> Server Y -> Sepeti Redis' ten okur.
```

**Hangi sunucuya düşerse düşsün, client' ın durumu ortak bir yerde tutulur.**

* Soru : Bu örnekte State'i merkezi tutmak için en mantıklı yer neresi olabilir ? 

1. Database(Sepet tablosu içersinde tutmak)
2. Redis(Hızlı cache depolama)
3. JWT Token(Sepeti client tarafına encode edip cookie içerisinde taşımak)

* Cevap : 
```
Ben 2. cevabı seçerdim redis' te tutmak daha mantıklı. Database' de tutmak da mantıklı olabilir.
Ancak tek mantıksız olan yol token' da saklamak olduğunu düşünüyorum.
Çünkü token süresinin belirli expire süresi var ve bu süre belki 1 hafta sonra dolabilir ve sepet datası kaybolabilir.
```

**Cevap değerlendirmesi:** 

| `Seçenek`          | `Avantaj`                              | `Dezavantaj`                                                            | `Değerlendirme`                             |
| ---                | ---                                    | ---                                                                     | ---                                         |
| DB' de saklamak    | Kalıcı, güvenli                        | Yavaş (her sepet aksiyonunda DB sorgusu döner -> yük bindirir.)         | Mantıklı ama ağır olabilir.                 |
| Redis' te saklamak | Çok hızlı(in-memory) TTL ayarlanabilir.| Redis çökse data kaybolabilir -> **ama backup/replication yapılabilir** | Sepet gibi "geçici state" için ideal tercih |
| Token' da saklamak | Sunucusuz, stateless                   | Token şişer (büyük payload), expire olursa sepet kaybolur.              | Riskli                                      |

**Yani Redis, üçünün arasında en dengeli olanı** -- hem stateless sistemi destekler, hem de yüksek trafikte hızlı çalışır.

---

# Cache & Database Stratejileri (Redis+DB İlişkisi)

Bu derste şu netleştirilecek.

> "Veri nereye yazılmalı? Cache nereye, database nereye? Hangisi önce okunmalı?"

**3 çeşit cache stratejisi vardır.**

1. **Read-Through Cache:** Client DB' ye gitmez. -> Cache' e gider. Cache' de yoksa DB' den çeker. (Kullanıcı verileri, profil bilgisi vb.)
2. **Write-Through Cache:** Yazılan her veri önce cache' e, sonra DB' ye yazılır. (Kalıcı olması gereken data)
3. **Write-Back (Write-Behind):** Veri önce sadece cache'e yazılır, sonra arkadan DB' ye flush edilir. (Sepet/ Like sistemi gibi geçici, toleranslı veriler)

Sistem ve kullanılan cache stratejisi olarak incelersek: 

| `Sistem` | `Cache Stratejisi`|
| --- | --- | 
| User Profile | **Read-Through** (cache varsa ordan oku, yoksa DB' den çek.) |
| Sepet | **Write-Back** (Redis'te tut -> Db' ye periyodik yazabilirsin veya hiç yazmayabilirsin.) |
| Banka Bakiyesi | Asla cache yazarken DB' ye gecikmeli gitmemeli -- **Write-Through** yada direkt DB  |

- Cache, verinin kopyasıdır. Database' in yerine geçmez.
- Redis gibi cache' ler volatile (uçucu) olabilir; tamamen kritik olan veriler için DB garanti noktasıdır.
- Sepet gibi geçici ama hızlı olması gereken şeyleri Redis' te tutmak en makul yoldur.

**Sepet sipariş mantığında 'Sepet = geçici state', 'Sipariş = kalıcı state' olarak değerlendirilebilir.**

---

# Replication & Consistency (Master-Slave/ Primary-Replica/ Strong vs Eventual)

Şimdi sistemleri çoğalttık (scaling), cache' e aldık (Redis). Sıradaki soru: 

> Database' i nasıl ölçeklendiririz?
> Yani High Load altında okumalar ve yazmalar nasıl dağıtılır? 

## Database Replication Nedir ? 

Bir DB' yi birden fazla kopya olarak tutmak demektir.

**Roller:**
1. Primary(Master): Tüm yazmalar (INSERT/UPDATE) buraya yapılır.
2. Replica(Slave/Read Replica): Sadece okumalar yapılır.(SELECT) 

Yazma/ Okuma Ayrımı Böyle Çalışır: 

```
Client(Write) -> Primary DB
Client(Read)  -> Replica DB' ler
```

Böylece yük paylaşılmış olur.

⚠️ Ancak burada bir sorun var : **Replication Lag**

- Primary DB' ye veri yazılır.
- Replica bu veriyi hemen mi alır? -> *Hayır, çoğu zaman gecikmeli gelir.*

Bu yüzden ortaya iki tip tutarlılık modeli çıkar:

1. **Strong Consistency:** Yazılan veri anında tüm DB' lerde görünür. (Örn: Bankacılık Sistemleri) 
2. **Eventual Consistency:** Veriler kısa süreliğine farklı olabilir, ama sonunda eşitlenir. (Örn: Sosyal Medya like sayıları)

### Eventual Consistency' nin Klasik Örnekleri:

Sen instagramda bir postu like' ladın.

- Hemen altta "12502 -> 12503 beğeni" olur.
- Arkadaşın telefonunda hala 12502 görünüyor.

**Bu bir bug değil eventual consistency' dir.**

* Soru: Bir e-ticaret sitesinde "ürün stok sayısı" sence hangi modelle tutulmalı? (Strong mu, Eventual mı?) Neden ? 

* Cevap: Bir e-ticaret sitesinde ürünün stok sayısı strong consistency ile tutulabilir. 
          Çünkü sepete ekleyip sipariş verildiğinde eğer bir gecikme yaşanırsa sonrasında siparişin iptali gibi
          müşterilerde olumsuz durumlara yol açma riski vardır.

* Soru: **Kullanıcı profiline ait gösterilen takipçi sayısı** sence strong consistency mi gerektirir yoksa eventual olabilir mi?

* Cevap: Kullanıcı profiline ait gösterilen takipçi sayısı eventual consistency ile işlem yapılması daha sağlıklı olur. 
         Çünkü 1. cevapdaki gibi çok çok olumsuz durumlara ve sonrasında yaşanacak krizlere sebebiyet verecek bir öncelik olmadığını düşünüyorum.

**Kritik olmayan datalarda Eventual Consistency, kritik datalarda Strong Consistency seçimi yapılır. High Load sistem tasarımı tam da bu dengeyi kurmak demektir.**

---

# Partitioning/ Sharding (Gerçek Ölçeklenme Burada Başlar.)

Cache ekledik. Replication yaptık ama bir noktada tek bir Primary DB bile yetmeyecek.

İşte bu durumda yapılacak şey: 

> Database'i bölmek.

## Partitioning/ Sharding Nedir? 

**Devasa bir tabloyu veya database' iki ya da daha fazla parçaya bölmek.**

Böylece:

```
instread of -> 1 DB with 1M users
we do -> 10 DB each with 10K users
```

### Sharding Yöntemleri:

1. **Range Sharding:** Alfabetik veya ID aralığına göre *(Örn: A-M -> DB1, N-Z -> DB2)*
2. **Hash Sharding:** ID' yi hash' leyip mod alarak *(Örn: user_id % 4 -> 4 farklı shard)*
3. **Geo Sharding:** Lokasyona göre *(Örn: Avrupa ayrı DB, Amerika ayrı DB)*
4. **Feature/Entity Based:** Veri tipine göre *(Örn: dbo.Users tablosu farklı, dbo.Orders farklı DB' de )*

* Örnek: Kullanıcıları Hash ile Shard Et:

```
Shard = user_id % 4 

user_id = 341 => 341 % 4 = 1 -> Database A 
user_id = 762 => 762 % 4 = 2 -> Database B

```

**Her sunucu sadece kendi parçasını bilir -> yük dağıtılır,sorgular hızlanır.**

**Sharding Sorunları:**

1. **Cross-Shard Query:** "Bir kullanıcının 5 siparişi, 3 yorumu ve 2 adresi var." -> hepsi farklı DB' de ise toplamak zorlaşır.
2. **Shard Rebalancing:** "Shard 1 çok doldu,2 boş kaldı" gibi durumlarda yeniden dağıtım gerektirir.
3. **ID Uniqueness:** Farklı shard' lardaki veriler çakışabilir. -> *global_id* sistemi gerekebilir.

* Soru: Sence “Kullanıcı tablosu”nu sharding yapmak mantıklıdır ama “Ürün kategorileri” tablosunu sharding yapmak mantıklı mıdır? Neden?

* Cevap: Bence ürün kategorileri tablosunu sharding yapmak çok değildir. Çünkü kategorilerin mutlak bir sınırı vardır. Bu yüzden datanın bölünmesi gerektiğini düşünmüyorum.

* Soru: Sharding yapıldığında "user_id = 1234" için hangi shard kullanılabileceğini nasıl bilebiliriz. (Senin önerdiğin yöntem)

* Cevap: Çok büyük bir trafik düşündüğümüzde mesela amazon gibi. Önce Geo Sharding yöntemi ile bölgelere ayırıp ardından aynı bölgelerdeki kullanıcılar için de Hash Sharding kullanarak daha verimli bir bölümleme ile sistemin daha stabil ve hızlı çalışabileceğini düşünmüyorum.
Mesela Türkiye' de kayıt olan bir amazon kullanıcısı Geo Sharding yöntemi ile TR sunucularına kayıt olacak ardından TR' de bulunan istanbul, Ankara veya İzmir gibi illerde bulunan sunucularında Hash Sharding kullanarak veri paylaşımı yapmak daha sağlıklı olabilir. 

Cevap Amerika için yine Amazon gibi yüksek trafikli bir sistemden bahsedecek olursak ilk olarak Geo sharing ile kıta bazlı ABD sunucusu olarak böler sonrasında yine Geo sharding ile bu sefer ABD içerisindeki sunucuları eyalet bazlı bölerdim ve eyalatlerde bulunan sunucuları Hash Sharding ile bölmeyi düşnürdüm. Mesela Washington için aktif 30 sunucu varsa bu 30 sunucu da Hash Sharding yöntemi uygulardım. 

> **Eğer data çok küçükse -> bölmek yerine tüm sunuculara kopyalamak daha mantıklıdır. (Replication)**
> **Eğer data büyükse -> parçalamak zorundayız. (Sharding)**






