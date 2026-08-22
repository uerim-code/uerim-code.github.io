# ADR-0004 — Tekrarlayan ders serisi birinci sınıf domain kavramıdır

**Durum:** Kabul edildi · **Bağlayıcı:** Ajan bu kararı değiştiremez.

## Karar

Ders serisi ayrı bir tablo olarak modellenir; `lessons` satırları seriden **materialize edilir**
(sanal/hesaplanmış değil, gerçek satır). Her ders satırı `series_id` (nullable) taşır.

```
lesson_series
  id, tenant_id, business_unit_id, branch_id, service_id,
  student_id | group_id, instructor_id, room_id,
  rrule (RFC 5545 subset: FREQ=WEEKLY;BYDAY=..;INTERVAL=..),
  starts_on, ends_on | occurrence_count,
  timezone, status, created_by

series_exceptions
  id, series_id, occurrence_date,
  type: SKIPPED | MOVED | DETACHED,
  moved_lesson_id
```

Düzenleme kapsamı her zaman açıkça belirtilir:

```ts
type SeriesEditScope = 'THIS_ONLY' | 'THIS_AND_FOLLOWING' | 'ALL'
```

## Gerekçe

Müzik ve sanat okullarında ders "her Çarşamba 17:00, 12 hafta" olarak satılır ve planlanır.
Tek tek ders modeli ile bu iş yapılamaz. v1.0'da bu kavram hiç yoktu; Gün 4'te fark edilseydi
şema geriye dönük değişecekti.

Materialize etmemek (uçuşta hesaplamak) çekici görünür ama exclusion constraint'i imkânsız kılar —
var olmayan satır çakışma üretemez. ADR-0001'in geçerli kalması için satırların gerçekten var olması şart.

## Sonuçlar

- Seri oluşturma **tek transaction**: tüm occurrence'lar için scheduling engine çalışır.
  Bir tanesi bile çakışırsa iki seçenek sunulur: çakışan tarihleri atla, veya tüm seriyi iptal et.
  Kısmi ve sessiz oluşturma yoktur.
- `THIS_ONLY` düzenlemesi ilgili occurrence'ı seriden koparır (`DETACHED`), `series_id` korunur ama
  seri güncellemeleri artık o dersi etkilemez.
- `THIS_AND_FOLLOWING` mevcut seriyi `ends_on` ile kapatır ve yeni seri açar. Geçmiş dersler değişmez.
- Tatil/kapalı gün çakışması `series_exceptions` içinde `SKIPPED` olarak durur; ders üretilmez,
  paketten düşüm olmaz.
- Paket düşümü **ders bazındadır, seri bazında değildir**. Seri oluşturulurken `LESSON_RESERVED`
  event'i her occurrence için ayrı yazılır; entitlement yetmiyorsa seri kısaltılır ve kullanıcıya bildirilir.
- Seri iptalinde yalnızca **gelecekteki** dersler iptal edilir ve kredileri iade edilir;
  tamamlanmış dersler dokunulmaz.
