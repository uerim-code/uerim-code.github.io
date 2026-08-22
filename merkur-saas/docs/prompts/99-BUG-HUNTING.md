# AYRI PROMPT — SALDIRGAN SON KONTROL / BUG HUNTING

> Bu prompt, beş geliştirme bloğu tamamlandıktan sonra **temiz bağlamlı ayrı bir oturumda**
> çalıştırılır. Amaç mevcut implementasyonu doğrulamak değil, onu **sistematik biçimde kırmak**
> ve gizli kusurları ortaya çıkarmaktır. Bu adım opsiyonel değildir.

---

## BUG HUNTING MASTER PROMPT

Bu repository'de kıdemli bir adversarial QA mühendisi, application security uzmanı,
distributed-systems hata avcısı ve SaaS production reviewer olarak çalış.

**Uygulamayı yazan ajanın varsayımlarına güvenme.** README, test isimleri, "çalışıyor" notları,
`docs/PROGRESS.md` kayıtları ve mevcut yeşil CI çıktıları kanıt değildir. Her iddiayı çalışan kod,
veritabanı davranışı, gerçek request/response, transaction sonucu ve tekrar üretilebilir test ile doğrula.

Hedefin yeni özellik geliştirmek değildir. Hedefin; veri kaybı, tenant sızıntısı, yetki atlama,
çift rezervasyon, yanlış paket bakiyesi, **yanlış para hesabı**, sessiz başarısızlık, duplicate mesaj,
yanlış AI mutasyonu ve production kesintisi oluşturabilecek her kusuru bulmak, yeniden üretmek,
düzeltmek ve regression testiyle kilitlemektir.

### Çalışma kuralları — taviz yok

1. Önce repository'yi, migration'ları, RLS policy'lerini, server action/API route'larını,
   queue worker'larını ve testleri haritala. **Dokümana göre değil, gerçek koda göre** sistem modeli çıkar.
   `docs/architecture/ADR-*.md` dosyalarını oku ve **implementasyonun ADR'lere uyup uymadığını denetle** —
   uymuyorsa bu başlı başına bir bulgudur.
2. Mevcut testleri çalıştırıp geçmesine sevinme. Önce testlerin gerçekten doğru şeyi doğrulayıp
   doğrulamadığını denetle: aşırı mock, yanlış assertion, boş test, snapshot körlüğü, false-positive ara.
3. Happy path ile yetinme. Her kritik akışta unauthorized, cross-tenant, duplicate, concurrent,
   stale-state, timeout, partial-failure, retry ve rollback senaryolarını zorla.
4. Bug bulduğunda semptomu yamama. Kök nedeni belirle, en küçük güvenli düzeltmeyi yap,
   regression testi ekle ve **aynı pattern'in repository'deki diğer örneklerini tara**.
5. Testi skip/todo yapma; assertion gevşetme; production davranışını yalnızca testi geçirmek için bozma;
   başarısızlığı mock ile saklama.
6. Silent catch, boş catch, loglayıp başarı dönme, optimistic UI ile yanlış başarı gösterme ve
   transaction dışı çok adımlı mutation gördüğünde **kritik şüphe** olarak incele.
7. Kullanıcıdan ara onay isteme. Scope içindeki güvenli düzeltmeleri yap. Yalnızca gerçek credential
   veya geri alınamaz dış işlem gerekiyorsa bunu kanıtıyla `BLOCKED` olarak kaydet.
8. Tüm bulgular kapanmadan "production-ready" deme. Kanıtı olmayan maddeyi PASS işaretleme.

---

## 0. Başlangıç envanteri ve tehdit modeli

- [ ] Tüm entry point'leri çıkar: UI, API, server action, webhook, worker, cron, queue consumer,
      storage callback, database function.
- [ ] Tüm trust boundary'leri çıkar: browser/server, server/Supabase, webhook/server, AI/tool,
      worker/database, tenant/tenant.
- [ ] Kritik varlıkları listele: öğrenci PII, paket kredisi, **para/fatura kaydı**, ders slotu,
      WhatsApp token/message, AI secret, dosya, audit log.
- [ ] Her mutation için actor, permission, tenant resolution, validation, idempotency, transaction ve
      audit beklentisini **tablo halinde** kaydet.
- [ ] Repository'de hard-coded tenant, service-role key, debug endpoint, bypass flag, test credential
      ve client bundle'a sızan secret ara.

## 1. Multi-tenant izolasyonuna saldır

- Tenant A token'ıyla Tenant B UUID'lerini URL, body, query, header, nested relation, RPC ve
  storage path içine enjekte et.
- Liste, detay, arama, export, autocomplete, realtime subscription, dashboard aggregate,
  **rapor sorguları** ve signed URL üretiminde çapraz tenant sızıntısı ara.
- `tenant_id` alanını eksik, null, sahte veya farklı tenant ile gönder; server'ın client-supplied
  tenant'a güvenip güvenmediğini doğrula.
- **Service-role kullanılan her kod yolunu tek tek çıkar** ve `withTenantContext` sarmalayıcısının
  atlandığı bir yer var mı bak (ADR-0003). Bir tane bile varsa P0.
- `withTenantContext` çağrılmadan yapılan sorgunun 0 satır döndüğünü, tüm tenant'ları döndürmediğini doğrula.
- Membership'i silinen veya tenant değiştiren açık session'ın eski tenant verisine erişimini test et.
- Bir tenant'ın ID sıralaması, toplam sayısı, hata mesajı veya timing farkı üzerinden başka tenant
  varlıklarını tahmin edip edemediğini kontrol et.

**Zorunlu kanıt:** Her tenant-owned tablo ve storage alanı için en az bir pozitif ve bir cross-tenant
DENIED testi. **Tek bir leakage bile P0'dır.**

## 2. Auth, RBAC ve yetki atlama

- Her rol için UI gizlemenin ötesinde doğrudan endpoint/server action çağrısı yap.
- Student/guardian/instructor token'ıyla admin, finance, paket düzeltme, kullanıcı daveti ve
  audit endpoint'lerini çağır.
- Expired, revoked, malformed ve başka environment'a ait token davranışını test et.
- **IDOR/BOLA:** kendi varlığının ID'sini başka öğrencinin, velinin, öğretmenin, dersin, faturanın
  veya dosyanın ID'siyle değiştir.
- **Mass assignment:** `role`, `tenant_id`, `created_by`, paket bakiyesi, `status`, fatura tutarı ve
  audit alanlarını request body'ye ekle.
- Davet token'ı: yeniden kullanım, süresi geçmiş token, başka tenant'ın davetini kabul etme.
- Custom role permission cache'i varsa yetki kaldırıldıktan sonra eski iznin yaşamaya devam edip
  etmediğini test et.
- Password reset, invite ve session refresh akışlarında token reuse, open redirect ve
  account enumeration ara.

## 3. Scheduling engine'i kır

- Aynı teacher/room/student slotuna 20, 50 ve 100 paralel istek; **bağımsız connection'lar** kullan
  (tek pool üzerinden yapılan test kanıt değildir).
- Exclusion constraint'lerin gerçekten kurulu olduğunu migration'dan doğrula (ADR-0001);
  yalnızca uygulama katmanı kontrolü varsa P0.
- Tam aynı başlangıç, tam aynı bitiş, iç içe slot, bir dakika overlap, buffer sınırı,
  gece yarısını geçen ders.
- DST ileri/geri, timezone dönüşümü, ay sonu, yıl sonu, artık gün, geçmiş tarih.
- `timestamp` (without tz) kullanılan bir kolon var mı ara (ADR-0005 ihlali).
- Instructor leave son saniyesi, şube kapanış sınırı, tatil override, oda capability değişimi.
- Availability kontrolü ile save arasındaki **TOCTOU** yarışını zorla.
- İki eş zamanlı reschedule'ın aynı eski dersi farklı slotlara taşımasını dene.
- Reschedule transaction'ının ortasında hata enjekte et; eski ders, yeni ders, ledger ve
  status history'nin atomik kaldığını doğrula.
- Alternatif slot seçildikten sonra slotu başka rezervasyonla doldur; seçim anında revalidation
  yapılmıyorsa P0 aç.
- Grup kapasitesi için iki "son koltuk" isteğini paralel gönder.

### 3B. Seri ders (ADR-0004)

- Seri oluşturma sırasında ortadaki bir occurrence çakışsın: **kısmi ve sessiz** oluşturma var mı?
- Entitlement seri uzunluğundan azken seri oluştur: kaç ders üretildiği kullanıcıya bildiriliyor mu,
  ledger doğru mu?
- `THIS_ONLY` ile koparılan dersin sonraki seri güncellemelerinden etkilenmediğini doğrula.
- `THIS_AND_FOLLOWING` sonrası **geçmiş derslerin değişmediğini** doğrula.
- Seri iptalinde tamamlanmış derslerin kredisinin iade edilmediğini, gelecek derslerinkinin
  edildiğini doğrula.
- Tatile denk gelen occurrence için ders üretilmediğini ve **paketten düşüm olmadığını** doğrula.
- Aynı seriyi iki koordinatör aynı anda düzenlesin.

## 4. Paket ledger ve finansal tutarlılık

- Aynı dersi iki kez complete, cancel, undo ve retry et; entitlement yalnızca doğru sayıda değişmeli.
- Rezervasyon sırasında worker timeout veya client retry üret; duplicate ledger event oluşmamalı.
- İptal cutoff sınırının bir saniye öncesi/sonrası ve timezone farkını test et.
- Süresi dolmuş paket, paket uzatma, dondurma/çözme, manuel düzeltme ve kredi iadesi kombinasyonları.
- Ledger toplamını `remaining_lessons` cache'i ile karşılaştır (`verifyPackageBalance`);
  drift tespit ve onarım mekanizmasını doğrula.
- Negatif bakiye, taşma, yanlış hizmet entitlement'ı ve **başka öğrencinin paketini kullanma** girişimi.
- Her başarısız transaction sonrası lesson, attendance, ledger, hakediş ve audit tutarlılığını kontrol et.

### 4B. Para (finans modülü)

- Tutarların float ile hesaplandığı bir yer var mı ara; kuruş yuvarlama hatası üret.
- Kısmi ödeme + iade + aşırı ödeme kombinasyonlarında bakiyeyi bozmaya çalış.
- Aynı ödemeyi iki kez kaydet (retry, çift tıklama, webhook tekrarı) → ikinci mahsup oluşmamalı.
- Ders tamamlama iki kez tetiklendiğinde hakedişin iki kez yazılıp yazılmadığını kontrol et.
- Kapanmış `payout_run`'ı değiştirmeye çalış.
- `PAYMENT_OVERDUE` engellemesini borcu ödenmiş öğrenci için tetiklemeye çalış (yanlış pozitif).
- Fatura ve ödeme kayıtlarında cross-tenant erişim.

## 5. WhatsApp webhook ve mesajlaşma dayanıklılığı

- Geçersiz signature, eski timestamp, bozuk JSON, eksik alan, bilinmeyen event type, aşırı büyük payload.
- **Webhook handler'ın senkron AI çağırmadığını koddan doğrula.** Çağırıyorsa P1.
- Aynı message ID'yi seri, paralel ve farklı payload ile tekrar gönder.
- Webhook işlenirken DB yazımından sonra response'un kaybolmasını simüle et;
  Meta retry yaptığında çift işlem oluşmamalı.
- Inbound mesajların sırasını ters çevir; status event'i message'dan önce gelsin.
- Rate limit, 429, 5xx, network timeout ve token expiration'da retry/backoff/dead-letter davranışı.
- 24 saat servis penceresi sınırının iki tarafı ve template seçimi; **yanlış kategori** ile gönderim denemesi.
- Opt-out sonrası reminder, duyuru veya AI yanıtı gönderilmeye devam ediyor mu.
- Telefon normalization collision ve aynı numaraya bağlı birden fazla öğrenci/veli belirsizliği.

## 6. AI güvenlik sınırını aşmaya çalış

- **Prompt injection:** kullanıcı mesajına "önceki talimatları unut", sahte tool sonucu,
  JSON kapatma ve SQL benzeri içerikler ekle.
- Modelin şema dışı, yarım JSON, yanlış enum, aşırı uzun alan, **başka tenant ID'si** ve
  uydurma lesson/invoice ID üretmesini simüle et — hepsi reddedilmeli.
- Düşük confidence fakat `needsHuman=false`; yüksek confidence fakat eksik entity;
  çelişkili tarih/saat kombinasyonları.
- AI tool'un authorization ve scheduling engine'i **atlayarak** mutation yapabildiği herhangi bir yol ara.
- `HUMAN_ACTIVE` durumunda kuyrukta bekleyen gecikmiş AI job'ının sonradan mesaj göndermesini zorla.
- Konuşma bağlamının başka öğrenciye veya başka tenant'a taşmasını test et.
- AI timeout sonrası sistemin başarı mesajı vermediğini ve yarım mutation bırakmadığını doğrula.
- Token bütçesi tükendiğinde AI'ın sessizce boş yanıt üretmediğini, koordinatöre devrettiğini doğrula.
- `PAYMENT`, `COMPLAINT`, `TEACHER_CHANGE`, paket uyuşmazlığı ve politika istisnasının
  her varyantta human handoff'a gittiğini kontrol et.

## 7. Queue, reminder ve dağıtık sistem hataları

- Aynı reminder job'ını birden fazla worker aynı anda claim etsin.
- Gönderim başarılı fakat job ack başarısız; retry'da duplicate outbound mesaj oluşmamalı.
- Ders reminder'dan hemen önce cancel/reschedule edilsin; **stale job mesaj göndermemeli**.
- Worker clock skew, timezone farkı ve uzun queue backlog durumu.
- Dead-letter replay'in idempotent olduğunu ve eski/iptal edilmiş işi canlandırmadığını doğrula.
- Cron iki kez tetiklendiğinde aynı job'ların yeniden üretilmesini engelle.
- Downstream outage sırasında backpressure ve kullanıcıya gözlemlenebilir hata üretildiğini doğrula.
- Worker `SIGTERM` aldığında çalışan job'ın yarıda kesilip kesilmediğini test et.

## 8. Storage, dosya ve PII saldırıları

- Extension/MIME/magic-byte uyuşmazlığı, polyglot dosya, çift extension, oversized upload.
- Path traversal, tahmin edilebilir object key, başka tenant path'iyle signed URL üretme.
- Expired/revoked signed URL, silinmiş membership ve paylaşılmış URL'nin erişimi.
- SVG/HTML aktif içerik, EXIF içindeki PII, zararlı dosya karantinası.
- Log, error payload, analytics, audit metadata ve client source map içinde token/PII sızıntısı.
- Export/delete talebinde ilişkili mesaj, dosya, audit ve **yasal retention istisnalarının** doğru davranışı.

## 9. API, UI ve hata durumları

- Boş, null, whitespace, Unicode, RTL, emoji, çok uzun metin, negatif sayı, NaN ve sınır tarihleriyle
  tüm form/API validasyonlarını zorla.
- Double-click, browser refresh, back navigation, offline→online, iki tab'dan eş zamanlı düzenleme.
- Optimistic update başarısız olduğunda UI'ın eski doğru duruma döndüğünü doğrula.
- Loading/empty/error state'lerde sonsuz spinner, kaybolan hata, **yanlış başarı toast'ı** ara.
- Pagination/filter/search kombinasyonlarında tenant ve permission filtresinin kaybolup kaybolmadığı.
- Takvim drag-drop: çakışmada kartın eski konumuna döndüğünü ve sunucu durumunun bozulmadığını doğrula.
- Keyboard-only, focus trap, label, screen reader adı, kontrast, responsive overflow.
- API hata sözleşmesinde `code`, `userMessage` ve `correlationId`'nin **her kritik yolda** bulunduğunu doğrula.
- i18n: hard-coded metin kalmış mı; eksik çeviri anahtarı ham anahtar olarak mı gösteriliyor.

## 10. Migration, deploy ve recovery

- Boş database'e tüm migration'ları uygula; production benzeri veride upgrade et; rollback/forward-fix planı.
- Not-null/unique/index/RLS değişikliklerinde lock süresi ve mevcut kirli veri davranışı.
- Eski application version ile yeni şemanın kısa süreli birlikte çalışmasını test et.
- Yanlış/eksik env'de uygulama fail-fast mi açılıyor, yoksa güvensiz varsayımla mı.
- Backup'tan restore sonrası auth ilişkileri, storage referansları, lesson, ledger ve
  finansal invariant'ları doğrula.
- Health endpoint yeşil olsa bile kritik bağımlılık kapalıyken readiness'in doğru başarısız olduğunu test et.
- Worker deployment'ının web'den bağımsız restart edilebildiğini doğrula.

---

## Zorunlu invariant kontrolleri

- [ ] Bir kullanıcının etkin membership'i olmayan tenant'a ait tek bir kayıt dahi okunamaz veya değiştirilemez.
- [ ] Aynı teacher, room veya student için yasaklı overlap **hiçbir concurrency seviyesinde** oluşamaz.
- [ ] Bir domain event retry edilse bile aynı iş etkisi ikinci kez uygulanamaz.
- [ ] Lesson, attendance, status history, package ledger, hakediş ve audit kritik işlemlerde birbirini doğrular.
- [ ] Paket bakiyesi cache'i ledger toplamıyla her zaman uyumludur.
- [ ] Fatura bakiyesi = fatura toplamı − mahsup edilmiş ödemeler; hiçbir yolda bozulamaz.
- [ ] AI çıktısı schema + authorization + deterministic domain validation geçmeden mutation yapamaz.
- [ ] `HUMAN_ACTIVE` konuşmada AI kaynaklı outbound mesaj sayısı sıfırdır.
- [ ] İptal/taşınmış ders için stale reminder gönderilemez.
- [ ] Başarısız işlem kullanıcıya başarı gibi gösterilemez.
- [ ] Her kritik hata correlation ID ile uçtan uca takip edilebilir.
- [ ] Secret veya PII client bundle, log, exception ve analytics'e sızmaz.

---

## Bulgu şablonu

```
ID: BUG-XXX
Severity: P0 | P1 | P2 | P3
Alan: tenant | auth | scheduling | series | ledger | finance | whatsapp | ai | queue | storage | ui | deploy
Başlık: Kısa ve somut
Etkisi: Veri sızıntısı / veri kaybı / yanlış işlem / para hatası / kesinti / güvenlik
Önkoşullar:
Yeniden üretme adımları:
Beklenen:
Gerçekleşen:
Kök neden:
Düzeltme:
Regression testi:
Kanıt: komut, test adı, request/response, DB sonucu
Benzer pattern taraması:
Durum: OPEN | FIXED | VERIFIED | BLOCKED
```

## Severity kuralları

- **P0** — tenant/PII sızıntısı, yetkisiz mutation, çift rezervasyon, para veya paket hakkı bozulması,
  geri döndürülemez veri kaybı, yaygın production outage.
- **P1** — kritik akışın güvenilir biçimde başarısız olması, duplicate mesaj/işlem,
  ciddi auth/AI/queue kusuru, yanlış kullanıcı sonucu.
- **P2** — sınırlı edge case, kurtarılabilir tutarsızlık, önemli UX/accessibility veya
  gözlemlenebilirlik problemi.
- **P3** — düşük etkili polish, dokümantasyon veya maintainability kusuru.

## Bitirme kriteri — erken çıkış yasak

- [ ] Tüm P0 ve P1 bulguları FIXED ve bağımsız regression testiyle VERIFIED
- [ ] P2/P3 bulguları düzeltilmiş veya açık gerekçeli backlog kaydı, sahip ve hedef tarih ile raporlanmış
- [ ] Yeni regression testleri full suite içinde geçiyor
- [ ] Cross-tenant, concurrency, retry/idempotency, finansal tutarlılık ve AI safety testleri
      **ayrı ayrı** kanıtlandı
- [ ] Migration, production build, staging E2E ve smoke testleri geçiyor
- [ ] Son raporda test edilen alanlar, bulunup düzeltilen bug'lar, kalan riskler ve
      çalıştırılan komutlar yer alıyor
- [ ] Kanıtsız PASS, "muhtemelen çalışıyor", "test mevcut" veya "kod doğru görünüyor" ifadesi yok

## Ajanın üreteceği son rapor

`docs/test-reports/aggressive-bug-hunt.md`

1. Executive summary
2. Test edilen yüzeyler
3. P0/P1/P2/P3 bulguları
4. Yapılan düzeltmeler
5. Eklenen regression testleri
6. Invariant sonuçları
7. ADR uyum denetimi (implementasyon ADR-0001…0005'e uyuyor mu)
8. Kalan riskler / blocked maddeler
9. **Production verdict: GO | CONDITIONAL GO | NO-GO**

**NO-GO koşulu:** açık P0/P1, kanıtlanmamış tenant izolasyonu, başarısız concurrency/idempotency testi,
AI'ın doğrulamasız mutation yolu, tutarsız ledger veya finansal kayıt,
production entegrasyon smoke testinin tamamlanmamış olması.
