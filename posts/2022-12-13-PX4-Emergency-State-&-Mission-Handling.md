# İçindekiler

| 1 GIRIŞ                                                                 | 5  |
|-------------------------------------------------------------------------|----|
| 2 ÖZET                                                                  | 6  |
| 3 METODOLOJİ                                                            |    |
| 3.1 Uygulama Esnasında Karşılaşılabilecek Olası Durumların Ele Alınması | 7  |
| 3.1.1 Eklenilmesi Planlanan Yeni Özellikler                             |    |
| 3.1.2 Ele Alınması Gereken Acil Durumlar                                | 8  |
| 3.1.3 Hata Durumları                                                    | 9  |
| 3.2 Eklenilmesi Planlanan Yeni Özelliklerin Uygulama Konsepti           | 10 |
| 3.2.1 Cihazın Coğrafi Sınır Toleransının Ayarlanması                    |    |
| 3.2.2 Cihazın Göreve Kaldığı Noktadan Devam Etmesi                      | 12 |
| 3.2.2 Püskürtme Hızının Dinamik Olarak Değişebilmes                     |    |
| 3.3 Ele Alınması Gereken Acil Durumlar                                  |    |
| 3.3.1 Gps Gürültüsünün İzlenmesi                                        | 15 |
| 3.3.2 Batarya Doluluk Oranının İzlenmesi                                | 16 |
| 3.3.3 İlaç Tankı Doluluk Oranı                                          | 17 |
| 3.3.4 Link Kaybı Durumunda Karar Mekanizması                            |    |
|                                                                         |    |

# **1** GİRİŞ

zirai amaçlı ilaçlama kopterinde kullanılacak olan otopilot yazılımına, uygulama sırasında meydana gelebilecek olası aksaklıkları işleme ve karar verebilme özelliği kazandırılmak istenmektedir. Buna istinaden planlar yapılmış ve bu raporda derlenmiştir.

Yapılan çalışmalar kapsamında cihazın, görevini ;

- Kalkış
- Uçuş
- İniş

olmak üzere üç ana aşamada tamamladığı görülmüş ve öngörülen problemler bu aşamalar altında incelenmiştir.

Amaçlanan, otopilot yazılımına eklenen yeni işlevler ile öngörülen durumların üstesinden gelebilmektir. Yazılım, geliştirme aşamasında olduğundan söz konusu yeni işlevler sözde kod yapısında ve algoritmalar yardımı ile incelenecek olup konsept niteliğindedir.

Bu sebeple çeşitli bölümlerde tam sayısal değerler, tüm olası durumlar, işlenmesi gereken komutlar belirtilmemiş olabilir. Kod dizimi ve algoritmik yapılar yüzeysel kalabilir/gerçeği yansıtmayabilir. Amaç, genel fikre dair bir konsept oluşturmaktır.

# **2** ÖZET

zirai amaçlı ilaçlama kopteri, bir zirai arazisini ilaçlama görevini üç aşamada gerçekleştirmektedir. Bu aşamalar;

- Kalkış
- Uçuş
- İniş

şeklinde sıralanabilir. Kopterde kullanılan otopilot yazılımı şu ana kadar, yer kontrol istasyonu ile harita üzerinde oluşturulan sanal coğrafi sınırın içerisinde belirli aralıklar ile bir araziyi tarama, tarama rotası üzerinde iken bu sınırın dışarısında çıkıp çıkmadığını anlayabilme, buna göre ilaç püskürtme ve yer kontrol istasyonuna belirli değerler gönderme özelliklerine sahiptir.

Buna nazaran cihazın kalkış noktasının zirai arazisini temsil eden sanal coğrafi sınır içerisinde bulunması durumunda kalkış yapmadan veya görev başlangıç noktasına varmadan ilaç püskürtme işleminin başlatılması, coğrafi sınır verilerinin halen çok sığ bir şekilde işlenmesi ve ilaç tankı doluluk oranı, batarya doluluk oranı gibi belirli parametrelerin işlenmemesinden kaynaklı sorunlar mevcut.

Dolayısıyla söz konusu problemlerin giderilmesi, cihazın görevini istenildiği gibi yerine getirebilmesi ve müşteri beklentilerini karşılayabilmesi için Otopilot yazılımında görev aşamaları iyi belirtilmeli ve gerekli ek parametreler hesaba katılmalıdır.

Bu çalışmada, cihaza "zirai amaçlı ilaçlama kopteri" olma özelliğini kazandıran Otopilot yazılımında ki zirai modulu fonksiyonları irdelenerek, modüle kazandırılacak yeni özellikler ve gerekli olan moduller arası iletişim altyapısı otopilot yazılımında ki './src/modules/navigator/navigator\_main.cpp' adresinde ki "\_modulu\_main" fonksiyonuna eklenmiş gibi sözde kod ve algoritmalar yardımı ile incelenmiştir.

Konsept aşamasında tasarlanan çözüm önerileri arasından gerçekleştirilmesine karar verilenler, otopilot yazılımına eklenerek öncelikle QGroundControl yer kontrol istasyonu programı üzerinde test edilecektir. Yapılan testlerin başarı ile tamamlanması halinde gerçek dünya testlerine geçilecektir.

# **3** METODOLOJİ

Planlama çalışmaları sonucu öngörülen problemlerin otopilot yazılımında ki navigator modülünde hali hazırda mevcut olan "\_modulu\_main" fonksiyonuna eklenecek yeni işlevler sayesinde Otopilot yazılımına kazandırılacak yeni özellikler ile çözülmesi amaçlanmaktadır.

Yapılacak kodlama çalışmaları Visual Studio Code kod editörü yazılımı ile gerçekleştirilecektir.

Simülasyon çalışmaları Gazebo simülasyon ve QGroundControl yer kontrol istasyonu yazılımları aracılığı ile gerçekleştirilecektir.

Bahsi geçen çalışmalar VirtualBox yazılımında Ubuntu 20.04 LTS işletim sistemi kurulu bir sanal makine üzerinde gerçekleştirilecektir.

Otopilot yazılımı içerisinde ki modüllerin haberleşmesinde uORB, otopilot üzerinden yer kontrol istasyonuna mesaj gönderilmesi için kullanılan haberleşme yöteminde ise MAVLink iletişim protokolü ile kullanılacaktır.

Yer kontrol istasyonundan otopilota mesaj gönderme işlemi ise geliştirilen MAVLink-Mesajlaşma yazılımı ile simüle edilecektir.

Çalışmalar konsept niteliğinde olup sözde kod ve algoritmalar yardımı ile planlanacaktır, ardından simülasyon çalışmalarına geçilecektir. Konsept çalışma planının oluşturulabilmesi için öncelikle yaşanması muhtemel problemler ve ele alınması gereken durumlar belirlenmeli ve bir öncelik sırasına konulmalıdır.

# **3.1 Uygulama Esnasında Karşılaşılaşılabilecek Olası Durumların Ele Alınması**

Görevi uygulama sırasında cihaz tarafından kontrol edilmesi istenilen durumlar, planlamalar doğrultusunda ;

- Eklenilmesi planlanan yeni özelliklere dair durumlar
- Acil durumlar

olmak üzere iki başlık altında toplanabilir.

Eklenilmesi istenilen yeni özelliklere dair olan durumlar, cihazın, müşteri memnuniyetini arttırmak, ilaçlama verimliliğini arttırmak ve güvenliği gözetmek amaçları doğrultusunda belirlenmiştir.

Acil durumlar ise cihazın, ilaçlama sırasında istenilenin dışında hareket etmesi, fiziksel donanımında bir problem yaşanması, bağlantı kopması, ilaç/batarya doluluk oranı gibi belirli parametreleri gözeten durumları ele alır. Bunlar haricinde var olan yazılımda düzeltilmesi gereken hatalarda belirli bir plan dahilinde sıralanarak çözülecektir.

# **3.1.1 Eklenilmesi Planlanan Yeni Özellikler**

zirai amaçlı ilaçlama kopterinde, var olan otopilot yazılımında ki zirai modulune eklenmesi planlanan yeni özellikler şunlardır;

- Cihazın, sanal coğrafi sınırdan ±5m uzaklaşması halinde pilottan komut beklemek üzere 5 sn için HOLD-MODE (cihazın son konumunda sabit kalmasını sağlar) moduna geçişmesi. 5 sn içerisinde herhangi bir komut alınmaması durumunda cihazın yer kontrol noktasına otomatik olarak geri dönmesi.
- Görevin herhangi bir sebeple (acil durum, komut vb.) yer kontrol noktasına geri dönmesi durumunda son konumunu kaydetmesi ve tekrar kullanıma hazır hale geldiğinde kaldığı son noktadan devam edebilmesi.
- Yer hızı ve yükseklik parametrelerine göre ilaç püskürtme hızının dinamik olarak değişebilmesi.

# **3.1.2 Ele Alınması Gereken Acil Durumlar**

zirai amaçlı ilaçlama kopterinde, görev uygulama anında gerçekleşebilecek muhtemel aksaklıklar şu şekilde sıralanmıştır ;

#### • GPS Jamming değerinin gözetilmesi

Bu acil durum, GPS verilerinin alınmasında ki gürültü miktarına göre değişiklik gösterir. GPS Jamming değerinin yüksek olması cihazın konum bilgisinin elde edilememesine, dolayısıyla cihazın düşmesine, kaybolmasına vb. durumlara sebebiyet verebilir. Güvenlik ölçütleri şu şekilde değerlendirilmiştir ;

- 0 40 : GPS Jamming değeri güvenli aralıktatır.
- 40 60 : GPS Jamming değeri risk teşkil eder. Yer kontrol istasyonuna uyarı gönderilecektir.
- 60 > : GPS Jamming değeri tehlike arz eder. Yer kontrol istasyonuna uyarı gönderilecek ve pilottan komut beklenecektir.

#### • Batarya doluluk oranı

Batarya doluluk oranının gözetilmemesi cihazın bataryası bittiğinde düşmesine, donanımının doğru çalışmamasına vb. problemlere yol açabilir. Güvenlik ölçütleri şu şekilde değerlendirilmiştir ; (Yüzdelik oran belirlenmemiştir)

- Düşük : Yer kontrol istasyonuna "Yer kontrol noktasına dön" uyarısı verilecektir.
- Kritik : 5 sn için HOLD-MODE moduna geç ve pilottan komut bekle. Komut gelmezse yer kontrol noktasına dön.
- Tehlike : Olduğun yere iniş yap. Yer kontrol istasyonuna uyarı gönder.

#### • İlaç tankı doluluk oranı

İlaç tankının boşalması durumunda (ilaç bitmiş olabilir, ilaç tankı bağlantısında problem olabilir vb.) cihaz görevini gerçekleştiremez. İlaç tankının tekrar doldurulması için güvenlik ölçütleri şu şekilde değerlendirilmiştir;

- Dolu : İlaç tankında ilaç var
- Boş : İlaç tankı boş, yer kontrol istasyonuna uyarı gönder, yer kontrol noktasına dön.

### • Link kaybı

Cihazın, pilot tarafından kontrol edilmesini sağlayan bağlantının kesildiği anlamına gelir. Güvenlik ölçütleri şu şekilde değerlendirilmiştir ;

- GPS Var: Yer kontrol istasyonuna mesaj gönder ve yer kontrol noktasına dön
- GPS Yok: Bulunduğun noktaya acil iniş yap

Yukarıda sıralanmış olan öngördüğümüz acil durumlar cihaz, uçuş esnasında iken yaşanabilecek acil durumları arz eder. Kalkış ve iniş modlarında gerçekleşebilecek olası acil durumlar ise hata olarak nitelendirilmiştir.

# **3.1.3 Hata Durumları**

zirai amaçlı ilaçlama kopterinde, uçuş ve iniş modlarının aktif olduğu anlarda hataya sebebiyet veren durumlar için ;

Cihaz kalkış yaparken, görev başlangıç noktasına giderken, görevi bitirdikten sonra belirlenen noktaya dönerken ya da iniş yaparken

• İlaç püskürtme aktif ise

,aslında yerine getirilen işlevlerin, istenilen görev sırasına göre gerçekleşmediği ve otopilot yazılımından, yer kontrol istasyonu konfigürasyonlarından ya da donanımsal bir arızadan kaynaklanan hata olduğunu gösterir.

# **3.2 Eklenilmesi Planlanan Yeni Özelliklerin Uygulama Kosepti**

zirai amaçlı ilaçlama kopterinin otopilot yazılımı olan Otopilot'unda, QGroundControl yer kontrol istasyonundan yapılandırılan bir coğrafi sınır (geofence) parametresi mevcuttu. Otopilot yazılımının alt yapısı olan PX4 te bu özellik, cihazın belirlenen coğrafi sınırın dışına çıkmaması için kullanılıyordu. Otopilotunda ise zirai modülüne implemente edilen bu özellik, cihazın, verilen coğrafi sınırsın içinde mi dışında mı olduğunu anlamaya yarayan bir değer döndürüyor.

Eknenecek yeni özellikler burada bahsedilen zirai modülü üzerine inşaa edilecektir.

# **3.2.1 Cihazın Coğrafi Sınır Toleransının Ayarlanması**

"Uygulama Esnasında Karşılaşılabilecek Olası Durumların Ele Alınması" başlığı altında eklenilmesi planlanan yeni özelliklerden ilki cihazın, sanal coğrafi sınırdan ±5m uzaklaşması halinde pilottan komut beklemek üzere 5 sn için HOLD-MODE moduna geçişmesi, ardından 5 sn içerisinde herhangi bir komut alınmaması durumunda cihazın yer kontrol noktasına otomatik olarak geri dönmesi özelliği idi.

Hali hazırda zirai modülünde var olan coğrafi sınır kontrolünde cihazın anlık pozisyonuna göre 'navigator' modülününde 'geofence' bloğunda bulunan 'isInsidePolygonOrCircle' fonksiyonunda ki "Point Inclusion in Polygon Test – PNPOLY" algoritması kullanılır.

Ve PX4 otopilot yazılımı olarak yapılandırılmadan önce coğrafi sınır noktasına yaklaştığını anlayan ve uyarı verip cihazı durduran bir başka algoritmaya daha sahipti (zirai modülü alt yapısı hazırlanırken kullanılmadığından iptal edildi!). Bu algoritma, otopilot yazılımının 'navigator' modülünde zirai modülünün de bulunduğu 'navigator\_main.cpp' bloğunda ve 'GeofenceBreachAvoidance' toplu bloğunda yer alan 'geofence\_breach\_checker' ve 'getFenceViolationTestPoint' fonksiyonları sayesinde gerçekleştirilmektedir. Algoritma basitçe cihazın bulunduğu noktadan coğrafi sınıra olan yen yakın noktaya olan uzaklığı hesaplayarak (Distance of point from line) bir tampon mesafesi belirler. İki nokta arasında ki uzaklık tampon mesafesinden az ise cihaz sınırı ihlal etmemek için hız keser ve mod değiştirir.

![](_page_9_Figure_8.jpeg)

*Şekil 1 : [https://d1whtlypfis84e.cloudfront.net/guides/wp-content/uploads/2018/05/23153857/distance1-](https://d1whtlypfis84e.cloudfront.net/guides/wp-content/uploads/2018/05/23153857/distance1-300x194.png) [300x194.png](https://d1whtlypfis84e.cloudfront.net/guides/wp-content/uploads/2018/05/23153857/distance1-300x194.png)*

Söz konusu coğrafi sınır toleransı özelliğinin eklenebilmesi için bu iki algoritmadan yararlanılmalıdır. Poligonun içinde kalan kısım zaten zirai arazisi olacağından ve cihaz bu kısımda görev rotasını takip ediyor olacağından coğrafi sınırın içerisine doğru tolerans vermeye gerek yok, sadece poligonun dışına doğru 5 metrelik bir tampon bölge oluşturulmalı. Dolayısıyla şartlandırma cihazın poligon dışarısında ve 5 metre uzaklıkta olmasına bağlı.

Bu durumda;

SÖZDE KOD:

```
1. // ./src/modules/navigator/navigator_main.cpp
2. void Navigator::_modul_main(bool &have_geofence_position_data, double lat, double lon, double alt)
3. {
4. #######
5. if(!(PoligonunIcinde()))
6. {
7. İf(Cografi_Sinir_Uzakligi >= 5m)
8. {
9. _puskurtme_kapali();
10. HOLD_MODE();
11. Yer_Kontrol_istasyonuna_Mesaj_Gonder();
12. if(Komut_Geldi() && (Mesaj_Gonderme_Suresi < 5000ms))
13. {
14. Komutu_isle();
15. }
16. else
17. {
18. Yer_Kontrol_Noktasina_Don();
19. }
20. }
21. else
22. {
23. İlac_puskurtme_kapali();
24. }
25. }
26. else
27. {
28. İlac_puskurtme_aktif();
29. }
30. #######
31. }
32.
```

Otopilota kazandırmaya çalıştığımız özelliğin genel şartlandırması bu şekilde olacaktır.

Bu şartlandırma şunu sağlar ;

- Eğer cihaz coğrafi sınır içerisinde ise ilaç püskürtme işlemine devam eder.
- Eğer cihaz coğrafi sınır dışarısında ise VE coğrafi sınıra olan uzaklığı 5 metre veya daha fazla İSE ilaç püskürtme işlemi sonlandırılır ve HOLD\_MODE moduna geçerek yer kontrol istasyonuna, pilottan komut almak üzere bir uyar mesajı gönderilir. Mesaj gönderildiği zaman bilgisi hafızada tutulur. Eğer mesaj gönderilmesinin üzerinden 5 sn geçmedi VE pilottan komut geldi İSE komut işlenir, eğer mesaj gönderme süresi 5 sn den fazla sürmüş İSE cihaz yer kontrol noktasına döner.

# **3.2.2 Cihazın Göreve Kaldığı Noktadan Devam Etmesi**

Cihaz, uygulama esnasında herhangi bir acil durumla karşılaşabilir (batarya/ilaç bitmesi, GPS gürültüsü vs.), bu durumda cihazın alması gereken kararlar "Ele Alınması Gereken Acil Durumlar" başlığı altında paylaşılmıştır. Fakat alınan kararlar cihaz konumunda değişikliğe sebep olursa zirai arazisinin ilaçlanma işlemine en baştan başlanması istenmez. Bu nedenle cihaz, her döngüde bulunduğu konumu bir hafıza yapısında kayıt altına almalıdır.

Konum bilgisini hafızaya kaydetmenin birden fazla yöntemi olabilir (dizi/matris oluşturma, harici depolama aygıtına kaydetme vb.). Ayrıca yapılan işlemlerin kaydınında alınması istenildiğinden (seyir defteri) her bir durum ve alınan kararda konum bilgisinin hafızada tutulması önem arz eder.

Cihaz çalışırken gerçekleşen işlemlerin her biri cihaza bağlı bir SD karta kaydedilir. Dolayısıyla öngörülen durumların her birini bir diziye atmalı, her bir durum kendine kayıtlı olan dizi elemanını güncellemelidir ve bu dizi sürekli SD karta aktarılmalıdır.

#### O zaman ;

- Acil durum halinde konum bilgilerini (enlem, boylam, yükseklik) tutacak ve gerekli indexte saklanmasını sağlayacağımız bir matris oluşturulmalıdır. ( double hafıza\_konum [3][acil\_durum\_s size] )
- Oluşturulan 3 x acil durum sayısı kadar eleman içeren matrisin her bir sütunu bir acil durumu temsil eder. Her bir acil durum struct bazında tanımlanabilir ve vektörel şekilde hafızada saklanabilir.

### double son\_konum [ ][\_\_ ]

```
aslında = |
    . 
    . 
    . 
         . 
         . 
         . 
               . 
               . 
               . 
                  … |
```

Bu durumda mekanizmanın sözde kodu;

```
1. void Navigator::_modul_main(bool &have_geofence_position_data, double lat, double lon, double alt)
2. {
3. double son_konum[3][acil durum sayisi];
4. struct konum_parametreleri_s konum;
5. struct acil_durum_s acilDurum;
6. #######
7. if(!(Poligonunİçinde()))
8. {
9. if(acil_durum())
10. {
11. _puskurtme_kapali();
12. acil_durum_işlevleri();
13. konum_kaydet();
14. }
15. else
16. {
17. _puskurtme_kapali();
18. }
19. else
20. {
21. ilaç_puskurtme_aktif();
22. }
23. #######
24. }
25.
26. void Navigator::konum_kaydet()
27. {
```

```
28. double
29. for(i=0;i<konum_boyutu;i++)
30. {
31. for(j=0;j<acilDurum_boyutu;j++)
32. {
33. son_konum[i][j] = son_konum[i.konum_parametresi][j.acil_durum];
34. }
35. }
36. kaydet(son_konum[i][j]);
37. }
38.
```

Sözde kodda tanımlanan 'konum\_kaydet' işlevi kısaca şunu sağlar ;

• Eğer cihaz poligon dışında iken bir acil durum meydana geldiyse acil\_durum\_s yapısının içerisinde ki hangi acil durum yaşanıyorsa o acil durumun indeksine kadar sayarak teker teker enlem, boylam ve yükseklik bilgilerini o indeksteki vektörün elemanlarına eşitler.

# **3.2.3 Püskürtme Hızının Dinamik Olarak Değişebilmesi**

Cihazın uygulama verimliliğini arttırmak için ilaç püskürtülen alanın genlik değeri belirli bir oranda tutulmalı ve belirli miktarda ilaç püskürtülmelidir. Püskürtme hızının dinamik olarak değiştirilebilmesi özelliği, ilaç israfını azaltma, hedef arazinin sınırları dışında kalan alanların ilaçlanmaması ve mahsüle göre değişkenlik göstererek ilaçlama verimliliğini arttırır.

Püskürtme işleminin değişken olarak gerçekleştirilebilmesi için;

- Yer hızı : Cihazın hızı püskürtme verimliliğini etkiler, hız arttıkça püskürtme hızı artmalıdır
- Yükseklik : Cihazın yüksekliği püskürtme genliğini etkiler, yükseklik arttıkça ilaç püskürtülen alan belli bir noktaya kadar artış gösterir.
- Arazi şekli (eğim, dar alanlar vb.) : zirai arazisi belirli alanlarda daralıp genişliyor olabilir, daralan alanlarda yüksekliğin ve püskürtme hızının doğru orantılı olarak azaltılması gerekir.
- Ekin türü : Ekilen mahsüle göre püskürtme işleminin farklı hızlarda farklı yüksekliklerden yapılması gerekebilir.

Bunlar dikkat edilmesi gereken önemli parametrelerdir. Ne var ki ekin türü ne göre hız ve yükseklik parametreleri yer kontrol istasyonu aracılığı ile pilot tarafından belirlenecek olup şu an için otopilot yazılımında önem arz etmemektedir. Arazi şekli ise "Cihazın Coğrafi Sınır Toleransının Ayarlanması" başlığı altında olduğu gibi cihazın poligona en yakın notası ile arasındaki uzaklığa bağlı olarak hesaplanacaktır. Dolayısıyla yer hızı ve yükseklik parametrelerine bağlı değişim birbirlerine orantılıyken arazi şekline bağlı püskürtme hızında ki değişiklik diğer parametrelerden bağımsızdır.

Çalışma konsept çerçevesinde gerçekleştirileceğinden ve yer hızı ile yükseklik parametreleri birbirlerini etkilediğinden ilaç pompasına verilen PWM sinyalinin değer aralığıda ilk etapta bu iki parametreye göre ayarlanacak ve 5:5 oranlanacaktır. Yani eğer 0-1000 arasında ki görev dönüsüne sahip bir PWM değeri mevcut ise bu parametreler yardımı ile ayarlanacak olan PID algoritması maksimum ±250 (500) kadar bir görev döngüsü değişikliğine sebep olabilecektir.

Püskürtme hızının arazi şekline göre değişiklik göstermesi ise "Cihazın Coğrafi Sınır Toleransının Ayarlanması" başlığı altında ki poligon dışından 5 m uzaklaşma koşulunun poligon içine uygulanmasıdır. Farklı olarak PID ayarı yapmak yerine poligona yakınlığına göre PWM değer aralığını bir MAP fonksiyonu aracılığı ile değiştirir.

Buna göre pompaya verilen PWM sinyalini ayarlayan PID algoritması şu şekilde oluşturulur ;

```
1. void Navigator::_modul_main(bool &have_geofence_position_data, double lat, double lon, double alt)
2. {
```

```
3. #######
4. if(!(Poligonunİçinde()))
5. {
6. #######
7. else
8. {
9. ilaç_puskurtme_aktif(); // ilaç püskürtme aktif
10. PWM = yerVPID() + altPID();
11. _pompasi_PWM = map_shape(PWM, minPWM, maxPWM, 0, 5m);
12. }
13. #######
14. }
15. void yerVPID()
16. {
17. son_zaman = anlik_zaman;
18. anlik_zaman = zamantut();
19. gecen_zaman = anlik_zaman - son_zaman;
20.
21. hata = anlik_puskurtme_orani - hedef_puskurtme_orani;
22. hiz_PID_oran = oran_hiz * hata;
23. if(hata < x)
24. {
25. hiz_PID_integral = hiz_PID_integral + (integral_hiz * hata);
26. }
27. hiz_PID_turev = turev_hiz * ((hata - eski_hata) / gecen_zaman);
28. hiz_PID = hiz_PID_oran + hiz_PID_integral + hiz_PID_turev;
29. eski_hata = hata;
30. }
31. void altPID()
32. {
33. son_zaman = anlik_zaman;
34. anlik_zaman = zamantut();
35. gecen_zaman = anlik_zaman - son_zaman;
36.
37. hata = anlik_puskurtme_orani - hedef_puskurtme_orani;
38. alt_PID_oran = oran_alt * hata;
39. if(hata < x)
40. {
41. alt_PID_integral = alt_PID_integral + (integral_alt * hata);
42. }
43. alt_PID_turev = turev_alt * ((hata - eski_hata) / gecen_zaman);
44. alt_PID = alt_PID_oran + alt_PID_integral + alt_PID_turev;
45. eski_hata = hata;
46. }
47. uint64_t map_shape(int pwmIn, int pwmMin, int pwmMax, double distMin, double distMax)
48. {
49. return (pwmIn - pwmMin) * (distMax - distMin) / (pwmMax - pwmMin) + distMin;
50. }
51.
```

Değişken püskürtme özelliğinin kazandırılmasına yönelik yapılması planlanan kodlama çalışmasına dair konsept fikir bu şekilde ifade edilmektedir.

# **3.3 Ele Alınması Gereken Acil Durumlar**

Otopilot yazılımına kazandırılması gereken özellikler haricinde müşteri beklentilerinin tam anlamı ile karşılanabilmesi ve verimliliğin arttırılabilmesi için cihazın belirli durumları değerlendirebilmesi ve doğru kararları döndürebilmesi gerekmektedir. Yapılan planlamalar sonucu bu durumlar 'acil durumlar' olarak adlandırılmış olup cihazın yazılımsal ve donanımsal gereklerinden kaynaklanan hataları çözmeye yönelik bir çalışmadır.

Var olan hataların kaynakları ;

- Hali hazırda var olan zirai modülü, ilaç püskürtme işlemini sadece coğrafi sınır içerisinde bulunup bulunmadığı bilgisine göre gerçekleştirmekte olup uçuş modu gözetmez.
- Batarya doluluğu ve ilaç tankı doluluğu izlenmediğinden cihaz ilaç bitse dahi ilaç püskürtüyormuş gibi görevine devam etmeye çalışacak ya da batarya bittiğinde pilotu bilgilendirmeden arazide düşüp kalacaktır.
- GPS gürültüsü izlenmediğinden konum bilgisi kaybı durumunda cihaz rüzgar, yanlış haritalandırma vb. sebeplerle başka arazilere savrulabilir.

olarak gösterilebilir.

Acil durumlarda ele alınan durumlar farklı modüllerde bulunmaktadır fakat zirai modülü "navigator" modülü içerisinde yer almaktadır. Dolayısıyla acil durumların hepsinde aynı mekanizmayı zirai modulune implemente etmek için modüller arası etkileşim gerekiyor. Bu nedenle acil durum bilgisini alacağımız modüller ile zirai modülünü barındıran 'navigator' modülü arasında bir uORB ağı kurulmalıdır. Amaç her bir acil durumu gözeten kapsamlı bir şartlandırma inşa etmektir. Daha sonra elde edilen veriler MAVLink protokolü aracılığı ile 'navigator' modülünden yer kontrol istasyonuna iletilebilir.

# **3.3.1 GPS Gürültüsünün İzlenmesi**

GPS gürülsünden kaynaklı durumlara karşı alınacak kararlar "navigator" modülünde ki "gpsfailture" bloğunda ele alınmıştır. Söz konusu mekanizmaların, "navigator\_main" bloğunda ki zirai modülüne bağlanabilmesi için gps kaybı durumları sonucu bir değer ataması yapılarak "\_MODULU" adı altında uORB kanalında yayınlanmalıdır. Bu değere "gps\_fail" adını verelim ;

```
1. void Navigator::_modul_main(bool &have_geofence_position_data, double lat, double lon, double alt)
2. {
3. #######
4. struct _modulu_nav_s Nav;
5. uORB::Subscription __modulu_sub
6. {
7. ORB_ID(_modulu), 0
8. };
9. #######
10. if(!(Poligonunİçinde()))
11. {
12. if(acil_durum())
13. {
14. _puskurtme_kapali();
15. switch acil_durum
16. case acil_durum == gps_fail
17. if(40 < gpsjam < 60)
18. {
19. Yer_Kontrol_istasyonuna_Mesaj_Gonder();
20. }
21. if(gpsjam > 60)
22. {
23. HOLD_MODE();
24. Yer_Kontrol_istasyonuna_Mesaj_Gonder();
25. if(Komut_Geldi() && (Mesaj_Gonderme_Suresi < 5000ms))
26. {
```

```
27. Komutu_isle();
28. }
29. else
30. {
31. Yer_Kontrol_Noktasina_Don();
32. }
33. }
34. else
35. {
36. break;
37. }
38. break;
39. #######
40. }
41. else
42. {
43. _puskurtme_kapali();
44. }
45. }
46. else
47. {
48. ilaç_puskurtme_aktif();
49. }
50. #######
51. }
52.
```

# **3.3.2 Batarya Doluluk Oranının İzlenmesi**

Cihaz yazımı, son hali ile batarya doluluk oranını gözetmiyor fakat temel PX4 yapısında batarya doluluk oranına göre otopilot için bir karar mekanizması mevcut. Bu karar mekanizmasına göre batarya doluluk oranı düşük olduğunda (varsayılan : %50 den az ise) kullanıcıya iniş uyarısı yapılıyor. Kritik düzeyde ise kontrol noktasına dönüş uyarısı yapılıyor ve otomatik olarak kontrol noktasına dönüyor ve tehlike düzeyinde ise acil iniş yapılıyor.

```
1. void Navigator::_modul_main(bool &have_geofence_position_data, double lat, double lon, double alt)
2. {
3. #######
4. struct _modulu_nav_s Nav;
5. uORB::Subscription __modulu_sub
6. {
7. ORB_ID(_modulu), 0
8. };
9. #######
10. if(!(Poligonunİçinde()))
11. {
12. if(acil_durum())
13. {
14. _puskurtme_kapali();
15. switch acil_durum
16. #######
17. case acil_durum == batarya
18. if(30 < batarya_durum < 50)
19. {
20. Yer_Kontrol_istasyonuna_Mesaj_Gonder();
21. }
22. if(10 < batarya_durum < 30)
23. {
24. HOLD_MODE();
25. Yer_Kontrol_istasyonuna_Mesaj_Gonder();
26. if(Komut_Geldi() && (Mesaj_Gonderme_Suresi < 5000ms))
27. {
28. Komutu_isle();
29. }
30. else
```

```
31. {
32. Yer_Kontrol_Noktasina_Don();
33. }
34. }
35. if(batarya_durum < 10)
36. {
37. LAND_MODE();
38. Yer_Kontrol_istasyonuna_Mesaj_Gonder();
39. }
40. else
41. {
42. break;
43. }
44. break;
45. #######
46. }
47. else
48. {
49. _puskurtme_kapali();
50. }
51. }
52. else
53. {
54. ilaç_puskurtme_aktif();
55. }
56. #######
57. }
58.
```

# **3.3.3 İlaç Tankı Doluluk Oranının İzlenmesi**

İlaç tankında ki ilaç doluluk oranına bakmaksızın sadece ilaç olup olmamasını gözetiyoruz. Dolayısıyla sadece bir "\_yok" acil durumu eklenecek ;

```
1. void Navigator::_modul_main(bool &have_geofence_position_data, double lat, double lon, double alt)
2. {
3. #######
4. struct _modulu_nav_s Nav;
5. uORB::Subscription __modulu_sub
6. {
7. ORB_ID(_modulu), 0
8. };
9. #######
10. if(!(Poligonunİçinde()))
11. {
12. if(acil_durum())
13. {
14. _puskurtme_kapali();
15. switch acil_durum
16. #######
17. case acil_durum == _yok
18. Yer_Kontrol_istasyonuna_Mesaj_Gonder();
19. Yer_Kontrol_Noktasina_Don();
20. break;
21. #######
22. }
23. else
24. {
25. _puskurtme_kapali();
26. }
27. }
28. else
29. {
```

```
30. ilaç_puskurtme_aktif();
31. }
32. #######
33. }
34.
```

# **3.3.4 Link Kaybı Durumunda Karar Mekanizması**

Cihaz ile yer kontrol istasyonu arasında sürekli bir "HEARTBEAT" sinyali alışverişi vardır. Bu sinyal sürekli belirli bir sürede tekrarlanarak gönderilir. Sinyalin gelmesi uzun sürerse/gelmezse bu link kaybı yaşandığı anlamına gelir. Dolayısıyla son şartlandırmada "HEARTBEAT" mesajı gözetilmelidir.

```
1. void Navigator::_modul_main(bool &have_geofence_position_data, double lat, double lon, double alt)
2. {
3. #######
4. struct _modulu_nav_s Nav;
5. uORB::Subscription __modulu_sub
6. {
7. ORB_ID(_modulu), 0
8. };
9. #######
10. if(!(Poligonunİçinde()))
11. {
12. if(acil_durum())
13. {
14. _puskurtme_kapali();
15. switch acil_durum
16. #######
17. case acil_durum == HEARTBEAT_YOK
18. if(gps_fail)
19. {
20. LAND_MODE();
21. }
22. else
23. {
24. Yer_Kontrol_Noktasina_Don();
25. }
26. break;
27. default;
28. }
29. else
30. {
31. _puskurtme_kapali();
32. }
33. }
34. else
35. {
36. ilaç_puskurtme_aktif();
37. }
38. #######
39. }
40.
```

Link kaybı acil durumuna göre aynı anda GPS kaybı varsa bulunduğu konuma iniş yapar. Eğer GPS kaybı yoksa yer kontrol noktasına döner.