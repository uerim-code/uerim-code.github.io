# GÜN 2 — SCHEDULING ENGINE + SERİ DERS + TAKVİM + PAKET KURALLARI

**Hedef:** Sistemin kritik deterministic planlama motorunu, tekrarlayan ders serilerini,
takvim UI'ını ve concurrency güvenliğini tamamlamak.

**Ön koşul:** Gün 1'in 1.2, 1.3, 1.6, 1.7, 1.8 maddeleri DONE.
**Bağlayıcı:** ADR-0001 (exclusion constraint), ADR-0004 (seri), ADR-0005 (zaman).

---

## 2.1 — Lesson modeli  *(2A)*

`lessons` · `lesson_participants` · `lesson_status_history` · `attendance` · `lesson_series` · `series_exceptions`

`lessons` alanları: `tenant_id, business_unit_id, branch_id, service_id, instructor_id, room_id,
start_at, end_at, period (generated tstzrange), status, source, series_id, created_by`.

`source`: `ADMIN | COORDINATOR | STUDENT_PORTAL | WHATSAPP_AI | API | SERIES`

Bireysel derste de `lesson_participants` kullanılır (tek satır) — grup dersiyle aynı kod yolu çalışsın.

**Exclusion constraint'ler ADR-0001'e göre kurulur.** Bu migration'ı yazmadan 2.2'ye geçme.

## 2.2 — Scheduling Engine  *(2A)*

`packages/scheduling` — framework bağımsız, React'sız test edilebilir.

```ts
checkAvailability(input: {
  tenantId, studentIds: string[], serviceId, requestedStart, durationMinutes,
  requestedInstructorId?, requestedRoomId?, branchId?, excludeLessonId?
}): AvailabilityResult
```

**Kontroller (sırayla, ilk başarısızlıkta reason code ile döner):**
1. Öğrenci ve business unit aktifliği
2. Geçerli öğrenci paketi ve kalan entitlement
3. Hizmetin pakete dahil olması
4. Öğretmen yetkinliği (`instructor_services`), çalışma saati, izin, çakışma
5. Oda capability, availability, çakışma
6. Öğrenci çakışması
7. Şube çalışma saati ve tatil
8. Ders süresi ve gerekli buffer
9. Grup kapasitesi

**Sonuç:** `AVAILABLE | UNAVAILABLE | REQUIRES_APPROVAL`

**Reason code'lar:** `PACKAGE_NOT_FOUND · PACKAGE_EXPIRED · PACKAGE_EXHAUSTED · PACKAGE_FROZEN ·
INSTRUCTOR_CONFLICT · INSTRUCTOR_UNAVAILABLE · INSTRUCTOR_ON_LEAVE · INSTRUCTOR_NOT_QUALIFIED ·
ROOM_CONFLICT · ROOM_UNAVAILABLE · ROOM_CAPABILITY_MISMATCH · STUDENT_CONFLICT ·
OUTSIDE_OPENING_HOURS · HOLIDAY · SERVICE_NOT_ALLOWED · GROUP_CAPACITY_FULL`

**`REQUIRES_APPROVAL` somutlaştırması** (v1.0'da tanımsızdı): politika istisnası gerektiren istekler
(cutoff sonrası iptal, dolu paket üstüne ders, kapanış saati dışı) reddedilmez —
koordinatör onay kuyruğuna düşer. `approval_requests` tablosu: talep, gerekçe, talep eden, karar, karar veren.

## 2.3 — Alternatif slotlar  *(2A)*

```ts
findAlternativeSlots(input, limit = 5): Slot[]
```
Öncelik: aynı öğretmen → aynı gün → istenen saate yakınlık → aynı şube → uygun oda.

**Kritik:** Alternatif seçildiğinde slot **yeniden doğrulanır**. Öneri ile seçim arasında geçen sürede
slot dolmuş olabilir. Revalidation yoksa bu P0'dır.

## 2.4 — Tekrarlayan seri  *(2A)* — ADR-0004

```ts
createSeries(input): { series, createdLessons, skippedDates, reason }
rescheduleLesson(lessonId, newSlot, scope: SeriesEditScope)
cancelSeries(seriesId, scope, refundPolicy)
```

- Seri oluşturma **tek transaction**. Occurrence çakışırsa iki seçenek: çakışanları atla, veya tümünü iptal et.
  Kısmi ve sessiz oluşturma yoktur.
- Tatil/kapalı gün → `series_exceptions.SKIPPED`, ders üretilmez, paketten düşüm olmaz.
- Entitlement yetmezse seri kısaltılır ve kullanıcıya kaç ders üretildiği açıkça bildirilir.
- `THIS_ONLY` düzenlemesi occurrence'ı seriden koparır (`DETACHED`).
- `THIS_AND_FOLLOWING` mevcut seriyi kapatır, yeni seri açar; geçmiş dersler değişmez.

## 2.5 — Concurrency, reschedule ve iptal  *(2A)*

- Çift rezervasyon garantisi ADR-0001'dedir. Constraint ihlali (`23P01`) ilgili reason code'a çevrilir;
  ham SQL hatası UI'a çıkamaz.
- `rescheduleLesson()` atomiktir: eski ders status'ü + yeni ders + ledger event + status history
  aynı transaction'da. Başarısızlıkta mevcut rezervasyon **bozulmaz**.
- **Tenant-configurable iptal politikası**: `cancellation_cutoff_hours` öncesinde `CREDIT_RETURNED`,
  sonrasında kredi tüketilmiş sayılır. Hard-code etme. Cutoff hesabı tenant timezone'ında (ADR-0005).
- **Telafi (makeup)**: politika izin veriyorsa `MAKEUP_GRANTED` event'i yazılır; makeup hakkının
  geçerlilik süresi ve adedi `tenant_settings.makeup_policy`'den okunur.
- **Paket dondurma**: `PACKAGE_FROZEN` / `PACKAGE_UNFROZEN`. Dondurulmuş paketle rezervasyon yapılamaz;
  dondurma süresi paket son kullanma tarihini uzatır.

## 2.6 — Bekleme listesi ve deneme dersi  *(2A)*

- `waitlist_entries`: dolu slot/grup için sıra. Yer açıldığında ilk sıradakine bildirim gider,
  belirli süre (tenant ayarı) içinde yanıt gelmezse sıradakine geçer.
- Deneme dersi: `TRIAL_LESSON` ledger event'i, paketi olmayan öğrenci için tek seferlik izin.
  Ücretli/ücretsiz olması tenant ayarı.

## 2.7 — Takvim UI  *(2B)*

- Görünümler: Gün · Hafta · Ay · Öğretmen · Oda
- Filtreler: business unit, şube, öğretmen, oda, hizmet, durum
- Kart: öğrenci, hizmet, öğretmen, oda, saat, durum, seri rozeti
- **Drag-drop:** bırakıldığında önce `checkAvailability` → başarılıysa kaydet.
  Çakışmada kart **eski konumuna döner** ve reason code kullanıcı diline çevrilmiş biçimde gösterilir.
- Seri derse drag-drop yapıldığında kapsam sorulur (`THIS_ONLY | THIS_AND_FOLLOWING | ALL`).
- Koordinatör onay kuyruğu ekranı (2.2'deki `approval_requests`).

---

## Gün 2 testleri

**Unit (scheduling paketi, DB'siz):** her reason code için en az bir vaka; buffer sınırı;
grup kapasitesi; alternatif slot sıralaması; rrule occurrence üretimi.

**Integration:**
- öğretmen/oda/öğrenci çakışması; izin günü; çalışma saati dışı; tatil
- paket tükenmiş, süresi dolmuş, dondurulmuş, hizmet kapsam dışı
- geçerli rezervasyon; reschedule başarı ve **rollback**
- iptal: cutoff öncesi kredi iadesi, sonrası kredi tüketimi, makeup verilmesi
- seri oluşturma, `THIS_ONLY`, `THIS_AND_FOLLOWING`, `ALL`, tatil atlama, entitlement yetmemesi
- alternatif slot seçimi öncesi slotun dolması → revalidation reddeder

**Zaman testleri (ADR-0005):** DST ileri/geri, gece yarısını geçen ders, ay sonu, yıl sonu, artık gün.

**Concurrency (CI'da zorunlu, bağımsız connection'lar):**
- Aynı öğretmen + aynı slot: 20, 50 ve 100 paralel istek → **1 SUCCESS, geri kalanı CONFLICT**,
  veritabanında tam 1 ders.
- Grup dersinde son 2 koltuk için 2 paralel istek → kapasite aşılmaz.
- İki eş zamanlı reschedule aynı dersi farklı slotlara taşımaya çalışır → biri başarılı, biri reddedilir,
  ders tek yerde kalır.
- Availability kontrolü ile save arasına yapay gecikme enjekte edilerek TOCTOU zorlanır.

**Kapı:** lint · typecheck · unit · integration · concurrency (CI) · RLS (CI) ·
rapor: `docs/test-reports/gun-2.md`

## Blok bitmezse

2.1, 2.2, 2.5 Gün 3'ün ön koşuludur — AI bunları çağıracak. 2.4 (seri) bitmezse Gün 3'te
"seriyi taşı" senaryosu kapsam dışına alınır ve PROGRESS'e yazılır. 2.6 ve 2.7 Gün 4'e taşınabilir.
