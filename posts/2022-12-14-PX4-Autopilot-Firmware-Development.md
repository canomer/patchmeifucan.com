## İçindekiler

| 1 GİRİŞ                                                         | 5  |
|-----------------------------------------------------------------|----|
| 2 ÖZET                                                          | 6  |
| 3 METODOLOJİ                                                    | 7  |
| 4  MODULU İŞLEVLERİ (HATALI)                               | 10 |
| 5  MODULU ANA İŞLEVİ - navigator                           | 17 |
| 6  MODULU YER KONTROL İSTASYONUNA MESAJ GÖNDERME - MAVLink | 24 |
| 7 ACİL DURUMLARIN ELE ALINMASI - commander                      | 28 |
| 7.1 Acil Durumlar                                               | 28 |
| 7.1.1 Batarya Doluluğu                                          | 29 |
| 7.1.2 GPS Jamming                                               | 32 |
| 7.1.3 Link Kaybı                                                | 33 |
| 7.1.4 Pilot Müdahalesi                                          | 38 |
| 7.2 Karar Verme                                                 | 39 |
| 8 SONUÇ                                                         | 44 |
| 8.1 Tüm Kod                                                     | 44 |
| 8.1.1  modulu.msg                                          | 44 |
| 8.1.2 _commander.msg                                       |    |
| 8.1.3 msg/CMakeLists.txt                                        | 44 |
| 8.1.4 navigator_main.cpp                                        | 45 |
| 8.1.5 navigator.h                                               | 50 |
| 8.1.6 MODUL_poligon.cpp                                  | 52 |
| 8.1.7 MODUL_poligon.h                                    | 57 |
| 8.1.8 navigator/CMakeLists.txt                                  | 57 |
| 8.1.9 commander.cpp                                             | 58 |
| 8.1.10 commander.hpp                                            | 61 |
| 8.1.10 state_machine_helper.h                                   | 61 |
| 8.1.12 state_machine_helper.cpp                                 | 62 |
| 8.1.13 mavlink_main.cpp                                         | 65 |
| 8.1.14 mavlink_messages.cpp                                     | 65 |
| 8.1.15 mavlink_receiver.cpp                                     | 65 |
| 8.1.16 mavlink_receiver.h                                       | 66 |
| 8.1.17 MODUL.hpp                                         |    |
| 8.1.18 mavlink_mission.cpp                                      | 68 |
| 8.1.19 navigation.h                                             | 74 |
| O 2 Marlan                                                      | 75 |

## **1** GİRİŞ

Zirai amaçlı lama kopteri geliştirilecektir. Cihazın görevini beklendiği üzere gerçekleştirebilmesi için gerekli görevlerin otopilot yazılımı Modül sayesinde gerçekleştirilmesi hedeflenmektedir.

Zirai lama kopterinde kullanılacak olan PX4 tabanlı otopilot yazılımının oluşturulması için PX4 otopilot yazılımına ek işlevsellikler kazandırılması gerekmektedir.

Modül yazılımı, ilk sürümü itibari ile yer kontrol istasyonu aracılığı ile belirlenen coğrafi sınırlar ve uçuş rotası bilgilerini kullanarak cihazın rota üzerinde iken coğrafi sınırlar içerisinde  püskürtmesi ve coğrafi sınırları ihlal etmesi halinde  püskürtme işlemini sonlandırması özelliği kazandırılmıştır.

Sonrasında cihazın çalışması esnasında yaşanması olası aksaklıklar ve cihaza kazandırılması istenen yeni özellikler tartışılmıştır ve tartışmalara istinaden planlar yapılmış, konsept çevçevesinde incelenmiştir. cihazda otopilot yazılımı ile olası acil durumları tanıma ve önlem alarak bulunduğu son konumu kaydedebilme özellikleri kazandırılmıştır.

Son olarak, otopilot yazılımının temeli olan PX4 ve yer kontrol istasyonu yazılımı QGroundControl yazılımlarının varolan özelliklerini değiştirmeden/kaldırmadan işlevlerinin kazandırılması için işlevlerin ayrı kod bloklarından ve Hamdi EVKAYA tarafından kodlama çalışmaları yürütülen yeni QGroundControl yazılımından gönderilen coğrafi sınır MAVLink mesajı aracılığı ile gerçekleştirilmesi hedeflenmiştir.

## **2** ÖZET

Otopilot yazılımı Modül işlevlerinden beklenen, zirai lama kopterinin yer kontrol istasyonundan sağlanan coğrafi sınır (tarım arazisi) lokasyon bilgileri doğrultusunda araziyi belirli aralıklarla tarayarak lama yapmasını sağlamasıdır. Bunun için cihazın ;

- Tarım arazisinin içindemi dışındamı bulunduğunu anlık olarak kontrol etmesi
- Tarım arazisi sınırları içerisinde ise lama yapması
- Tarım arazisi sınırları dışarısında ise  püskürtmemesi
- Olası arızalanma, aksaklık vb. durumlarda lamayı sonlandırması ve yer kontrol noktasına dönmesi ya da yer kontrol istasyonuna uyarı göndermesi

İşlevlerini yerine getirebilmesi beklenmektedir.

Cihazın, arazi sınırları içerisinde bulunup bulunmadığını kontrol etmesi 'Navigator' modülünün 'geofence' bloğunda ki işlevler ile anlaşılır. 'geofence' işlevleri, GPS donanım modülünden alınan anlık lokasyon bilgilerini ve yer kontrol istasyonundan gönderilen arazi sınırları bilgilerini kullanarak belirli algoritmalar yardımı ile cihazın arazi sınırları içerisinde olup olmadığı bilgisini döndüren algoritmalar kullanmaktadır.

Modül kod bloğunun, bu işlevleri kullanarak bir uORB kanalı aracılığı ile arazi sınırları içerisinde bulunup bulunmadığı bilgisini MAVLink modülü ile paylaşmaktadır. MAVLink modülü, MAVLink iletişim protokolünü kullanarak, anlık olarak lama yapılıp yapılmadığı bilgisini yer kontrol istasyonuna göndermektedir ve yer kontrol istasyonundan anlık olarak izlenebilmektedir.

Söz konuzu işlevler kullacını beklentilerinin tam anlamı ile karşılanabilmesi için yeterli olmaması sebebiyle cihazın kullanımı sırasında karşılaşılabilecek olası aksaklıklar ve arızalar hesaba katılarak yeni işlevler otopilot yazılımına eklenmelidir.

Yapılan çalışmalar kapsamında belirlenen ;

- GPS gürültü değerleri
- Kumanda bağlantısı kopması
- Batarya doluluk oranı
- İlaç tankı doluluk oranı

Olası aksaklık durumları ve ;

- Coğrafi sınır toleransı
- Herhangi bir aksaklık durumunda cihazın bulunduğu son konumu kaydetmesi ve bu durumların çözülmesi durumunda görevine kaldığı yerden devam edebilmesi
- Yer hızı, yükseklik ve sınır kenarlarına olan uzaklık parametrelerine bağlı olarak  püskürtme hızının dinamik olarak değiştirilmesi

Özellikleri, otopilot yazılımının 'Commander' modülünden yararlanılarak uORB kanalı üzerinden 'Navigator' modülüne iletilmiş ve Modül işlevlerinde kullanılmıştır.

Son olarak, işlevler doğrultusunda elde edilen veriler kullanılmak üzere yer kontrol istasyonuna iletilmiştir.

## **3** METODOLOJİ

Planlama çalışmaları sonucu öngörülen problemlerin otopilot yazılımında ki navigator modülünde hali hazırda mevcut olan "_modul\_main" fonksiyonuna eklenecek yeni işlevler sayesinde Otopilot yazılımına kazandırılacak yeni özellikler ile çözülmesi amaçlanmaktadır.

Yapılacak kodlama çalışmaları Visual Studio Code kod editörü yazılımı ile gerçekleştirilmiştir.

Simülasyon çalışmaları Gazebo simülasyon ve QGroundControl yer kontrol istasyonu yazılımları aracılığı ile gerçekleştirilmiştir.

Bahsi geçen çalışmalar VirtualBox yazılımında Ubuntu 20.04 LTS işletim sistemi kurulu bir sanal makine üzerinde gerçekleştirilmiştir.

Otopilot yazılımı içerisinde ki modüllerin haberleşmesinde uORB, otopilot üzerinden yer kontrol istasyonuna mesaj gönderilmesi için kullanılan haberleşme yöteminde ise MAVLink iletişim protokolü ile kullanılmıştır.

Yer kontrol istasyonundan otopilota mesaj gönderme işlemi ise geliştirilen MAVLink-Mesajlaşma yazılımı ile simüle edilmiştir.

### Çalışmalar ;

- Çalışma amacının belirlenmesi
- Amaca yönelik bir plan oluşturulması
- Plan doğrultusunda konsept niteliğinde çalışmaların yapılması
- Kodlama çalışmaları
- Test ve doğrulama çalışmaları
- Sonuçlar

#### Şeklinde ilerletilmiştir.

Amaç, zirai lama kopterinde kullanılmak üzere bir otopilot yazılımı geliştirmektir. Bahsi geçen otopilot yazılımı PX4 tabanlı olup eklenen işlevler ve yapılandırılmalar doğrultusunda olarak isimlendirilecektir. Söz konusu işlevleri de otopilot yazılımı içerisinde bulunan modül sağlamaktadır.

Modül, QGroundControl yer kontrol istasyonundan alınan arazi sınırları koordinat bilgileri doğrultusunda cihazın lama yapıp yapmayacağını belirler ve herhangi bir aksaklık yaşanması durumunda cihazın alacağı kararı belirleyen bir mekanizmaya sahiptir.

Arazi sınırlarının ihlal edilip edilmediği bilgisi hali hazırda PX4 otopilot yazılımında bulunan 'Navigator' modülünün 'geofence' bloğunda ki işlevler vasıtası ile elde edilebilmektedir. Modül aynı işlevleri, 'Navigator' modülünde ki '_modul\_poligon' bloğundan çekerek '_modul\_main' kod bloğunda kullanmaktadır. Modülü kod bloğu 'navigator\_main' bloğunda yer almakta olup asıl program akışı buradan gerçekleşir.

Arazi sınırı ihlal bilgili alındıktan sonra '_modul' uORB kanalı üzerinden 'MAVLink' modülüne iletilir ve ayrıştırılır. Ardından yer kontrol istasyonuna gönderilir.

![](_page_8_Figure_1.jpeg)

Aksaklıklar karşısında karar alma mekanizması ise 'Commander' modülüne bağlı '\_commander' ve 'sensor\_gps' uORB kanalları üzerinden elde edilen verilen doğrultusunda gerçekleşir.

GPS gürültüsü 'sensor\_gps' uORB kanalı üzerinden izlenmektedir ve belirli aralıklarda ki gürültü değerleri izlenerek 'Commander' modülüne alınması gereken karara dair '_modul' üzerinden veri gönderilir.

Link bağlantısı, batarya seviyesine göre karar alma gibi durumlar ise 'Commander' modülünde izlenerek herhangi bir aksiyon alınması gerektiğinde önce '\_commander' üzerinden 'Navigator' modülüne gönderilir, ardından alınacak aksiyon bilgisi 'Navigator' modülünden 'Commander' modülüne gönderilir. 'Commander' modülünde, gelen veri doğrultusunda işlenecek karar mekanizması çalıştırılır.

Ayrıca herhangi bir aksaklık yaşanması vb. durumlarda cihaz '_modul\_main' içerisinde bulunduğu son konumu bir diziye kaydeder.

![](_page_8_Figure_6.jpeg)

## **4**  MODULU İŞLEVLERİ (HATALI)

## **UYARI : 10.12.22 tarihinde değiştirildi, bu doküman değiştirmeden önceki halini baz almaktadır.**

Modülü işlevlerini gerçekleştirmek üzere kullanılan fonksiyonlar PX4 otopilot yazılımında hali hazırda var olan 'geofence' bloğunda ki işlevlerle aynı olup farklı parametreleri kullandıklarından '_modul\_poligon' bloğu olarak ayrılmıştır.

Modülünün temel işlevi olan arazi sınırlarının ihlali kontrolü ve arazi sınır/köşe noktalarına olan uzaklık hesaplamaları bu blokta ki işlevler aracılığı ile gerçekleştirilir.

Arazi sınırlarının bir 'polygon' olarak indekslenmesi ;

```
./src/modules/navigator/MODUL_poligon.cpp
void Poligon::zs_updateFence()
{
 zs__poligon_s zs__poligon;
 mission_stats_entry_s stats;
 int ret2 = dm_read(DM_KEY_ZS__POLIGON, 0, &stats, sizeof(mission_stats_entry_s));
 int sizret = sizeof(mission_stats_entry_s);
 int num_fence_items = 0;
 if (ret2 == sizeof(mission_stats_entry_s)) {
 num_fence_items = stats.num_items;
 _update_counter = stats.update_counter;
 }
 _num_polygons = 0;
 int current_seq = 1;
 while (current_seq <= num_fence_items) {
 if (dm_read(DM_KEY_ZS__POLIGON, current_seq, &zs__poligon, si
 zeof(zs__poligon_s)) !=
 sizeof(zs__poligon_s)) {
 break;
 }
 switch (zs__poligon.nav_cmd) {
 case NAV_CMD_LAMA_KOSE:
 PX4_WARN("update case de");
 if (_polygons) {
 PX4_WARN("update if poligonda");
 PolygonInfo *new_polygons = new PolygonInfo[_num_poly
 gons + 1];
 if (new_polygons)
 {
 memcpy(new_polygons, _polygons, sizeof(Polygo
 nInfo) * _num_polygons);
 }
                        delete[](_polygons);
                        _polygons = new_polygons;
 }
                   else
                   {
 _polygons = new PolygonInfo[1];
 }
```

```
 if (!_polygons)
                {
 _num_polygons = 0;
 PX4_ERR("alloc failed");
 return;
 }
                PolygonInfo &polygon = _polygons[_num_polygons];
 polygon.dataman_index = current_seq;
 polygon.fence_type = zs__poligon.nav_cmd;
 polygon.vertex_count = zs__poligon.vertex_count;
 current_seq += zs__poligon.vertex_count;
 ++_num_polygons;
                PX4_WARN("poligon : %d num_polygon %d", polygon.vertex_count, 
 _num_polygons);
 break;
 }
 }
}
```

'zs\_updateFence()' işlevi, DM\_READ işlevi ile okunan değerler './src/navigator/navigation.h' konumunda tanımlı 'zs\_\_poligon\_s' yapısına eşitlendikten sonra, her bir arazi köşe noktası için bir 'polygon' tanımlar. Köşe sayısı değeri okunan değer doğrulandıktan sonra çekilir.

Coğrafi sınır ihlali kontrolü ;

```
./src/modules/navigator/MODUL_poligon.cpp
bool Poligon::isInsidePolygonOrCircle(double lat, double lon, float altitude)
{
 mission_stats_entry_s stats;
 int ret = dm_read(DM_KEY_ZS__POLIGON, 0, &stats, sizeof(mission_stats_entry_s));
 if (ret == sizeof(mission_stats_entry_s) && _update_counter != stats.update_counter)
 {
 zs_updateFence();
 }
 for (int polygon_idx = 0; polygon_idx < _num_polygons; ++polygon_idx)
 {
 bool inside = insidePolygon(_polygons[polygon_idx], lat, lon, altitude);
 if (inside)
 {
 _zs_inside__poligon = true;
 }
```

```
 else
 {
 _zs_inside__poligon = false;
 }
 }
 return _zs_inside__poligon;
}
bool Poligon::insidePolygon(const PolygonInfo &polygon, double lat, double lon, float alti-
tude)
{
 zs__poligon_s temp_vertex_i;
 zs__poligon_s temp_vertex_j;
 bool c = false;
 for (unsigned i = 0, j = polygon.vertex_count - 1; i < polygon.vertex_count; j = i++) {
 if (dm_read(DM_KEY_ZS__POLIGON, polygon.dataman_index + i, &temp_vertex_i,
 sizeof(zs__poligon_s)) != sizeof(zs__poligon_s)) {
 break;
 }
 if (dm_read(DM_KEY_ZS__POLIGON, polygon.dataman_index + j, &temp_vertex_j,
 sizeof(zs__poligon_s)) != sizeof(zs__poligon_s)) {
 break;
 }
 if (temp_vertex_i.frame != NAV_FRAME_GLOBAL && temp_vertex_i.frame != 
 NAV_FRAME_GLOBAL_INT
 && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT
 && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT_INT) {
 PX4_ERR("Frame type %i not supported <-- ./navigator/MODUL_poli
 gon/'insidePolygon'", (int)temp_vertex_i.frame);
 break;
 }
 if (((double)temp_vertex_i.lon >= lon) != ((double)temp_vertex_j.lon >= lon) &&
 (lat <= (double)(temp_vertex_j.lat - temp_vertex_i.lat) * (lon - (do
 uble)temp_vertex_i.lon) / (double)(temp_vertex_j.lon - temp_vertex_i.lon) + 
 (double)temp_vertex_i.lat)) {
 c = !c;
 }
 }
 return c;
}
```

'isInsidePolygonOrCircle' işlevi önce 'mission\_stats\_entry\_s' yapısının 'update\_counter' parametresinde bir değişiklik olup olmadığını kontrol eder. Eğer değişiklik varsa bu, arazi sınırları bilgilerinde bir güncelleme işlemi yapıldığı anlamına gelir ve eğer güncelleme yapıldı ise yeni bilgileri günceller.

'isInsidePolygon' işlevi, data\_manager dan okuduğu verileri indexleyerek arazinin iki köşesi üzerinde sanal bir çizgi çeker ve sanal bir çokgen elde eder. Cihaz koordinatlarına bakarak cihazın bu çokgenin içinde mi yoksa dışında mı bulunduğu bilgisine göre 'boolean' bir değer döndürür.

### Arazi sınırına uzaklık ; (**Kullannılmadı)**

```
./src/modules/navigator/MODUL_poligon.cpp
double Poligon::poligonUzaklik(double lat, double lon)
{
 zs__poligon_s temp_vertex_i;
 zs__poligon_s temp_vertex_j;
 double uzaklik = 0;
 double coefA =0;
 double coefB =0;
 double coefC =0;
 for (int polygon_idx = 0; polygon_idx < _num_polygons; ++polygon_idx) {
 for (unsigned i = 0, j = _polygons[polygon_idx].vertex_count - 1; i < _poly-
gons[polygon_idx].vertex_count; j = i++)
 {
 if (dm_read(DM_KEY_ZS__POLIGON, _polygons[polygon_idx].dataman_index 
+ i, &temp_vertex_i, sizeof(zs__poligon_s)) != sizeof(zs__poligon_s))
 {
 PX4_ERR("UYARI : Poligon Bilgisi %i Alinamadi", (int)temp_ver-
tex_i.frame);
 break;
 }
 if (dm_read(DM_KEY_ZS__POLIGON, _polygons[polygon_idx].dataman_index 
+ j, &temp_vertex_j, sizeof(zs__poligon_s)) != sizeof(zs__poligon_s))
 {
 PX4_ERR("UYARI : Poligon Bilgisi %i Alinamadi", (int)temp_ver-
tex_j.frame);
 break;
 }
 if (temp_vertex_i.frame != NAV_FRAME_GLOBAL && temp_vertex_i.frame != 
NAV_FRAME_GLOBAL_INT && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT && temp_ver-
tex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT_INT)
 {
 PX4_ERR("Poligon Bilgisi %i Desteklenmiyor", (int)temp_ver-
tex_i.frame);
 break;
 }
```

```
 // Ax + By + C = 0 : dogru denkleminde ki A, B, C katsayilarinin hesap-
lanmasi :
 coefA = (double)temp_vertex_i.lon - (double)temp_vertex_j.lon; // A = 
x.(yi) - x.(yj) : A katsayisi
 coefB = (double)temp_vertex_i.lat - (double)temp_vertex_j.lat; // B = 
y.(xi) - y.(xj) : B katsayisi
 coefC = ((double)temp_vertex_i.lat * (double)temp_vertex_j.lon) - ((do-
uble)temp_vertex_j.lat * (double)temp_vertex_i.lon); // C = (xi).(yj) - (xj).(yi) : C katsayisi
 //Cihazin, en yakin poligon sinirina uzakligi
                    uzaklik = ((coefA * lat) + (coefB * lon) + coefC) / (double)sqrtf((coefA
* coefA) + (coefB * coefB));
 // Cihazin Poligonda kendisine en yakin noktanin uzakligi : d = 
(A.cihaz_lat + B.cihaz_lon + C) / sqrt(A*A + B*B)
 PX4_WARN("temp-i poligonu %f %f temp-j poligonu %f %f Uzaklik %f", (do-
uble)temp_vertex_i.lat, (double)temp_vertex_i.lon, (double)temp_vertex_j.lat, (double)temp_ver-
tex_j.lon, uzaklik);
 }
 }
 return uzaklik;
}
```

'poligonUzaklik' işlevi, analitik geometride ki 'bir noktanın bir doğruya olan uzaklığı' teoremine dayanır.

![](_page_13_Figure_3.jpeg)

A noktası (xi, yi), B noktası (xj, yj), N noktası (kx, ky) olsun. Burada A ve B arazinin o anki uzaklık hesaplanan sınırının köşe noktalarını ifade ederken N noktası cihazın o anki konumunu ifade eder.

```
I. Alan(ABN) = ½ . d . |AB| → d = (2.Alan(ABN)) / |AB|
    II. Alan(ABN) = ½ . kx . (yj – yi) + xj . (yi – ky) + xi . (ky – yi)
    III. |AB| = |C/AB| . sqrt(A . A + B . B)
dir.
     d = (|C/AB| . |Akx + Bky + C|) / (|C/AB| . sqrt(A . A + B . B)
                                           = (| Akx + Bky + C|) / sqrt(A . A + B . B)
```

Ayrıca A ve B noktalarını bildiğimizden yine analitik geometriden "iki noktası bilinen doğrunun denklemi" ;

```
 A = x.(yi) – x.(yj)
     B = y.(xi) – y.(xj)
dir.
```

Köşe uzaklığının hesaplanması ; **(Kullanılmadı)**

```
./src/modules/navigator/MODUL_poligon.cpp
double Poligon::koseUzaklik(double lat, double lon, int numvertex)
{
 zs__poligon_s temp_vertex_i;
 double vertexDist = 0;
 double msgDist = 0;
 double distArray[numvertex];
 for (int polygon_idx = 0; polygon_idx < _num_polygons; ++polygon_idx)
 {
 for (unsigned i = 0; i < _polygons[polygon_idx].vertex_count; i++)
 {
 if (dm_read(DM_KEY_ZS__POLIGON, _polygons[polygon_idx].dataman_index 
                        + i, &temp_vertex_i, sizeof(zs__poligon_s)) != 
                        sizeof(zs__poligon_s))
 {
 PX4_ERR("UYARI : Poligon Bilgisi %i Alinamadi", (int)temp_ver
                             tex_i.frame);
 break;
 }
 if (temp_vertex_i.frame != NAV_FRAME_GLOBAL && temp_vertex_i.frame != 
                        NAV_FRAME_GLOBAL_INT && temp_vertex_i.frame != 
                             NAV_FRAME_GLOBAL_RELATIVE_ALT && temp_vertex_i.frame != 
                                  NAV_FRAME_GLOBAL_RELATIVE_ALT_INT)
 {
 PX4_ERR("Poligon Bilgisi %i Desteklenmiyor", (int)temp_ver
                             tex_i.frame);
 break;
 }
 vertexDist = (double)sqrtf(((temp_vertex_i.lat - lat) * (temp_ver
              tex_i.lat - lat)) + ((temp_vertex_i.lon - lon) * (temp_vertex_i.lon - lon)));
 distArray[i] = vertexDist;
 PX4_WARN("poligon sayisi vertex_count : %d", _polygons[polygon_idx].ver
```

```
tex_count);
 PX4_WARN("poligon sayisi numvertex : %d", numvertex);
 PX4_WARN("poligon koses is correct? : %f", distArray[i]);
 }
 }
 return 0;
}
```

Bu işlev sürekli olarak arazinin tüm köşeleri ile arasında ki uzaklığı ölçerek bir diziye atar. Uzaklık hesabı analitik geometriden 'koordinatları bilinen iki noktanın birbirine olan uzaklığı" teoremine dayanır ;

A(xi, yi) = i. Köşe koordinatı ve B(xj, yj) = cihazın o anki koordinatları olsun;

Cihazın köşe noktasına olan uzaklığı = sqrt(A . A + B . B)

dir.

Tüm bu işlevlerin tanımlamaları aynı şekilde başlık dosyasında da yapılmalıdır ;

```
./src/modules/navigator/MODUL_poligon.h
public :
...
virtual bool isInsidePolygonOrCircle(double lat, double lon, float altitude);
...
void zs_updateFence();
...
double poligonUzaklik(double lat, double lon);
double koseUzaklik(double lat, double lon, int numvertex);
...
private :
...
bool insidePolygon(const PolygonInfo &polygon, double lat, double lon, float alti
              tude);
...
```

## **5**  MODULU ANA İŞLEVİ – NAVIGATOR

Modülnün ana program akışı 'Navigator' modülünde ki 'navigator\_main.cpp' bloğu üzerinden sağlanmaktadır. Bu blokta oluşturulan '_modul\_main' işlevi, bu blokta ki 'RUN' işlevi tarafından sürekli olarak çağrılmaktadır.

Modülün burada oluşturulmasının sebebi, yürütülmesi esnasında ihtiyaç duyulabilecek hemen hemen her parametrenin hali hazırda 'navigator\_main.cpp' bloğunda kullanılıyor olup tekrar çağırma, güncelleme vb. gereksinimler duymamamızı sağlamasıdır.

Modül kod bloğu ;

```
./src/modules/navigator/navigator_main.cpp
void Navigator::MODUL_main(double lat, double lon, double alt)
{
     //Acil durum olup olmadığını anlamak için '_commander' kanalı aboneliği
 __commander_sub.update(&__commander);
     //GPS gürültüsü değerini alabilmek için 'sensor_gps' kanalı aboneliği
 _sensor_gps_sub.update(&_sensor_gps);
     //'MODUL' kanalında ki parametreleri içeren yapı tanımı
 MODUL_nav_s Nav;
 memset(&Nav, 0, sizeof(Nav));
 //QGC den belirlenen arazi sınırları bilgilerini içerecek yapı – dm_read ile okuduğum veriler
 mission_stats_entry_s stats;
 int ret = dm_read(DM_KEY_ZS__POLIGON, 0, &stats, sizeof(mission_stats_entry_s));
 int Kapa[4] = {(__commander.batarya_durum), (_sensor_gps.jamming_indicator), 
(__commander.kumanda_durum), (__commander.heart_beat)}; //acil durumları içeren dizi
 double konum[3] = {lat, lon, alt}; //anlık konum bilgileri
 double sonKonum[3]; //son konum bilgileri
 int zs__lama =0, fenceVertexNum =0;
 int leniK = sizeof(Kapa) / sizeof(int); //for için
 int lensK = sizeof(sonKonum) / sizeof(double); //for için
 if (ret == sizeof(mission_stats_entry_s))
 {
 fenceVertexNum = stats.num_items; //arazi köşe sayısı
 }
     //Acil durum tespiti
 for(int i=0; i < leniK; i++)
 {
 if((Kapa[0] != 0) && !_msgAcilDurum)
 {
```

```
 _acil_durum = 1;
               for(int j=0; j < lensK; j++)
 {
 sonKonum[j] = konum[j];
 }
               PX4_WARN("Acil Durum --> Batarya, Son Konum : Enlem(%f)/Boylam(%f)/Yuk-
seklik(%f)", sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
 _msgAcilDurum = true;
               break;
 }
 else if((Kapa[1] > 39) && !_msgAcilDurum)
 {
 _acil_durum = 2;
               for(int j=0; j < lensK; j++)
 {
 sonKonum[j] = konum[j];
 }
               PX4_WARN("Acil Durum --> GPS Jamming, Son Konum : Enlem(%f)/Boy-
lam(%f)/Yukseklik(%f)", sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
 _msgAcilDurum = true;
               break;
 }
 else if((Kapa[2] == 2) && !_msgAcilDurum)
 {
 _acil_durum = 3;
               for(int j=0; j < lensK; j++)
 {
 sonKonum[j] = konum[j];
 }
               PX4_WARN("Acil Durum --> Kumanda Baglantisi, Son Konum : Enlem(%f)/Boy-
lam(%f)/Yukseklik(%f)", sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
 _msgAcilDurum = true;
               break;
 }
 else if((Kapa[3] == 2) && !_msgAcilDurum)
 {
 _acil_durum = 4;
               for(int j=0; j < lensK; j++)
 {
 sonKonum[j] = konum[j];
 }
               PX4_WARN("Acil Durum --> Yer Kontrol Istasyonu Baglantisi, Son Konum : 
Enlem(%f)/Boylam(%f)/Yukseklik(%f)", sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
 _msgAcilDurum = true;
               break;
 }
 }
```

```
 if(!(_MODUL_poligon.isInsidePolygonOrCircle(lat,lon,alt)) || !(fenceVertexNum >=3) 
|| (_acil_durum != 0)) //lamanin kapanmasina sebep olan durumlar ve in kapanmasi
 {
 _Aktif = false;
 switch (_acil_durum)
 {
 case 0: // Cografi Sinir ihlali ya da poligon olusturulmamis olmasi du-
rumlari
 if(!(fenceVertexNum >=3) && !_msgNonAcil)
 {
 PX4_WARN("Arazi Sinirini Kontrol edin");
 }
               else if (!(_geofence.isInsidePolygonOrCircle(lat, lon, alt)) && 
!_msgNonAcil)
 {
 PX4_WARN("Cihaz Cografi Sinir Disinda");
 }
               else if(!_msgNonAcil)
 {
 PX4_WARN("HATA");
 }
               _msgNonAcil = true;
               break;
 case 1: //Batarya kaynakli acil durumlar
 switch (__commander.batarya_durum)
               {
               case 1: // Batarya durumu dusuk ise
 Nav.commander_action = 1;
 if(_geofence.isInsidePolygonOrCircle(lat, lon, alt)) //Batarya 
durumu dusuk VE sınır icinde ise
 {
 //_Aktif = true;
                         _acil_durum = 0;
 }
                    break;
 case 2: // Batarya durumu kritik ise
 if(!_msgBatarya)
                    {
 _wait_for = hrt_absolute_time();
                         Nav.commander_action = 2;
 _msgAcilDurum = false;
                         _msgBatarya = true;
 }
 else if((hrt_absolute_time() - _wait_for > 15_s) && _msgBatarya 
&& (__commander.pilot_kontrol == 2)){ //15sn boyunda pilottan komut gelmedi ise
```

```
 Nav.commander_action = 3;
 }
                  break;
 case 3: // Batarya durumu tehlike arz ediyor ise
 if(_msgBatarya)
                  {
 Nav.commander_action = 4;
 _msgAcilDurum = false;
                       _msgBatarya = false;
 }
                  break;
 default:
 break;
 }
              break;
 case 2: //GPS Jamming kaynakli acil durumlar 0-39 : Güvenli / 40-59 : 
Riskli / 60 > : Tehlike
 if(_sensor_gps.jamming_indicator >= 60)
 {
 Nav.commander_action = 7;
 _msgAcilDurum = false;
                  _acil_durum = 0;
 }
              else if((40 <= _sensor_gps.jamming_indicator) && (_sensor_gps.jamming_in-
dicator < 60) && !_msgGPSJam)
 {
 _wait_for = hrt_absolute_time();
 Nav.commander_action = 5;
 _msgAcilDurum = false;
                  _msgGPSJam = true;
 }
              if((hrt_absolute_time() - _wait_for > 15_s) && _msgGPSJam && (_sen-
sor_gps.jamming_indicator < 60) && (__commander.pilot_kontrol ==2)){ //15sn boyunda pi-
lottan komut gelmedi ise
 Nav.commander_action = 6;
 }
              break;
 case 3: //Kumanda Link Kaybi kaynakli acil durumlar Default : 0 / RC 
Link var : 1 / RC Link yok : 2
 if(__commander.kumanda_durum == 2)
 {
 Nav.commander_action = 8;
 _msgAcilDurum = false;
```

```
 }
             else
             {
 _acil_durum = 0;
 }
             break;
 case 4: //Yer Kontrol Istasyonu Baglantisi kaynakli acil durumlar 
Default : 0 / QGC Baglantisi var : 1 / QGC Baglantisi yok : 2
 if(__commander.heart_beat == 2)
 {
 Nav.commander_action = 9;
 _msgAcilDurum = false;
 }
             else
             {
 _acil_durum = 0;
 }
             break;
 default:
 break;
 }
 }
 else
 {
 _Aktif = true;
 //_msgAcilDurum = false;
 _msgNonAcil = false;
 _msgBatarya = false;
 _acil_durum = 0;
 if((fenceVertexNum >=3) && !_msgFenceVertex) //köşe sayısını 1 kez uyarı gön-
der
 {
 PX4_WARN("Poligon Sayisi : %d", fenceVertexNum);
 _msgFenceVertex = true;
 }
 }
 if(_Aktif)
 {
 zs__lama = 1;
 //PX4_WARN("lama AKtif");
 }
 else
 {
 zs__lama = 2;
```

```
 //PX4_WARN("lama Kapali");
 }
 PX4_WARN("Cihazin poligona uzakligi %f",_MODUL_poligon.poligonUzaklik(konum[0], 
konum[1]));
     //Sonuc verilerin 'MODUL' kanalinda paylasimi
 _MODUL.commander_action = Nav.commander_action;
 _MODUL.timestamp = hrt_absolute_time();
 _MODUL._durum = zs__lama;
 _MODUL_pub.publish(_MODUL);
}
```

'_modul\_main()' kod bloğu içerisinde ilk olarak işlevde kullanılacak yapıların, değişkenlerin ve dizilerin tanımlamaları yapılmıştır.

Ardından ilk olarak arazi koşullarını anlamak ve  püskürtme işlemini başlatmak/durdurmak için gerekli olan arazi köşe sayısı bilgisi çekilmiştir.

Sonrasında, çalışmalar doğrultusunda belirlenen acil durumların yaşanması halinde hangi acil durumun gerçekleştiği ve cihazın son konum bilgilerini almak amacı ile bir acil durum tespit mekanizması kurulmuştur.

Devamında artık  püskürtmenin başlatılması/durdurulması işlemine karar veren temel IF-ELSE bloğu kurulmuş ve eğer püskürtmenin kapatılması sebeplerinden birisi bir acil durum ise sistemin yaşanan acil duruma istinaden karar alabilmesi için gerekli SWITCH-CASE yapısı oluşturulmuştur.

Son bölümde ise kod bloğu çıktısı olan son veriler '_modul' uORB kanalında paylaşılmıştır.

Söz konusu Modül işlevlerinin gerçekleştirilebilmesi için başlık dosyasında yapılan tanımlamalar ;

```
./src/modules/navigator/navigator.h
#include "MODUL_poligon.h"
…
#include <uORB/topics/MODUL.h>
#include <uORB/topics/_commander.h>
#include <uORB/topics/sensor_gps.h>
...
public :
…
 void MODUL_main(double lat, double lon, double alt);
 struct MODUL_nav_s {
 uint64_t timestamp;
 uint16_t _durum;
 uint16_t commander_action;
 };
…
Poligon &get_() { return _MODUL_poligon; }
…
private :
…
```

```
 uORB::Subscription __commander_sub{ORB_ID(_commander)};
 uORB::Subscription _sensor_gps_sub{ORB_ID(sensor_gps)};
…
 uORB::Publication<MODUL_s> _MODUL_pub{ORB_ID(MODUL)};
…
 _commander_s __commander{};
 sensor_gps_s _sensor_gps{};
…
 MODUL_s _MODUL{};
…
 Poligon _MODUL_poligon;
 bool _msgFenceVertex{false};
 bool _msgAcilDurum{false};
 bool _msgNonAcil{false};
 bool _msgBatarya{false};
 bool _msgGPSJam{false};
 bool _Aktif{false};
 hrt_abstime _wait_for = {0};
 uint8_t _acil_durum{};
…
```

Öncelikle Modül işlevlerini barındıran '_modul\_poligon' ve işlevlerde kullanılan parametreleri içeren uORB kanalları eklenmiştir.

Ardından sırası ile 'public' olarak '_modul\_main' işlevi, uORB kanalı parametreleri kontrol etmek için kullanılan '_modul\_nav\_s' yapısı tanımlanmıştır.

'private' kısımda uORB kanalı aboneliklerimiz ve paylaşımlarımız ayarlanmıştır. Bu kanallara tanımlı yapılar tanımlanmıştır. Ve son olarak işlevlerimizi barındıran '_modul\_poligon' bloğunun sınıfı ve Modülnde kullandığımız değişkenler tanımlanmıştır. 'bool' değişkenler, bir işlevin sürekli tekrarlanmaması / döngüye girmemesi için kullanılmıştır.

## **6**  MODULU YER KONTROL İSTASYONUNA MESAJ GÖNDERME – MAVLink

Yeni bir MAVLink mesajı oluşturmak için PX4 otopilot yazılımında 'mavlink/include/mavlink/v2.0/message\_definition s/' dizininde ki 'common.xml' dosyasında düzenlemeye gitmemiz gerekmektedir. 'common.xml' MAVLink mesajlarının tanımlarını/kurallarını içeren dosyadır.

```
./mavlink/include/mavlink/v2.0/message_definitions/common.xml
 </message>
 <message id="15000" name="MODUL">
 <description>MODUL MESAJI</description>
 <field type="uint64_t" name="time_usec" units="us">Timestamp (UNIX Epoch time or time since system boot). The receiving end can infer 
timestamp format (since 1.1.1970 or since system boot) by checking for the magnitude of the number.</field>
 <field type="uint16_t" name="lama_durumu">Hava aracının lama durumu bilgisini verir. 1 = İlaçlama Açık, 2 = İlaçlama Kapalı</field>
 </message>
 </messages>
```

'common.xml' de tanımladığımız bir MAVLink mesajı benzersiz mesaj kimliğine sahip olmalıdır. 'common.xml' dosyasında yaptığımız mesaj tanımına istinaden, gerekli iletişim tanımlamalarının yapılması ve fonksiyonların oluşturulması için 'https://github.com/mavlink/mavlink' adresinden indirdiğimiz 'mavgenerate.py' programını kullanıyoruz. Sonrasında program çıktısı olan klasörü otopilot yazılımımızda ki 'mavlink/include/mavlink/v2.0/' dizinine kopyalıyoruz.

Sonrasında uORB kanalında yayınladığımız verileri alacak ve bu bilgileri yer kontrol istasyonuna gönderecek olan sistemi kurmamız gerekmektedir. 'src/modules/mavlink/streams/' dizininde, abonelik, mesaj gönderme ve işlemlerini gerçekleştiren fonksiyonların tanımlarının yapılacağı '_modul.hpp' başlık dosyasını, hali hazırda çalışır durumda olan ve MAVLink mesajlaşmaları için kullanılan diğer modüllerden yararlanarak oluşturuyorum.

```
./src/modules/mavlink/streams/MODUL.HPP
#ifndef __MODUL_HPP
#define __MODUL_HPP //mavlink_messages.cpp de tanımlanıyor
#include <uORB/topics/MODUL.h> //topic tanımlanıyor
class MavlinkStreamModulu : public MavlinkStream
{
public:
 static MavlinkStream *new_instance(Mavlink *mavlink)
 {
 return new MavlinkStreamModulu(mavlink); // Bir üye işlev bildiriminde veya tanı-
mında, geçersiz kılma belirteci şunları sağlar: işlev sanaldır ve temel sınıftan bir sanal işlevi 
geçersiz kılıyor.
 }
 static constexpr const char *get_name_static() // constexpr belirteci, derleme zamanında iş-
levin veya değişkenin değerinin değerlendirilmesinin mümkün olduğunu bildirir.
 {
 return "MODUL";
 } //stream etmek etiket
 static constexpr uint16_t get_id_static()
```

```
 {
 return MAVLINK_MSG_ID_MODUL;
 } //msj id sini get_id_static e etiketliyor
 const char *get_name() const override
 {
 return get_name_static();
 } //get_name = MODUL from 13 ?
 uint16_t get_id() override
 {
 return get_id_static();
 } //etiketlediği msj id ni çağırıyor ?
 unsigned get_size() override
 {
 return _MODUL_sub.advertised() ? (MAVLINK_MSG_ID_MODUL_LEN + MAV-
LINK_NUM_NON_PAYLOAD_BYTES) : 0; //advertised dan dönen OK ise sub ediyor
 }
private:
 explicit MavlinkStreamModulu(Mavlink *mavlink) : MavlinkStream(mavlink) {}
 uORB::Subscription _MODUL_sub
 {
 ORB_ID(MODUL), 0
 }; //_lama_modulu ID sine sub oluyor
 bool send() override
 {
 MODUL_s NavStruct; //ORB un oluşturdu struct // // _lama_modulu'nun 
uORB konunuzun tanımı olduğundan emin olun
 if (_MODUL_sub.update(&NavStruct))
 {
 mavlink_MODUL_t msg__modul{}; //mmavlink struct ı // // mavlink__mo-
dulu_t, özel MAVLink mesajınızın tanımıdır
 msg__modul.time_usec = NavStruct.timestamp;
 msg__modul.lama_durumu = NavStruct._durum; //lama_durumu
 mavlink_msg_MODUL_send_struct(_mavlink->get_channel(), &msg__modul);
//mavlink mesajının gönderilmesi
 return true;
 }
 return false;
```

```
 }
};
#endif // __MODUL_HPP
```

- 2 ve 3. Satırlarda yer alan '#ifndef' ve '#define' kısımları, MAVLink mesajlarını yöneten 'mavlink\_messages.cpp' bloğu için gereklidir.
- '#include kısmında, uORB kanalı tanım dosyasını ekliyoruz.
- 'MavlinkStreamModulu' sınıfının 'Public' bloğunda, mesaj içeriğini oluşturan fonksiyonların, tanımlamaların ve içerdiği parametrelerin, mesaj gönderileceği esnada hangi fonksiyonlar/bloklar tarafından çağrılacağını belirlemek için etiketleme işlemi yapılıyor.
- 'Private' bloğunda, oluşturduğumuz uORB mesajına abone olunuyor. Ardından MAVLink mesajının içeriğini barındıran bir struct (mavlink\__modul\_t) oluşturuluyor. uORB mesajında içeriği gönderilen '_modul\_nav\_s' struct parametrelerinin değerleri 'mavlink\__modul\_t' parametrelerine atanıyor ve mesaj MAVLink aracılığı için gönderilmek üzere paketleniyor.

Gönderme işlemini ise '/src/modules/mavlink/' dizininde ki 'mavlink\_messages.cpp' ve 'mavlink\_main.cpp' blokları ile gerçekleştiriyoruz ;

```
./src/modules/mavlink/mavlink_messages.cpp
#include <uORB/topics/MODUL.h>
…
#include "streams/MODUL.hpp"
…
#if defined(__MODUL_HPP)
        create_stream_list_item<MavlinkStreamModulu>()
#endif // __MODUL_HPP
```

```
./src/modules/mavlink/mavlink_main.cpp
configure_stream_local("MODUL", 1.0f);
```

## **7** YENİ ÖZELLİKLER VE ACİL DURUMLARIN ELE ALINMASI – commander

Tarım amaçlı lama kopteri, bir tarım arazisini lama görevini üç aşamada gerçekleştirmektedir. Bu aşamalar;

- Kalkış (Kalkış ve Yükseliş)
- Uçuş
- İniş (Alçalma ve İniş)

şeklinde sıralanabilir.

Kopterde kullanılan otopilot yazılımı şu ana kadar, yer kontrol istasyonu ile harita üzerinde oluşturulan sanal coğrafi sınırın içerisinde belirli aralıklar ile bir araziyi tarama, tarama rotası üzerinde iken bu sınırın dışarısında çıkıp çıkmadığını anlayabilme, buna göre  püskürtme ve yer kontrol istasyonuna belirli değerler gönderme özelliklerine sahiptir.

Bununla birlikte cihazın kalkış noktasının tarım arazisini temsil eden sanal coğrafi sınır içerisinde bulunması durumunda kalkış yapmadan veya görev başlangıç noktasına varmadan  püskürtme işleminin başlatılması, acil durumların göz ardı edilmesi ve köşe uzaklıklarının hesaplanmaması sebebi ile püskürtülen ın başka ekinlere zarar vermesi, müşteri beklentileri karşılayamaması,  israfı vb. bir çok durumu önlemek üzere Modülne bu durumlar karşısında işlemesi üzere bir karar mekanizması eklenmiştir.

Bu karar mekanizması, 'commander' modülünden ve gps sensöründen elde edilen verileri veya sorunları '\_commander' kanalı aracılığı ile yayınlar. Bu kanala abone olan 'navigator' modülü ise hangi durum karşısında hangi önlemin alınacağını belirler ve '_modul' kanalı aracılığı ile alınacak kararı yayınlar. '_modul' kanalına abone olan 'commander' modülü de gelen karar verisine göre aksiyon alır / kararı işler.

Bahsi geçen işlemlerin gerçekleştirilebilmesi için 'commander' modülünde yapılan tanımlamalar ve başlık dosyası ;

```
./src/modules/commander/commander.cpp
#include <unistd.h>
#include <stdio.h>
#include <poll.h>
#include <string.h>
#include <uORB/uORB.h>
#include <uORB/topics/_commander.h>
```

```
./src/modules/commander/commander.hpp
#include <uORB/topics/_commander.h>
#include <uORB/topics/MODUL.h>
...
 bool _msgAct{false};
 bool _msgRC{false};
 bool _msgPlt{false};
...
 _commander_s __commander{};
 MODUL_s _MODUL{};
...
 uORB::Subscription _MODUL_sub{ORB_ID(MODUL)};
...
      uORB::Publication<_commander_s> 
                           __commander_pub{ORB_ID(_commander)};
```

## **7.1 Acil Durumlar**

Tarım amaçlı lama kopterinde, görev uygulama anında gerçekleşebilecek muhtemel aksaklıklar şu şekilde sıralanmıştır ;

## • GPS Jamming değerinin gözetilmesi

Bu acil durum, GPS verilerinin alınmasında ki gürültü miktarına göre değişiklik gösterir. GPS Jamming değerinin yüksek olması cihazın konum bilgisinin elde edilememesine, dolayısıyla cihazın düşmesine, kaybolmasına vb. durumlara sebebiyet verebilir. Güvenlik ölçütleri şu şekilde değerlendirilmiştir ;

- 0 40 : GPS Jamming değeri güvenli aralıktatır.
- 40 60 : GPS Jamming değeri risk teşkil eder. Yer kontrol istasyonuna uyarı gönderilecektir.
- 60 > : GPS Jamming değeri tehlike arz eder. Yer kontrol istasyonuna uyarı gönderilecek ve pilottan komut beklenecektir.

## • Batarya doluluk oranı

Batarya doluluk oranının gözetilmemesi cihazın bataryası bittiğinde düşmesine, donanımının doğru çalışmamasına vb. problemlere yol açabilir. Güvenlik ölçütleri şu şekilde değerlendirilmiştir ; (Yüzdelik oran belirlenmemiştir)

- Düşük : Yer kontrol istasyonuna "Yer kontrol noktasına dön" uyarısı verilecektir.
- Kritik : 5 sn için HOLD-MODE moduna geç ve pilottan komut bekle. Komut gelmezse yer kontrol noktasına dön.
- Tehlike : Olduğun yere iniş yap. Yer kontrol istasyonuna uyarı gönder.

### • İlaç tankı doluluk oranı

İlaç tankının boşalması durumunda ( bitmiş olabilir,  tankı bağlantısında problem olabilir vb.) cihaz görevini gerçekleştiremez. İlaç tankının tekrar doldurulması için güvenlik ölçütleri şu şekilde değerlendirilmiştir;

- Dolu : İlaç tankında  var
- Boş : İlaç tankı boş, yer kontrol istasyonuna uyarı gönder, yer kontrol noktasına dön.

## • Link kaybı

Cihazın, pilot tarafından kontrol edilmesini sağlayan bağlantının kesildiği anlamına gelir. Güvenlik ölçütleri şu şekilde değerlendirilmiştir ;

- GPS Var: Yer kontrol istasyonuna mesaj gönder ve yer kontrol noktasına dön
- GPS Yok: Bulunduğun noktaya acil iniş yap

Yukarıda sıralanmış olan öngördüğümüz acil durumlar cihaz, uçuş esnasında iken yaşanabilecek acil durumları arz eder.

'commander' modülü çalıştığında her seferinde zaman sayacı başlatılır

```
./src/modules/commander/commander.cpp
void Commander::run()
{
… 
__commander.timestamp = hrt_absolute_time();
…
```

## **7.1.1 Batarya Doluluğu**

Batarya durumuna göre aksiyon alabilmek için otopilot yazılımının belirli batarya durumlarında konsolda verdiği mesajlar incelenmiştir. PX4 otopilot yazılımına göre, herhangi bir batarya acil durumunda 'commander' modülünde ki 'state\_machine\_helper' işlevleri kullanılmaktadır. Bu durum diğer acil durum senaryoları içinde geçerlidir.

state\_machine\_helper.cpp ;

```
./src/modules/commander/state_machine_helper.cpp
int battery_failsafe(orb_advert_t *mavlink_log_pub, const vehicle_status_s &status,
 const vehicle_status_flags_s &status_flags, 
                     commander_state_s &internal_state, const uint8_t battery_warning,
      const low_battery_action_t low_battery_action)
{
...
      int TAR_BATARYA_STS = 0;
 switch (battery_warning) {
 case battery_status_s::BATTERY_WARNING_NONE:
 break;
 case battery_status_s::BATTERY_WARNING_LOW:
                TAR_BATARYA_STS = 1;
                break;
case battery_status_s::BATTERY_WARNING_CRITICAL:
          ...
 switch (low_battery_action) {
 case LOW_BAT_ACTION::WARNING:
 TAR_BATARYA_STS = 2;
 break;
          ...
...
case battery_status_s::BATTERY_WARNING_EMERGENCY:
          ...
 switch (low_battery_action) {
 case LOW_BAT_ACTION::WARNING:
 TAR_BATARYA_STS = 3;
          ...}
```

```
./src/modules/commander/state_machine_helper.hpp
int battery_failsafe(orb_advert_t *mavlink_log_pub, const vehicle_status_s &status,
 const vehicle_status_flags_s &status_flags, 
                       commander_state_s &internal_state, const uint8_t battery_warning,
 const low_battery_action_t low_bat_action);
...
bool _action(int action, commander_state_s &internal_state, const vehicle_status_flags_s
&status_flags, bool _msgLow);
```

Buna göre olası batarya durumlarında 'navigator' modülüne gönderilecek olan bir 'TAR\_BATARYA\_STS' değişkeni oluşturulmuş ve normalde 'void' yapısında olan 'battery\_failsafe' fonksiyonu 'int' yapısına çevirilerek 'TAR\_BATARYA\_STS' değeri döndürülmüştür.

```
./src/modules/commander/commander.cpp
void Commander::battery_status_check()
{
...
if (battery_warning_level_increased_while_armed) {
      __commander.batarya_durum = battery_failsafe(&_mavlink_log_pub, _status, 
                    _status_flags, _internal_state, _battery_warning, 
                    (low_battery_action_t)_param_com_low_bat_act.get());
}
int _modul_action;
_MODUL_sub.update(&_MODUL);
_modul_action = _MODUL.commander_action;
_msgAct = _action(_modul_action, _internal_state, _status_flags, _msgAct);
...
```

'battery\_failsafe' fonksiyonundan dönen değer 'navigator' modülüne '\_commander' kanalı aracılığı ile iletilmiştir. '\_action' fonksiyonu, herhangi bir acil durum gönderilmiş ise tekrar tekrar gönderilerek döngüye girilmemesini sağlar.

```
void Navigator::MODUL_main(double lat, double lon, double alt)
{
        __commander_sub.update(&__commander);
        _sensor_gps_sub.update(&_sensor_gps);
        MODUL_nav_s Nav;
        memset(&Nav, 0, sizeof(Nav));
...
        int Aktif, fenceVertexNum =0, Kapa[4] = {(__commander.batarya_durum), (_sen-
sor_gps.jamming_indicator), (__commander.kumanda_durum), (__commander.heart_beat)};
        double konum[3] = {lat, lon, alt}, sonKonum[3]; //anlik ve son konum bilgileri
        int leniK = sizeof(Kapa) / sizeof(int); //Kapa dizesinin boyutu
        int lensK = sizeof(sonKonum) / sizeof(double); //sonKonum dizesinin boyutu
...
        for(int i=0; i < leniK; i++) //acil durum tespiti
        {
...
                if((Kapa[0] != 0) && !_msgAcilDurum)
```

```
{
                        _acil_durum = 1;
                        for(int j=0; j < lensK; j++)
                        {
                                sonKonum[j] = konum[j];
                        }
                        PX4_WARN("Acil Durum --> Batarya, Son Konum : Enlem(%f)/Boylam(%f)/Yukseklik(%f)", 
                             sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
                        _msgAcilDurum = true;
                        break;
                }
...
        if (!(_geofence.isInsidePolygonOrCircle(lat, lon, alt)) || !(fenceVertexNum >=3) || 
              (_acil_durum != 0)) {
                Aktif = 2;
                switch (_acil_durum)
                {
              ...
                case 1: //Batarya kaynakli acil durumlar
                        switch (__commander.batarya_durum)
                        {
                        case 1: // Batarya durumu dusuk ise
                                Nav.commander_action = 1;
                                if(_geofence.isInsidePolygonOrCircle(lat, lon, alt))
//Batarya durumu dusuk VE sınır icinde ise
                                {
                                        Aktif = 1;
                                }
                                break;
                        case 2: // Batarya durumu kritik ise
                                if(!_msgBatarya)
                                {
                                        _wait_for = hrt_absolute_time();
                                        Nav.commander_action = 2;
                                        _msgAcilDurum = false;
                                        _msgBatarya = true;
                                }
                                else if((hrt_absolute_time() - _wait_for > 15_s) && _msgBatarya && 
                                           (__commander.pilot_kontrol == 2)){
//15sn boyunda pilottan komut gelmedi ise
                                        Nav.commander_action = 3;
                                }
                                break;
                        case 3: // Batarya durumu tehlike arz ediyor ise
                                if(_msgBatarya)
                                {
                                        Nav.commander_action = 4;
                                        _msgAcilDurum = false;
                                        _msgBatarya = false;
                                }
                                break;
                        default:
                                break;
```

```
}
                         break;
                      ...
        _MODUL.commander_action = Nav.commander_action;
        _MODUL.timestamp = hrt_absolute_time();
        _MODUL._durum = Aktif;
        _MODUL_pub.publish(_MODUL);
}
```

'\_commander' kanalından okunan 'batarya\_durum' parametresi 0, 1, 2 ve 3 olmak üzere 4 değer almaktadır. "0" değeri varsayılan olup herhangi bir batarya kaynaklı acil durum olmadığında dönen değerdir. Diğer değerler, bir batarya acil durumu olduğunda 'battery\_failsafe' fonksiyonundan dönen ve 'batarya\_durum' parametresine yazılan değerler olup;

- 1 değeri düşük düzeyde batarya anlamına gelir, yer kontrol istasyonuna uyarı mesajı gönderilir.
- 2 değeri kritik seviyede batarya doluluğu anlamına gelir, önce yer kontrol istasyonuna uyarı mesajı gönderilir ve cihaz HOLD moduna geçer. 15 saniye boyunca pilottan herhangi bir komut gelmemesi durumunda cihaz yer kontrol noktasına döner.
- 3 değeri tehlikeli düzeyde batarya doluluğu anlamına gelir. Cihaz bu durumda bulunduğu konuma acil iniş yapar.

Bu aksiyonları belirten parametre '_modul' kanalında yayınlanan ve '_modul\_main' işlevleri ile değer alan 'commander\_action' parametresidir.

## **7.1.2 GPS Jamming**

GPS gürültü değerleri diğer acil durumların aksine GPS sensörü verilerini barındıran 'sensor\_gps' kanalından direkrt olarak alınıp işlenmektedir.

```
./src/modules/navigator/navigator_main.cpp
void Navigator::MODUL_main(double lat, double lon, double alt)
{
       ...
        _sensor_gps_sub.update(&_sensor_gps);
        MODUL_nav_s Nav;
        memset(&Nav, 0, sizeof(Nav));
...
        int Aktif, fenceVertexNum =0, Kapa[4] = {(__commander.batarya_durum), (_sen-
sor_gps.jamming_indicator), (__commander.kumanda_durum), (__commander.heart_beat)};
        double konum[3] = {lat, lon, alt}, sonKonum[3]; //anlik ve son konum bilgileri
        int leniK = sizeof(Kapa) / sizeof(int); //Kapa dizesinin boyutu
        int lensK = sizeof(sonKonum) / sizeof(double); //sonKonum dizesinin boyutu
...
        for(int i=0; i < leniK; i++) //acil durum tespiti
        {
       ...
                else if((Kapa[1] > 39) && !_msgAcilDurum)
                {
                        _acil_durum = 2;
                        for(int j=0; j < lensK; j++)
                        {
                                sonKonum[j] = konum[j];
```

```
}
                        PX4_WARN("Acil Durum --> GPS Jamming, Son Konum : Enlem(%f)/Boylam(%f)/Yuksek
                             lik(%f)", sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
                        _msgAcilDurum = true;
                        break;
                }
       }
...
case 2: //GPS Jamming kaynakli acil durumlar 0-39 : Güvenli / 40-59 : Riskli / 60 > : Tehlike
       if(_sensor_gps.jamming_indicator >= 60)
       {
              Nav.commander_action = 7;
              _msgAcilDurum = false;
       }
       else if((40 <= _sensor_gps.jamming_indicator) && (_sensor_gps.jamming_indicator < 60) && 
                                                                                              !_msgGPSJam)
       {
              _wait_for = hrt_absolute_time();
              Nav.commander_action = 5;
              _msgAcilDurum = false;
              _msgGPSJam = true;
       }
       if((hrt_absolute_time() - _wait_for > 15_s) && _msgGPSJam && (_sensor_gps.jamming_indicator < 60) 
              && (__commander.pilot_kontrol ==2)){ //15sn boyunda pilottan komut gelmedi ise
              Nav.commander_action = 6;
       }
break;
...
}
```

GPS gürültü değerlerine göre aksiyon alırken 'sensor\_gps' kanalından okunan 'jamming\_indicator' parametresi izlenmektedir. 'Jamming\_indicator' parametresinin ;

- 0-39 arasında değer alması güvenli
- 40-59 arasında değer alması riskli
- 60 ve üzerinde değer alması tehlikeli

olarak belirlenmiştir.

Mekanizma, jamming\_indicator değeri 0-39 arasında iken herhangi bir aksiyon almaz, 40-59 arasında iken yer kontrol istasyonuna mesaj göndermek üzere aksiyon alır ve zaman tutar. Koda göre uyarı verilmesinden itibaren geçen zaman 15 saniyeden fazla ise cihaz yer kontrol noktasına döner.

## **7.1.3 Link Kaybı**

Cihaz ile yer kontrol istasyonu arasında sürekli bir "HEARTBEAT" sinyali alışverişi vardır. Bu sinyal sürekli belirli bir sürede tekrarlanarak gönderilir. Sinyalin gelmesi uzun sürerse/gelmezse bu yer istasyonu link kaybı yaşandığı anlamına gelir. Dolayısıyla son şartlandırmada "HEARTBEAT" mesajı gözetilmelidir.

```
./src/modules/commander/
void Commander::data_link_check()
{
…
…
if (telemetry.heartbeat_type_gcs) {
// Initial connection or recovery from data link lost
      if (_status.data_link_lost) {
             _status.data_link_lost = false;
             _status_changed = true;
             if (_datalink_last_heartbeat_gcs != 0) {
                    mavlink_log_info(&_mavlink_log_pub, "Data link regained");
                    __commander.heart_beat = 1;
             }
             if (!_armed.armed && !_status_flags.condition_calibration_enabled) {
             // make sure to report preflight check failures to a connecting GCS
                    PreFlightCheck::preflightCheck(&_mavlink_log_pub, _status, 
                                                            _status_flags, true, true,
                    hrt_elapsed_time(&_boot_timestamp));
             }
      }
      _datalink_last_heartbeat_gcs = telemetry.timestamp;
}
…
…
if (!_status.data_link_lost) {
      if ((_datalink_last_heartbeat_gcs != 0)
                          && hrt_elapsed_time(&_datalink_last_heartbeat_gcs) > 
                                              (_param_com_dl_loss_t.get() * 1_s)) {
             _status.data_link_lost = true;
             _status.data_link_lost_counter++;
             mavlink_log_info(&_mavlink_log_pub, "Connection to ground station 
                                                                                lost");
             __commander.heart_beat = 2;
 _status_changed = true;
      }
}
…
```

Yer kontrol istasyonu bağlantısı, Modülnde '\_commander' kanalının 'heart\_beat' parametresinden kontrol edilmektedir.

Uzaktan kumanda bağlantısının kaybı ise 'RC Lost' olarak ifade edilir, bağlantı durumunu gözetmek için RC kaybını belirten bayraklar izlenir.

```
./src/modules/commander/commander.cpp
void
Commander::run()
{
…
if (!_manual_control.isRCAvailable()) {
 // set RC lost
     if (_status_flags.rc_signal_found_once && !_status.rc_signal_lost) {
 // ignore RC lost during calibration
           if (!_status_flags.condition_calibration_enabled && !_status_flags.rc_input_blocked) 
           {
                 mavlink_log_critical(&_mavlink_log_pub, "Manual control lost");
 _status.rc_signal_lost = true;
 _rc_signal_lost_timestamp = _manual_control.getLastRcTimestamp();
 set_health_flags(subsystem_info_s::SUBSYSTEM_TYPE_RCRECEIVER, true, true, false, 
                                                                 _status);
 _status_changed = true;
           }
           _msgRC = true;
 }
}
if (_status.rc_signal_lost && _msgRC){
     __commander.kumanda_durum = 2;
}
else if(!_status.rc_signal_lost && _msgRC)
{
 __commander.kumanda_durum = 1;
}
data_link_check();
__commander_pub.publish(__commander);
```

Modülnde, kumanda durumu '\_commander' kanalının 'kumanda\_durum' parametresi üzerinden okunmaktadır.

```
./src/modules/navigator/navigator_main.cpp
void Navigator::MODUL_main(double lat, double lon, double alt)
{
 __commander_sub.update(&__commander);
...
 MODUL_nav_s Nav;
 memset(&Nav, 0, sizeof(Nav));
...
 int Kapa[4] = {(__commander.batarya_durum), (_sensor_gps.jamming_indicator), 
                    (__commander.kumanda_durum), (__commander.heart_beat)};
 double konum[3] = {lat, lon, alt};
 double sonKonum[3];
 int zs__lama =0, fenceVertexNum =0;
 int leniK = sizeof(Kapa) / sizeof(int);
 int lensK = sizeof(sonKonum) / sizeof(double);
...
 for(int i=0; i < leniK; i++) //acil durum tespiti
 {
     ...
 else if((Kapa[2] == 2) && !_msgAcilDurum)
 {
 _acil_durum = 3;
                  for(int j=0; j < lensK; j++)
 {
 sonKonum[j] = konum[j];
 }
                  PX4_WARN("Acil Durum --> Kumanda Baglantisi, Son Konum : Enlem(%f)/Boy
                    lam(%f)/Yukseklik(%f)", sonKonum[0],sonKonum[1],
                         sonKonum[2]);//_acil_durum
 _msgAcilDurum = true;
                  break;
 }
 else if((Kapa[3] == 2) && !_msgAcilDurum)
 {
 _acil_durum = 4;
                  for(int j=0; j < lensK; j++)
 {
 sonKonum[j] = konum[j];
 }
                  PX4_WARN("Acil Durum --> Yer Kontrol Istasyonu Baglantisi, Son Konum : 
                    Enlem(%f)/Boylam(%f)/Yukseklik(%f)", sonKonum[0],sonKonum[1],
                         sonKonum[2]);//_acil_durum
 _msgAcilDurum = true;
                  break;
 }
 }
...
```

```
 if(!(_MODUL_poligon.isInsidePolygonOrCircle(lat,lon,alt)) || !(fenceVertexNum >=3) 
|| (_acil_durum != 0))
 {
 _Aktif = false;
 switch (_acil_durum)
 {
        ...
 case 3: //Kumanda Link Kaybi kaynakli acil durumlar Default : 0 / RC 
Link var : 1 / RC Link yok : 2
 if(__commander.kumanda_durum == 2)
 {
 Nav.commander_action = 8;
 _msgAcilDurum = false;
 }
               else
               {
 _acil_durum = 0;
 }
               break;
 case 4: //Yer Kontrol Istasyonu Baglantisi kaynakli acil durumlar 
Default : 0 / QGC Baglantisi var : 1 / QGC Baglantisi yok : 2
 if(__commander.heart_beat == 2)
 {
 Nav.commander_action = 9;
 _msgAcilDurum = false;
 }
               else
               {
 _acil_durum = 0;
 }
               break;
 default:
 break;
 }
 }
    ...
 _MODUL.commander_action = Nav.commander_action;
 _MODUL.timestamp = hrt_absolute_time();
 _MODUL._durum = zs__lama;
 _MODUL_pub.publish(_MODUL);
}
```

Modülne göre herhangi bir şekilde RC Link ya da yer kontrol istasyonu bağlantısı kopması durumunda cihaz bulunduğu konuma acil iniş yapar, eğer bağlantı tekrar kazanılırsa acil durum sıfırlanır.

## **7.1.4 Pilot Müdahallesi**

Modülnde istenen özelliklerden bir diğeri ise pilotun, cihaza kumanda ile istediği zaman müdahale edebilmesidir.

```
./src/modules/commander/commander.cpp
void
Commander::run()
{
...
__commander.timestamp = hrt_absolute_time();
if ((_status.vehicle_type == vehicle_status_s::VEHICLE_TYPE_ROTARY_WING) &&
                                  _manual_control.wantsOverride(_vehicle_control_mode, _status))
{
      if (main_state_transition(_status, commander_state_s::MAIN_STATE_POSCTL, 
                                  _status_flags, _internal_state) == TRANSITION_CHANGED)
      {
             tune_positive(true);
             mavlink_log_info(&_mavlink_log_pub, "Pilot took over position control using 
                                                                                         sticks");
             _status_changed = true;
             _msgPlt = true;
      } else if (main_state_transition(_status, commander_state_s::MAIN_STATE_ALTCTL, 
                                         _status_flags, _internal_state) == TRANSITION_CHANGED)
      {
             tune_positive(true);
             mavlink_log_info(&_mavlink_log_pub, "Pilot took over altitude control using 
                                                                                         sticks");
             _status_changed = true;
             _msgPlt = true;
      }
}
if (!_msgPlt)
{
      __commander.pilot_kontrol = 2;
}
else
{
      __commander.pilot_kontrol = 1;
}
...
```

Burada, pilotun herhangi bir şekilde uçuşa müdahale etmesi durumunda '\_commander' kanalının 'pilot\_kontrol' parametresi değiştirilmiştir. Bu şekilde GPS jamming ve batarya durumlarında olduğu gibi cihaz bir uyarı gönderip HOLD moduna geçtiğinde başlayıp otomatik yer kontrol noktasına dönüşe kadar geçen süre içerisinde pilot bir komut verirse, yer kontrol istasyonuna dönüş iptal edilecektir.

## **7.2 Karar Verme**

Her bir acil durum anında '\_commander' kanalında bir parametre değerinde değişiklik yapılmıştır. Yapılan değişiklikler, '\_commander' kanalına abone olan 'navigator' modülü tarafından alınarak 'tarım\_modulu\_main' bloğu içerisinde işlenmiş ve '_modul' kanalının 'commander\_action' parametresi ile yapılacak bir sonraki işlem yayınlanmıştır.

'_modul' kanalına abone olan 'commander' modülü bu parametreyi 'state\_machine\_helper' bloğunda '\_action' işlevi altında kullanır. Bir acil durum esnasında yapılacak tüm işlemler buradadır.

```
./src/modules/commander/state_machine_helper.cpp
bool _action(int action, commander_state_s &internal_state, const vehicle_status_flags_s
&status_flags, bool _msgAct)
{
 switch (action)
 {
 case 1:
 if (!_msgAct)
 {
 PX4_WARN("UYARI : Dusuk Seviyede Batarya Dolulugu, Yer Kontrol Istasyo-
nuna Donun");
 _msgAct = true;
 }
 break;
 case 2:
 if (status_flags.condition_global_position_valid && status_flags.condi-
tion_home_position_valid)
 {
 if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_RTL 
              || internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || 
              internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                                 commander_state_s::MAIN_STATE_AUTO_LOITER;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, Kullanici 
                                                     Komutu Bekleniyor");
 }
 }
 else
 {
 if (!(internal_state.main_state == 
                        commander_state_s::MAIN_STATE_AUTO_LAND || 
                            internal_state.main_state == 
                                 commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                                 commander_state_s::MAIN_STATE_AUTO_LAND;
```

```
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, HATA! Inis 
                                                            Yapiliyor");
 }
 }
 break;
 case 3:
 if (status_flags.condition_global_position_valid && 
                                    status_flags.condition_home_position_valid)
 {
 if (!(internal_state.main_state == 
                       commander_state_s::MAIN_STATE_AUTO_RTL || 
                           internal_state.main_state == 
                                commander_state_s::MAIN_STATE_AUTO_LAND || 
                                    internal_state.main_state == 
                                    commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                                    commander_state_s::MAIN_STATE_AUTO_RTL;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, Yer Kontrol 
                                                  Noktasina Donuluyor");
 }
 }
 else
 {
 if (!(internal_state.main_state == 
                       commander_state_s::MAIN_STATE_AUTO_LAND || 
                           internal_state.main_state == 
                                commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                           commander_state_s::MAIN_STATE_AUTO_LAND;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, Donus Islemi 
                                              Basarisiz, Inis Yapiliyor");
 }
 }
 break;
 case 4:
 if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || 
                  internal_state.main_state == 
                       commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
```

```
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Tehlikeli Seviyede Batarya Dolulugu, Inis Yapiliyor");
 }
 break;
 case 5:
 if (status_flags.condition_global_position_valid && 
                  status_flags.condition_home_position_valid)
 {
 if (!(internal_state.main_state == 
                       commander_state_s::MAIN_STATE_AUTO_RTL || 
                            internal_state.main_state == 
                                commander_state_s::MAIN_STATE_AUTO_LAND || 
                                     internal_state.main_state == 
                                     commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                            commander_state_s::MAIN_STATE_AUTO_LOITER;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Riskli GPS Jamming Degeri, Kullanici Komutu 
                                                        Bekleniyor");
 }
 }
 else
 {
 if (!(internal_state.main_state == 
                       commander_state_s::MAIN_STATE_AUTO_LAND || 
                            internal_state.main_state == 
                                commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                            commander_state_s::MAIN_STATE_AUTO_LAND;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Riskli GPS Jamming Degeri, HATA! Inis 
                                                             Yapiliyor");
 }
 }
 break;
 case 6:
 if (status_flags.condition_global_position_valid && 
                                     status_flags.condition_home_position_valid)
 {
 if (!(internal_state.main_state == 
                       commander_state_s::MAIN_STATE_AUTO_RTL || 
                            internal_state.main_state == 
                                commander_state_s::MAIN_STATE_AUTO_LAND ||
```

```
internal_state.main_state == 
                                    commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                           commander_state_s::MAIN_STATE_AUTO_RTL;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Riskli GPS Jamming Degeri, Yer Kontrol 
                                                  Noktasina Donuluyor");
 }
 }
 else
 {
 if (!(internal_state.main_state == 
                      commander_state_s::MAIN_STATE_AUTO_LAND || 
                           internal_state.main_state == 
                                commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = 
                           commander_state_s::MAIN_STATE_AUTO_LAND;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Riskli GPS Jamming Degeri, Donus Islemi 
                                              Basarisiz, Inis Yapiliyor");
 }
 }
 break;
 case 7:
 if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || 
                  internal_state.main_state == 
                      commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Tehlikeli GPS Jamming Degeri, Inis Yapiliyor");
 }
 break;
 case 8:
 if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || 
                  internal_state.main_state == 
                      commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Kumanda Baglantisi Kayip, Inis Yapiliyor");
 }
 break;
```

```
 case 9:
 if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || 
                   internal_state.main_state == 
                       commander_state_s::MAIN_STATE_AUTO_PRECLAND))
 {
 internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
 internal_state.timestamp = hrt_absolute_time();
 PX4_WARN("UYARI : Yer Kontrol Istasyonu Baglantisi Yok, Inis Yapiliyor");
 }
 break;
 default:
 break;
 }
 return _msgAct;
}
```

'\_msgAct', bir acil durum yaşandıktan sonra '\_action' işlevinin döndürdüğü değer olup yaşandığı yerde tekrar aynı acil durumu vermemesi için kullanılmıştır.

## **8** SONUÇ

PX4 otopilot yazılımı altyapısına sahip otopilotu, ek olarak Modül işlevlerine sahip olup temel detayları ile bu raporda özetlenmiştir.

Modülnü oluşturan tüm kodlar, nesneler ve haritalama çalışmaları aşağıda verilmiştir.

## **8.1 Tüm Kod**

**uORB TOPIC Oluşturulması ; ./src/msg/..**

## **8.1.1 msg/_modul.msg**

#### msg\_modul.msg:

```
1: uint64 timestamp
2:
3: uint16 _durum
4: uint16 commander_action
```

## **8.1.2 msg/\_commander.msg**

#### msg\\_commander.msg:

```
1: uint64 timestamp
2:
3: uint8 batarya_durum
4: uint8 kumanda_durum
5: uint8 pilot_kontrol
6: uint8 heart_beat
```

## **8.1.3 msg/CmakeLists.txt**

#### msg\CMakeLists.txt:

```
191: MODUL.msg
192: _commander.msg
```

## **Modül İşlevleri ;**

## **8.1.4 ./src/modules/navigator/navigator\_main.cpp**

```
src\modules\navigator\navigator_main.cpp: uORB Kütüphane ve Kanallarının Eklenmesi STR[68-78]
  ...:
  68: #include <uORB/uORB.h>
  69: #include <uORB/topics/MODUL.h>
  70: #include <uORB/topics/_commander.h>
  ...:
  77: #include <uORB/Subscription.hpp>
  78: #include <uORB/SubscriptionMultiArray.hpp>
 ...:
src\modules\navigator\navigator_main.cpp:  MODULU İŞLEVİNİN ÇAĞRILMASI STR[165-754]
 ...:
  165: void
  166: Navigator::run()
  167: {
 ...:
  195: while (!should_exit()) {
 ...:
 535: MODUL_main(_global_pos.lat, _global_pos.lon, _global_pos.alt);
 ...:
 753: }
 754: }
src\modules\navigator\navigator_main.cpp:  MODULU İŞLEVİ STR[1495-1716]
 ...:
 1495:void Navigator::MODUL_main(double lat, double lon, double alt)
{
    __commander_sub.update(&__commander);
    _sensor_gps_sub.update(&_sensor_gps);
    MODUL_nav_s Nav;
    memset(&Nav, 0, sizeof(Nav));
    mission_stats_entry_s stats;
    int ret = dm_read(DM_KEY_ZS__POLIGON, 0, &stats, sizeof(mission_stats_entry_s));
    int Kapa[4] = {(__commander.batarya_durum), 
                     (_sensor_gps.jamming_indicator),
                     (__commander.kumanda_durum), 
                     (__commander.heart_beat)};
    double konum[3] = {lat, lon, alt};
    double sonKonum[3];
    int zs__lama =0, fenceVertexNum =0;
    int leniK = sizeof(Kapa) / sizeof(int);
    int lensK = sizeof(sonKonum) / sizeof(double);
    if (ret == sizeof(mission_stats_entry_s))
    {
        fenceVertexNum = stats.num_items;
    }
    for(int i=0; i < leniK; i++) //acil durum tespiti
    {
```

```
/*
        if((Kapa[i] != 0) && !_msgAcilDurum) //" i " != 0 ise
        {
            _acil_durum = Kapa[i];
            for(int j=0; j < lensK; j++)
            {
                sonKonum[j] = konum[j];
            }
            PX4_WARN("Acil Durum Kod: %d, Son Konum : Enlem(%f)/Boylam(%f)/Yukseklik(%f)", Kapa[i], 
sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
            _msgAcilDurum = true;
            break;
        }
        */ //" x " != 0 ise
        if((Kapa[0] != 0) && !_msgAcilDurum)
        {
            _acil_durum = 1;
            for(int j=0; j < lensK; j++)
            {
                sonKonum[j] = konum[j];
            }
            PX4_WARN("Acil Durum --> Batarya, Son Konum : Enlem(%f)/Boylam(%f)/Yukseklik(%f)", sonKo-
num[0],sonKonum[1],sonKonum[2]);//_acil_durum
            _msgAcilDurum = true;
            break;
        }
        else if((Kapa[1] > 39) && !_msgAcilDurum)
        {
            _acil_durum = 2;
            for(int j=0; j < lensK; j++)
            {
                sonKonum[j] = konum[j];
            }
            PX4_WARN("Acil Durum --> GPS Jamming, Son Konum : Enlem(%f)/Boylam(%f)/Yukseklik(%f)", sonKo-
num[0],sonKonum[1],sonKonum[2]);//_acil_durum
            _msgAcilDurum = true;
            break;
        }
        else if((Kapa[2] == 2) && !_msgAcilDurum)
        {
            _acil_durum = 3;
            for(int j=0; j < lensK; j++)
            {
                sonKonum[j] = konum[j];
            }
            PX4_WARN("Acil Durum --> Kumanda Baglantisi, Son Konum : Enlem(%f)/Boylam(%f)/Yukseklik(%f)", 
sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
            _msgAcilDurum = true;
            break;
        }
        else if((Kapa[3] == 2) && !_msgAcilDurum)
        {
            _acil_durum = 4;
            for(int j=0; j < lensK; j++)
            {
                sonKonum[j] = konum[j];
```

```
}
            PX4_WARN("Acil Durum --> Yer Kontrol Istasyonu Baglantisi, Son Konum : Enlem(%f)/Boy-
lam(%f)/Yukseklik(%f)", sonKonum[0],sonKonum[1],sonKonum[2]);//_acil_durum
            _msgAcilDurum = true;
            break;
        }
    }
    if(!(_MODUL_poligon.isInsidePolygonOrCircle(lat,lon,alt)) || !(fenceVertexNum >=3) || 
(_acil_durum != 0))
    {
        _Aktif = false;
        switch (_acil_durum)
        {
        case 0: // Cografi Sinir ihlali ya da poligon olusturulmamis olmasi durumlari
            if(!(fenceVertexNum >=3) && !_msgNonAcil)
            {
                PX4_WARN("Arazi Sinirini Kontrol edin");
            }
            else if (!(_geofence.isInsidePolygonOrCircle(lat, lon, alt)) && !_msgNonAcil)
            {
                PX4_WARN("Cihaz Cografi Sinir Disinda");
            }
            else if(!_msgNonAcil)
            {
                PX4_WARN("HATA");
            }
            _msgNonAcil = true;
            break;
        case 1: //Batarya kaynakli acil durumlar
            switch (__commander.batarya_durum)
            {
            case 1: // Batarya durumu dusuk ise
                Nav.commander_action = 1;
                if(_geofence.isInsidePolygonOrCircle(lat, lon, alt)) //Batarya durumu dusuk VE sınır 
icinde ise
                {
                    //_Aktif = true;
                    _acil_durum = 0;
                }
                break;
            case 2: // Batarya durumu kritik ise
                if(!_msgBatarya)
                {
                    _wait_for = hrt_absolute_time();
                    Nav.commander_action = 2;
                    _msgAcilDurum = false;
                    _msgBatarya = true;
                }
                else if((hrt_absolute_time() - _wait_for > 15_s) && _msgBatarya && (__commander.pi-
lot_kontrol == 2)){ //15sn boyunda pilottan komut gelmedi ise
                    Nav.commander_action = 3;
                }
                break;
```

```
case 3: // Batarya durumu tehlike arz ediyor ise
                if(_msgBatarya)
                {
                    Nav.commander_action = 4;
                    _msgAcilDurum = false;
                    _msgBatarya = false;
                }
                break;
            default:
                break;
            }
            break;
        case 2: //GPS Jamming kaynakli acil durumlar 0-39 : Güvenli / 40-59 : Riskli / 60 > : Teh-
like
            if(_sensor_gps.jamming_indicator >= 60)
            {
                Nav.commander_action = 7;
                _msgAcilDurum = false;
                _acil_durum = 0;
            }
            else if((40 <= _sensor_gps.jamming_indicator) && (_sensor_gps.jamming_indicator < 60) && 
!_msgGPSJam)
            {
                _wait_for = hrt_absolute_time();
                Nav.commander_action = 5;
                _msgAcilDurum = false;
                _msgGPSJam = true;
            }
            if((hrt_absolute_time() - _wait_for > 15_s) && _msgGPSJam && (_sensor_gps.jamming_indicator < 
60) && (__commander.pilot_kontrol ==2)){ //15sn boyunda pilottan komut gelmedi ise
                Nav.commander_action = 6;
            }
            break;
        case 3: //Kumanda Link Kaybi kaynakli acil durumlar Default : 0 / RC Link var : 1 / RC Link 
yok : 2
            if(__commander.kumanda_durum == 2)
            {
                Nav.commander_action = 8;
                _msgAcilDurum = false;
            }
            else
            {
                _acil_durum = 0;
            }
            break;
        case 4: //Yer Kontrol Istasyonu Baglantisi kaynakli acil durumlar Default : 0 / QGC Baglan-
tisi var : 1 / QGC Baglantisi yok : 2
            if(__commander.heart_beat == 2)
            {
                Nav.commander_action = 9;
                _msgAcilDurum = false;
            }
```

```
else
           {
               _acil_durum = 0;
           }
           break;
       default:
           break;
       }
   }
   else
   {
       _Aktif = true;
       //_msgAcilDurum = false;
       _msgNonAcil = false;
       _msgBatarya = false;
       _acil_durum = 0;
       if((fenceVertexNum >=3) && !_msgFenceVertex) //köşe sayısını 1 kez uyarı gönder
       {
           PX4_WARN("Poligon Sayisi : %d", fenceVertexNum);
           _msgFenceVertex = true;
       }
   }
   if(_Aktif)
   {
       zs__lama = 1;
       //PX4_WARN("lama AKtif");
   }
   else
   {
       zs__lama = 2;
       //PX4_WARN("lama Kapali");
   }
   //PX4_WARN("fence vertex num %d", fenceVertexNum);
   PX4_WARN("Cihazin poligona uzakligi %f",_MODUL_poligon.poligonUzaklik(konum[0], konum[1]));
   _MODUL.commander_action = Nav.commander_action;
   _MODUL.timestamp = hrt_absolute_time();
   _MODUL._durum = zs__lama;
   _MODUL_pub.publish(_MODUL);
1716:}
```

## **8.1.5 ./src/modules/navigator/navigator.h**

```
src\modules\navigator\navigator.h: Yardımcı İşlevlerin Eklenmesi STR:[50]
...:
 50: #include "MODUL_poligon.h"
...:
src\modules\navigator\navigator.h: uORB Kanallarının Eklenmesi STR:[93-96]
...:
 93: #include <uORB/topics/MODUL.h>
 94: #include <uORB/topics/_commander.h>
...:
 96: #include <uORB/topics/sensor_gps.h>
...:
src\modules\navigator\navigator.h: Class 'public' başlangıcı STR:[108-110]
 ...:
 108:class Navigator : public ModuleBase<Navigator>, public ModuleParams
 109:{
 110:public:
 ...:
src\modules\navigator\navigator.h: Modül İşlevinin ve PUB Edilen Yapının Tanımı STR[164-169]
 ...:
 164: void MODUL_main(double lat, double lon, double alt);
 165: struct MODUL_nav_s {
 166: uint64_t timestamp;
 167: uint16_t _durum;
 168: uint16_t commander_action;
 169: };
...:
src\modules\navigator\navigator.h: Class 'private' başlangıcı STR:[345]
 ...:
 345:private:
 ...:
src\modules\navigator\navigator.h: uORB Abonelikleri STR:[385-387]
 ...:
 385: uORB::Subscription __commander_sub{ORB_ID(_commander)};
 386:
 387: uORB::Subscription _sensor_gps_sub{ORB_ID(sensor_gps)};
 ...:
src\modules\navigator\navigator.h: uORB Yayınları STR:[401]
 ...:
 401: uORB::Publication<MODUL_s> _MODUL_pub{ORB_ID(MODUL)};
 ...:
src\modules\navigator\navigator.h: Abone Olunan uORB Kanalı Yapıları STR:[424-426]
 ...:
 424: _commander_s __commander{};
 425:
 426: sensor_gps_s _sensor_gps{};
 ...:
src\modules\navigator\navigator.h: Yayını Yapılan uORB Kanalı Yapıları STR:[439]
 ...:
 439: MODUL_s _MODUL{};
 ...:
```

src\modules\navigator\navigator.h: '_modul\_main' içerisinde Kullanılan Değişken Tanımları STR:[451- 458]

```
 ...:
 451: Poligon _MODUL_poligon;
 452: bool _msgFenceVertex{false};
 453: bool _msgAcilDurum{false};
 454: bool _msgNonAcil{false};
 455: bool _msgBatarya{false};
 456: bool _msgGPSJam{false};
 457: hrt_abstime _wait_for = {0};
 458: uint8_t _acil_durum{};
 ...:
```

## **8.1.6 ./src/modules/navigator/_modul\_poligon.cpp (HATALI)**

```
src\modules\navigator\MODUL_poligon.cpp: Uzaklik Hesaplama İşlevleri için STR:[53]
...:
 53: #include <lib/mathlib/mathlib.h>
...:
src\modules\navigator\MODUL_poligon.cpp: Poligon Güncelleme STR:[96-173]
  96: void Poligon::zs_updateFence()
  97: {
  98: zs__poligon_s zs__poligon;
  99: mission_stats_entry_s stats;
 100: int ret2 = dm_read(DM_KEY_ZS__POLIGON, 0, &stats, sizeof(mission_stats_entry_s));
 101: int sizret = sizeof(mission_stats_entry_s);
 102: int num_fence_items = 0;
 ...:
 105: if (ret2 == sizeof(mission_stats_entry_s)) {
 106: num_fence_items = stats.num_items;
 107: _update_counter = stats.update_counter;
 108: }
 109:
 110 _num_polygons = 0;
 111: int current_seq = 1;
 112:
 113: while (current_seq <= num_fence_items) {
 114:
 ...:
 117: if (dm_read(DM_KEY_ZS__POLIGON, current_seq, &zs__poligon, sizeof(zs__poli-
gon_s)) !=
 118: sizeof(zs__poligon_s)) {
 119: break;
 120: }
 121:
 122: switch (zs__poligon.nav_cmd) {
 123:
 124: case NAV_CMD_LAMA_KOSE:
 125: PX4_WARN("update case de");
 126: if (_polygons) {
 127:
 128: PX4_WARN("update if poligonda");
 129:
 130: PolygonInfo *new_polygons = new PolygonInfo[_num_polygons + 1];
 131:
 132: if (new_polygons)
 133: {
 134: memcpy(new_polygons, _polygons, sizeof(PolygonInfo) * _num_polygons);
 135: }
 136:
 137: delete[](_polygons);
 138: _polygons = new_polygons;
 139 }
 140: else
 141: {
 142: _polygons = new PolygonInfo[1];
 143: }
 144:
 145: if (!_polygons)
```

```
146: {
 147: _num_polygons = 0;
 148: PX4_ERR("alloc failed");
 149: return;
 150: }
 151:
 152: PolygonInfo &polygon = _polygons[_num_polygons];
 153: polygon.dataman_index = current_seq;
 154: polygon.fence_type = zs__poligon.nav_cmd;
 155:
...:
 163: polygon.vertex_count = zs__poligon.vertex_count;
 164: current_seq += zs__poligon.vertex_count;
...:
 167: ++_num_polygons;
 168: PX4_WARN("poligon : %d num_polygon %d", polygon.vertex_count, _num_polygons);
 169:
 170: break;
 171: }
 172: }
 173: }
src\modules\navigator\MODUL_poligon.cpp: Arazi İhlali Kontrol İşlevi STR:[302-332]
...:
 302: static bool _zs_inside__poligon = {false};
 303:
 304: bool Poligon::isInsidePolygonOrCircle(double lat, double lon, float altitude)
 305: {
 306: mission_stats_entry_s stats;
 307:
 308: int ret = dm_read(DM_KEY_ZS__POLIGON, 0, &stats, sizeof(mission_stats_entry_s));
 309:
 310: if (ret == sizeof(mission_stats_entry_s) && _update_counter != stats.update_counter)
 311: {
 312: zs_updateFence();
 313: }
 314:
 315: for (int polygon_idx = 0; polygon_idx < _num_polygons; ++polygon_idx)
 316: {
 317:
 318: bool inside = insidePolygon(_polygons[polygon_idx], lat, lon, altitude);
 319:
 320: if (inside)
 321: {
 322: //PX4_WARN("icerde");
 323: _zs_inside__poligon = true;
 324: }
 325: else
 326: {
 327: //PX4_WARN("disarda");
 328: _zs_inside__poligon = false;
 329: }
 330: }
 331: return _zs_inside__poligon;
 332: }
```

```
src\modules\navigator\MODUL_poligon.cpp: Çokgen Arazi Sınırlarına Göre İhlal Kontrolü STR:[334-367]
...:
 334: bool Poligon::insidePolygon(const PolygonInfo &polygon, double lat, double lon, float alti-
tude)
 335: {
 336: zs__poligon_s temp_vertex_i;
 337: zs__poligon_s temp_vertex_j;
 338: bool c = false;
 339:
 340: for (unsigned i = 0, j = polygon.vertex_count - 1; i < polygon.vertex_count; j = i++) {
 341: if (dm_read(DM_KEY_ZS__POLIGON, polygon.dataman_index + i, &temp_vertex_i,
 342: sizeof(zs__poligon_s)) != sizeof(zs__poligon_s)) {
 343: break;
 344: }
 345:
 346: if (dm_read(DM_KEY_ZS__POLIGON, polygon.dataman_index + j, &temp_vertex_j,
 347: sizeof(zs__poligon_s)) != sizeof(zs__poligon_s)) {
 348: break;
 349: }
 350:
 351: if (temp_vertex_i.frame != NAV_FRAME_GLOBAL && temp_vertex_i.frame != NAV_FRAME_GLOBAL_INT
 352: && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT
 353: && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT_INT) {
 354: // TODO: handle different frames
 355: PX4_ERR("Frame type %i not supported <-- ./navigator/MODUL_poligon/'insidePoly-
gon'", (int)temp_vertex_i.frame);
 356: break;
 357: }
 358:
 359: if (((double)temp_vertex_i.lon >= lon) != ((double)temp_vertex_j.lon >= lon) &&
 360: (lat <= (double)(temp_vertex_j.lat - temp_vertex_i.lat) * (lon - (double)temp_ver-
tex_i.lon) /
 361: (double)(temp_vertex_j.lon - temp_vertex_i.lon) + (double)temp_vertex_i.lat)) {
 362: c = !c;
 363: }
 364: }
 365:
 366: return c;
 367: }
...:
src\modules\navigator\MODUL_poligon.cpp: Arazi Köşe Noktasına Olan Uzaklık Hesaplama İşlevi 
STR[402-438]
 402: double Poligon::koseUzaklik(double lat, double lon, int numvertex)
 403: {
 404: zs__poligon_s temp_vertex_i;
 405:
 406: double vertexDist = 0;
 407: double msgDist = 0;
 408: double distArray[numvertex];
 409:
 410: for (int polygon_idx = 0; polygon_idx < _num_polygons; ++polygon_idx)
 411: {
 412: for (unsigned i = 0; i < _polygons[polygon_idx].vertex_count; i++)
 413: {
 414: if (dm_read(DM_KEY_ZS__POLIGON, _polygons[polygon_idx].dataman_index + i, &temp_ver-
tex_i, sizeof(zs__poligon_s)) != sizeof(zs__poligon_s))
```

```
415: {
 416: PX4_ERR("UYARI : Poligon Bilgisi %i Alinamadi", (int)temp_vertex_i.frame);
 417: break;
 418: }
 419:
 420: if (temp_vertex_i.frame != NAV_FRAME_GLOBAL && temp_vertex_i.frame != NAV_FRAME_GLOBAL_INT 
&& temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELA-
TIVE_ALT_INT)
 421: {
 422: PX4_ERR("Poligon Bilgisi %i Desteklenmiyor", (int)temp_vertex_i.frame);
 423: break;
 424: }
 425:
 426:
 427: vertexDist = (double)sqrtf(((temp_vertex_i.lat - lat) * (temp_vertex_i.lat - lat)) + 
((temp_vertex_i.lon - lon) * (temp_vertex_i.lon - lon)));
 428:
 429: distArray[i] = vertexDist;
 430: PX4_WARN("poligon sayisi vertex_count : %d", _polygons[polygon_idx].vertex_count);
 431: PX4_WARN("poligon sayisi numvertex : %d", numvertex);
 432: PX4_WARN("poligon koses is correct? : %f", distArray[i]);
 433:
 434: }
 435:
 436: }
 437: return 0;
 438: }
src\modules\navigator\MODUL_poligon.cpp: Arazi Kenarlarına Olan Uzaklık Hesaplama İşlevi STR:[442-
486]
 442: double Poligon::poligonUzaklik(double lat, double lon)
 443: {
 444: zs__poligon_s temp_vertex_i;
 445: zs__poligon_s temp_vertex_j;
 446: double uzaklik = 0;
 447: double coefA =0;
 448: double coefB =0;
 449: double coefC =0;
 450:
 451: for (int polygon_idx = 0; polygon_idx < _num_polygons; ++polygon_idx) {
 452:
 453: for (unsigned i = 0, j = _polygons[polygon_idx].vertex_count - 1; i < _polygons[poly-
gon_idx].vertex_count; j = i++)
 454: {
 455: if (dm_read(DM_KEY_ZS__POLIGON, _polygons[polygon_idx].dataman_index + i, &temp_ver-
tex_i, sizeof(zs__poligon_s)) != sizeof(zs__poligon_s))
 456: {
 457: PX4_ERR("UYARI : Poligon Bilgisi %i Alinamadi", (int)temp_vertex_i.frame);
 458: break;
 459: }
 460:
 461: if (dm_read(DM_KEY_ZS__POLIGON, _polygons[polygon_idx].dataman_index + j, &temp_ver-
tex_j, sizeof(zs__poligon_s)) != sizeof(zs__poligon_s))
 462: {
 463: PX4_ERR("UYARI : Poligon Bilgisi %i Alinamadi", (int)temp_vertex_j.frame);
 464: break;
 465: }
```

```
466:
 467: if (temp_vertex_i.frame != NAV_FRAME_GLOBAL && temp_vertex_i.frame != NAV_FRAME_GLOBAL_INT 
&& temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELA-
TIVE_ALT_INT)
 468: {
 469: PX4_ERR("Poligon Bilgisi %i Desteklenmiyor", (int)temp_vertex_i.frame);
 470: break;
 471: }
 472:
 473: // Ax + By + C = 0 : dogru denkleminde ki A, B, C katsayilarinin hesaplanmasi :
 474: coefA = (double)temp_vertex_i.lon - (double)temp_vertex_j.lon; // A = x.(yi) - x.(yj) : A 
katsayisi
 475: coefB = (double)temp_vertex_i.lat - (double)temp_vertex_j.lat; // B = y.(xi) - y.(xj) : B 
katsayisi
 476: coefC = ((double)temp_vertex_i.lat * (double)temp_vertex_j.lon) - ((double)temp_ver-
tex_j.lat * (double)temp_vertex_i.lon); // C = (xi).(yj) - (xj).(yi) : C katsayisi
 477:
 478: //Cihazin, en yakin poligon sinirina uzakligi
 479: uzaklik = ((coefA * lat) + (coefB * lon) + coefC) / (double)sqrtf((coefA * coefA) + (coefB 
* coefB));
 480: // Cihazin Poligonda kendisine en yakin noktanin uzakligi : d = (A.cihaz_lat + B.ci-
haz_lon + C) / sqrt(A*A + B*B)
 481:
 482: PX4_WARN("temp-i poligonu %f %f temp-j poligonu %f %f Uzaklik %f", (double)temp_ver-
tex_i.lat, (double)temp_vertex_i.lon, (double)temp_vertex_j.lat, (double)temp_vertex_j.lon, uzaklik);
 483: }
 484: }
 485: return uzaklik;
 486: }
```

## **8.1.7 ./src/modules/navigator/_modul\_poligon.h**

src\modules\navigator\_modul\_poligon.h Class 'public' başlangıcı STR:[60-62]

```
 ...:
  60: class Poligon : public ModuleParams
  61: {
  62: public:
 ...:
src\modules\navigator\MODUL_poligon.h İşlev tanımları STR:[158-159]
 ...:
  158: double poligonUzaklik(double lat, double lon);
  159: double koseUzaklik(double lat, double lon, int numvertex);
 ...:
src\modules\navigator\MODUL_poligon.h Class 'private' başlangıcı STR:[164]
 ...:
  164: private:
 ...:
src\modules\navigator\MODUL_poligon.h İşlev tanımı STR:[206]
 ...:
  206: void zs_updateFence();
 ...:
```

## **8.1.8 ./src/modules/navigator/CMakeList.txt**

```
src\modules\navigator\CMakeLists.txt:
```

```
 ...:
 54: MODUL_poligon.cpp
 ...:
```

## **Aksiyon Alma İşlevleri ;**

## **8.1.9 ./src/modules/commander/commander.cpp**

```
src\modules\commander\Commander.cpp: uORB Kanalı Eklendi
 ...:
 85 
 86: #include <uORB/uORB.h>
 87: #include <uORB/topics/_commander.h>
 ...:
src\modules\commander\Commander.cpp: Commander blok başlangıcı STR:[1606-2842]
 ...:
 1606: void
 1607: Commander::run()
 1608: {
 ...:
 2061: __commander.timestamp = hrt_absolute_time();
 ...:
src\modules\commander\Commander.cpp: Pilotun Kontrolü Ele Alması STR:[2305-2331]
 ...:
 1606: void
 1607: Commander::run()
 1608: {
 ...:
 ...:
 2305: __commander.timestamp = hrt_absolute_time();
 2306: // abort autonomous mode and switch to position mode if sticks are moved significantly
 2307: if ((_status.vehicle_type == vehicle_status_s::VEHICLE_TYPE_ROTARY_WING) &&
/*!in_low_battery_failsafe && !_geofence_warning_action_on &&*/ _manual_control.wantsOverride(_ve-
hicle_control_mode, _status))
 2308: {
 2309: if (main_state_transition(_status, commander_state_s::MAIN_STATE_POSCTL, _sta-
tus_flags, _internal_state) == TRANSITION_CHANGED)
 2310: {
 2311: tune_positive(true);
 2312: mavlink_log_info(&_mavlink_log_pub, "Pilot took over position control using 
sticks");
 2313: _status_changed = true;
 2314: _msgPlt = true;
 2315:
 2316: } else if (main_state_transition(_status, commander_state_s::MAIN_STATE_ALTCTL, 
_status_flags, _internal_state) == TRANSITION_CHANGED)
 2317: {
 2318: tune_positive(true);
 2319: mavlink_log_info(&_mavlink_log_pub, "Pilot took over altitude control using 
sticks");
 2320: _status_changed = true;
 2321: _msgPlt = true;
 2322: }
 2323: }
 2324: if (!_msgPlt)
 2325: {
 2326: __commander.pilot_kontrol = 2;
 2327: }
 2328: else
 2329: {
```

```
2330: __commander.pilot_kontrol = 1;
 2331: }
 ...:
src\modules\commander\Commander.cpp: Kumanda Bağlantı Kontrolü STR:[2411-2443]
 ...:
 1606: void
 1607: Commander::run()
 1608: {
 ...:
 ...:
 2411: if (!_manual_control.isRCAvailable()) {
 2412: // set RC lost
 2413: if (_status_flags.rc_signal_found_once && !_status.rc_signal_lost) {
 2414: // ignore RC lost during calibration
 2415: if (!_status_flags.condition_calibration_enabled && !_status_flags.rc_input_bloc-
ked) {
 2416: mavlink_log_critical(&_mavlink_log_pub, "Manual control lost");
 2417: _status.rc_signal_lost = true;
 2418: _rc_signal_lost_timestamp = _manual_control.getLastRcTimestamp();
 2419: set_health_flags(subsystem_info_s::SUBSYSTEM_TYPE_RCRECEIVER, true, true, 
false, _status);
 2420: _status_changed = true;
 2421: }
 2422: _msgRC = true;
 2423: }
 2424: }
 2425:
 2426: if (_status.rc_signal_lost && _msgRC)
 2427: {
 2428: __commander.kumanda_durum = 2;
 2429: }
 2430: else if(!_status.rc_signal_lost && _msgRC)
 2431: {
 2432: __commander.kumanda_durum = 1;
 2433: }
 2438: // data link checks which update the status
 2439: data_link_check();
 2443: __commander_pub.publish(__commander);
 ...:
src\modules\commander\Commander.cpp: Yer Kontrol İstasyonu Bağlantısı Kontrolü STR:[3576-3648]
3539: void Commander::data_link_check()
3540: {
 ...:
 3576 
 3577: if (telemetry.heartbeat_type_gcs) {
 3578: // Initial connection or recovery from data link lost
 3579: if (_status.data_link_lost) {
 3580: _status.data_link_lost = false;
 3581: _status_changed = true;
 3582:
 3583: if (_datalink_last_heartbeat_gcs != 0) {
 3584: mavlink_log_info(&_mavlink_log_pub, "Data link regained");
 3588: __commander.heart_beat = 1;
 3592: }
```

```
3593:
 3594: if (!_armed.armed && !_status_flags.condition_calibration_enabled) {
 3595: // make sure to report preflight check failures to a connecting GCS
 3596: PreFlightCheck::preflightCheck(&_mavlink_log_pub, _status, _status_flags, 
true, true,
 3597: hrt_elapsed_time(&_boot_timestamp));
 3598: }
 3599: }
 ...:
 ...:
 3629 
 3630: // GCS data link loss failsafe
 3631: if (!_status.data_link_lost) {
 3632: if ((_datalink_last_heartbeat_gcs != 0)
 3633: && hrt_elapsed_time(&_datalink_last_heartbeat_gcs) > (_param_com_dl_loss_t.get() * 
1_s)) {
 3634:
 3635: _status.data_link_lost = true;
 3636: _status.data_link_lost_counter++;
 3637:
 3638: mavlink_log_info(&_mavlink_log_pub, "Connection to ground station lost");
 3642: __commander.heart_beat = 2;
 3646: _status_changed = true;
 3647: }
 3648: }
 ...:
src\modules\commander\Commander.cpp: Batarya Durum Kontrolü STR:[3823-3831]
 3716: void Commander::battery_status_check()
 3717: {
 ...:
 3823: if (battery_warning_level_increased_while_armed) {
 3824: __commander.batarya_durum = battery_failsafe(&_mavlink_log_pub, _status, 
                  _status_flags, _internal_state, _battery_warning, 
                        (low_battery_action_t)_param_com_low_bat_act.get());
 3825: }
 3826: int _modul_action;
 3827:
 3828: _MODUL_sub.update(&_MODUL);
 3829:
 3830: _modul_action = _MODUL.commander_action;
 3831: _msgAct = _action(_modul_action, _internal_state, _status_flags, _msgAct);
 ...:
```

## **8.1.10 ./src/modules/commander/commander.hpp**

```
src\modules\commander\Commander.hpp: uORB Kanalları Eklendi
 ...:
  93: #include <uORB/topics/_commander.h>
  94: #include <uORB/topics/MODUL.h>
 ...:
src\modules\commander\Commander.hpp: Class 'private' blok başlangıcı 
 ...:
  104: class Commander : public ModuleBase<Commander>, public ModuleParams
  105: {
  106: public:
 ...:
 ...:
  131: private:
src\modules\commander\Commander.hpp: İşlevlerin sonsuz döngüye girmesi engellendi
 ...:
  333: bool _msgAct{false};
  334: bool _msgRC{false};
  335: bool _msgPlt{false};
 ...:
src\modules\commander\Commander.hpp: uORB Kanal Yapılarının Tanımları
 ...:
  405: _commander_s __commander{};
  406: MODUL_s _MODUL{};
 ...:
src\modules\commander\Commander.hpp: uORB 'MODUL' Aboneliği
 ...:
  438: uORB::Subscription _MODUL_sub{ORB_ID(MODUL)};
 ...:
src\modules\commander\Commander.hpp: uORB '_commander' Yayını
 ...:
  478: uORB::Publication<_commander_s> __commander_pub{ORB_ID(_commander)};
 ...:
./src/modules/commander/state_machine_helper.h
src\modules\commander\state_machine_helper.h: Aksiyon Alma İşlevi Tanımı
 ...:
  166: bool _action(int action, commander_state_s &internal_state, const vehicle_status_flags_s
&status_flags, bool _msgLow);
```

## **8.1.11 ./src/modules/commander/state\_machine\_helper.cpp**

```
src\modules\commander\state_machine_helper.cpp: Aksiyon Alma İşlevi
 ...:
  1215: bool _action(int action, commander_state_s &internal_state, const vehicle_status_flags_s
&status_flags, bool _msgAct)
{
    switch (action)
    {
    case 1:
        if (!_msgAct)
        {
            PX4_WARN("UYARI : Dusuk Seviyede Batarya Dolulugu, Yer Kontrol Istasyonuna Donun ");
            _msgAct = true;
        }
        break;
    case 2:
        if (status_flags.condition_global_position_valid && status_flags.condition_home_position_valid)
        {
            if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_RTL || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || internal_state.main_state == comman-
der_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LOITER;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, Kullanici Komutu Bekleniyor ");
            }
        }
        else
        {
            if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, HATA! Inis Yaoiliyor");
            }
        }
        break;
    case 3:
        if (status_flags.condition_global_position_valid && status_flags.condition_home_position_valid)
        {
            if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_RTL || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || internal_state.main_state == comman-
der_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_RTL;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, Yer Kontrol Noktasina Donuluyor ");
            }
        }
        else
        {
```

```
if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Kritik Seviyede Batarya Dolulugu, Donus Islemi Basarisiz, Inis Yapili-
yor");
            }
        }
        break;
    case 4:
        if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
        {
            internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
            internal_state.timestamp = hrt_absolute_time();
            PX4_WARN("UYARI : Tehlikeli Seviyede Batarya Dolulugu, Inis Yapiliyor");
        }
        break;
    case 5:
        if (status_flags.condition_global_position_valid && status_flags.condition_home_position_valid)
        {
            if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_RTL || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || internal_state.main_state == comman-
der_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LOITER;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Riskli GPS Jamming Degeri, Kullanici Komutu Bekleniyor");
            }
        }
        else
        {
            if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Riskli GPS Jamming Degeri, HATA! Inis Yaoiliyor");
            }
        }
        break;
    case 6:
        if (status_flags.condition_global_position_valid && status_flags.condition_home_position_valid)
        {
            if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_RTL || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || internal_state.main_state == comman-
der_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_RTL;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Riskli GPS Jamming Degeri, Yer Kontrol Noktasina Donuluyor");
            }
```

```
}
        else
        {
            if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
            {
                internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
                internal_state.timestamp = hrt_absolute_time();
                PX4_WARN("UYARI : Riskli GPS Jamming Degeri, Donus Islemi Basarisiz, Inis Yapiliyor ");
            }
        }
        break;
    case 7:
        if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
        {
            internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
            internal_state.timestamp = hrt_absolute_time();
            PX4_WARN("UYARI : Tehlikeli GPS Jamming Degeri, Inis Yapiliyor");
        }
        break;
    case 8:
        if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
        {
            internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
            internal_state.timestamp = hrt_absolute_time();
            PX4_WARN("UYARI : Kumanda Baglantisi Kayip, Inis Yapiliyor");
        }
        break;
    case 9:
        if (!(internal_state.main_state == commander_state_s::MAIN_STATE_AUTO_LAND || inter-
nal_state.main_state == commander_state_s::MAIN_STATE_AUTO_PRECLAND))
        {
            internal_state.main_state = commander_state_s::MAIN_STATE_AUTO_LAND;
            internal_state.timestamp = hrt_absolute_time();
            PX4_WARN("UYARI : Yer Kontrol Istasyonu Baglantisi Yok, Inis Yapiliyor");
        }
        break;
    default:
        break;
    }
    return _msgAct;
}
```

## **.MAVLink Aracılığı ile Mesaj Gönderme**

## **8.1.12 ./src/modules/mavlink/mavlink\_main.cpp**

```
src\modules\mavlink\mavlink_main.cpp: QGC ye Yapılacak Yayın ve Frekansı STR:[1556]
 ....:
  1494: int
  1495: Mavlink::configure_streams_to_default(const char *configure_single_stream)
  1496: {
 ....:
 ....:
  1556: configure_stream_local("MODUL", 1.0f);
 .....:
```

## **8.1.13 ./src/modules/mavlink/mavlink\_messages.cpp**

```
src\modules\mavlink\mavlink_messages.cpp: Yayın Dosyası ve uORB Kanalları Eklendi STR:[61/124]
.....:
  61: #include <uORB/topics/MODUL.h>
.....:
  124: #include "streams/MODUL.hpp"
.....:
src\modules\mavlink\mavlink_messages.cpp: Yapılacak Yayın ve Yayın Listesi İşlevleri
.....:
  331: static const StreamListItem streams_list[] = {
  332: #if defined(HEARTBEAT_HPP)
 ....:
  557: #if defined(__MODUL_HPP)
  558: create_stream_list_item<MavlinkStreamModulu>()
  559: #endif // __MODUL_HPP
.....:
  563: };
```

## **8.1.14 ./src/modules/mavlink/mavlink\_receiver.cpp**

```
src\modules\mavlink\mavlink_receiver.cpp: 'MAVLinkMesajlaşma' Yazılımı Deneme Mesajı İşlev Yürütme
  108: void
  109: MavlinkReceiver::handle_message(mavlink_message_t *msg)
  110: {
.....:
  272: case MAVLINK_MSG_ID_MODUL:
  273: handle_message_MODUL (msg);
  274: break;
.....:
src\modules\mavlink\mavlink_receiver.cpp: 'MAVLinkMesajlaşma' Yazılımı Deneme Mesajı Handle İşlevi
.....:
  3258: void MavlinkReceiver::handle_message_MODUL (mavlink_message_t *msg)
  3259: {
  3260: mavlink_MODUL_t NavStruct_my;
  3261: mavlink_msg_MODUL_decode(msg, &NavStruct_my);
  3262:
  3263: struct MODUL_s uorb_tm;
  3264: memset (&uorb_tm, 0, sizeof(uorb_tm));
  3265:
```

```
3266: uorb_tm.timestamp = hrt_absolute_time();
3267: //uorb_tm.lat = NavStruct_my.lama_durumu;
3268:
3269: if(NavStruct_my.lama_durumu==8)
3270: {
3271: PX4_WARN("QGC den mesaj geldi");
3272: }
3273:
3274: _NavStruct_pub.publish(uorb_tm);
3275: }
```

## **8.1.15 ./src/modules/mavlink/mavlink\_receiver.h**

```
src\modules\mavlink\mavlink_receiver.h: Yayını Yapılan uORB Kanalı Eklendi
.....:
  110: #include <uORB/topics/MODUL.h>
.....:
src\modules\mavlink\mavlink_receiver.h: Class 'Private' bloğu başlangıcı
.....:
  204:class MavlinkReceiver : public ModuleParams
  204:{
  204:public:
.....:
  204:private:
.....:
src\modules\mavlink\mavlink_receiver.h: Handle İşlevinin Tanımı ve uORB Yayını
.....:
  204: void handle_message_MODUL(mavlink_message_t *msg);
.....:
  315: uORB::Publication<MODUL_s> _NavStruct_pub{ORB_ID(MODUL)};
.....:
```

## **8.1.16 ./src/modules/mavlink/streams/_modul.hpp**

```
src\modules\mavlink\streams\MODUL.hpp:
```

```
1: #ifndef __MODUL_HPP
  2: #define __MODUL_HPP //mavlink_messages.cpp de tanımlanıyor
  3:
  4: #include <uORB/topics/MODUL.h> //topic tanımlanıyor
  5:
  6: class MavlinkStreamModulu : public MavlinkStream
  7: {
  8: public:
  9: static MavlinkStream *new_instance(Mavlink *mavlink)
  10: {
  11: return new MavlinkStreamModulu(mavlink); // Bir üye işlev bildiriminde veya tanımında, 
geçersiz kılma belirteci şunları sağlar: işlev sanaldır ve temel sınıftan bir sanal işlevi geçersiz kılı-
yor.
  12: }
  13:
  14: static constexpr const char *get_name_static() // constexpr belirteci, derleme zamanında işlevin 
veya değişkenin değerinin değerlendirilmesinin mümkün olduğunu bildirir.
```

```
15: {
 16: return "MODUL";
 17: } //stream etmek etiket
 18:
 19: static constexpr uint16_t get_id_static()
 20: {
 21: return MAVLINK_MSG_ID_MODUL;
 22: } //msj id sini get_id_static e etiketliyor
 23:
 24: const char *get_name() const override
 25: {
 26: return get_name_static();
 27: } //get_name = MODUL from 13 ?
 28:
 29: uint16_t get_id() override
 30: {
 31: return get_id_static();
 32: } //etiketlediği msj id ni çağırıyor ?
 33:
 34: unsigned get_size() override
 35: {
 36: return _MODUL_sub.advertised() ? (MAVLINK_MSG_ID_MODUL_LEN + MAV-
LINK_NUM_NON_PAYLOAD_BYTES) : 0; //advertised dan dönen OK ise sub ediyor
 37: }
 38:
 39: private:
 40: explicit MavlinkStreamModulu(Mavlink *mavlink) : MavlinkStream(mavlink) {}
 41:
 42: uORB::Subscription _MODUL_sub
 43: {
 44: ORB_ID(MODUL), 0
 45: }; //_lama_modulu ID sine sub oluyor
 46:
 47: bool send() override
 48: {
 49: MODUL_s NavStruct; //ORB un oluşturdu struct // // _lama_modulu'nun 
uORB konunuzun tanımı olduğundan emin olun
 50:
 51: if (_MODUL_sub.update(&NavStruct))
 52: {
 53: mavlink_MODUL_t msg__modul{}; //mmavlink struct ı // // mavlink__mo-
dulu_t, özel MAVLink mesajınızın tanımıdır
 54:
 55: msg__modul.time_usec = NavStruct.timestamp;
 56: msg__modul.lama_durumu = NavStruct._durum; //lama_durumu
 57:
 58: mavlink_msg_MODUL_send_struct(_mavlink->get_channel(), &msg__modul); //mavlink 
mesajının gönderilmesi
 59:
 60: return true;
 61: }
 62: return false;
 63: }
 64: };
 65: #endif // __MODUL_HPP
 66:
```

## **.MAVLink Aracılığı ile Mesaj Alma (HATALI)**

## **8.1.17 ./src/modules/mavlink/mavlink\_mission.cpp**

```
src\modules\mavlink\mavlink_mission.cpp:
.....:
  68: uint16_t MavlinkMissionManager::_zs__poligon_update_counter = 0;
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
  86: void
  87: MavlinkMissionManager::init_offboard_mission()
  88: {
.....:
  123: zs_load__poligon();
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
  150: int
  151: MavlinkMissionManager::zs_load__poligon()
  152: {
  153: mission_stats_entry_s stats;
  154: int ret = dm_read(DM_KEY_ZS__POLIGON, 0, &stats, sizeof(mission_stats_entry_s));
  155:
  156: if (ret == sizeof(mission_stats_entry_s)) {
  157: _count[MAV_MISSION_TYPE_MISSION] = stats.num_items;
  158: _zs__poligon_update_counter = stats.update_counter;
  159: }
  160:
  161: return ret;
  162: }
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
  260: int
  261: MavlinkMissionManager::zs_update__poligon(unsigned count)//#count?
  262: {
  263: mission_stats_entry_s stats;
  264: stats.num_items = count;
  265: stats.update_counter = ++_zs__poligon_update_counter;
  266:
  267: int res = dm_write(DM_KEY_ZS__POLIGON, 0, DM_PERSIST_POWER_ON_RESET, &stats, sizeof(mis-
sion_stats_entry_s));
  268:
  269: if (res == sizeof(mission_stats_entry_s)) {
  270: _count[MAV_MISSION_TYPE_MISSION] = count;
  271:
  272: } else {
  273:
  274: if (_filesystem_errcount++ < FILESYSTEM_ERRCOUNT_NOTIFY_LIMIT) {
  275: _mavlink->send_statustext_critical("Mission storage: Unable to write to microSD");
  276: }
  277:
  278: return PX4_ERROR;
  279: }
  280:
```

```
281: return PX4_OK;
 282: }
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
 364: void
 365: MavlinkMissionManager::send_mission_item(uint8_t sysid, uint8_t compid, uint16_t seq)
 366: {
 367: mission_item_s mission_item{};
 368: int read_result = 0;
 369:
 370: switch (_mission_type) {
 374: case MAV_MISSION_TYPE_MISSION: {
 375: if(mission_item.nav_cmd == NAV_CMD_LAMA_KOSE)
 376: {
 377: zs__poligon_s zs__poligon;
 378:
 379: read_result = dm_read(DM_KEY_ZS__POLIGON, seq + 1, 
                         &zs__poligon, sizeof(zs__poligon_s)) == sizeof(zs__poligon_s);
 380:
 381: mission_item.nav_cmd = zs__poligon.nav_cmd;
 382: mission_item.frame = zs__poligon.frame;
 383: mission_item.lat = zs__poligon.lat;
 384: mission_item.lon = zs__poligon.lon;
 385: mission_item.altitude = zs__poligon.alt;
 386:
 387: mission_item.vertex_count = zs__poligon.vertex_count;
 388: }
 389: else
 390: {
 391: read_result = dm_read(_dataman_id, seq, &mission_item, 
                         sizeof(mission_item_s)) == sizeof(mission_item_s);
 392: }
 393: }
 394: break;
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
 769: void
 770: MavlinkMissionManager::handle_mission_request_list(const mavlink_message_t *msg)
 771: {
.....:
 793: switch (_mission_type) {
 797: case MAV_MISSION_TYPE_MISSION:
 798: zs_load__poligon();
 799: break;
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
 947: void
 948: MavlinkMissionManager::handle_mission_count(const mavlink_message_t *msg)
 949: {
.....:
 973: if (wpc.count == 0) {
 974: PX4_DEBUG("WPM: MISSION_COUNT 0, clearing waypoints list and staying in state MAV-
LINK_WPM_STATE_IDLE");
```

```
975:
 976: switch (_mission_type) {
 977: case MAV_MISSION_TYPE_MISSION:
 978:
 979: /* alternate dataman ID anyway to let navigator know about changes */
 980:
 981: if (_dataman_id == DM_KEY_WAYPOINTS_OFFBOARD_0) {
 982: update_active_mission(DM_KEY_WAYPOINTS_OFFBOARD_1, 0, 0);
 983:
 984: } else {
 985: update_active_mission(DM_KEY_WAYPOINTS_OFFBOARD_0, 0, 0);
 986: }
 990: zs_update__poligon(0);
 994: break;
 995:
 996: case MAV_MISSION_TYPE_FENCE:
 997: update_geofence_count(0);
 998: break;
 1027: mission_item_s mission_item{};
 1028:
 1029: if((_mission_type == MAV_MISSION_TYPE_MISSION) && (mission_item.nav_cmd == 
NAV_CMD_LAMA_KOSE))
 1030: {
 1031: PX4_DEBUG("locking fence dataman items");
 1032:
 1033: int ret = dm_lock(DM_KEY_ZS__POLIGON);
 1034:
 1035: if (ret == 0) {
 1036: _geofence_locked = true;
 1037:
 1038: } else {
 1039: PX4_ERR("locking failed (%i)", errno);
 1040: }
 1041: }
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
 1106: void
 1107: MavlinkMissionManager::zs_switch_to_idle_state()
 1108: {
 1109: // when switching to idle, we *always* check if the lock was held and release it.
 1110: // This is to ensure we don't end up in a state where we forget to release it.
 1111: if (_geofence_locked) {
 1112: dm_unlock(DM_KEY_ZS__POLIGON);
 1113: _geofence_locked = false;
 1114:
 1115: PX4_DEBUG("unlocking zs  geofence");
 1116: }
 1117:
 1118: _state = MAVLINK_WPM_STATE_IDLE;
 1119: }
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
 1148: static uint16_t _zs__poligon_counter = {0};
```

```
1149: static uint16_t _zs__poligon_transfer_count = {0};
 1153: void
 1154: MavlinkMissionManager::handle_mission_item_both(const mavlink_message_t *msg)
 1155: {
 1156:
 1223: switch (_mission_type) {
 1227: case MAV_MISSION_TYPE_MISSION: {
 1228: // check that we don't get a wrong item (hardening against wrong client implemen-
tations, the list here
 1229: // does not need to be complete)
 1230: struct zs__poligon_s zs__poligon;
 1231:
 1232: if (mission_item.nav_cmd == MAV_CMD_NAV_FENCE_POLYGON_VERTEX_INCLUSION ||
 1233: mission_item.nav_cmd == MAV_CMD_NAV_FENCE_POLYGON_VERTEX_EXCLUSION ||
 1234: mission_item.nav_cmd == MAV_CMD_NAV_FENCE_CIRCLE_INCLUSION ||
 1235: mission_item.nav_cmd == MAV_CMD_NAV_FENCE_CIRCLE_EXCLUSION ||
 1236: mission_item.nav_cmd == MAV_CMD_NAV_RALLY_POINT) {
 1237: check_failed = true;
 1238: }
 1239: else if(mission_item.nav_cmd == MAV_CMD_LAMA_KOSESI)
 1240: {
 1241: zs__poligon.nav_cmd = mission_item.nav_cmd;
 1242: zs__poligon.lat = mission_item.lat;
 1243: zs__poligon.lon = mission_item.lon;
 1244: zs__poligon.alt = mission_item.altitude;
 1245:
 1246: zs__poligon.vertex_count = mission_item.vertex_count;
 1247:
 1248: if (mission_item.vertex_count < 3)
 1249: {
 1250: //zs__poligon.circle_radius = mission_item.circle_radius;
 1251: zs_update__poligon(0);
 1252: PX4_WARN(" vertex_count < 3");
 1253: }
 1254:
 1255: zs__poligon.frame = mission_item.frame;
 1256:
 1257: if (!check_failed)
 1258: {
 1259: if (dm_write(DM_KEY_ZS__POLIGON, ++_zs__poligon_counter, DM_PER-
SIST_POWER_ON_RESET, &zs__poligon, sizeof(zs__poligon_s)) != sizeof(zs__poligon_s))
 1260: { //^wp.seq + 1 --> DM yapisinde struct lari dizi gibi değil 
belirli anahtarlari olan listeler gibi indexliyor
 1261: //misal bir struct 1249 numarasi ile indexlenerek listeye 
atildi ise yine sadece 1249 numarasi ile cekilebilir
 1262: PX4_WARN("ZS dmwrite calismadi");
 1263: }
 1264: else if (dm_read(DM_KEY_ZS__POLIGON, _zs__poligon_counter, 
&zs__poligon, sizeof(zs__poligon_s)) != sizeof(zs__poligon_s))
 1265: {
 1266:
 1267: //PX4_ERR("ZS dm_read failed HR %lu %lu", dm_read(DM_KEY_ZS_TA-
RIM_POLIGON, wp.seq + 1, &zs__poligon, sizeof(zs__poligon_s)), sizeof(zs__poligon_s));
 1268: }
```

```
1269: }
 1270:
 1271: PX4_WARN("Current seq coordinates from MAV MISSION : %f % f %d AND wp.seq no : 
%d AND  poligon no : %d", zs__poligon.lat, zs__poligon.lon, zs__poligon.vertex_count, 
wp.seq, _zs__poligon_counter);
 1272:
 1273: }
 1274: else { //@hamdi MAV_CMD_LAMA_KOSESI poligon köşesinde de buraya 
giriyor
 1275: _zs__poligon_transfer_count = _zs__poligon_counter;
 1276: _zs__poligon_counter = 0; // ZS_TAR
 1277: dm_item_t dm_item = _transfer_dataman_id;
 1278:
 1279: write_failed = dm_write(dm_item, wp.seq, DM_PERSIST_POWER_ON_RESET, &mis-
sion_item, sizeof(struct mission_item_s)) != sizeof(struct mission_item_s);
 1280:
 1281: if (!write_failed) {
 1282: /* waypoint marked as current */
 1283: if (wp.current) {
 1284: _transfer_current_seq = wp.seq;
 1285: }
 1286: }
 1287: }
 1291: } // end of case
 1292: break;
 1293:
 1294: case MAV_MISSION_TYPE_FENCE: { // Write a geofence point
 1295 mission_fence_point_s mission_fence_point;
.....:
.....:
 1367 
 1368: if (_transfer_seq == _transfer_count) {
 1369: /* got all new mission items successfully */
 1370: PX4_DEBUG("WPM: MISSION_ITEM got all %u items, current_seq=%u, changing state to MAV
                                                               LINK_WPM_STATE_IDLE",
 1371: _transfer_count, _transfer_current_seq);
 1372:
 1373: ret = 0;
 1374:
 1375: switch (_mission_type) {
 1379: case MAV_MISSION_TYPE_MISSION:
 1380: PX4_WARN("mav mis type case de %d %d %d %d", mission_item.nav_cmd, _transfer_count, 
mission_item.vertex_count, _zs__poligon_transfer_count);
 1381: zs_update__poligon(_zs__poligon_transfer_count);
 1382: ret = update_active_mission(_transfer_dataman_id, _transfer_count, _transfer_cur-
rent_seq);
 1383: break;
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
 1424: void
 1425: MavlinkMissionManager::handle_mission_clear_all(const mavlink_message_t *msg)
 1426: {
.....:
 1432: struct mission_item_s mission_item = {};
```

```
1446: switch (wpca.mission_type) {
 1447: case MAV_MISSION_TYPE_MISSION:
 1451: if(mission_item.nav_cmd == MAV_CMD_LAMA_KOSESI)
 1452: {
 1453: ret = zs_update__poligon(0);
 1454: }
 1455: else
 1456: {
 1457: ret = update_active_mission(_dataman_id == DM_KEY_WAYPOINTS_OFFBOARD_0 ? 
DM_KEY_WAYPOINTS_OFFBOARD_1 :
 1458: DM_KEY_WAYPOINTS_OFFBOARD_0, 0, 0);
 1459: }
 1473: case MAV_MISSION_TYPE_ALL:
 1474: ret = update_active_mission(_dataman_id == DM_KEY_WAYPOINTS_OFFBOARD_0 ? 
DM_KEY_WAYPOINTS_OFFBOARD_1 :
 1475: DM_KEY_WAYPOINTS_OFFBOARD_0, 0, 0);
 1479: ret = zs_update__poligon(0) || ret;
.....:
src\modules\mavlink\mavlink_mission.cpp:
.....:
 1509:int
 1510:MavlinkMissionManager::parse_mavlink_mission_item(const mavlink_mission_item_t *mavlink_mis-
sion_item, struct mission_item_s *mission_item)
 1512:{
.....:
 1649: case MAV_CMD_LAMA_KOSESI:
 1650: mission_item->nav_cmd = (NAV_CMD)mavlink_mission_item->command;
 1651: mission_item->vertex_count = (uint16_t)(mavlink_mission_item->param4 + 0.5f);
 1652: PX4_INFO("MAV_CMD_LAMA_KOSESI");
 1653: break;
.....:
```

## **8.1.18 ./src/modules/navigator/navigation.h**

#### src\modules\navigator\navigation.h:

```
228: struct zs__poligon_s {
229: double lat;
230: double lon;
231: float alt;
232:
233: union {
234: uint16_t vertex_count;
235: float circle_radius;
236: };
237:
238: uint16_t nav_cmd;
239: uint8_t frame;
240:
241: uint8_t _padding0[5];
242: };
```

## **8.2 Notlar**

### Modül;

- QGC den belirlenen **geofence** sınırlarına göre arazi sınırları içerisinde olduğunu anlayabilme
- QGC'e  püskürtme durumuna dair MAVLink üzerinden bilgi gönderme
- GPS Jamming, RC Link kaybı, istasyon bağlantısı kaybı ve batarya acil durumlarını anlama ve aksiyon alma
- İlaçlama, acil durum, konum bilgilerini anlık olarak kullanıcıya döndürme
- Acil durum anında bulunduğu son konumu biz diziye kaydetme

özelliklerine sahiptir.

Bununla birlikte sahip olduğu problemler;

- Uçuş modlarından
  - ➢ Kalkış & yükseliş
  - ➢ İniş & alçalma

sırasında  püskürtme arazi sınırları içerisinde ise halen devam ediyor

- Acil durumlar tam anlamı ile sıfırlanamıyor, kullanıcı müdahale ederse veya görev o an için sonlandırılırsa sonra göreve otomatik dönemiyor.
- QGC'de oluşturulmuş olan yeni 'LAMA\_KOSELERI' mesajı düzgün şekilde alınamıyor.

dir.

Hataların düzeltilebilmesi için ;

- 'mavlink\_mission.cpp' işlevleri gözden geçirilmeli
- '_modul\_poligon' işlevleri gözden geçirilmeli