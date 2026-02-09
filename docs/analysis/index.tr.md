# Setuav Analiz Modülü

## Amaç

Setuav analiz modülü, tasarım tanımını mühendislik çıktısına dönüştüren
hesaplama katmanıdır. Belirli analiz koşulları altında (kütle, CG, irtifa,
sıcaklık, hava yoğunluğu) aerodinamik, itki ve uçuş performansı sonuçlarını üretir.

Pratikte şu iki katman arasında çalışır:

- **Girdi**: [Setuav Specification](https://setuav.github.io/specification/) içinde tanımlanan tasarım girdileri ve analiz koşulları
- **Çıktı**: [Setuav Specification](https://setuav.github.io/specification/) içinde tanımlanan performans verileri

## Nasıl Çalışır

1. **Girdiler hazırlanır**  
   Modül, uçak tanımını ve analiz koşullarını alır.

2. **Hesaplamalar yürütülür**  
   Aerodinamik, itki ve uçuş performansı hesapları sıralı şekilde çalıştırılır.

3. **Sonuçlar birleştirilir**  
   Çıktılar, raporlama ve arayüz gösterimi için tek bir analiz sonuç setinde toplanır.

## Çözücü Temeli

Setuav analiz modülü, fizik çözüm katmanı olarak AeroSandbox kullanır; bunun
üzerine Setuav'a özgü veri yönetimi ve raporlama adımlarını ekler.

Pratikte:

1. **AeroSandbox temel fizik çözümünü yürütür**  
   Atmosfer durumunun hesaplanması, aerodinamik katsayıların üretilmesi ve seçili elektrikli itki model yardımcılarının kullanılması.

2. **Setuav ürün seviyesindeki orkestrasyonu yürütür**  
   Tasarım/spesifikasyon eşlemesi, komponent kütüphanesi entegrasyonu, limit/koşul kontrolleri ve sonuçların arayüz/rapor formatına dönüştürülmesi.

## Girdiler

Analiz modülü üç girdi grubunu kullanır:

1. **Tasarım tanımı**  
   Gövde/kanat geometrisi, itki kurulumu ve komponent seçimleri.

2. **Analiz koşulları**  
   Uçuş kütlesi, ağırlık merkezi konumu, irtifa, sıcaklık ve hava yoğunluğu.

3. **Analiz ayarları (opsiyonel)**  
   Hesaplamanın nasıl yürütüleceğini belirleyen ayarlar; aerodinamik tarama aralığı ve çözünürlük, itki tarafındaki elektriksel/termal varsayımlar ve limitler ile uçuş performansında hız aralığı, adım sayısı ve güvenlik marjlarını kapsar.

## Çıktılar

Analiz modülü üç çıktı grubunu üretir:

1. **Aerodinamik sonuçlar**  
   Kaldırma, sürükleme ve verimlilik davranışını anlamak için polar veriler ve temel aerodinamik metrikler.

2. **İtki sonuçları**  
   İtki, güç, akım ve verimlilik eğilimlerini içeren statik ve eğri tabanlı itki verileri.

3. **Uçuş performansı sonuçları**  
   Stall hızı, cruise performansı, tırmanma kabiliyeti, menzil ve dayanım gibi operasyonel metrikler.

## Sonuçlar Nasıl Kullanılır

Analiz çıktıları karar desteği için kullanılmalıdır:

1. **Önce fizibiliteyi kontrol edin**  
   İtki, stall hızı ve temel performans hedeflerinin kabul edilebilir sınırlar içinde olduğundan emin olun.

2. **Tasarım alternatiflerini karşılaştırın**  
   Adil kıyas için farklı tasarımları aynı analiz koşulları altında değerlendirin.

3. **Sadece tek değere değil trendlere bakın**  
   Çalışma marjlarını ve ödünleşimleri anlamak için eğri davranışını birlikte okuyun.
