# **PX4 Otopilot Yazılımı için Modül Oluşturulması ve MAVLink İletişiminin Sağlanması**

# İçindekiler

| 1 GİRİŞ                                                              | 5 |
|----------------------------------------------------------------------|---|
| 2 ÖZET                                                               |   |
| 3 METADOLOJİ                                                         |   |
| 3.1 Modül için Yazılım Altyapısının Hazırlanması                     |   |
| 3.2 uORB Kanalı Üzerinden Haberleşme                                 |   |
| 3.2.1 uORB Mesajlaşması için Yayın Kanalı (TOPIC) Oluşturmak         |   |
| 3.2.2 uORB Kanalında Mesaj Yayınlamak                                |   |
| 3.3 uORB Aboneliği ve MAVLink Üzerinden Yer İstasyonu ile Haberleşme |   |
| 3.2.2 MAVLink Mesajı Oluşturmak                                      |   |
| 3.2.2 İuORB Aboneliği ve MAVLink Mesajının Gönderilmesi              |   |
| 4 SONUÇ                                                              |   |

## **1** GİRİŞ

PX4 programlama çalışmaları kapsamında bir sal ilaçlama kopteri geliştirilecektir. Cihazın görevini beklendiği üzere gerçekleştirebilmesi için gerekli görevler PX4 yazılımına eklenen modül tarafından işlenecektir. Söz konusu cihazın uçuş kontrol yazılımı PX4 otopilot tabanlı ve yer kontrol istasyonu QGroundControl tabanlı olup genel otopilot yazılımı. Bu raporda otopilot yazılımı içerisinde ki modülün nasıl oluşturulduğu ve PX4 otopilot yazılımında bir modül oluştururken nelere dikkat edilmesi gerektiği detayları ile özetlenmiştir.

sal ilaçlama cihazında kullanılacak olan PX4 tabanlı otopilot yazılımının oluşturulması için PX4 otopilot yazılımına ek işlevsellik kazandırılması gerekmektedir, bunun için PX4 yazılımında bir modül oluşturularak bu işlevsellikleri kazandıracak olan  modülü yazılımının temelleri oluşturulmuştur. Modül yazılımı ilk sürümü itibari ile yer kontrol istasyonu aracılığı ile belirlenen coğrafi sınırlar ve uçuş rotası bilgilerini kullanarak cihazın rota üzerinde iken coğrafi sınırlar içerisinde ilaç püskürtmesi ve coğrafi sınırları ihlal etmesi halinde ilaç püskürtme işlemini sonlandırma özelliklerine sahiptir.

# **2** ÖZET

PX4 otopilot yazılımının 'navigator' modülünde ki 'geofence' fonksiyonlarından yararlanılarak cihazın anlık konum bilgilerine göre coğrafi sınır ihlali olup olmadığı belirlenmiştir. Ardından cihazın ilaç püskürtme işlemini başlatıp durduracağı koşullar belirlenmiştir, buna göre cihazın coğrafi sınırları ihlal etmesi durumunda ilaç püskürtme işlemi sonlandırılacaktır. Koşul durumlarının yer kontrol istasyonunda anlık görüntülenebilmesi için 'navigator' modülü içerisinde tanımlanan 'modül' fonksiyonunun MAVLink aracılığı ile iletişim kurabilmesi gerekmektedir. Bunun için ' modülü' nden alınan koşul durumu bilgileri uORB aracılığı ile yayınlanmış ve 'MAVLink' modülünden alınmıştır. 'MAVLink' modülünden alınan bilgiler yer kontrol istasyonuna iletilmiştir.

uORB yayını yapılabilmesi için öncelikle PX4-otopilot yazılımında ki ./msg dizininde bir '\_modulu.msg' dosyası oluşturulmuş ve aynı dizinde ki 'CMakeList.txt' dosyasında tanımlanarak bir uORB konusu (iletişim kanalı) oluşturulmuştur. Yapılan değişikliklerin ardından yazılımın derlenmesi gerekmektedir. Aksi taktirde yayın kanalının kullanılması için gerekli parametreleri içeren fonksiyon ve başlık dosyaları oluşturulamaz. Ardından 'navigator' modülünde tanımlanan '\_modulu\_lama' fonksiyonundan işlemin başlangıcından itibaren geçen süre ve coğrafi sınır ihlali bilgileri uORB üzerinden diğer modüller ile paylaşıma açık hale gelir.

uORB üzerinden yayınlanan bilgiyi alıp yer kontrol istasyonuna iletebilmek için yayın yapılan kanala abone olunması, ardından alınan bilgileri MAVLink üzerinden yer kontrol istasyonuna iletmek gerekiyor. Dolayısıyla kanala abone olacak ve yer kontrol istasyonuna iletecek olan modül otopilot yazılımında ki 'mavlink' modülüdür. İşlemi gerçekleştirebilmek için gerekli tanımlamaları yaptığımız '\_MODULU.hpp' dosyasını 'mavlink' modülünün içerisinde ki 'streams' dizininde oluşturulması gerekiyor. Sonraki süreçte tanımlamalarını yaptığımız mesaj yayınlama, uORB aboneliği, MAVLink mesajı gönderme gibi özellikleri yine 'mavlink' modülü içerisinde ki 'mavlink\_main' ve 'mavlink\_messages' dosyalarına eklenmiştir.

### **3** METADOLOJİ

Otopilot yazılımına eklenen modülün yazılımının kodlama çalışmaları Visual Studio Code editörü kullanıla rak gerçekleştirilmiştir. Simülasyon çalışmaları Gazebo simülasyon programı ve QGroundControl yer kontrol istasyonnu programı kullanılarak Ubuntu işletim sisteminin 20.04 LTS sürümü üzerinde gerçekleştirilmiştir. Otopilot içerisinde ki modüller arası haberleşme uORB ile, otopilot ile yer kontrol istasyonu arası haberleşme ise MAVLink protokolü ile gerçekleştirilmektedir.

### **3.1 Modül için Yazılım Altyapısının Hazırlanması**

Amaç  arazilerinin etkili ilaçlanması ve ilaç israfını, çevreye verilen zarar etkilerini azaltmak olduğundan cihazın sadece belirlenen alanları ilaçlaması gereklidir.  arazisi sınırlarının belirlenmesi için QGroundControl yer kontrol istasyonu yazılımının 'fence' özelliği kullanılarak bir poligon oluşturulur ve otopilot yazılımına gönderilir.

Otopilot yazılımı, 'navigator' modülünün 'geofence' bloğunda (./src/modules/navigator/geofence.cpp) cihazın poligon sınırlarını işlediği ve poligon sınırlarının ihlal edilip edilmediğine dair 'navigator' modülüne "bool" tipinde değer döndüren fonksiyonlar vardır. Aşağıda çokgen bir poligonun, enlem, boylam ve yükseklik bilgileri ile birlikte cihazın poligon sınırları içerisinde olması durumunda :TRUE değeri döndüren 'insidePolygon' fonksiyonu görülmektedir ;

```
1. /* ./src/modules/navigator/geofence.cpp */
2.
3. bool Geofence::insidePolygon(const PolygonInfo &polygon, double lat, double lon, float altitude)
4. {
5. mission_fence_point_s temp_vertex_i;
6. mission_fence_point_s temp_vertex_j;
7. bool c = false;
8.
9. for (unsigned i = 0, j = polygon.vertex_count - 1; i < polygon.vertex_count; j = i++) {
10. if (dm_read(DM_KEY_FENCE_POINTS, polygon.dataman_index + i, &temp_vertex_i,
11. sizeof(mission_fence_point_s)) != sizeof(mission_fence_point_s)) {
12. break;
13. }
14.
15. if (dm_read(DM_KEY_FENCE_POINTS, polygon.dataman_index + j, &temp_vertex_j,
16. sizeof(mission_fence_point_s)) != sizeof(mission_fence_point_s)) {
17. break;
18. }
19.
20. if (temp_vertex_i.frame != NAV_FRAME_GLOBAL && temp_vertex_i.frame != NAV_FRAME_GLO-
  BAL_INT
21. && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT
22. && temp_vertex_i.frame != NAV_FRAME_GLOBAL_RELATIVE_ALT_INT) {
23. // TODO: handle different frames
24. PX4_ERR("Frame type %i not supported", (int)temp_vertex_i.frame);
25. break;
26. }
27.
28. if (((double)temp_vertex_i.lon >= lon) != ((double)temp_vertex_j.lon >= lon) &&
29. (lat <= (double)(temp_vertex_j.lat - temp_vertex_i.lat) * (lon - (double)temp_ver-
  tex_i.lon) /
30. (double)(temp_vertex_j.lon - temp_vertex_i.lon) + (double)temp_vertex_i.lat)) {
31. c = !c;
32. }
33. }
34. return c;
35. }
```

Poligonlar çokgen veya dairesel olarak farklı şekillere sahip olabilir, bu sebeple sınır ihlalini kontrol etmeden önce poligon tipinin ne olduğu bilmeli ve "Point Inclusion in Polygon Test – PNPOLY" algoritmasını bu bilgilere göre uygulamalıyız. Poligon tipinin ne olduğunu anlamak için 'Modülü'müzün bel kemiğini oluşturan "navigator" modülünün "geofence" bloğunda ki "isInsidePolygonOrCircle" fonksiyonundan yararlanıyoruz ;

```
1. /* ./src/modules/navigator/geofence.cpp */
2.
3. bool Geofence::isInsidePolygonOrCircle(double lat, double lon, float altitude)
4. {
5. if (dm_trylock(DM_KEY_FENCE_POINTS) != 0) {
6. return true;
7. }
8.
9. mission_stats_entry_s stats;
10. int ret = dm_read(DM_KEY_FENCE_POINTS, 0, &stats, sizeof(mission_stats_entry_s));
11.
12. if (ret == sizeof(mission_stats_entry_s) && _update_counter != stats.update_counter) {
13. _updateFence();
14. }
15.
16. if (isEmpty()) {
17. dm_unlock(DM_KEY_FENCE_POINTS);
18. return true;
19. }
20.
21. if (_altitude_max > _altitude_min) {
22. if (altitude > _altitude_max || altitude < _altitude_min) {
23. dm_unlock(DM_KEY_FENCE_POINTS);
24. return false;
25. }
26. }
27.
28. bool outside_exclusion = true;
29. bool inside_inclusion = false;
30. bool had_inclusion_areas = false;
31.
32. for (int polygon_idx = 0; polygon_idx < _num_polygons; ++polygon_idx) {
33. if (_polygons[polygon_idx].fence_type == NAV_CMD_FENCE_CIRCLE_INCLUSION) {
34. bool inside = insideCircle(_polygons[polygon_idx], lat, lon, altitude);
35.
36. if (inside) {
37. inside_inclusion = true;
38. }
39.
40. had_inclusion_areas = true;
41.
42. } else if (_polygons[polygon_idx].fence_type == NAV_CMD_FENCE_CIRCLE_EXCLUSION) {
43. bool inside = insideCircle(_polygons[polygon_idx], lat, lon, altitude);
44.
45. if (inside) {
46. outside_exclusion = false;
47. }
48.
49. } else {
50. bool inside = insidePolygon(_polygons[polygon_idx], lat, lon, altitude);
51.
52. if (_polygons[polygon_idx].fence_type == NAV_CMD_FENCE_POLYGON_VERTEX_INCLUSION) 
  {
53. if (inside) {
54. inside_inclusion = true;
55. }
56.
57. had_inclusion_areas = true;
58.
59. } else {
60. if (inside) {
```

```
61. outside_exclusion = false;
62. }
63. }
64. }
65. }
66.
67. dm_unlock(DM_KEY_FENCE_POINTS);
68.
69. return (!had_inclusion_areas || inside_inclusion) && outside_exclusion;
70. }
71.
```

'Modül' de kullanılan temel fonksiyonlarımız bunlar olmakla birlikte tanımlamaları sadece 'geofence' bloğunda mevcuttur. Fonksiyonlar oto pilot yazılımında 'navigator\_main' bloğunun "geofence\_breach\_check" fonksiyonunun altında, cihazın poligon sınırlarına yaklaştığında "Approaching on Geofence" uyarısını yer kontrol istasyonuna göndermek için kullanılmaktadır. Yani kullanım şekli poligon ihlaline değil, poligon sınırlarına yaklaşması koşuluna bağlı olup 'GeofenceBreachAvoidance" bloğunda belirlenen test nokta/uzaklıklarına göre belirlenir.

Test mekanizması ;

```
1. /* /src/modules/navigator/GeofenceBreachAvoidance/geofence_breach_avoidance.cpp */
2.
3. TEST_F(GeofenceBreachAvoidanceTest, waypointFromBearingAndDistance)
4. {
5.
6. GeofenceBreachAvoidance gf_avoidance(nullptr);
7. struct map_projection_reference_s ref = {};
8. Vector2d home_global(42.1, 8.2);
9. map_projection_init(&ref, home_global(0), home_global(1));
10.
11. Vector2f waypoint_north_east_local(1.0, 1.0);
12. waypoint_north_east_local = waypoint_north_east_local.normalized() * 10.5;
13. Vector2d waypoint_north_east_global = gf_avoidance.waypointFromBearingAndDistance(home_global, 
   M_PI_F * 0.25f, 10.5);
14.
15. float x, y;
16. map_projection_project(&ref, waypoint_north_east_global(0), waypoint_north_east_global(1), &x, 
   &y);
17. Vector2f waypoint_north_east_reprojected(x, y);
18.
19. EXPECT_FLOAT_EQ(waypoint_north_east_local(0), waypoint_north_east_reprojected(0));
20. EXPECT_FLOAT_EQ(waypoint_north_east_local(1), waypoint_north_east_reprojected(1));
21.
22. Vector2f waypoint_south_west_local = -waypoint_north_east_local;
23.
24. Vector2d waypoint_southwest_global = gf_avoidance.waypointFromBearingAndDistance(home_global, 
   M_PI_F * 0.25f, -10.5);
25.
26. map_projection_project(&ref, waypoint_southwest_global(0), waypoint_southwest_global(1), &x, &y);
27. Vector2f waypoint_south_west_reprojected(x, y);
28.
29. EXPECT_FLOAT_EQ(waypoint_south_west_local(0), waypoint_south_west_reprojected(0));
30. EXPECT_FLOAT_EQ(waypoint_south_west_local(1), waypoint_south_west_reprojected(1));
31.
32. Vector2d same_as_home_global = gf_avoidance.waypointFromBearingAndDistance(home_global, M_PI_F * 
   0.25f, 0.0);
33.
34. EXPECT_LT(Vector2d(home_global - same_as_home_global).norm(), 1e-4);
35. }
36.
```

#### Uyarı Mekanizması ;

```
1. /* /src/modules/navigator/navigator_main.cpp */
2. void Navigator::geofence_breach_check(bool &have_geofence_position_data)
3. {
4. ……
5. ……
6. ……
7. gf_violation_type.flags.fence_violation = !_geofence.isInsidePolygonOrCircle(fence_violation_test_po-
   int(0),
8. fence_violation_test_point(1),
9. _global_pos.alt);
10. ……
11. ……
12. ……
13. if (gf_violation_type.value) {
14. _geofence_result.geofence_violated = true;
15.
16. if (!_geofence_violation_warning_sent && _vstatus.arming_state == vehicle_sta-
   tus_s::ARMING_STATE_ARMED) {
17. mavlink_log_critical(&_mavlink_log_pub, "Approaching on Geofence");
18. ……
19. ……
20. ……
21.
```

Uyarı mekanizması kısmında görüleceği üzere "geofence\_breach\_check" fonksiyonu, yer kontrol istasyonundan sağlanan bir poligon verisi olup olmadığına bağlıdır. Uyarı mesajı ise "gf\_violation\_type" isminde poligon ihlallerine göre sınıflandırma yapan bir yapıdır ve "geofence" bloğunda ki 'isInsidePolygonOrCircle" fonksiyonunun döndürece değere göre değer alır. Aldığı değer doğrultusunda eğer bir poligon sınırına yaklaşılmış ise yarı mesajı yer kontrol istasyonuna "mavlink\_log\_critical" işlevi aracılığı ile iletilir.

'isInsidePolyhonOrCircle" fonksiyonunda ki 'lat', 'lon' parametreleri (burada 'fence\_violation\_test\_point(x)') test mekanizmasında ki fonksiyonlar aracılığı ile oluşturulup 2 boyutlu bir vektör yapısına indekslenerek burada kullanılıyor. Cihazın anlık yükseklik bilgisi ('alt') ise '\_global\_pos' yapısından alınıyor. Bizim istediğimiz anlık olarak poligon ihlalini kontrol etmek olduğundan 'Modül' fonksiyonumuzda 'lat' ve 'lon' parametrelerini de anlık olarak '\_global\_pos' yapısından almamız gerekiyor.

Modülde kullanılacak gerekli fonksiyonlar ve parametreler bu şekilde belirlendi. Modül ise 'navigator\_main' bloğunda oluşturuldu. 'navigator\_main' bloğunda oluşturma sebebimiz, ihtiyaç duyacağımız tüm tanımlamaların bu blokta hali hazırda yapılmış olmasıdır.

Modül alt yapısı;

```
1. /* /src/modules/navigator/navigator_main */
2. void Navigator::_modul_lama(bool &have_geofence_position_data, double lat, double lon, double
   alt)
3. {
4. bool ihlal = _geofence.isInsidePolygonOrCircle(lat, lon, alt);
5. int Aktif;
6.
6. if (!ihlal) {
7. Aktif = 2;
8. }
9. else {
10. Aktif = 1;
11. }}
```

Modülün temelini oluşturan ilk fonksiyon bu şekildedir. '\_modul\_lama' fonksiyonu, poligon verisini tanımlamak üzere "bool &have\_geofence\_position\_data", cihaz konumunu ve "geofence" fonksiyonlarını gerçekleştirmek üzere de "double lat", "double lon" ve "double alt" (enlem, boylam, yükseklik) parametrelerine ihtiyaç duymaktadır.

"ihlal", enlem, boylam ve yükseklik bilgilerine göre cihazın poligon sınırları içerisinde bulunup bulunmadığına dair 'bool' tipinde değer alır. Eğer cihaz poligon sınırları içerisinde ise TRUE döndürür ve "Aktif" tam sayı değişkeninin 1 değerini alması ile cihazın ilaç püskürtmesi istendiği anlaşılır. Aksi taktirde cihazın ilaç püskürtmemesi gerektiği anlaşılır.

'navigator' modülünün tüm işlevleri 'navigator\_main' bloğunda ki 'RUN' fonksiyonu üzerinden gerçekleştirilir. Öyleyse '\_modulu\_lama' fonksiyonu da burada ve başlık dosyasında tanımlanmalıdır ;

#### Yürütme ;

```
1. void
2. Navigator::run()
3. {
4. ……
5. ……
6. ……
7. _modul_lama(have_geofence_position_data, _global_pos.lat, _global_pos.lon, _global_pos.alt);
8. ……
9. ……
10. ……
11.
```

#### Başlık Dosyasında Tanımlama ;

```
1. class Navigator : public ModuleBase<Navigator>, public ModuleParams
2. {
3. public:
4. ……
5. ……
6. ……
7. void _modul(bool &have_geofence_position_data, double lat, double lon, double alt);
8. ……
9. ……
10. ……
11.
```

Modülün temel alt yapısı bu şekilde oluşturulmuştur.

# **3.2 uORB Kanalı Üzerinden Haberleşme**

Modülün fonksiyonel alt yapısı oluşturulduktan sonra, ilaç püskürtme, yer kontrol istasyonu ile haberleşme vb. işlevlerini fiziki olarak yerine getirebilmesi için diğer modüller ile iletişim halinde olması gerekir. PX4 otopilot yazılımında modüller arası iletişim uORB yapısı ile sağlanır. uORB, PX4 otopilot yazılımı içine gömülü olan yayınla/abone ol şeklinde çalışan özelleşmiş bir mesajlaşma protokolüdür.

![](_page_11_Figure_3.jpeg)

*Şekil 1 uORB Diyagramı - https://i0.wp.com/uav-lab.org/wp-content/uploads/2016/08/uORB-1 e1489045507250.jpg?fit=554%2C385&ssl=1*

Modül için ;

- uORB mesajlaşması için bir yayın kanalı (TOPIC) oluşturulacaktır.
- '\_modulu\_lama' fonksiyonunda "Aktif" değişkeninin aldığı değer, uORB mesajlaşması için oluşturulan struct yapısının bir parametresine eşitlenecektir.
- Eşitlenen değer, struct yapısında ki parametreleri içeren bir uORB kanalında yayınlanacaktır.

### **3.2.1 uORB Mesajlaşması için Yayın Kanalı (TOPIC)Oluşturmak**

uORB kanalı oluşturmak için otopilot yazılımında 'PX4/msg/' dizinine geliyorum ve oluşturacağım kanal ismi ile bir '.msg' dosyası oluşturuyorum. Modül iletişimi bu kanal üzerinden gerçekleşeceğinden '\_modulu.msg' dosyasını burada oluşturuyorum.

uORB mesajlaşması için kullanılan kanallar mutlaka bir 'timestamp' parametresi içerir. Bu parametre, kanal üzerinde bir yayın yapılmasından itibaren geçen süre bilgisini tutar, bu şekilde sistem modüllere istenilen bilgileri çok bekletmeden iletebilir. uORB Kanalı ;

- 1. # ./PX4-Autopilot/msg/\_modulu.msg
- 2. uint64 timestamp
- 3. uint16 \_durum
- 4.

Bir mesaj dosyası oluştururken şu hususlara dikkat edilmelidir :

- Mutlaka 'uint64' tipinde 'timestamp' parametresini içermelidir
- Tanımlanan parametrelertamamen küçük ya da tamamen büyük harften oluşmalıdır. Parametre "Durum" şeklinde isimlendirilemez

Daha sonra oluşturulan '\_modulu.msg' dosyasını yine 'PX4/msg/' dizininde ki 'CMakeList.txt' bloğunda tanımlanmalıdır. 'CMakeList.txt', sistemde ki modüller, bloklar gibi yapıların sistem içerisinde nerede, ne amaçla kullanılacağını belirleyen kural tanımlayıcısıdır.

#### CMakeList.txt ;

```
1. # ./PX4-Autopilot/msg/CMakeList.txt
2. cmake_policy(SET CMP0057 NEW)
3.
4. set(msg_files
5. …
6. …
7. _modulu.msg
8. …
9. …
10.
```

Bu işlemlerden sonra, sistemin 'CMakeList.txt' te tanımladığımız '\_modulu.msg' bloğuna göre gerekli yazılım/tanım dosyalarını oluşturması için otopilot yazılımın derlenmesi gerekmektedir. Aksi taktirde gerekli tanım dosyaları oluşturulamaz ve uORB mesajlaşması sağlanamaz.

### **3.2.2 uORB Kanalında Mesaj Yayınlamak**

Mesaj dosyamı oluşturduktan sonra tekrar '\_modul\_lama' fonksiyonuma gelip uORB üzerinde yayınlayacağım parametreleri içeren struct ve uORB da mesaj yayınlama işine geçiyorum.

- uORB üzerinde yayınlamak üzere struct oluştururken struct isminin oluşturulan mesaj dosyası ismi ile aynı olmasına dikkat edilmelidir.
- Oluşturulan struct, mesaj dosyası ile aynı parametreleri içermelidir.

Struct Tanımı ;

```
1. /* /src/modules/navigator/navigator.h */
2. class Navigator : public ModuleBase<Navigator>, public ModuleParams
3. {
4. public:
5. ……
6. void _modul(bool &have_geofence_position_data, double lat, double lon, double alt);
7. struct _modulu_nav_s {
8. uint64_t timestamp;
9. uint16_t Durum;
10. } ;
11. ……
12.
```

\_modul\_lama Fonksiyonu ;

```
1. /* /src/modules/navigator/navigator_main.cpp */
2. void Navigator::_modul_lama(bool &have_geofence_position_data, double lat, double lon, double
   alt)
3. {
4. bool ihlal = _geofence.isInsidePolygonOrCircle(lat, lon, alt);
5.
5. int Aktif;
7.
6. if (ihlal) {
7. Aktif = 2;
```

```
10. }
11. else {
12. Aktif = 1;
13. }
14.
14. struct _modulu_s NavStruct;
16.
15. memset(&NavStruct, 0, sizeof(NavStruct));
18.
16. orb_advert_t _modulu_pub = orb_advertise(ORB_ID(_modulu), &NavStruct);
20.
17. NavStruct.timestamp = hrt_absolute_time();
18. NavStruct._durum = Aktif;
23.
19. orb_publish(ORB_ID(_modulu), _modulu_pub, &NavStruct);
20. }
26.
```

Yukarıdaki, '\_modul\_lama' fonksiyonunun tam kodudur. '\_modulu\_s' isimli bir struct oluşturulmuştur ve bu struct, mesaj bloğunda tanımlanan 'timestamp' ve '\_durum' parametrelerini içermektedir. Struct, fonksiyon içerisinde 'NavStruct' şeklinde kullanılmıştır. Struct oluşturulduktan sonra, öncelikli olarak içerdiği parametreler sıfırlanmıştır, aksi taktirde rastgele değerler alabilirler. Sonrasında yayını yapılacak olan struct öncelikle uORB için bir 'publish' yapısı olarak belirlenmiştir. Fonksiyonun çalıştırılmasından itibaren süre tutulması ve cihazın poligon sınırları içerisinde ilaç püskürtmesi durumunu belirten 'Aktif' parametrelleri struct parametrelerine eşitlenmiş ve struct, '\_modulu' kanalı üzerinde yayınlanmıştır.

Otopilot konsolunda 'listener <module\_name>' komut sözdizimi ile bir uORB kanalı oluşturulduğunu ve yayın yapıldığını doğrulayabiliriz ;

Görüldüğü üzere 'timestamp' ve '\_durum' parametreleri "\_modulu" konu başlığı altında yayınlanıyor.

# **3.3 uORB Aboneliği ve MAVLink Üzerinden Yer İstasyonu ile Haberleşme**

### **3.3.1 MAVLink Mesajı Oluşturmak**

Yeni bir MAVLink mesajı oluşturmak için PX4 otopilot yazılımında 'mavlink/include/mavlink/v2.0/message\_definition s/' dizininde ki 'common.xml' dosyasında düzenlemeye gitmemiz gerekmektedir. 'common.xml' MAVLink mesajlarının tanımlarını/kurallarını içeren dosyadır.

```
1. -- mavlink/include/mavlink/v2.0/message_definitions/common.xml
2. ……
3. <messages>
4. ……
5. …… 
6. <message id="15000" name="_MODULU">
7. <description>MODUL MESAJI</description>
8. <field type="uint64_t" name="time_usec" units="us">Timestamp (UNIX Epoch time or time since 
   system boot). The receiving end can infer timestamp format (since 1.1.1970 or since system boot) by chec-
   king for the magnitude of the number.
9. </field>
10. <field type="uint16_t" name="lama_durumu">Hava aracının ilaçlama durumu bilgisini verir. 
   1 = İlaçlama Açık, 2 = İlaçlama Kapalı
11. </field>
12. </message>
13. ……
14. …… 
15. </messages>
16. ……
```

'common.xml' de tanımladığımız bir MAVLink mesajlar, benzersiz mesaj kimliğine sahip olmalıdır. Kimlik numaraları MAVLink sürümlerine göre değişiklik gösterebilir. MAVLink v1.x sürümü 1 byte (8-bit) olup en fazla 0-255 arasında mesaj kimlik numarası verilebilir, MAVLink v2.x sürümünde ise 4 byte (32-bit) olup 3 byte kadar kısmı kimlik numarası vermek için kullanılabilir. Biz MAVLink v2.x sürümünü kullanıyoruz ve mesaj kimlik numaramız 15000.

'common.xml' dosyasında yaptığımız mesaj tanımına istinaden, gerekli iletişim tanımlamalarının yapılması ve fonksiyonların oluşturulması için 'https://github.com/mavlink/mavlink' adresinden indirdiğimiz 'mavgenerate.py' programını kullanıyoruz. Sonrasında program çıktısı olan klasörü otopilot yazılımımızda ki 'mavlink/include/mavlink/v2.0/' dizinine kopyalıyoruz.

# **3.3.1 uORB Aboneliği ve MAVLink MesajınınGönderilmesi**

MAVLink mesajını oluşturduk, şimdi uORB kanalında yayınladığımız verileri alacak ve bu bilgileri yer kontrol istasyonuna gönderecek olan sistemi kurmamız gerekmektedir. 'src/modules/mavlink/streams/' dizininde, abonelik, mesaj gönderme ve işlemlerini gerçekleştiren fonksiyonların tanımlarının yapılacağı '\_MODULU.hpp' başlık dosyasını, hali hazırda çalışır durumda olan ve MAVLink mesajlaşmaları için kullanılan diğer modüllerden yararlanarak oluşturuyorum.

\_MODULU.hpp ;

```
1. /* /src/modules/mavlink/streams/_MODULU.hpp */
2. #ifndef __MODUL_HPP
3. #define __MODUL_HPP
4.
4. #include <uORB/topics/_modulu.h>
6.
5. class MavlinkStreamModulu : public MavlinkStream
6. {
7. public:
8. static MavlinkStream *new_instance(Mavlink *mavlink)
```

```
11. {
12. return new MavlinkStreamModulu(mavlink); 
13. }
14.
15. static constexpr const char *get_name_static
16. {
17. return "_MODULU";
18. }
19. static constexpr uint16_t get_id_static()
20. {
21. return MAVLINK_MSG_ID__MODULU;
22. } 
23. const char *get_name() const override
24. {
25. return get_name_static();
26. } 
27. uint16_t get_id() override
28. {
29. return get_id_static();
30. } 
31. unsigned get_size() override
32. {
33. return __modulu_sub.advertised() ? (MAVLINK_MSG_ID__MODULU_LEN + MAVLINK_NUM_NON_PAY-
   LOAD_BYTES) : 0; 
34. }
35.
36. private:
37. explicit MavlinkStreamModulu(Mavlink *mavlink) : MavlinkStream(mavlink) {}
38.
38. uORB::Subscription __modulu_sub
39. {
40. ORB_ID(_modulu), 0
41. }; 
42. bool send() override
43. {
44. _modulu_nav_s NavStruct; 
46.
45. if (__modulu_sub.update(&NavStruct))
46. {
47. mavlink__modulu_t msg__modul{}; 
50.
48. msg__modul.time_usec = NavStruct.timestamp;
49. msg__modul.lama_durumu = NavStruct.Durum; 
53.
50. mavlink_msg__modulu_send_struct(_mavlink->get_channel(), &msg__modul); 
55.
51. return true;
52. }
53. return false;
54. }
55. };
56. #endif
62.
```

- 2 ve 3. Satırlarda yer alan '#ifndef' ve '#define' kısımları, MAVLink mesajlarını yöneten 'mavlink\_messages.cpp' bloğu için gereklidir.
- '#include <uORB/topics/\_modulu.h> kısmında, uORB kanalı tanım dosyasını ekliyoruz.
- 'MavlinkStreamModulu' sınıfının 'Public' bloğunda, mesaj içeriğini oluşturan fonksiyonların, tanımlamaların ve içerdiği parametrelerin, mesaj gönderileceği esnada hangi fonksiyonlar/bloklar tarafından çağrılacağını belirlemek için etiketleme işlemi yapılıyor.

• 'Private' bloğunda, oluşturduğumuz uORB mesajına abone olunuyor. Ardından MAVLink mesajının içeriğini barındıran bir struct (mavlink\_\_modulu\_t) oluşturuluyor. uORB mesajında içeriği gönderilen '\_modulu\_nav\_s' struct parametrelerinin değerleri 'mavlink\_\_modulu\_t' parametrelerine atanıyor ve mesaj MAVLink aracılığı için gönderilmek üzere paketleniyor.

Artık mesajımız gönderilmek üzere etiketlendi ve paketlendi. Gönderme işlemini ise '/src/modules/mavlink/' dizininde ki 'mavlink\_messages.cpp' ve 'mavlink\_main.cpp' blokları ile gerçekleştiriyoruz.

mavlink\_messages.cpp ;

```
1. ……
2. #include <uORB/topics/_modulu.h>
3. ……
4. #include "streams/_MODULU.hpp"
5. ……
6. ……
7. #if defined(__MODUL_HPP)
8. create_stream_list_item<MavlinkStreamModulu>()
9. #endif // __MODUL_HPP
10. ……
11. ……
12.
```

#### mavlink\_main.cpp ;

```
1. void
2. Mavlink::send_protocol_version()
3. {
4. ……
5. switch (_mode) {
6. case MAVLINK_MODE_NORMAL:
7. ……
8. ……
9. configure_stream_local("_MODULU", 1.0f);// configure_stream_local(<message_def_name>, HZ cinsinden gönderme hızı);
10. ……
11.
```

### **4** SONUÇ

Cihazın görevini beklendiği üzere gerçekleştirebilmesi için gerekli görevler PX4 yazılımına eklenen Modül tarafından işlenecektir. Söz konusu cihazın uçuş kontrol yazılımı PX4 otopilot tabanlı ve yer kontrol istasyonu QGroundControl tabanlıdır. Bu raporda otopilot yazılımı içerisinde ki modülünün nasıl oluşturulduğu ve PX4 otopilot yazılımında bir modül oluştururken nelere dikkat edilmesi gerektiği detayları ile özetlenmiştir.

sal ilaçlama cihazında kullanılacak olan PX4 tabanlı otopilot yazılımının oluşturulması için PX4 otopilot yazılımına ek işlevsellik kazandırılması gerekmektedir, bunun için PX4 yazılımında bir modül oluşturularak bu işlevsellikleri kazandıracak olan  modülü yazılımının temelleri oluşturulmuştur. modül yazılımı ilk sürümü itibari ile yer kontrol istasyonu aracılığı ile belirlenen coğrafi sınırlar ve uçuş rotası bilgilerini kullanarak cihazın rota üzerinde iken coğrafi sınırlar içerisinde ilaç püskürtmesi ve coğrafi sınırları ihlal etmesi halinde ilaç püskürtme işlemini sonlandırma özelliklerine sahiptir.

Bu raporda, sal ilaçlama kopteri ve PX4 tabanlı otopilot yazılımının bir parçası olan yazılımın temel yapısına ilişkin kodlama ve PX4 otopilot yazılımına yeni modül eklenmesi çalışmaları kapsamında gerçekleştirilen ;

- Bir uORB kanalı için mesaj bloğu/dosyası oluşturulması ve gerekli tanımlamaların yapılması
- Cihazın, modülün işlevlerini gerçekleştirebilmesi için gerekli fonksiyonların yazılması ve uORB üzerinde sonuçlarının mesaj olarak yayınlanması
- uORB kanalı üzerinden yayınlanan mesajdan gerekli parametrelerin çekilerek MAVLink üzerinden yer kontrol istasyonuna gönderilmesi ve kullanılması

işlerine değinilmiş ve detaylıca incelenmiştir.

### **5** KAYNAKÇA

- <https://nxp.gitbook.io/hovergames/developerguide/px4-tutorial-example-code/hg-px4-example-lab-1>
- <https://nxp.gitbook.io/hovergames/developerguide/px4-tutorial-example-code/hg-px4-example-lab-2>
- <https://nxp.gitbook.io/hovergames/developerguide/px4-tutorial-example-code/hg-px4-example-lab-3>
- <https://nxp.gitbook.io/hovergames/developerguide/px4-tutorial-example-code/hg-px4-example-lab-4>
- <https://nxp.gitbook.io/hovergames/developerguide/px4-tutorial-example-code/hg-px4-example-lab-5>
- <https://www.hackster.io/mdobrea/communication-through-custom-uorb-and-mavlink-messages-269ebf>
- <https://github.com/mavlink/mavlink>
- <https://uav-lab.org/2016/08/02/px4-research-log4-a-first-look-at-px4-architecture/>
- <https://px4.io/px4-uorb-explained-part-1/>
- <https://px4.io/px4-uorb-explained-part-2/>
- <https://px4.io/px4-uorb-explained-part-3-the-deep-stuff/>