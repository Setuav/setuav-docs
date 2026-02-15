# Uçuş Performansı

Uçuş performansı analizi, aerodinamik ve itki çıktılarından hız zarfını, görev hızlarını, tırmanma kabiliyetini, menzil ve dayanımı türetir. Bu katmanda temel amaç, her hız noktasında seviye uçuş için gereken itki/güç ile aynı hız noktasında sistemin sağlayabildiği itki/gücü karşılaştırarak fiziksel olarak tutarlı bir görev resmi üretmektir.

## Yöntem

Setuav bu problemi hız taraması üzerinde bağlı bir çözüm olarak ele alır. Önce aerodinamik tarafta seviye uçuş için $C_L$, $C_D$, sürükleme ve gereken güç hesaplanır; ardından aynı hız noktaları için itki tarafında tek model kullanılır. Bu tek model, `PropulsionAnalyzer.solve_required_thrust_sweep(...)` çağrısıdır ve itki analizinde kullanılan aynı motor-pervane-batarya denge çözücüsünü kullanır. Böylece uçuş performansındaki menzil/dayanım, cruise ve zarf metrikleri ile itki analizindeki elektriksel model birbiriyle uyumlu kalır.

## Hazırlık

### Girdi Hazırlığı

Uçuş performansı çözücüsü aşağıdaki girdileri kullanır ve çözüm değişkenlerine dönüştürür:

| Girdi | Çözüm karşılığı |
| --- | --- |
| `aero.cl_max` | Stall hızı hesabında $C_{L,\max}$ |
| `aero.area` | Referans alan $S$ (`mm^2 -> m^2`) |
| `aero.polars.cl_values`, `aero.polars.cd_values` | $C_D(C_L)$ enterpolasyonu |
| `aero.cd_min` | Polar yoksa sabit sürükleme fallback'i |
| `aero.ld_max` | `max_ld_ratio`, `glide_ratio` |
| `aero.operating_velocity` | `best_ld` hız çıktısı |
| `total_mass` | Ağırlık $W = mg$ |
| `air_density` | Yoğunluk $\rho$ |
| `battery_capacity` | Enerji tabanlı menzil/dayanım hesabı (`mAh -> Ah`) |
| `design.propulsion` | Gerilim uyumluluğu kontrolü ve itki çözücü kurulumu |

### Konfigürasyon Girdileri

Bu sınıfta doğrudan kullanılan ayarlar aşağıdadır:

| Konfigürasyon alanı | Çözüm karşılığı |
| --- | --- |
| `config.performance.velocity_min` | Hız taraması alt sınırı |
| `config.performance.velocity_max` | Hız taraması üst sınırı |
| `config.performance.velocity_steps` | Hız noktası sayısı |
| `config.performance.stall_margin` | Minimum güvenli hız: $V_{\min}=V_{\text{stall}}\cdot\text{stall\_margin}$ |
| `config.propulsion.usable_capacity_ratio` | Kullanılabilir batarya enerjisi katsayısı |

## Çözüm

Çözüm, hız ekseninde aynı adımların tekrarıyla yürür ve tüm eğriler aynı hız dizisi üzerinde üretilir.

### Stall Hızı ve Hız Aralığı

Stall hızı standart kaldırma dengesiyle hesaplanır:

$$
V_{\text{stall}}=\sqrt{\frac{2W}{\rho S C_{L,\max}}}
$$

Tarama başlangıcı doğrudan konfigürasyon alt sınırı değildir; güvenli başlangıç hızı
$V_{\text{start}}=\max(V_{\text{stall}}\cdot\text{stall\_margin},\,V_{\min,config})$
şeklinde seçilir.

### Seviye Uçuş Güç Gereksinimi

Her hız noktasında önce seviye uçuş için gereken kaldırma katsayısı bulunur:

$$
C_{L,\text{req}} = \frac{W}{0.5\rho V^2 S}
$$

Ardından $C_D$, polar tablosundan $C_D(C_L)$ enterpolasyonu ile hesaplanır; polar sınırlarının dışına çıkılırsa parabolik model kullanılır:

$$
C_D = C_{D0} + k C_L^2
$$

Sürükleme ve gereken güç:

$$
D = 0.5\rho V^2 S C_D,
\qquad
P_{\text{req}} = D\,V
$$

### İtki ile Bağlı Çözüm (Tek Model)

Aynı hız dizisi için itki çözücüsüne gereken itki eğrisi verilir:

$$
[T_{\text{avail}}(V),\;P_{\text{avail}}(V),\;P_{\text{batt,req}}(V),\;\text{feasible}(V)]
\leftarrow
\texttt{solve\_required\_thrust\_sweep}
$$

Buradaki $P_{\text{avail}}$ itki tarafında kullanılabilir **itici güç**tür (propulsive power),
$P_{\text{batt,req}}$ ise gereken uçuş noktasını tutmak için batarya tarafında çekilen güçtür.

### Dayanım ve Menzil Eğrileri

Enerji hesabı tek elektrik modelinden gelen $P_{\text{batt,req}}$ ile yapılır. Kullanılabilir batarya enerjisi:

$$
E_{\text{batt}} = V_{\text{nom}}\,Q_{\text{Ah}}\,\eta_{\text{usable}}
$$

Sadece geçerli ve güç gereksinimi pozitif olan hız noktalarında:

$$
\text{Endurance}(V) = \frac{E_{\text{batt}}}{P_{\text{batt,req}}(V)},
\qquad
\text{Range}(V) = V\cdot 3.6\cdot \text{Endurance}(V)
$$

### Optimal Hızlar

Geçerli görev maskesi için:

- `best_endurance`: $P_{\text{batt,req}}$ minimum olan hız
- `best_range`: `range_curve` maksimum olan hız
- `best_climb`: $(P_{\text{avail}}-P_{\text{req}})$ maksimum olan hız
- `best_ld`: doğrudan `aero.operating_velocity`

Bu sınıfta raporlanan `cruise_speed`, ham hesap içinde `best_range` olarak atanır.
Yani seçim ölçütü doğrudan menzilin maksimize edilmesidir; ayrı bir UI/operasyon
kuralı bu adıma dahil değildir.

Ham veri açısından alternatif cruise tanımları:

1. `best_endurance` tabanlı cruise:  
   $P_{\text{batt,req}}(V)$ minimum hız seçilir; amaç maksimum havada kalış süresidir.
2. `best_climb` tabanlı cruise:  
   $(P_{\text{avail}}-P_{\text{req}})$ maksimum hız seçilir; amaç tırmanış fazı odaklıdır.
3. `best_ld` tabanlı cruise:  
   Aerodinamik verimlilik referans hızı (`aero.operating_velocity`) seçilir.

### Performans Metrikleri ve Cruise Noktası

Maksimum hız, geçerli maskede son hız noktası olarak alınır. Tırmanma oranı:

$$
\text{RoC}(V)=\frac{P_{\text{avail}}(V)-P_{\text{req}}(V)}{W}
$$

En iyi tırmanma açısı, en yüksek excess-power noktasında
$\sin\gamma=\text{RoC}/V$ ile hesaplanır.

Cruise hesabı da ayrı bir yaklaşık model değil, yine aynı itki çözücüsünün
`solve_required_thrust_operating_point(...)` çağrısı ile yapılır.

### Geçersizlik ve Fallback Davranışı

Batarya gerilimi motor/ESC penceresi ile uyumsuzsa analiz sıfırlanmış bir sonuç döndürür.
İtki çözücüsü yoksa güç/itki kullanılabilir eğrileri sıfır kalır ve powered metrikler sınırlanır.
Batarya kapasitesi yoksa menzil/dayanım eğrileri ve bu metrikler sıfır kalır.
