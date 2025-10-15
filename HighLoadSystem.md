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

* Yazılım Bazlı

- nginx
- HAProxy

* Cloud Managed

- AWS ELB/ ALB
- Google LB
- Azure LB


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
Ben 2. cevabı seçerdim redis' te tutmak daha mantıklı. Database' de tutmak da mantıklı olabilir ancak tek mantıksız olan yol token' da saklamak olduğunu düşünüyorum
çünkü token süresinin belirli expire süresi var ve bu süre belki 1 hafta sonra dolabilir ve sepet datası kaybolabilir.
```

**Cevap değerlendirmesi:** 

| `Seçenek`          | `Avantaj`                              | `Dezavantaj`                                                            | `Değerlendirme`                             |
| ---                | ---                                    | ---                                                                     | ---                                         |
| DB' de saklamak    | Kalıcı, güvenli                        | Yavaş (her sepet aksiyonunda DB sorgusu döner -> yük bindirir.)         | Mantıklı ama ağır olabilir.                 |
| Redis' te saklamak | Çok hızlı(in-memory) TTL ayarlanabilir.| Redis çökse data kaybolabilir -> **ama backup/replication yapılabilir** | Sepet gibi "geçici state" için ideal tercih |
| Token' da saklamak | Sunucusuz, stateless                   | Token şişer (büyük payload), expire olursa sepet kaybolur.              | Riskli                                      |

**Yani Redis, üçünün arasında en dengeli olanı** -- hem stateless sistemi destekler, hem de yüksek trafikte hızlı çalışır.

---

