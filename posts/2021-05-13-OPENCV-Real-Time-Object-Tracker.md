# **OPENCV İLE GERÇEK ZAMANLI NESNE TAKİBİ**

# İÇİNDEKİLER

| BEYAN                                                       | iii |
|-------------------------------------------------------------|-----|
| ÖZET                                                        | iv  |
| SİMGE VE KISALTMALAR LİSTESİ                                | v   |
| ŞEKİLLER LİSTESİ                                            | vi  |
| TABLOLAR LİSTESİ                                            | vii |
| BÖLÜM 1. GİRİŞ                                              | 1   |
| 1.1. Problemin Tanımı                                       | 1   |
| 1.2. Çalışmanın Amacı                                       | 2   |
| 1.3. Çalışmanın Kapsamı                                     | 2   |
| 1.4. Literatür Özeti                                        | 2   |
| 1.5. Çalışmanın Gerçekçi Kısıtlar Açısından Analizi         | 6   |
| BÖLÜM 2. MATEMATİKSEL YÖNTEM ve TASARIM                     | 7   |
| 2.1. Yöntem Hakkında Genel Bilgi                            | 7   |
| 2.2. Tasarım (Yöntemin Problemin Çözümünde Kullanımı)       | 11  |
| BÖLÜM 3. UYGULAMA ÇALIŞMALARI                               | 12  |
| 3.1. Uygulamada Kullanılan Araç ve Gereçler                 | 12  |
| 3.2. Uygulamanın Gerçekleştirilme Aşamaları                 | 12  |
| 3.3. Uygulama Sonuçları ve Yorumlanması                     | 17  |
| BÖLÜM 4. SONUÇLAR ve ÖNERİLER                               | 18  |
| 4.1. Sonuçlar                                               | 18  |
| 4.2. Öneriler                                               | 18  |
| 4.3. Sonuçların Sağlık, Çevre ve Güvenlik Açısından Analizi | 18  |
| BÖLÜM 5. KAYNAKLAR                                          | 19  |
| BÖLÜM 6. ÖZ GEÇMİŞ                                          | 21  |
| BÖLÜM 7. EKLER                                              | 22  |
| EK A. Deney Tasarımı Açıklamaları                           | 22  |
| EK B. IEEE Etik Kurallar Onay Formu                         | 23  |
| EK C Seminer Fuar Konferans Katılım Belgesi                 | 24  |

| EK D. Çalışma Yönetimi                 | 25 |
|----------------------------------------|----|
| EK E. Çalışma ile İlişkili Diğer Ekler | 27 |
| EK E.1. Python Yazılım Kodları         | 27 |
| EK E.2. Arduino Yazılım Kodları        | 31 |

# **ÖZET**

Görüntü işleme dijital bir görüntü içindeki önemli bilgilerin çıkartılması ve işlenmesidir. Görüntülerde bulunana nesneleri sırasıyla nesnelerin tespiti, tanımlanması, sınıflandırılması ve takibi işlemlerini yapacak birçok yöntem geliştirilmiştir. Her bir basamaktaki başarı oranı bir sonraki basamaktaki başarı oranını etkilemektedir. Takip edilecek nesnenin değişken bir alan içinde bulunması bu yöntemleri zorlaştıran temel problemdir. Bu problemi çözmek ve nesnenin başarılı bir şekilde takibini yapabilmek için birçok farklı yöntem geliştirilmiştir. Bu çalışmada kameradan alınan gerçek zamanlı görüntüyü işlemek için Python ve OpenCV kullanılmıştır. Nesnenin ya da kameranın hareket etmesi durumunda hedef nesnenin görüntü çerçevesinden çıkmasını engellemek için Arduino ile kameraya iki eksende bağlı motorları sürerek hedef seçilen nesnenin takibini yapılmıştır. Uygulama sonucunda hedef seçme ve hedefin takip işlemi başarılı olmuştur. Yedi algoritmanında kullanım alanları farklı olduğu için birbirine göre başarı oranları daha farklıdır. Boosting daha yavaş ve zaman zaman hedefi kaybetmektedir. MIL, Boosting'e göre daha iyi doğruluk oranına sahip. KCF, ilk iki algoritmaya göre daha iyi fakat tıkanma durumunda nesneyi takip edemiyor. Median Flow hızlı nesneleri takip ederken bazen nesneyi kaybedebiliyor. TLD tıkanma durumunda en iyi sonucu veriyor. Mosse yüksek hata oranına sahip. CSRT yüksek doğruluk oranına sahip.

# **SİMGE VE KISALTMALAR LİSTESİ**

**İHA** : İnsansız Hava Aracı

**MHT** : Çoklu Hipotez Takibi

**LBP** : Local Binary Pattern

**CSRT** : Kanal ve Mekansal Güvenilirlik Takipçisi

**HOG** : Yönelimli Gradyanların Histogramı

**KCF** : Çekirdekleştirilmiş İlinti Süzgeci

**MIL** : Çoklu Örnek Öğrenme

**TLD** : Takip, Öğrenme ve Tespit

**NCC** : Düzgelenmiş Çapraz İlinti

**OpenCV** : Açık Kaynak Kodlu Görüntü İşleme Kütüphanesi

**FPS** : Saniyedeki Kare

# **ŞEKİLLER LİSTESİ**

| Şekil 1.1. | Nesne takip uygulamalarının<br>genel<br>blok<br>diyagramı<br> | 1  |
|------------|---------------------------------------------------------------|----|
| Şekil 1.2. | Nesne takip algoritmalarının genel diyagramı                  | 2  |
| Şekil 1.3. | Nesne tespit yöntemlerinin genel başlıkları                   | 3  |
| Şekil 1.4. | Nesne tespiti için arka plan çıkarma yöntemi                  | 4  |
| Şekil 3.1. | Algoritma seçme menüsü                                        | 12 |
| Şekil 3.2. | Hedef nesne seçme ekranı                                      | 13 |
| Şekil 3.3. | Hedefin takip edilmesi<br>                                    | 14 |
| Şekil 3.4. | Arduino ile motor kontrol<br>                                 | 14 |
| Şekil 3.5. | Uygulamanın python<br>blok diyagramı<br>                      | 15 |
| Şekil 3.6. | Uygulamanın arduino blok diyagramı<br>                        | 16 |

# **TABLOLAR LİSTESİ**

| Tablo         | Nesne sınıflandırma çalışmalarında kullanılan öznitelikler       |
|---------------|------------------------------------------------------------------|
| 1.1.          | 4                                                                |
| Tablo<br>1.2. | Sınıflandırma yöntemlerinin kullandığı özniteliklere göre        |
|               | sınıflandırılması<br>5                                           |
| Tablo         | Literatürde nesne takibi için yaygın olarak kullanılan yöntemler |
| 1.3.          | 6                                                                |

# **BÖLÜM 1. GİRİŞ**

Çalışmanın giriş bölümünde, ele alınan problemin tanımlanması, amaç ve hedeflerin belirtilmesi, çalışmanın sınırlarının ve kapsamının belirlenmesi, literatürde yer alan kaynakların araştırılarak önceden yapılmış çalışmalar hakkında bilgiler sunulması hedeflenmiştir.

### **1.1. Problemin Tanımı**

Nesne tespit ve nesne tanıma uzun yıllardır üzerinden çalışılan konulardır. Nesne tespiti ve nesne tanıma için farklı algoritmalar geliştirilmiştir. Son yıllarda grafik işleme birimlerindeki gelişmeler ve derin öğrenme algoritmaları sayesinde daha fazla doğruluk oranına sahip yöntemler geliştirildi. Nesne takip genel olarak dört farklı aşamada ele alınabilir. Bunlar ön işlem, nesne tespiti, nesne sınıflandırması ve nesne takibidir. Bu dört aşamanın her biri bir sonraki aşamanın başarı oranını etkilemektedir [1]. Günümüzde nesne tespit ve takibi uygulamalarını askeri alanda iha, casus uçak, uydu; sosyal hayatta akıllı telefonların yüz tanıması, araç plaka tanıma sistemleri gibi birçok kullanım alanına sahiptir.

![](_page_10_Figure_4.jpeg)

Şekil 1.1. Nesne takip uygulamalarının genel blok diyagramı

# **1.2. Çalışmanın Amacı**

Bu çalışmayı yapmamdaki amacım Python ve OpenCV ile kameradan aldığımız görüntüyü işleyerek, bir hedef belirlemek ve bu hedefi takip etmektir. Cisimlerin kameranın kadrajından çıkması durumunda Python ile Arduino'nun haberleşmesini sağlayarak kamerayı hareket ettirerek takibini sürdürmeye devam etmektir. Bu sayede kamera veya cismin hareket etmesi durumunda bile cismi takip etmeye devam edebilmektir.

### **1.3. Çalışmanın Kapsamı**

Görüntü işleme yöntemleri ile görüntü bulunan nesneler sırasıyla nesnelerin tanımlanması, tespiti, sınıflandırılması ve takibi için birçok yol vardır. Her bir basamağın kendisine ait algoritmaları ve formülü vardır. Her bir basamaktaki başarı bir sonraki basamağı etkilemektedir. İşlem yapılacak görüntüdeki çözünürlük, ışık durumu, hareketli nesne sayısı, görüntüyü kayda alan eleman gibi etkenler çalışmanın başarısını etkilemektedir. Askeri uygulamalarda kullanılan elektro-optik algılayıcı, son yıllarda yapay zeka tabanlı derin öğrenme algoritmaları ile güçlendirilerek hareketli ve sabit hedeflerin belirlenmesinde ve takibinde hem daha hızlı hem de daha kesin olarak çalışmaktadır. Bu çalışmada Python ve OpenCV ile kameradan alınan gerçek zamanlı görüntüyü işleyerek, nesnenin ya da kameranın hareket etmesi durumunda hedef nesnenin görüntü çerçevesinden çıkmasını engellemek için Arduino ile kameraya iki eksende bağlı motorları sürerek hedef seçilen nesnenin takibini yapmaktır.

### **1.4. Literatür Özeti**

Literatür çalışmalarını incelersek nesne takibi yapan çalışmalar 4 farklı aşamadan geçmektedir. Bu aşamalar Şekil 1'de gibidir.

![](_page_11_Figure_6.jpeg)

Şekil 1.2. Nesne takip algoritmalarının genel diyagramı [3]

Görüntü işlemede işleme başarısını arttırmak ve işlemci yükünü azaltmak için görüntü önişlemden geçer. Bu işlemi gerçekleştirecek yöntemler, görüntüdeki gürültüyü yok etme, nesne kesimleme, renk temelli filtreleme gibi işlemleri içerir [2].

Nesne tespiti için literatürde birçok yol vardır. Bunları 3 ana grupta gruplarsak bunlar Şekil 1'deki gibi olur.

![](_page_12_Figure_2.jpeg)

Şekil 1.3. Nesne tespit yöntemlerinin genel başlıkları [3]

Bir videodaki veya görüntüdeki nesnenin takibi tespiti için 2 temel bilgi gerekir. Bunlar; nesnenin rengi, dokusu, şekli gibi özellikleri olan nesnenin özniteliği ve hareket bilgisidir.

Nesne tespiti için kullanılan yöntemlerden birisi videoda arka arkaya gelen resimler arasındaki geçici değişiklikleri bulma yöntemidir (temporal frame differencing). Kısaca özetlersek bir videodaki arka arkaya gelen iki resmin birbirinden çıkartılmasıdır. Bu sayede hareket eden cismin farkına varabiliriz. Fakat bu yöntemde ışık değişikliğinin ve gürültünün artması durumunda başarı oranı düşecektir [3].

Bir diğer yöntem ise belirli nesnenin olduğu görüntüden sadece arka planı çıkarmaktır. Sonrasında görüntü belirli bir eşlik değerinden geçirilir ve istenmeyen nesneler silinir, görüntüdeki gürültüleri azaltmak için morfolojik işlemleri uyguladıktan sonra istenilen nesne ortaya çıkmış olur [3].

![](_page_13_Picture_0.jpeg)

Şekil 1.4. Nesne tespiti için arka plan çıkarma yöntemi [4]

Nesne tespiti için kullanılan bir diğer yöntem ise Gaussian Ortalama (Gaussian Average) yöntemidir. Mevcut görselden arka planı çıkarmak için her bir pikselin olasılık yoğunluk dağılımları bulunarak nesnenin arka plana dahil olup olmadığı bulunur. Arka plana dahilse arka planla ile birlikte oda çıkartılır ve böylece elimizde sadece nesne kalır.

Nesne tespiti yapıldıktan sonra elde kalan nesnenin sınıflandırılması gerekir. Sınıflandırma işlemini cismin özniteliği bakarak yapılır. Nesne takibi için kullanılan en yaygın öznitelikler renk, kenar, doku, derinlik, süper piksel, hareket ve optik akıştır [4].

Tablo 1.1. Nesne sınıflandırma çalışmalarında kullanılan öznitelikler [3]

| Öznitelik Tipi | Kullanım Biçimi | Avantajı                     | Dezavantajı            |
|----------------|-----------------|------------------------------|------------------------|
| Renk           | Renk Histogramı | Etkili                       | Işık değişikliğine     |
|                |                 |                              | hassas                 |
| Eğim           | Hog, Shift      | İnsan takibi için kullanışlı | Gürültüye karşı hassas |
| Doku           | Lbp, Shift      | Yüksek doğruluk oranı        | İşlem zamanı yüksek    |
| Hareket        | Hog, Shift      | Nesne takip sırasında oluşan | İşlem zamanı yüksek    |
|                |                 | kesilmeler                   |                        |
| Zamansal       | Süper Piksel,   | Nesne takip sırasında oluşan | İşlem zamanı yüksek,   |
| Değişimler     | Süper Voxel     | kesilmeler ve nesnenin durum | Yüksek öznitelik       |
|                |                 | çeşitliliği                  | boyutu                 |

Nesne sınıflandırma çalışmalara 4 temel gruba ayrılmıştır. Bunlar hareket tabanlı, doku tabanlı, şekil tabanlı ve renk tabanlı algoritmalardır [5].

Hareket tabanlı sınıflandırma yönteminde, cismin hareket bilgilerini kullanarak bir sonraki hareketinde cismin nerede olacağı üzerine tahmin yürütür bu sayede cismin her hareketinde cismin aranacağı alan azaltılmış olur. Doku tabanlı sınıflandırmalarda cismin yapısına ait öznitelikler çıkartılır ve kullanılır. Renk tabanlı sınıflandırmalar işlem zamanını çok azaltır ve aynı zamanda en yaygın kullanıma sahiptir. Şekil tabanlı sınıflandırmalar ise cisimleri geometrik şekillerine göre sınıflandırır [3].

Tablo 1.2. Sınıflandırma yöntemlerinin kullandığı özniteliklere göre sınıflandırılması [3]

| Nesne Sınıflandırma | Hesaplama Zamanı | Başarı Oranı | Yorum                                 |
|---------------------|------------------|--------------|---------------------------------------|
| Yöntemi             |                  |              |                                       |
| Şekil tabanlı       | Düşük            | Orta-Yüksek  | Model uydurma algoritmaları           |
|                     |                  |              | barındırır. Değişen şekle sahip olan  |
|                     |                  |              | nesnelerde zayıflar.                  |
| Hareket tabanlı     | Yüksek           | Orta         | Hareketli nesneleri tanımayı amaçlar. |
| Renk tabanlı        | Yüksek           | Yüksek       | Yüksek başarılar yanında işlem        |
|                     |                  |              | zamanı artmaktadır.                   |
| Doku tabanlı        | Yüksek           | Yüksek       | Öznitelik çıkartmada işlem maliyeti   |
|                     |                  |              | yükselmekte.                          |

Görüntü işleme ile nesne tespiti ve takibi yapmanın 2 yolu vardır: sabit kamera ile, hareketli kamera ile. Sabit kameralı sistemlerde nesne tespitindeki problemler genellikle ışıktaki değişimler, kameradaki titreşimler, gürültüler ve ağaç yapraklı gibi hızlı değişen artalana ait nesnelerden kaynaklanır. Hareketli kamerada ise sabit kameradaki problemlere ek olarak kameranın hareketi sebebiyle artalanın kayması problemi ile karşılaşılır [6].

Nesne takibi yöntemleri genel olarak 3 gruba ayrılır. Bunlar nokta tabanlı, çekirdek tabanlı ve siluet tabanlı takip etmedir.

Nokta tabanlı takipte her bir nesne tek bir nokta ilişkilendirilir. Bu yöntem ile güncel resimde takip edilecek nesne seçildikten sonra, nesnenin diğer noktalarla olan veri bağı ilişkisi kurulur. Sonraki resimde tespit edilen noktalar arasında kurmuş olduğumuz veri bağı benzerliği hesaplanır [7].

Çekirdek tabanlı takip yönteminde nesnenin şekli ve görüntüsüne dikkat edilir. Bu yöntemde nesnenin ardışık görüntülerinde çekirdeğinin hareketi olasılık yoğunluk dağılımı hesaplanır ve bu hesaplamaya bakılarak ardışık görüntülerde izlenir. Takip edilecek nesne bir geometrik şekil içine alınır [7-8].

Siluet tabanlı takip etmede basit geometrik şekillerle belirtilemeyen el, ayak, yüz gibi karmaşık şekilleri takip etmek için kullanılır. Önceki resimler kullanılarak elde edilen nesne modelini güncel resimlerde renk histogramını, nesne kenarını ve sınır şeklini kullanarak bulmaya çalışır [7].

Tablo 1.3. Literatürde nesne takibi için yaygın olarak kullanılan yöntemler [3]

| Nesne Takip Yöntemi |                        | Kullanımı                   | İşlem Zamanı | Başarı Oranı |  |  |
|---------------------|------------------------|-----------------------------|--------------|--------------|--|--|
| Kalman filtresi     |                        | Kalman filtre algoritmaları | Düşük-Orta   | Orta         |  |  |
| Nokta tabanlı       | Parçacık filtresi      | Öz yinelemeli bayes         | Orta-Yüksek  | Orta         |  |  |
|                     |                        | filtresi                    |              |              |  |  |
|                     | Çoklu hypothesis takip | MHT algoritması             | Düşük        | Düşük-Orta   |  |  |
|                     | Temel şablon uydurma   | Video içinde uyum arama     | Düşük-Orta   | Düşük        |  |  |
|                     |                        | algoritmaları               |              |              |  |  |
| Çekirdek            | Destek vektör makinesi | Görüntü içindeki            | Orta         | Orta         |  |  |
| tabanlı             | (dvm)                  | piksellerin nesne ve arka   |              |              |  |  |
|                     |                        | plan olarak                 |              |              |  |  |
|                     |                        | sınıflandırılması           |              |              |  |  |
|                     | Uydurma tabanlı        | Şekil içindeki piksel       | Orta         | Orta-Yüksek  |  |  |
|                     | sınıflandırma          | yoğunluklarına bakılır      |              |              |  |  |
| Siluet              | Kenar kesiştirme       | Gradient descent            | Orta         | Orta-Yüksek  |  |  |
|                     |                        | algoritmaları               |              |              |  |  |
| tabanlı             | Şekil uydurma          | Hought dönüşümü             | Yüksek       | Yüksek       |  |  |

### **1.5. Çalışmanın Gerçekçi Kısıtlar Açısından Analizi**

Çalışmada ekonomik maliyet çıkaracak nesneler bir adet kamera, bir adet arduino ve iki adet motordur. Burada maliyeti en fazla olan nesne kameradır. Kameranın özellikleri olan optik lens kalitesi, video çözünürlüğü ve saniyede tazeleme oranına göre maliyeti değişmektedir. Bu özelliklerinin daha iyi olması sonucu görüntü girdisinin kalitesi artar dolayısıyla çerçevede gözüken cisimlerin tespiti, sınıflandırılması ve takibi daha iyi bir şekilde yapılır. Bu çalışmada kullanılan malzemelerin toplam maliyeti beş yüz Türk lirasıdır.

Çalışmanın üretilebilirliği gayet kolaydır. Üretim aşaması motorların kameraya montajı, motorların arduino ile bağlantısı, kamera ve arduino'nun bilgisayara bağlanmasından oluşur. Çalışmanın kullanıcıya ya da etrafa sağlık bakımından herhangi bir zararı bulunmamaktadır.

# **BÖLÜM 2. MATEMATİKSEL YÖNTEM ve TASARIM**

Bu bölümde çalışmada kullanılan yöntem ya da yöntemlere ait bilgilendirme ve yöntemin çalışmada kullanımına ilişkin literatür araştırmalarına yer verilmiştir. Yöntem veya yöntemlerin problemin çözümünde kullanımı ve tasarım aşamaları da bu bölümde verilmektedir.

# **2.1. Yöntem Hakkında Genel Bilgi**

Görsel denetim ve nesne takibi yapmak için birçok yol vardır. Nesne denetim için 3 farklı yol vardır. Bunlar çerçeveler arasındaki fark, optik akış, arka plan modeli çıkarmadır. Nesne sınıflandırma için ise cismin öznitelikleri olan renk, eğim, doku, hareket ve zamansal değişiklere bakılır. Nesne sınıflandırma yöntemleri ise 4 temel gruba ayrılır şekil tabanlı, hareket tabanlı, renk tabanlı ve doku tabanlıdır. Nesne takip yöntemleri ise nokta tabanlı çekirdek tabanlı ve siluet tabanlı olmak üzere 3 genel gruba ayrılır. Görsel denetim ve nesne takibi yapmak için C++, python, matlab kullanılan programlama dillerinden bazılarıdır. Bu çalışmada Python ile OpenCV kütüphanesin 7 farklı nesne takip algoritmasını kullanıldı.

### **2.1.1. CSRT Tracker**

CSRT takip algoritması, takip için çerçeveden seçilen bölgenin bir kısmını filtre desteğini ayarlamak için uzamsal güvenilirlik haritası kullanır. CSRT, kanallar arasında korelasyon yanıtları elde etmek için yalnızca iki standart özellik kullanır. Bunlar HOG'lar ve renk adlarıdır [9].

### **2.1.2. KCF Tracker**

KCF takipçi algoritması fikri basit olmasına rağmen, son zamanlarda en iyi performansı gösteren takipçiler arasında, en hızlı ve en yüksek performansı elde eder. KCF takipçisinin anahtarı, yüksek verimlilik için sirkülasyon matrisinin yapısını keşfederken, takip detektör şemasının ayırt etme yeteneğini geliştirmek için negatif örneklerin çoğaltılmasını kullanır. Veriler dolaşım matrisleri denklem (2.1) ile ifade edilebilir. F,verileri Fourier alanına dönüştüren bir matristir.  $F^H$  F'nin Hermitian transpozudur. Dolaşım matrisinin ayrıştırılması, doğrusal regresyon çözümünü basitleştirmek için kullanılabilir formüle edilmiş hali denklem (2.2)'de verilmiştir. Denklem (2.3)'te verilen temel örneklemlerin doğrusal kombinasyonu, sırt regresyonunun yakın form çözümü ile denklem (2.4) haline gelir. Denklem (2.1) ile bölerek  $\hat{x}$  karmaşık kompleks sayısı elde edilir. Doğrusal olmayan regresyon durumunda, daha güçlü sınıflandırıcı kullanmak için denklem (2.5)'deki çekirdek numarası uygulanır. Bu sayede ikili uzay katsayısı  $\alpha$  denklem (2.6)'daki gibi bulunur. Burada  $k^{***}$  çekirdek korelasyonu olarak tanımlanır. Denklem (2.7)'deki dolaşım matris tekniğini kullanarak Gauss çekirdeğini buluruz. Nesnenin bir sonraki çerçevede aynı konumdaki z yaması, Fourier yanıtını hesaplamak için temel örnek olarak kabul edilir ve denklem (1.8) ile hesaplanır. Burada  $\hat{x}$  modelde öğrenilecek verileri gösterir.  $\hat{f}(z)$  tekrar uzaysal alana dönüştürüldüğünde maksimum tepkiye göre öteleme, izlenen hedefin hareketi olarak kabul edilir [10].

$$X = F^{H} diag(Fx)F (2.1)$$

$$min_w \sum_{i=1}^{n} (f(x_i) - y_i)^2 + \lambda ||w||$$
 (2.2)

$$f(x) = w^T x (2.3)$$

$$w = (X^T X + \lambda I)^{-1} X^T y \tag{2.4}$$

$$f(z) = w^T z = \sum_{i=1}^n \alpha_i \kappa(z, x_i)$$
 (2.5)

$$\hat{\alpha}^* = \frac{\hat{y}}{\hat{k}^{xx} + \lambda} \tag{2.6}$$

$$k^{xx'} = \exp\left(-\frac{1}{\sigma^2}(\|x\|^2 + \|x'\|^2) - 2F^{-1}(\hat{x} \odot \hat{x}'^*)\right)$$
 (2.7)

$$\hat{f}(z) = (\hat{k}^{\tilde{x}z})^* \odot \hat{\alpha} \tag{2.8}$$

#### 2.1.3. Boosting Tracker

Boosting takipçi bir çerçeve verildiğinde çerçevenin konum mahallesindeki her pikselde çalıştırılır ve sınıflandırma puanı kaydedilir. Nesnenin yeni konumu tekrar sınıflandırma yapıldığında puanın maksimum olduğu yerdedir. Nesne tahmini zayıf sınıflandırıcı tarafından oluşturulan  $h^{weak}$  hipotezi öğrenme algoritması kullanılarak

elde edilir. Denklem (2.9)'den optimizasyon kriterine göre bir m seçicisi seçilir. Aslında m için Denklem (2.11)'te gösterilen her bir  $h_i^{weak} \in H^{weak}$  için beklenen  $e_i$  değeri kullanılır. Bir dizi N zayıf sınıflandırıcısı verildiğinde denklem (2.12) ile seçicilerin doğrusal kombinasyonu ile güçlü sınıflandırıcı hesaplanır. Ayrıca marjin ile ilgili olan conf() değeri, güçlü sınıflandırıcının bir güven ölçüsüdür [11].

$$H^{weak} = \left\{ h_1^{weak}, \dots, h_M^{weak} \right\} \tag{2.9}$$

$$h^{sel}(x) = h_m^{weak}(x) (2.10)$$

$$m = argmin_i e_i (2.11)$$

$$hStrong(x) = sign(conf(x))$$
 (2.12)

$$conf(x) = \sum_{n=1}^{N} \alpha_n h_n^{sel}(x)$$
 (2.13)

#### 2.1.4. MIL Tracker

Mil yani çoklu örnek öğrenme takipçi birden çok pozitif ve negatif örnekleyici kullanarak bunları pozitif ve negatif torbalara koyar. Olabilirlik fonksiyonu sınıflandırır. Bir dizi görüntü kırpılır ve denklem (2.14) ile özellik vektörleri hesaplanır. Denklem (2.15) ile MIL sınıflandırıcısı kullanılır. Denklem (2.16) ile takipçi konumu güncellenir. Denklem (2.17) ile iki grup görüntüsü kırpılır. MIL görünüm modeli  $X^r$  pozitif torbası ve  $|X^{r,\beta}|$  negatif torbası ile güncellenir. Her biri  $X^{r,\beta}$  setinden tek bir görüntü içerir [12].

$$X^{s} = \{x|s > ||l(x) - l_{t-1}^{*}||\}$$
(2.14)

$$p(y=1|x) \tag{2.15}$$

$$l_t^* = l(argmax_{x \in X^S} p(y|x))$$
 (2.16)

$$X^{r} = \{x|r > ||l(x) - l_{t}^{*}||\}, X^{r,\beta}\{x|\beta > ||l(x) - l_{t}^{*}|| > r\}$$
 (2.17)

#### 2.1.5. TLD Tracker

TLD yöntemi takip etme problemini üç alt gövdeye ayırır. Bunlar izleme, öğrenme ve algılamadır. Her bir alt görev tek bir modül tarafından adreslenir ve tüm modüller aynı

anda çalışır. İzleme modülü medyan akış algoritması tarafından uygulanır. Medyan akış teoremi ilk olarak noktaları sınırlayıcı kutu içine alır ve yer değiştirmelerinin güvenirliğini tahmin eder. En çok yer değiştirmeye sahip noktaların yarısı özellik noktaları olarak seçilir. Nesnenin hareketi, özellik noktalarının hareketinin medyanı ile elde edilir. Yer değiştirmenin güvenilirliği Lucas-Kanade algoritmasından yararlanarak düzgelenmiş çapraz ilinti katsayısı (NCC) ile tahmin edilir. NCC izlenen noktayı çevreleyen yerleri son çerçeve ( $P_1$ ) ile şuan ki çerçeveyi ( $P_2$ ) karşılaştırır [13].

$$R(x,y) = \frac{\sum_{x',y'} (P_1(x',y') * P_2(x+x',y+y'))}{\sqrt{\sum_{x',y'} P_1(x',y')^2 \sum_{x',y'} P_2(x+x',y+y')}}$$
(2.18)

#### 2.1.6. Median Flow Tracker

Medyan akış algoritması birbiri ardına gelen video dizisi karelerinde seyrek optik akış yardımıyla bir nesnenin konumunu tahmin etmek için kullanılır. Nesnenin, tüm nesnenin hareketi ile senkronize hareket eden küçük ve sert bir şekilde bağlı yamalardan oluştuğu varsayılır. Denklem (2.19)'deki durum sağlandığında bitişik noktaların hareket etmesi gerektiği ve kabaca aynı yer değiştirmeye sahip olur. Burada  $N_i$  4 adet komşusu olan i noktalarına sahip bir nokta kümesidir. Parantez içi bağımsız değişkeni doğru ise 1'e aksi taktirde 0'a eşit olan bir fonksiyondur.  $\Delta_i$  i'inci noktanın yer değiştirmesidir.  $\epsilon$  yer değiştirme eşiğidir ve  $\epsilon$  < 0.5 olarak alınır.  $S_i^{Nh} < \theta$ ,  $\theta$  < 1 koşulu sağlandığında noktanın komşularıyla eş zamanlı olarak hareket ettiği kabul edilir [14].

$$S_i^{Nh} = \sum_{j \in N_i} (\left\| \Delta_i - \Delta_j \right\|^2 > \epsilon)$$
 (2.19)

#### 2.1.7. Mosse Tracker

Mosse takipçi algoritması daha az eğitim görüntüsü ile sentetik tam filtrelerin ortalaması yöntemi benzeri filtreler üretmek için kullanılır. Mosse takipçi algoritması için bir dizi eğitim görüntüsü  $f_i$  ve eğitim çıktıları olan  $g_i$  ihtiyaç vardır.  $g_i$  herhangi bir şekle girebilir. Bu durumda  $g_i$ , eğitim görüntüsünde hedef üzerinde merkezlenmiş kompakt 2D Gauss şekilli bir tepe noktasına sahip olacak şekilde temel gerçeklikten üretilir. Girdi ve çıktı arasındaki basit öğe temelli ilişkiden yararlanmak için Fourier

bağlantısı kullanılır. Büyük harf değişkenler olan  $F_i$ ,  $G_i$  ve filtre H'yi küçük harf karşılıklarının Fourier dönüşümü olarak tanımlanır. Eğitim girdilerini istenen eğitim çıktılarıyla eşleştiren bir filtre bulmak için, denklem (2.21) ile konvolüsyonun gerçek çıktısı ile istenen evrişim çıktısı arasındaki hata karesinin toplamını en aza indiren bir H filtresi bulunur. Takip etmede hedef her zaman ortalanmaz ve öğrenme çıktılarındaki tepe öğrenme girdilerindeki hedefi takip etmek için hareket eder. Genel olarak öğrenme çıktıları herhangi bir şekle sahip olabilir. Bu optimizasyon problemini çözmek için ilk olarak filtrenin  $\omega$  ve  $\nu$  tarafından indekslenen her bir elemanı bağımsız olarak çözülebilir çünkü Fourier alanındaki tüm işlemler eleman bazında gerçekleştirilir. Bu, işlevi  $H_{\omega\nu}$  ve  $H_{\omega\nu}^*$  cinsinden yazmayı içerir. Denklem (2.22) yapılarak,  $H_{\omega\nu}$  bağımsız bir değişken olarak ele alınır.  $H^*$  için çözüldüğünde, denklem (2.23)'teki Mosse filtresi için bir kapalı form ifadesi bulunur[15].

$$H_i^* = \frac{G_i}{F_i} \tag{2.20}$$

$$\min_{H^*} \sum_i |F_i \odot H^* - G_i|^2 \tag{2.21}$$

$$0 = \frac{\partial}{\partial H_{\omega v}^*} \sum_{i} |F_{i\omega v} H_{\omega v}^* - G_{i\omega v}|^2$$
(2.22)

$$H^* = \frac{\sum_i G_i \odot F_i^*}{\sum_i F_i \odot F_i^*}$$
 (2.23)

#### 2.2. Tasarım (Yöntemin Problemin Çözümünde Kullanımı)

Python derleyicisine OpenCV kütüphanesi indirildi. OpenCV kütüphanesi kullanılarak nesne takip algoritmaları kullanıldı. Nesnenin çerçevedeki konumu belirlendi. Konum bilgileri Python üzerinden Arduino'ya gönderildi. Arduino ile nesnenin konumu ile çerçevenin merkezi arasındaki uzaklık hesaplandı. Bulunan değere göre motorlar x,y ekseninde hareket ederek cisim çerçevenin merkezine getirildi.

# **BÖLÜM 3. UYGULAMA ÇALIŞMALARI**

### **3.1. Uygulamada Kullanılan Araç ve Gereçler**

Uygulamada görsel denetim ve nesne takibi yapabilmek için Python ve OpenCV kullanıldı. Python nesne yönelimli bir programlama dilidir. Python ile sistem otomasyonu, web, makine öğrenimi, veri bilimi, uygulama programlama arayüzü gibi birçok alanda çalışma yapılabilir. OpenCV açık kaynak kodlu görüntü işleme kütüphanesidir. OpenCV kütüphanesi görüntü işleme ve makine öğrenmesine ait algoritmalar barındırır. Nesnenin ekran dışına çıkması durumunda takibini yapabilmek için motor sürme işleminde Arduino ile kullanıldı. Arduino elektronik donanım ve yazılım temelli açık kaynaklı bir mikrokontrolcü platformudur.

### **3.2. Uygulamanın Gerçekleştirilme Aşamaları**

Uygulama nesneyi takip etmek için kullanıcıdan 7 farklı nesne takip algoritmasından birini seçmeni ister. Bu yapı Python'daki anahtar:değer çiftlerinden oluşan sözcükler kullanılarak tasarlandı. Artık nesne takibinde kullanacağımız takip algoritması belirlenmiş ve 'tracker' değişkeni tanımlanmış oldu.

Şekil 3.1. Algoritma seçme menüsü

Algoritma seçimi tamamlandıktan sonra kamera görüntü alma faaliyetine başlar ve alınan her görüntü "capture" değişkenine depolanır. Kullanıcı takip etmek istediği nesneyi seçmek için klavyeden "x" tuşuna basar. Seçme işlemi sırasında kullanıcının seçme işlemini gerçekleştirebilmesi için "capture" değişkenine depolanan son görüntü/frame kullanılır. Kullanıcı seçmek istediği nesneyi görüntü üzerinde bir dikdörtgen içerisine almak için faresini kullanır. Artık takip işleminin gerçekleşmeye başlaması için kullanıcının klavyeden herhangi bir tuşa basması yeterlidir.

![](_page_22_Picture_1.jpeg)

Şekil 3.2. Hedef nesne seçme ekranı

Takip işlemi başladığı zaman kullanılmakta olan takip algoritması, seçilen yani takip edilen objenin frame üzerindeki koordinatları ve servo motorların anlık konumu ekranda görüntülenir. Hem görüntü üzerindeki takip işleminin hem de fiziksel takip işleminin düzgün gerçekleşebilmesi için dikdörtgen içine alınan ve takip edilen nesnenin içine alındığı dikdörtgenin merkez noktası hem de ana frame'in yani çerçevenin merkez noktası hesaplanır.

![](_page_23_Picture_0.jpeg)

Şekil 3.3. Hedefin takip edilmesi

Seçilen hedefi görüntünün ortasına almak için hedef ile görüntü merkezi arasındaki uzaklık belirlenir. Fiziksel takip işlemi Arduino'ya gönderilen anlık konum bilgileri sayesinde servoların dönüşü ile gerçekleşir. Seçilen hedefin ya da kameranın hareket etmesi durumunda Python üzerinden Arduino ile iletişime geçilir. Arduino kodlarıyla servo motorlar yardımıyla kamera iki eksen üzerinde hareket ederek seçilen hedefi, görüntünün ortasında tutarak takip etmeye çalışır.

Eğer takip edilen nesne kadrajdan çıkarsa fiziksel takip işlemi cismin son konumuna kadar devam eder. Kadrajdan çıkan nesne tekrar aynı kadraja girene kadar fiziksel takip işlemi son konuma gitmeye devam eder.

![](_page_23_Picture_4.jpeg)

Şekil 3.4. Arduino ile motor kontrol

Hedeflenen cismin takip işlemi bittikten sonra q tuşuna basarak program bitirilir.

![](_page_24_Figure_0.jpeg)

Şekil 3.5. Uygulamanın Python blok diyagramı

![](_page_25_Figure_0.jpeg)

Şekil 3.6. Uygulamanın Arduino blok diyagramı

![[Pasted image 20260429134121.png]]
Şekil 3.7. Uygulamanın Sequence diyagramı

![[Pasted image 20260429134204.png]]
Şekil 3.8. Uygulamanın Usecase diyagramı

![[Pasted image 20260429134157.png]]
Şekil 3.9. Uygulamanın Relationship diyagramı
### **3.3. Uygulama Sonuçları ve Yorumlanması**

Uygulama sonucunda hedef seçme ve hedefin takip işlemi başarılı olmuştur. Boosting algoritması yavaş çalışmaktadır ve bazen hedefi kaybetmektedir. MIL algoritması Boosting algoritmasına göre daha iyi doğruluk ile çalışmaktadır. KCF algoritması Boosting ve MIL algoritmasına göre daha iyi çalışıyor fakat nesnelerin başka nesne tarafından gizlendiğinde yani tıkanma durumunda nesne takibini kaybedebiliyor. Median Flow algoritması çok hızlı nesneleri takip ederken nesne takibini kaybedebiliyor. TLD algoritması tıkanma koşullarında en iyi sonucu veriyor. Mosse algoritması çok hızlı çalışıyor ama yüksek hata oranına sahip. CSRT algoritması daha yüksek doğruluk oranına sahip fakat 25 fps'te çalışıyor.

# **BÖLÜM 4. SONUÇLAR VE ÖNERİLER**

Bu bölümünde tüm çalışmanın sonucunda elde edilen bulgular yorumlanarak çeşitli öneriler sunulmuştur. Çalışma sonuçlarının gerçekçi kısıtlar açısından analizi yapılmıştır.

### **4.1. Sonuçlar**

Bu çalışma ile anlık görüntüde hedef seçilen objenin takibi başarı ile yapılmıştır. Nesnenin hareketi durumunda motorlar yardımıyla nesne görüntü karesinin ortasında başarılı bir şekilde tutulmuştur. Teorik sonuçlar ortamdan bağımsız olduğu için uygulama sonuçları ile birebir aynı sonuçlar alınamamaktadır. Ortamdaki ışık, görüntü girdi kalitesi, doğru algoritma seçimi, hareketli nesne sayısı ve nesnelerin kameraya olan uzaklıkları sonuçları etkileyen faktörlerdir. Yedi farklı algoritmanın her birinin kullanım amacı farklı olduğu için kullanıcı duruma en uygun algoritmayı seçerek nesne tespit ve takip etme işlemini başarılı bir şekilde yapabilir.

# **4.2. Öneriler**

Hedef tespit ve algılama işlemlerini sadece kamera yerine elektro-optik sistemler kullanarak projenin başarısı kat kat arttırılabilir. Yazılımını derin öğrenme algoritmaları ile destekleyerek daha başarılı seviyelere ulaştırılabilir. Kamera, yazılım ve motorlar Raspberry ve bir drone ile birleştirerek bir İHA yapılabilir.

### **4.3. Sonuçların Sağlık, Çevre ve Güvenlik Açısından Analizi**

Çalışmanın kullanıcıya ve çevreye sağlık bakımından herhangi bir zararı bulunmamaktadır. Çalışmada çevre ile teması bulunan parçaları iki adet motora bağlı kamera ve kablolardır. Bunların çevreye herhangi bir zararı bulunmamaktadır. Çalışmanın insanları hayatına güvenlik açısından bir zarardan çok yararı vardır. Bu sistem yaşam alanlarında, ev ve iş yerlerinde, halka açık alanlarda güvenlik amaçlı kullanılabilir.

# **BÖLÜM 5. KAYNAKLAR**

- [1] Resul, D. A. Ş., Polat, B., & Tuna, G. (2019). Derin Öğrenme ile Resim ve Videolarda Nesnelerin Tanınması ve Takibi. Fırat Üniversitesi Mühendislik Bilimleri Dergisi, 31(2), 571-581.
- [2] Koksal, K., Surucu, D., Surucu, M., & Hacıoğlu, R. (2014, April). Visual line tracking with vector field guidance for UAV. In 2014 22nd Signal Processing and Communications Applications Conference (SIU) (pp. 646-649). IEEE.
- [3] HANBAY, K., & Hüseyin, Ü. Z. E. N. (2017). Nesne tespit ve takip metotları: Kapsamlı bir derleme. Türk Doğa ve Fen Dergisi, 6(2), 40-49.
- [4] Detecting Movement with Pairwise Frame Subtraction <https://sam-low.com/opencv/frame-differencing.html> Son erişim tarihi : 22.03.2021
- [5] Balaji, S. R., & Karthikeyan, S. (2017, January). A survey on moving object tracking using image processing. In 2017 11th international conference on intelligent systems and control (ISCO) (pp. 469-474). IEEE.
- [6] Surucu, D., Surucu, M., Koksal, K., & Hacıoğlu, R. (2015, May). Visual tracking and control of Unmanned Aerial Vehicle. In 2015 23nd Signal Processing and Communications Applications Conference (SIU) (pp. 1849- 1852). IEEE.
- [7] Talu M.F., Nesne Takip Yöntemlerinin Sınıflandırılması,İstanbul Ticaret Üniversitesi Fen Bilim. Derg., 18, 45–63, 2010
- [8] Nangre M.R., Havanoor S.S., A survey on multiple object detection and tracking, International Journal of Engineering Research & Technology (IJERT), pp. 3538–3540, 2014
- [9] Biswas, D., Su, H., Wang, C., & Stevanovic, A. (2019). Speed Estimation of Multiple Moving Objects from a Moving UAV Platform. ISPRS International Journal of Geo-Information, 8(6), 259.
- [10] George, M., Jose, B. R., & Mathew, J. (2018). Performance evaluation of kcf based trackers using vot dataset. Procedia Computer Science, 125, 560-567.
- [11] Grabner, H., Grabner, M., & Bischof, H. (2006). Real-time tracking via on-line boosting. In Bmvc (Vol. 1, No. 5, p. 6).

- [12] Babenko, B., Yang, M. H., & Belongie, S. (2009). Visual tracking with online multiple instance learning. IEEE Conference on computer vision and Pattern Recognition, pp. 983-990
- [13] Zhen, X., Fei, S., Wang, Y., & Du, W. (2020). A visual object tracking algorithm based on improved TLD. Algorithms, 13(1), 15.
- [14] Varfolomieiev, A., & Lysenko, O. (2016). An improved algorithm of median flow for visual object tracking and its implementation on ARM platform. Journal of Real-Time Image Processing, 11(3), 527-534.
- [15] Bolme, D. S., Beveridge, J. R., Draper, B. A., & Lui, Y. M. (2010, June). Visual object tracking using adaptive correlation filters. In 2010 IEEE computer society conference on computer vision and pattern recognition (pp. 2544- 2550). IEEE.

# **BÖLÜM 6. ÖZ GEÇMİŞ**

1 Mart 1998 yılında Osmaniye'nin Kadirli ilçesinde dünyaya gelmiştir. Asker bir baba ve ev hanımı bir annenin evladıdır, ailenin üç evladının ortanca evladıdır. İlköğretim hayatını sırasıyla İzmir Aliağa PETKİM İlköğretim Okulu, Elazığ Kovancılar Eti Holding İlköğretim Okulu, Şırnak Orgeneral Edip Başer İlköğretim Okulu, Osmaniye Kadirli 100.Yıl İlköğretim Okulu ve Ankara Sincan İMKB İlköğretim Okulu'nda yaptı. Liseyi Özel Ankara Sanayi Odası Mesleki ve Teknik Anadolu lisesinde okumuştur. Üniversite eğitimine Sakarya Üniversitesi Mühendislik Fakültesi Elektrik-Elektronik Mühendisliği bölümünde devam etmektedir.

# **BÖLÜM 7. EKLER**

### **EK A. Deney Tasarımı Açıklamaları**

Bu çalışmada görsel denetim ve nesne takibi yapabilmek için OpenCV yardımıyla 7 farklı algoritma kullanılmıştır. Bu çalışmada matematiksel yöntem ve tasarım bölümünde algoritmaların nesne tespiti ve takibi bakımından farklılıklarının bulunduğu anlaşılmış ve bu farklılıkları tespit etmek için her bir algoritmaya testler yapılmıştır. Bir nesne belirlenmiştir ve bu nesneyi farklı hızlarda hareket ettirerek, test yapılan odanın ışık düzeyini değiştirerek, seçilen nesnenin kameraya ulan uzaklığını değiştirerek, seçilen nesneyi önce duvar önünde sonra nesnelerin önünden geçirerek algoritmanın nesneyi takip etme başarısını gözlemlenmiştir. Daha sonra seçilen nesneyi farklı renkte ve farklı dokuda başka nesneler seçerek değiştirip, yukarıda belirtilen testleri uygulayarak yedi farklı algoritmanın hangi konularda başarılı olarak çalıştığı ve başarı olarak birbirlerinden farklılığını ölçmek amaçlanmıştır.

### **EK B. IEEE Etik Kurallar Onay Formu**

![](_page_32_Picture_1.jpeg)

IEEE üyeleri olarak bizler bütün dünya üzerinde teknolojilerimizin hayat standartlarını etkilemesindeki önemin farkındayız. Mesleğimize karşı şahsi sorumluluğumuzu kabul ederek, hizmet ettiğimiz toplumlara ve üyelerine en yüksek etik ve mesleki davranışta bulunmayı söz verdiğimizi ve aşağıdaki etik kuralları kabul ettiğimizi ifade ederiz.

- 1. Kamu güvenliği, sağlığı ve refahı ile uyumlu kararlar vermenin sorumluluğunu kabul etmek ve kamu veya çevreyi tehdit edebilecek faktörleri derhal açıklamak;
- 2. Mümkün olabilecek çıkar çatışması, ister gerçekten var olması isterse sadece algı olması, durumlarından kaçınmak. Çıkar çatışması olması durumunda, etkilenen taraflara durumu bildirmek;
- 3. Mevcut verilere dayalı tahminlerde ve fikir beyan etmelerde gerçekçi ve dürüst olmak;
- 4. Her türlü rüşveti reddetmek;
- 5. Mütenasip uygulamalarını ve muhtemel sonuçlarını gözeterek teknoloji anlayışını geliştirmek;
- 6. Teknik yeterliliklerimizi sürdürmek ve geliştirmek, yeterli eğitim veya tecrübe olması veya işin zorluk sınırları ifade edilmesi durumunda ancak başkaları için teknolojik sorumlulukları üstlenmek;
- 7. Teknik bir çalışma hakkında yansız bir eleştiri için uğraşmak, eleştiriyi kabul etmek ve eleştiriyi yapmak; hatları kabul etmek ve düzeltmek; diğer katkı sunanların emeklerini ifade etmek;
- 8. Bütün kişilere adilane davranmak; ırk, din, cinsiyet, yaş, milliyet, cinsi tercih, cinsiyet kimliği veya cinsiyet ifadesi üzerinden ayırımcılık yapma durumuna girişmemek;
- 9. Yanlış veya kötü amaçlı eylemler sonucu kimsenin yaralanması, mülklerinin zarar görmesi, itibarlarının veya istihdamlarının zedelenmesi durumlarının oluşmasından kaçınmak;
- 10. Meslektaşlara ve yardımcı personele mesleki gelişimlerinde yardımcı olmak ve onları desteklemek.

IEEE Yönetim Kurulu tarafından Ağustos 1990'da onaylanmıştır.

Bu çalışma IEEE Etik ilkelerine bağlı kalınarak hazırlanmıştır.

### **EK C. Seminer, Fuar, Konferans Katılım Belgesi**

BTK Kariyer Günleri, Dinlenesi Fikirler, 7 Mart 2018, Katılımcı Gazi Üniversitesi, G-Force Days, 25 Nisan 2018, Görevli

### **EK D. Çalışma Yönetimi**

Bu bölümde, iş-zaman çizelgesi ve/veya iş-görev paylaşımı, başarı ölçütleri ve risk yönetimi belirtilmektedir.

**İş Zaman Çizelgesi ve/veya İş Görev Paylaşımı**

| Hafta/İş             | 1   | 2   | 3   | 4   | 5   | 6   | 7   | 8   | 9   | 10  | 11  | 12  | 13  | 14  |
| -------------------- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Literatür Taraması   | X   | X   | X   | X   | X   |     |     |     |     |     |     |     |     |     |
| Rapor Haline Getirme |     |     |     |     | X   |     |     |     |     |     |     |     |     |     |
| Yöntem ve Tasarım    |     |     |     |     | X   | X   | X   |     |     |     |     |     |     |     |
| Uygulama             |     |     |     |     |     |     | X   | X   | X   | X   | X   |     |     |     |
| Rapor Haline Getirme |     |     |     |     |     |     |     |     |     |     | X   |     |     |     |
| Sonuç                |     |     |     |     |     |     |     |     |     |     | X   | X   | X   |     |
| Rapor Haline Getirme |     |     |     |     |     |     |     |     |     |     |     |     | X   |     |
| Rapor Teslimi        |     |     |     |     |     |     |     |     |     |     |     |     |     | X   |

## **Başarı Ölçütleri**

Çalışmanın başarıya ulaşması için en önemli bölüm uygulama kısmında bulunan nesnenin görsel denetimi ve nesnenin takibidir. Görsel denetim ve nesne takibi yapabilmek için 4 adım vardır. Gerçek zamanda aldığımız girdilerin ön işleme alınması, nesnelerin tespiti, nesnelerin sınıflandırılması ve hedef seçilen nesnenin takibidir. Her adım kendinden sonraki adımı etkilemektedir. Girdi, görüntü işleme başarısını arttırmak ve işlemci yükünü azaltmak için ön işlemden geçer. Girdinin son halinde bulunan nesneler tespiti, nesnenin özniteliği ve hareketine göre tespitleri yapılır. Tespit edilen nesneler belli özelliklere göre sınıflandırılır. Sınıflandırılan nesnelerden hedef seçilir ve hedef nesnenin istenilen algoritmaya göre takibi yapılır. Ön işleme kısmında başarısız olmak nesnenin tespitini zorlaştırmakta ve bazı nesnelerinin tespit edilememesine neden olmaktadır. Nesne tespitinde başarısız olmak girdideki nesnelerin sınıflandırılmasında hatalara yol açmaktadır. Nesnenin sınıflandırmasında başarısız olmak nesnenin takibini zorlaştırmakta veya takip edilememesine neden olmaktadır. Nesne takip algoritmalarında başarısız olmak nesnenin takip edilememesi neden olur.

### **Risk Yönetimi**

Çalışmanın devam etmesini engelleyecek öngörülmemiş durumlarda programlama dili olarak Python yerine C++ kullanmak planlanmıştır. Aynı şekilde Arduino ile iletişim kurma konusunda sıkıntı olması durumunda Raspberry kullanılması planlanmıştır.

# **Çalışmanın Sürdürülebilir Kalkınma Açısından Önemi**

Çalışma iha, radarlar gibi birçok kullanım alanı vardır. Şehirlerde, yaşam alanlarında yapay zeka desteği ile kullanarak çevreye zarar verecek davranışların yapan kişilerin takibi yapılabilir ve bu sayede insanların kendilerini daha huzurlu ve güvenli hissetmeleri sağlanabilir.

## **Maliyet Değerlendirmesi**

Çalışmada ekonomik maliyet çıkaracak nesneler bir adet kamera, bir adet arduino kartı ve iki adet motordur. Burada maliyeti en fazla olan nesne kameradır. Kameranın özellikleri olan optik lens kalitesi, video çözünürlüğü ve saniyede tazeleme oranına göre maliyeti değişmektedir. Arduino kartlarının fiyatı arduino kartının özelliklerine göre değişmektedir. Bu çalışmada kullanılan malzemelerin toplam maliyeti beş yüz Türk lirasıdır.

### **EK E. Çalışma ile İlişkili Diğer Ekler**

### **EK E.1. Python Yazılım Kodları**

```
import cv2
import time 
import serial
import struct
Algorithms = {
       "CSRT" : cv2.TrackerCSRT_create,
       "KCF" : cv2.TrackerKCF_create,
       "Boosting" : cv2.TrackerBoosting_create,
       "Mil" : cv2.TrackerMIL_create,
       "TLD" : cv2.TrackerTLD_create,
       "MedianFlow" : cv2.TrackerMedianFlow_create,
       "Mosse" : cv2.TrackerMosse_create}
NumAlgorithms = {
       "1" : "CSRT",
       "2" : "KCF",
       "3" : "Boosting",
       "4" : "MIL",
       "5" : "TLD",
       "6" : "MedianFlow",
       "7" : "Mosse"}
NameAlgorithm = ["Boosting"]
RunAlgorithm = Algorithms[NameAlgorithm[0]]()
choice = 0
print(''' 
1.CSRT
2.KCF
3.Boosting
4.MIL
5.TLD
6.MedianFlow
7.Mosse
''')
choice = input("Choose a Algorithm :")
keys = NumAlgorithms.keys()
if choice in keys:
      NameAlgorithm.append(NumAlgorithms[choice])
      NameAlgorithm.remove(NameAlgoritm[0])
      print(choice, NameAlgorithm)
      RunAlgorithm = Algorithms[NameAlgorithm[len(NameAlgorithm)-1]]()
else:
      print("Invalid Input Value {}" .format(NameAlgorithm))
cap = cv2.VideoCapture(0)
xloc = [90]
yloc = [90]
rsuc, img = cap.read()
while True:
      timer = cv2.getTickCount()
      rsuc, img = cap.read()
      rsuc, coorpln = RunAlgorithm.update(img)
      if rsuc:
             x, y, w, h = int(coorpln[0]), int(coorpln[1]), int(coorpln[2]), 
int(coorpln[3])
             mid_x = int(x+w/2)
             mid_y = int(y+h/2)
             print(" ")
             print("coorpln")
             cv2.rectangle(img, (x,y), ((x+w), (y+h)), (0,255,0), 5, 1)
             cv2.putText(img, "Object At :", (10,15), 
cv2.FONT_HERSHEY_PLAIN, 1.5, (0, 255, 0), 2)
             cv2.putText(img, "X ="+str(center_x), (140,15), 
cv2.FONT_HERSHEY_PLAIN, 1.5, (0, 255, 0), 2)
             cv2.putText(img, "Y ="+str(center_y), (240,15), 
cv2.FONT_HERSHEY_PLAIN, 1.5, (0, 255, 0), 2)
             cv2.putText(img, "Algorithm : "+NameAlgorithm[0], (10, 450), 
cv2.FONT_HERSHEY_PLAIN, 1.5, (0, 255, 0), 2)

if mid_x < 300 and mid_x >340:
       print("locked_x")
       xloc.append(xloc[len(xloc)-1])
       xloc.remove(xloc[0])
elif mid_x > 340:
       print("right")
       xloc.append(xloc[len(xloc)-1]-1)
       xloc.remove(xloc[0])
elif mid_x < 300:
       print("left")
       xloc.append(xloc[len(xloc)-1]+1)
       xloc.remove(xloc[0])
xpos=xloc[len(xloc)-1]
if mid_y < 220 and mid_y >260:
       print("locked_y")
       yloc.append(yloc[len(yloc)-1])
       yloc.remove(yloc[0])
elif mid_y > 260:
       print("down")
       yloc.append(yloc[len(yloc)-1]-1)
       yloc.remove(yloc[0])
elif mid_y < 220:
       print("up")
       yloc.append(yloc[len(yloc)-1]+1)
       yloc.remove(yloc[0])
ypos=yloc[len(yloc)-1]
if xpos >= 180:
       xloc.append(xloc[len(xloc)-1]-1)
       xloc.remove(xloc[0])
       print("x-axis boundary angle 180")
elif xpos <= 0:
       xloc.append(xloc[len(xloc)-1]+1)
       xloc.remove(xloc[0])
       print("x-axis boundary angle 0")
if ypos >= 180:
                    yloc.append(yloc[len(yloc)-1]-1)
                    yloc.remove(yloc[0])
                    print("y-axis boundary angle 180")
             if ypos <= 0:
                    yloc.append(yloc[len(yloc)-1]+1)
                    yloc.remove(yloc[0])
                    print("y-axis boundary angle 0")
             print(xpos, ypos)
      else:
             cv2.putText(img, "Press: X" , (10,15), 
cv2.FONT_HERSHEY_PLAIN, 1.5, (0, 255, 0), 2)
      cv2.imshow("Tracking", img)
      key = cv2.waitKey(1) & 0xFF
      if key == ord("x"):
             coorpln = cv2.selectROI("Tracking", img, False)
             RunAlgorithm.init(img, coorpln)
      elif key == ord("q"):
             break
cap.release()
cv2.destroyAllWindows()
```

## **EK E.2. Arduino Yazılım Kodları**

```
#include <Servo.h>
int data_x1 = 0;
int data_y1 = 0;
int data[1];
Servo myservo_x1;
Servo myserv0_y1;
void setup() {
       Serial.begin(9600);
       myservo_x1.attach(9);
       myservo_y1.attach(10);
       myservo_x1.write(90);
       myservo_y1.write(90);
}
void loop() {
       while(Serial.available() >= 2) {
               for(int i =0; i < 2; i++) {
                       data[i] = Serial.read();
               }
               myservo_x1.write(data[0]);
               myservo_y1.write(data[1]);
               Serial.printIn(data[0]);
               Serial.printIn(data[1]);
               }
}
```

* https://drive.google.com/file/d/1cEfQJ0En3jDUYZlLh6bAGXaDWkooeqkt/view

* https://drive.google.com/file/d/1cFODFqn3Lrw2_SRPcbI7_Z07xWquXr0F/view