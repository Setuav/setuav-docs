# Setuav Analiz Modülü

Setuav analiz modülü, tasarım tanımını mühendislik çıktısına dönüştüren hesaplama katmanıdır. Belirli analiz koşulları altında (kütle, CG, irtifa, sıcaklık, hava yoğunluğu) aerodinamik, itki ve uçuş performansı sonuçları üretir. Pratikte bu modül, [Setuav Specification](https://setuav.github.io/specification/) içindeki tasarım ve koşul girdilerini alır, aynı spesifikasyonda tanımlanan performans çıktısına dönüştürür.

Analiz akışı tek bir sırayla yürür: girdi hazırlığı, aerodinamik hesap, itki hesap, uçuş performansı hesabı ve sonuçların birleştirilmesi. Burada uçuş performansı aerodinamik sonuçlara doğrudan bağlıdır; itki verisi gereken senaryolarda itki çözümü başarısızsa itkiye bağlı performans çıktıları sınırlı kalır. Birleştirme adımı ise yalnızca mevcut alan çıktıları toplandıktan sonra çalışır.

| Aşama | Ana girdi | Elde edilen çıktı |
| --- | --- | --- |
| Girdi hazırlığı | Tasarım + koşullar | Analize hazır girdiler |
| Aerodinamik | Geometri + atmosfer durumu | Aerodinamik davranış ve verimlilik göstergeleri |
| İtki | İtki kurulumu + atmosfer durumu | İtki, güç, akım ve verimlilik çıktıları |
| Uçuş performansı | Aerodinamik (+ varsa itki) | Stall, cruise, tırmanma, menzil, dayanım metrikleri |
| Birleştirme | Alan çıktıları | Rapor ve arayüz için birleşik sonuç seti |

Fizik çözüm katmanında AeroSandbox kullanılır; Setuav tarafında ise tasarım/spesifikasyon eşlemesi, limit kontrolleri, komponent entegrasyonu ve sonuç paketleme yürütülür. Bu nedenle modelleme ve raporlama ayrışır: fizik çözüm tutarlılığı korunurken ürün seviyesinde okunabilir ve karşılaştırılabilir sonuç seti elde edilir.

Analiz girdileri üç ana grupta toplanır: tasarım tanımı, analiz koşulları ve opsiyonel analiz ayarları. Çıktılar da üç ana grupta sunulur: aerodinamik sonuçlar, itki sonuçları ve uçuş performansı sonuçları. Bu yapı, aynı koşullar altında farklı tasarımların adil biçimde karşılaştırılmasını sağlar.

Sonuçları değerlendirirken önce fizibiliteye bakmak, ardından alternatif tasarımları aynı koşullarda kıyaslamak ve tekil sayıların yanında eğri davranışlarını birlikte okumak daha doğru karar verir. Özellikle stall, itki marjı, tırmanma, menzil ve dayanım değerleri tek başına değil, birbirleriyle tutarlılık içinde yorumlanmalıdır.
