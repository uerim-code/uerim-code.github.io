# ADR-0001 — Çift rezervasyon garantisi veritabanı seviyesindedir

**Durum:** Kabul edildi · **Bağlayıcı:** Ajan bu kararı değiştiremez.

## Karar

Aynı öğretmen, aynı oda veya aynı öğrenci için zaman çakışması **PostgreSQL exclusion constraint**
ile engellenir. Uygulama katmanındaki kontrol yalnızca kullanıcıya erken ve anlamlı geri bildirim içindir;
**doğruluk garantisi veritabanındadır**.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE lessons ADD COLUMN period tstzrange
  GENERATED ALWAYS AS (tstzrange(start_at, end_at, '[)')) STORED;

ALTER TABLE lessons ADD CONSTRAINT lessons_no_instructor_overlap
  EXCLUDE USING gist (instructor_id WITH =, period WITH &&)
  WHERE (status NOT IN ('CANCELLED','STUDENT_CANCELLED','INSTRUCTOR_CANCELLED','POSTPONED'));

ALTER TABLE lessons ADD CONSTRAINT lessons_no_room_overlap
  EXCLUDE USING gist (room_id WITH =, period WITH &&)
  WHERE (room_id IS NOT NULL
         AND status NOT IN ('CANCELLED','STUDENT_CANCELLED','INSTRUCTOR_CANCELLED','POSTPONED'));
```

Öğrenci çakışması grup dersleri nedeniyle `lessons` üzerinde değil `lesson_participants`
üzerinde kurulur; aynı desen, denormalize edilmiş `period` kolonu ile.

## Gerekçe

- Advisory lock, `SELECT ... FOR UPDATE` ve `SERIALIZABLE` yaklaşımlarının hepsi doğru kullanılırsa çalışır,
  ama üçü de **kod disiplinine** bağlıdır. Yeni bir insert yolu (worker, import, admin script, AI tool)
  kontrolü atlarsa garanti sessizce kaybolur.
- Exclusion constraint hangi yoldan gelirse gelsin uygulanır; migration'a bakan herkes kuralı görür.
- Aynı slota 100 paralel istekte tam olarak 1'i başarılı olur — test edilebilir, ispatlanabilir davranış.

## Sonuçlar

- Buffer süresi (`service.buffer_minutes`) `period` hesabına dahil edilir; buffer da çakışma sayılır.
- Constraint ihlali `23P01` SQLSTATE döner. Application katmanı bunu `INSTRUCTOR_CONFLICT` /
  `ROOM_CONFLICT` / `STUDENT_CONFLICT` reason code'una **çevirmek zorundadır**; ham hata UI'a çıkamaz.
- Reschedule tek transaction: eski dersin `status` güncellemesi + yeni satır + ledger event + status history.
  Constraint patlarsa tüm transaction geri alınır, eski rezervasyon bozulmaz.
- Grup dersinde kapasite kontrolü constraint ile değil, `lesson_participants` üzerinde
  `SELECT ... FOR UPDATE` + sayım ile yapılır (aynı transaction içinde).

## Kabul testi

Aynı öğretmen ve aynı slot için **bağımsız connection'lardan** 100 paralel istek:
1 SUCCESS, 99 CONFLICT, veritabanında tam 1 ders. Tek connection pool üzerinden yapılan test **kanıt sayılmaz**.
