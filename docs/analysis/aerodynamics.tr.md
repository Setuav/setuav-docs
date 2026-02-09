# Aerodinamik

Aerodinamik analiz, uçağın hangi hız ve hücum açısı aralığında güvenli ve verimli uçabileceğini görmek için yapılır. Bu hesaplama; kaldırma, sürükleme ve verimlilik davranışını ortaya koyarak stall sınırlarını, uçuş marjlarını ve performans analizine girecek temel verileri üretir.

## Yöntem

Bu aşamada AeroSandbox yazılımı kullanılır. AeroSandbox, Peter D. Sharpe tarafından uçak tasarım optimizasyonunu hızlandırmak ve aerodinamik/itki/yapı/misyon gibi havacılık analizlerini aynı sayısal çerçevede çözmek amacıyla geliştirilmiş açık kaynaklı bir Python mühendislik yazılımıdır. Setuav içinde tercih edilme nedeni, parametrik hava aracı geometrisini ve uçuş koşullarını doğrudan modele alabilmesi ve hücum açısı taramasında tutarlı `CL`/`CD` üretmesidir. Güvenirlik açısından yöntem açık kaynak kodu, MIT lisansı, sürüm geçmişi, kamuya açık dokümantasyon ve doğrudan atıf verilen MIT tez çalışması üzerinden izlenebilir ve denetlenebilir durumdadır ([AeroSandbox](https://peterdsharpe.github.io/AeroSandbox/), [GitHub](https://github.com/peterdsharpe/AeroSandbox), [MIT 2021 Tezi](https://dspace.mit.edu/handle/1721.1/140023)).

Bu aşamada kullanılan başlıca AeroSandbox bileşenleri şunlardır:

- `Airplane`
- `Wing` ve `WingXSec`
- `Fuselage` ve `FuselageXSec`
- `Atmosphere`
- `OperatingPoint`
- `AeroBuildup`

## Hazırlık

### Geometri Hazırlığı

AeroSandbox tarafında geometri, üst seviyede bir `Airplane` nesnesi olarak temsil edilir; bu nesnenin içinde kanatlar `Wing` (kesitler `WingXSec`), gövde ise `Fuselage` (kesitler `FuselageXSec`) ile tanımlanır. Yani specification içindeki profil/kesit tabanlı geometri, analizde doğrudan kesit listeleri üzerinden tanımlanan bu nesnelere dönüştürülür ve her kesitte konum, boyut, açı ve profil bilgisi açıkça taşınır.

| Specification alanı | AeroSandbox karşılığı | Dönüşüm / Not |
| --- | --- | --- |
| `airframe.wings[].component.geometry.profiles[].position.(x,y,z)` | `WingXSec.xyz_le` | `mm -> m` dönüşümü uygulanır |
| `airframe.wings[].component.geometry.profiles[].chord` | `WingXSec.chord` | `mm -> m` dönüşümü uygulanır |
| `airframe.wings[].component.geometry.profiles[].rotation.x` | `WingXSec.twist` | Derece cinsinden aktarılır |
| `airframe.wings[].component.geometry.profiles[].airfoil` | `WingXSec.airfoil` | NACA / dosya / koordinat tipine göre `asb.Airfoil` oluşturulur |
| `airframe.wings[].attachment.position.(x,z)` | `WingXSec.xyz_le` ofseti | `mm -> m`; kanat bağlama `y` ofseti merkez hatta sabitlenir |
| `airframe.wings[].attachment.rotation.(x,y,z)` | `WingXSec.xyz_le` dönüşümü | XYZ dönüş matrisiyle uygulanır |
| `airframe.wings[].attachment.mirror` | `Wing.symmetric` | Modeldeki `mirror` değeri doğrudan kullanılır |
| `airframe.fuselage.geometry.segments[].sections[].position.(x,y,z)` | `FuselageXSec.xyz_c` | `mm -> m` dönüşümü uygulanır |
| `airframe.fuselage.geometry.segments[].sections[].profile` | `FuselageXSec.width`, `FuselageXSec.height` | `circle/ellipse/rectangle/rounded_rectangle` tipine göre boyutlar atanır |
| Aerodinamik referans noktası | `Airplane.xyz_ref` | Bu aşamada `[0.0, 0.0, 0.0]` olarak atanır |

### Uçuş Koşulu Hazırlığı

Geometri hazırlandıktan sonra analiz koşulları AeroSandbox'ın atmosfer ve uçuş durumu nesnelerine dönüştürülür. Bu adımda çevresel koşullar `Atmosphere` ile tanımlanır ve çözümde kullanılacak anlık durum `OperatingPoint` ile kurulur.

| Analiz koşulu alanı | AeroSandbox karşılığı | Dönüşüm / Not |
| --- | --- | --- |
| `conditions.altitude_msl` | `Atmosphere.altitude` | `m` biriminde doğrudan aktarılır |
| `conditions.temperature` | `create_atmosphere(..., temperature_c)` | `°C` girdisi atmosfer kurulumunda kullanılır |
| `conditions.air_density` | `Atmosphere.temperature_deviation` ile eşlenmiş atmosfer | Yoğunluk override varsa hedef yoğunluğa göre sıcaklık sapması sayısal olarak çözülür |
| Aerodinamik referans hız (analiz kurulumu) | `OperatingPoint.velocity` | Mevcut akışta başlangıç aerodinamik çözümü `15.0 m/s` ile kurulur |
| Başlangıç açıları | `OperatingPoint.alpha`, `OperatingPoint.beta` | Başlangıçta `0°` alınır; `alpha` taraması sonraki çözüm adımında yapılır |

## Çözüm

### Aerodinamik Katsayı Çözümü

Bu adımda `alpha` taraması oluşturulur ve her açı noktasında `AeroBuildup` çalıştırılarak katsayılar elde edilir. Elde edilen noktasal `CL`/`CD` dizileri, Setuav tarafında performans raporunda kullanılan türetilmiş metriklere dönüştürülür.

`AeroBuildup`, AeroSandbox içinde uçağın toplam aerodinamik davranışını bileşen katkılarını birleştirerek hesaplayan çözücüdür. Kanat ve gövde geometrisini, uçuş durumunu (`OperatingPoint`) ve atmosfer bilgisini birlikte kullanarak her `alpha` noktasında toplam kaldırma ve sürükleme katsayılarını üretir; bu üretimde viskoz etkiler, stall davranışı ve konfigürasyona göre dalga sürüklemesi etkileri de modele dahil edilir. Setuav, bu çözücüden doğrudan `CL` ve `CD` alıp kalan metrikleri kendi tarafında türetir.

| Çözüm adımı | Uygulama | Çıktı |
| --- | --- | --- |
| `alpha` taraması | `alpha_min`, `alpha_max`, `alpha_steps` ile doğrusal tarama oluşturulur | `alpha` noktaları |
| Çözücü konfigürasyonu | `include_wave_drag` ve çözünürlükten türetilen `model_size` (`small/medium/large`) ile `AeroBuildup` yapılandırılır | Konfigürasyona bağlı katsayı çözümü |
| Durum varsayımları | Her `alpha` noktasında `beta=0`, `p=0`, `q=0`, `r=0` kabulüyle çözüm çalıştırılır | Yalın (quasi-steady) katsayı seti |
| Noktasal katsayı çözümü | Her `alpha` için `AeroBuildup(airplane, op_point)` çalıştırılır | `CL(alpha)`, `CD(alpha)` |
| Verimlilik hesabı | Her noktada `L/D = CL / CD` hesaplanır | `L/D(alpha)` |
| Ana metrik çıkarımı | Dizi üzerinden `CL_max`, `CD_min`, `L/D_max` seçilir | Özet aerodinamik metrikler |
| Referans geometri | Kanat kesitlerinden span/alan hesaplanır; `mean_chord` ve `AR` türetilir | `span`, `area`, `mean_chord`, `aspect_ratio` |
| Reynolds hesabı | `Re = rho * V * mean_chord / mu` | `reynolds_number` |
| Rapor birim dönüşümü | Geometri çıktıları rapora yazılırken `m -> mm`, `m² -> mm²` çevrilir | Spesifikasyon birimlerinde rapor çıktısı |
