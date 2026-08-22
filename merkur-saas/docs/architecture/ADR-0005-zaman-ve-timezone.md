# ADR-0005 — Zaman UTC saklanır, tenant timezone tek doğruluk kaynağıdır

**Durum:** Kabul edildi · **Bağlayıcı:** Ajan bu kararı değiştiremez.

## Karar

- Tüm zaman kolonları `timestamptz`. `timestamp` (without time zone) kullanımı yasaktır.
- Saklama daima UTC. Sunum ve **iş kuralı yorumu** daima `tenant_settings.timezone`
  (IANA identifier, örn. `Europe/Istanbul`).
- Çalışma saatleri, tatiller, iptal cutoff süresi, reminder zamanlaması ve raporların gün sınırları
  tenant timezone'ında hesaplanır — sunucunun veya kullanıcı tarayıcısının timezone'ında değil.
- Ham `Date` aritmetiği (`new Date(t + 86400000)` gibi) yasaktır. Tarih hesapları TZ-aware
  kütüphane ile yapılır (Temporal API veya `date-fns-tz`).
- Çalışma saatleri **yerel duvar saati** olarak saklanır (`weekday, start_time, end_time`),
  UTC offset olarak değil. DST değişiminde "her Salı 17:00" kayar değil, sabit kalır.

## Gerekçe

DST geçişleri, ay/yıl sınırları ve artık gün v1.0'da test edilmesi isteniyordu ama saklama kuralı
yazılmamıştı. Kural yazılmazsa test yazılamaz: `timestamp` seçen bir implementasyonda DST testi
kökten anlamsızdır.

## Sonuçlar

- Türkiye şu an DST uygulamıyor (kalıcı UTC+3), ancak sistem çok tenant'lı ve başka ülkelerde
  kullanılabilir olduğu için DST doğruluğu kapsam içidir. "Türkiye'de DST yok" gerekçesiyle
  bu madde atlanamaz.
- Reminder `scheduled_for` değeri UTC olarak saklanır ama tenant TZ'de hesaplanır.
- Test kapsamı: DST ileri/geri geçiş günü, gece yarısını geçen ders, ay sonu, yıl sonu, artık gün.
