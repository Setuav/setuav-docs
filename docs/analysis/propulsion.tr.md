# İtki

İtki analizi, bir elektrikli tahrik sisteminin belirli bir uçuş hızında ne kadar itki üretebildiğini ve bunu hangi elektriksel maliyetle yaptığı hesaplamadır. Sonuç; motorun fiziksel özelliklerine ($K_v$, $R$, $I_0$, akım limiti), bataryanın elektriksel durumuna (nominal gerilim, iç direnç, verim, kullanılabilir kapasite), ESC dönüşüm verimine, pervanenin geometrisine ve aerodinamik karakteristiklerine ($C_t$, $C_p$), ayrıca çevre koşullarına (özellikle hava yoğunluğu $\rho$) doğrudan bağlıdır. Fiziksel olarak hesap; hız ve devirden ilerleme oranı $J$’nin bulunması, buradan pervane itki/tork yükünün türetilmesi, motorun gerilim-akım-tork dengesinin çözülmesi, ardından batarya/ESC kayıpları ve gerilim düşümünün geri besleme ile aynı noktaya taşınması şeklinde tek bir bağlı enerji-zorlanım zinciri olarak yürür.

## Yöntem

Setuav itki analizi bu problemi, verilen uçuş hızı $V$ ve throttle $u$ (veya gerekli itki) için pervane yükü, motor elektrik denklemleri ve batarya/ESC enerji dönüşümünü aynı anda sağlayan bağlı bir denge problemi olarak ele alır. Çözücü önce $J$ ile pervane çalışma noktasını kurar, buradan oluşan tork-itki talebini motor tarafındaki $K_v$-$R$-$I_0$ dengesiyle iteratif olarak eşleştirir. Her iterasyonda gerilim düşümü ve verim kayıpları geri beslenerek tek bir yakınsak işletim noktası bulunur; böylece statik ve dinamik çıktılar aynı fizik modelinden üretilir.

## Hazırlık

### Bileşen Hazırlığı

Bu adımın amacı, itki sistemini çözücüye hazır ve fiziksel olarak tutarlı bir giriş setine dönüştürmektir. Kontrol sonrası hesap, tek bir bileşen yığınıyla yürür; analiz sırasında eksik/çelişkili veriyle ilerlenmez.

Hazırlıkta üç temel grup normalize edilir: motor elektrik parametreleri, batarya paket parametreleri ve pervane geometrisi. Böylece sonraki çözüm adımında her işletim noktası aynı veri yapısı üzerinden çözülür.

| Girdi | Çözüm karşılığı |
| --- | --- |
| `propulsion.motors[].kv` | $K_v$ |
| `propulsion.motors[].resistance` | $R$ (sıcaklıkla ölçeklenmiş motor direnci) |
| `propulsion.motors[].no_load_current` | $I_0$ |
| `propulsion.motors[].current_max` | $I_{\max}$ |
| `propulsion.batteries[].voltage_nominal` | $V_{\mathrm{nom}}$ |
| `propulsion.batteries[].cells_series` | $n_{\mathrm{cells}}$ |
| `propulsion.batteries[].cells_parallel` | Paralel kol sayısı ($n_{\mathrm{parallel}}$) |
| `propulsion.batteries[].cell_resistance` | $R_{\mathrm{pack}}$ bileşeni |
| `propulsion.propellers[].(diameter,pitch,blade_count)` | Pervane çalışma noktası için geometri girdisi |

### Koşul Hazırlığı

Koşul hazırlığının amacı, analizde verilen çevre ve görev bilgilerini itki denklemlerinin doğrudan kullandığı çözüm değişkenlerine dönüştürmektir. Bu adım özellikle atmosfer durumu ve rapor metrikleri için gerekli alanları normalize eder.

| Analiz girdisi | Çözüm karşılığı |
| --- | --- |
| `conditions.altitude_msl` | Atmosfer modeli irtifa girdisi |
| `conditions.temperature` | Atmosfer modeli sıcaklık girdisi |
| `conditions.air_density` | $\rho$ (doğrudan yoğunluk override değeri) |
| `conditions.total_mass` | $\mathrm{thrust\_to\_weight}$ raporlaması için kütle |

### Analiz Konfigürasyon Girdileri

Tasarım ve atmosfer girdilerine ek olarak, aşağıdaki tabloda arayüzde doğrudan sunulan analiz parametreleri ve çözümdeki karşılıkları verilmiştir.

| Konfigürasyon alanı | Çözüm karşılığı |
| --- | --- |
| `config.propulsion.use_battery_internal_resistance` | Sag model anahtarıdır; batarya iç direncinin iterasyona dahil edilmesini açar/kapatır. |
| `config.propulsion.motor_efficiency_default` | Motor elektrik gücü için alt sınır verimidir; $P_{\mathrm{motor,elec}}$ hesabında $\max\!\left(V_{\mathrm{motor}}I_{\mathrm{motor}},\, P_{\mathrm{shaft}}/\eta_{\mathrm{default}}\right)$ şeklinde kullanılır. |
| `config.propulsion.back_emf_scale` | $\eta_{\mathrm{bemf}}$ kalibrasyon katsayısıdır; daha düşük değer aynı RPM için daha yüksek gerilim/akım gerektirir. |
| `config.propulsion.usable_capacity_ratio` | Kullanılabilir enerji katsayısıdır; itki çıktılarındaki batarya enerji bütçesini (özellikle süre tahmini) ölçekler. |
| `config.propulsion.battery_discharge_efficiency` | $\eta_{\mathrm{batt}}$ verimidir; motor elektrik talebini batarya tarafı güç talebine dönüştürür. |
| `config.propulsion.esc_efficiency` | $\eta_{\mathrm{esc}}$ verimidir; ESC üzerindeki dönüşüm kayıplarını batarya çekişine yansıtır. |
| `config.propulsion.rpm_steps` | Statik sweep çözünürlüğüdür; statik çözümde örneklenen işletim noktası sayısını belirler. |
| `config.propulsion.motor_max_temperature` | $T_{\max}$ limitidir; geçerli işletim noktası için motor sıcaklık üst sınırını belirler. |
| `config.propulsion.motor_thermal_resistance` | $R_{\mathrm{th}}$ katsayısıdır; motor kayıp gücünü sıcaklık artışına çevirir. |
| `config.propulsion.cooling_level` | Soğutma seviyesidir (1-5); etkin termal direnci çarpar: 1→1.00, 2→0.95, 3→0.80, 4→0.75, 5→0.70. |

## Çözüm

Çözüm, bağlı bir işletim noktası araması olarak yürütülür: önce hız ve devirden pervane çalışma noktası kurulur, ardından oluşan tork-itki talebi motorun gerilim-akım dengesiyle eşleştirilir. Bu denge batarya/ESC kayıpları ve gerilim düşümü geri beslemesiyle tekrar güncellenir; sınır kontrollerini geçen nokta geçerli çözüm olarak raporlanır.

Güç zinciri, çözüm içinde doğrudan aşağıdaki enerji dönüşümü olarak ele alınır:

$$
P_{\mathrm{bat}}
\;\xrightarrow[\mathrm{ESC}]{\eta_{\mathrm{esc}}}\;
P_{\mathrm{motor}}
\;\xrightarrow[\mathrm{Motor}]{I^2R,\;I_0,\;\eta_{\mathrm{bemf}}}\;
P_{\mathrm{shaft}}
\;\xrightarrow[\mathrm{Pervane}]{C_t,\;C_p,\;J}\;
T
$$

Burada $P_{\mathrm{bat}}$ bataryadan çekilen gücü, $P_{\mathrm{motor}}$ motor elektrik giriş gücünü, $P_{\mathrm{shaft}}$ şaft gücünü ve $T$ itki çıktısını temsil eder. Okların üzerindeki terimler, her geçişte baskın olan kayıp veya dönüşüm mekanizmasını ifade eder.

Bu geçişlerin kayıp özeti:

| Aşama | Geçiş | Kayıp mekanizması |
| --- | --- | --- |
| ESC | $P_{\mathrm{bat}} \rightarrow P_{\mathrm{motor}}$ | Anahtarlama/iletim kayıpları, $P_{\mathrm{motor}} \approx P_{\mathrm{bat}}\eta_{\mathrm{esc}}$ |
| Motor | $P_{\mathrm{motor}} \rightarrow P_{\mathrm{shaft}}$ | Bakır kaybı $P_{\mathrm{Cu}} = I_{\mathrm{motor}}^2R$, boşta kayıp $P_0 \approx V_{\mathrm{motor}}I_0$, ek manyetik/mekanik kayıplar |
| Pervane | $P_{\mathrm{shaft}} \rightarrow T$ | Aerodinamik kayıplar; çalışma noktası $C_t/C_p$ ve $J$ ile belirlenir |

### Pervane İşletim Noktası

Hesap, pervane ilerleme oranı $J$ (Advance Ratio) hesaplanarak başlar. Uçuş hızı $V$, devir $n$ ve çap $D$ ile $J$ bulunur:

$$
J = \frac{V}{nD}
$$

J, pervanenin havada ne kadar "verimli" ilerlediğini gösteren boyutsuz bir parametredir. Pervanenin bir devirde aldığı yol ile çapının oranıdır.

Bu denklemde $J$, pervanenin ileri uçuşta ne kadar "boşta" veya ne kadar "yükte" çalıştığını belirleyen temel parametredir. $J$ arttıkça aynı devirde pervane diskinden geçen eksenel akış artar; bu da tipik olarak $C_t$ ve $C_p$ davranışını değiştirir.

Bulunan $J$ değeri ve $\mathrm{RPM}$ ile veritabanından katsayılar okunur:

$$
C_t = C_t(\mathrm{RPM}, J),
\qquad
C_p = C_p(\mathrm{RPM}, J)
$$

### Aerodinamik Yükten Tork ve Güç

Katsayılar okunduktan sonra aynı işletim noktasında tork $Q_{\mathrm{prop}}$, itki $T$ ve şaft gücü $P_{\mathrm{shaft}}$ türetilir. Burada $\rho$ hava yoğunluğudur:

$$
Q_{\mathrm{prop}} = \frac{C_p \rho n^2 D^5}{2\pi}
$$

$$
T = C_t \rho n^2 D^4
$$

$$
P_{\mathrm{shaft}} = C_p \rho n^3 D^5
$$

Bu üç denklem birlikte, pervane aerodinamiğinin elektrik tarafına bindirdiği yükü tanımlar. Özellikle $D$ çapının kuvvetli üstel etkisi ($D^4$, $D^5$), küçük çap değişimlerinin bile tork ve güç gereksinimini belirgin şekilde değiştirmesine neden olur.

### Motor Elektriksel Denge

Motor denge adımında tork-akım ve gerilim bağı birlikte çözülür. Bu adımda $K_v$ hız sabiti, $K_t$ tork sabiti, $I_0$ boşta akım, $R$ ise sıcaklıkla ölçeklenmiş motor direncidir:

$$
I_{\mathrm{motor}} = \frac{Q_{\mathrm{prop}}}{K_t} + I_0
$$

$$
V_{\mathrm{back}} = \frac{\mathrm{RPM}}{K_v \eta_{\mathrm{bemf}}}
$$

$$
V_{\mathrm{motor}} \approx V_{\mathrm{back}} + I_{\mathrm{motor}}R
$$

Buradaki $\eta_{\mathrm{bemf}}$, gelişmiş itki ayarlarındaki `back_emf_scale` parametresidir. Bu adımın fiziksel anlamı şudur: pervane torku arttıkça $I_{\mathrm{motor}}$ artar, akım arttıkça hem $I^2R$ kaybı hem de gerekli terminal gerilim artar. Çözüm, bu dengeyi aynı anda sağlayan $\mathrm{RPM}$ ve akım kombinasyonunu iteratif olarak arar.

### Batarya ve ESC Güç Eşlemesi

Motor tarafındaki elektrik gücü batarya tarafına, ESC ve batarya verimleri üzerinden taşınır. Burada $\eta_{\mathrm{esc}}$ ESC verimi, $\eta_{\mathrm{batt}}$ batarya deşarj verimidir. Çözücü, motor elektrik gücünü hem gerilim-akım çarpımına hem de şaft/verim alt sınırına göre muhafazakâr biçimde seçer:

$$
P_{\mathrm{motor,elec}} =
\max\!\left(
V_{\mathrm{motor}} I_{\mathrm{motor}},
\frac{P_{\mathrm{shaft}}}{\eta_{\mathrm{default}}}
\right)
$$

$$
P_{\mathrm{batt}} = \frac{P_{\mathrm{motor,elec}}}{\eta_{\mathrm{esc}} \eta_{\mathrm{batt}}}
$$

$$
I_{\mathrm{pack}} = \frac{P_{\mathrm{batt}}}{V_{\mathrm{pack}}}
$$

Bu dönüşümde kritik nokta, kayıpların ardışık olarak birikmesidir: aynı şaft gücü ihtiyacı için $P_{\mathrm{batt}}$, her verim katmanında büyüyerek daha yüksek batarya akımı gerektirir.

### Gerilim Düşümü Geri Beslemesi

Yük akımı belirlendikten sonra efektif paket direnci üzerinden gerilim düşümü güncellenir ve çözüme geri beslenir. Burada $R_{\mathrm{pack}}$, etkinleştirilmişse hücre iç direnci bileşenini ve kablo direncini birlikte içerir:

$$
R_{\mathrm{pack}} = R_{\mathrm{int}} + R_{\mathrm{wire}}
$$

$$
V_{\mathrm{pack,new}} =
\max\!\left(
V_{\mathrm{nom}} - I_{\mathrm{pack}}R_{\mathrm{pack}},
0.5\,V_{\mathrm{nom}}
\right)
$$

Bu geri besleme, akım yükseldikçe voltajın düşmesini modele dahil eder; voltaj düşünce aynı itki için daha zor bir motor çalışma noktası oluşur ve çözüm bu etkiyi otomatik yakalar.

### Geçerlilik Sınırları

Bu adımda işletim noktasının kabul edilip edilmeyeceği termal ve akım limitleriyle denetlenir. Burada $T_{\mathrm{amb}}$ ortam sıcaklığı, $R_{\mathrm{th}}$ termal direnç, $k_{\mathrm{cool}}$ soğutma seviyesi katsayısı ve $T_{\max}$ limit sıcaklıktır:

$$
T_{\mathrm{motor}} =
T_{\mathrm{amb}} +
\left(P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}\right)
R_{\mathrm{th}}\,k_{\mathrm{cool}}
$$

$$
I_{\mathrm{motor}} > I_{\max}
\quad \text{veya} \quad
T_{\mathrm{motor}} > T_{\max}
$$

Sıcaklık denklemi, şafta aktarılamayan elektrik gücünü ısıl kayıp olarak kabul eder. Bu yüzden $P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}$ farkı büyüdükçe motor sıcaklığı yükselir; $k_{\mathrm{cool}}$ küçüldükçe aynı kayıp gücü için sıcaklık artışı azalır. Eşiklerden herhangi biri aşılırsa işletim noktası geçersiz kabul edilir.

### Raporlanan Verim

Yalnızca geçerli işletim noktaları için sistem verimi gram/Watt biçiminde raporlanır. $T_g$ gram cinsinden itki olmak üzere metrik aşağıdaki gibi hesaplanır:

$$
\eta_{g/W} = \frac{T_g}{P_{\mathrm{batt}}}
$$

Bu metrik, bataryadan çekilen her Watt başına üretilen itkiyi doğrudan verdiği için farklı hız noktaları arasında kıyaslamayı netleştirir. Aynı akış tüm hız noktalarına uygulanarak dinamik eğrileri üretir; statik sonuçlar ise bu çözüm setinin sıfıra yakın hız kesitinden alınır.
