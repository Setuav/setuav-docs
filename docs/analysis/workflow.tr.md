# Analiz İş Akışı

Bu sayfa, bir analiz çalışmasının girdiden nihai sonuca kadar nasıl ilerlediğini açıklar.
Amaç, kullanıcıya her aşamada ne olduğunu ve sonuçları değerlendirirken nereye
odaklanması gerektiğini net göstermektir.

## Akış Özeti

Süreç sıralı şekilde çalışır:

1. **Girdi hazırlığı**
2. **Aerodinamik hesaplama**
3. **İtki hesaplama**
4. **Uçuş performansı hesaplama**
5. **Sonuçların birleştirilmesi**

## Bağımlılık Mantığı

- Uçuş performansı, aerodinamik çıktılara bağlıdır.
- İtki verisi gerektiğinde itki analizi başarısız olursa, itkiye bağlı performans çıktıları sınırlı kalır.
- Birleştirme adımı, mevcut alan çıktıları toplandıktan sonra çalışır.

## Ön Koşullar

Analiz çalıştırmadan önce şunların hazır olması gerekir:

- Tasarım dosyasında tam bir uçak tanımı
- İtki/performance hesapları için geçerli bir itki kurulumu
- Analiz koşulları (kütle, ağırlık merkezi konumu, irtifa, sıcaklık, hava yoğunluğu)

## Aşama Haritası

| Aşama | Ana girdi | Elde edilen çıktı |
| --- | --- | --- |
| Girdi hazırlığı | Tasarım + koşullar | Analize hazır girdiler |
| Aerodinamik | Geometri + atmosfer durumu | Aerodinamik davranış ve verimlilik göstergeleri |
| İtki | İtki kurulumu + atmosfer durumu | İtki, güç, akım ve verimlilik çıktıları |
| Uçuş performansı | Aerodinamik (+ varsa itki) | Stall, cruise, tırmanma, menzil, dayanım metrikleri |
| Birleştirme | Alan çıktıları | Rapor ve arayüz için birleşik sonuç seti |

## Aşamaya Göre Kontrol Noktaları

| Aşama | Önerilen kullanıcı kontrolü |
| --- | --- |
| Çalıştırma öncesi | Girdilerin tamlığı ve koşulların gerçekçiliği |
| Aerodinamik sonrası | Kaldırma/sürükleme davranışı ve hedef hız rejimiyle uyum |
| İtki sonrası | Görev ihtiyacına göre itki ve güç marjı |
| Performans sonrası | Stall, cruise, tırmanma, menzil, dayanım gereksinim uyumu |
| Nihai gözden geçirme | Tüm sonuç grupları arasında genel tutarlılık |
