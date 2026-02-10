# İtki

## Yöntem

Setuav itki analizi hız-bağımlı tek bir çözüm akışı kullanır. Her işletim noktası uçuş hızı $V$ ve throttle $u$ ile tanımlanır. Pervane katsayıları veritabanından $\mathrm{RPM}$ ve ilerleme oranı $J$ üzerinden alınır; motor tarafında $K_v$, $R$, $I_0$ ile elektrik-mekanik denge kurulur; batarya tarafında gerilim düşümü ve verim kayıpları aynı döngü içinde hesaba katılır. Böylece statik ve dinamik sonuçlar iki ayrı modelden değil aynı fizik zincirinden üretilir.

## Hazırlık

### Bileşen Hazırlığı

Çözümden önce itki sistemi bileşenleri hem tamlık hem de fiziksel geçerlilik açısından doğrulanır. Bu adımda motor, batarya ve pervane verileri tek çözücüye girecek ortak bir sisteme dönüştürülür.

| Tasarım alanı | Çözümde karşılığı | Kontrol / Not |
| --- | --- | --- |
| `propulsion.motors[].kv` | $K_v$ | Zorunlu, $> 0$ |
| `propulsion.motors[].resistance` | $R$ (sıcaklık ölçekli) | Zorunlu, $> 0$, `motor_resistance_temp_factor` ile ölçeklenir |
| `propulsion.motors[].no_load_current` | $I_0$ | Zorunlu, $\ge 0$ |
| `propulsion.motors[].current_max` | $I_{\max}$ | Eksikse yüksek bir sınırla ikame edilir |
| `propulsion.batteries[].voltage_nominal` | $V_{\mathrm{nom}}$ | Zorunlu, $> 0$ |
| `propulsion.batteries[].cells_series` | $n_{\mathrm{cells}}$ | Sag hesabında kullanılır; yoksa voltajdan türetilir |
| `propulsion.batteries[].cell_resistance` | $R_{\mathrm{pack}}$ bileşeni | Yoksa config fallback kullanılır |
| `propulsion.propellers[].(diameter,pitch,blade_count)` | DB lookup girdisi | Veritabanı eşleşmesi zorunlu |

Tork sabiti konfigürasyonda override edilmemişse doğrudan hız sabitinden türetilir:

$$
K_t = \frac{60}{2\pi K_v}
$$

Bu adım sonunda çözücü; motor, batarya, pervane ve motor sayısını tek bir sistem spesifikasyonu olarak kullanır.

### Koşul Hazırlığı

İtki denklemlerinde kullanılacak atmosfer durumu analiz koşullarından üretilir.

| Analiz koşulu alanı | Çözümde karşılığı | Not |
| --- | --- | --- |
| `conditions.altitude_msl` | Atmosfer modeli | Yoğunluk durumuna girer |
| `conditions.temperature` | Atmosfer modeli | Yoğunluk ve ses hızı durumunu etkiler |
| `conditions.air_density` | $\rho$ | Varsa doğrudan override etkisi uygulanır |
| `conditions.total_mass` | $\mathrm{thrust\_to\_weight}$ raporu | Çekirdek prop çözücüden çok rapor metriğinde kullanılır |

Çözüm döngüsünde pervane tarafının ana girdisi $V$, elektrik tarafının ana girdisi $V_{\mathrm{pack}}$ olur ve $V_{\mathrm{pack}}$ her iterasyonda sag geri beslemesi ile güncellenir.

## Çözüm

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

Katsayılar okunduktan sonra aynı işletim noktasında tork, itki ve şaft gücü türetilir. Burada $\rho$ hava yoğunluğudur:

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

Bu adımın fiziksel anlamı şudur: pervane torku arttıkça $I_{\mathrm{motor}}$ artar, akım arttıkça hem $I^2R$ kaybı hem de gerekli terminal gerilim artar. Çözüm, bu dengeyi aynı anda sağlayan $\mathrm{RPM}$ ve akım kombinasyonunu iteratif olarak arar.

Motor tarafındaki elektrik gücü batarya tarafına, ESC ve batarya verimleri üzerinden taşınır. Burada $\eta_{\mathrm{esc}}$ ESC verimi, $\eta_{\mathrm{batt}}$ batarya deşarj verimidir:

$$
P_{\mathrm{motor,elec}} \approx V_{\mathrm{motor}} I_{\mathrm{motor}}
$$

$$
P_{\mathrm{batt}} = \frac{P_{\mathrm{motor,elec}}}{\eta_{\mathrm{esc}} \eta_{\mathrm{batt}}}
$$

$$
I_{\mathrm{pack}} = \frac{P_{\mathrm{batt}}}{V_{\mathrm{pack}}}
$$

Bu dönüşümde kritik nokta, kayıpların ardışık olarak birikmesidir: aynı şaft gücü ihtiyacı için $P_{\mathrm{batt}}$, her verim katmanında büyüyerek daha yüksek batarya akımı gerektirir.

Yük akımı belirlendikten sonra paket iç direnci $R_{\mathrm{pack}}$ ile gerilim düşümü güncellenir ve çözüme geri beslenir:

$$
V_{\mathrm{pack,new}} = V_{\mathrm{nom}} - I_{\mathrm{pack}}R_{\mathrm{pack}}
$$

Bu geri besleme, akım yükseldikçe voltajın düşmesini modele dahil eder; voltaj düşünce aynı itki için daha zor bir motor çalışma noktası oluşur ve çözüm bu etkiyi otomatik yakalar.

Aynı iterasyon içinde termal ve akım güvenliği denetlenir. Burada $T_{\mathrm{amb}}$ ortam sıcaklığı, $R_{\mathrm{th}}$ termal direnç, $T_{\max}$ limit sıcaklıktır:

$$
T_{\mathrm{motor}} = T_{\mathrm{amb}} + \left(P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}\right)R_{\mathrm{th}}
$$

$$
I_{\mathrm{motor}} > I_{\max}
\quad \text{veya} \quad
T_{\mathrm{motor}} > T_{\max}
$$

Sıcaklık denklemi, şafta aktarılamayan elektrik gücünü ısıl kayıp olarak kabul eder. Bu yüzden $P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}$ farkı büyüdükçe motor sıcaklığı hızla yükselir.

Bu eşiklerden biri aşılırsa nokta geçersiz kabul edilir. Geçerli noktalarda raporlanan sistem verimi gram/Watt biçimindedir ve $T_g$ gram cinsinden itki olmak üzere şöyle hesaplanır:

$$
\eta_{g/W} = \frac{T_g}{P_{\mathrm{batt}}}
$$

Bu metrik, bataryadan çekilen her Watt başına üretilen itkiyi doğrudan verdiği için farklı hız noktaları arasında kıyaslamayı netleştirir. Aynı akış tüm hız noktalarına uygulanarak dinamik eğrileri üretir; statik sonuçlar ise bu çözüm setinin sıfıra yakın hız kesitinden alınır.
