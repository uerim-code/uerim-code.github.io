# v1.0 Prompt Setinin İncelemesi ve v2.0 Gerekçeleri

Kaynak belge: *Merkür SaaS — 5 Günlük Uçtan Uca Kodlama Prompt Seti, Sürüm 1.0, Ağustos 2026*

---

## 1. Korunan güçlü yanlar

Bunlar v2.0'da aynen korundu, bazıları güçlendirildi.

**AI'ın mutasyon yetkisinin kesilmesi.** `AI → Application Service → Scheduling Engine → Authorization → DB Transaction`
zinciri ve "LLM SQL çalıştıramaz, veritabanına yazamaz" kuralı doğru mimari sınırı çiziyor.
Structured output + schema validation + "invalid AI response hiçbir mutation başlatamaz" bu sınırı kilitliyor.
Bu, LLM tabanlı planlama sistemlerinde en sık yapılan hatanın panzehiri.

**Deterministic scheduling'in AI'dan ayrılması.** AI niyeti çözer, uygunluk/çakışma/entitlement hesabını
deterministic motor yapar. v2.0'da bu ayrım daha da sertleştirildi: AI tool'u scheduling engine'i
çağırmak zorunda, engine'in döndürdüğü `reason code` dışında bir gerekçe üretemez.

**Ledger'ın event-sourced ve immutable olması.** `remaining_lessons` kolonuna güvenilmemesi ve
bakiyenin event toplamından türetilmesi finansal tutarlılığın temeli. Bug-hunt bölümünde
cache/ledger drift tespiti aranması da doğru.

**Reminder idempotency key formülü.** `lesson_id + reminder_rule_id + scheduled_for` somut ve uygulanabilir;
çoğu belgede bu madde "duplicate göndermeyin" temennisi olarak kalır.

**Yapılandırılmış hata sözleşmesi.** `{code, message, userMessage, correlationId, metadata?}` +
silent failure yasağı. v2.0'da bu sözleşmeye uymayan hata dönüşü lint kuralıyla yakalanır.

**Kanıt zorunluluğu.** "Kanıtsız PASS yok", "test mevcut demek yeterli değil", testi skip/todo/mock ile
geçmiş gösterme yasağı. Exit checklist'in tamamı delil talep ediyor.

**Bug hunting promptu.** Belgenin en olgun parçası. TOCTOU yarışı, prompt injection, mass assignment,
polyglot dosya, worker clock skew, dead-letter replay idempotency — bunlar deneyimden yazılmış maddeler.
v2.0'da yalnızca yeni modüller (recurring series, finans) için bölüm eklendi, mevcut maddelere dokunulmadı.

---

## 2. Tespit edilen boşluklar ve v2.0'daki karşılıkları

### B1 — Süre gerçekçiliği
**Sorun.** 5 günde bu kapsam production-ready olmuyor. Meta WhatsApp Business doğrulaması ve
template onayı takvimsel süre ister; kod hızıyla ilgisi yok. v1.0 bunu "credential yoksa placeholder üret"
ile geçiştiriyor, ama Gün 3 ve Gün 5'in büyük kısmı tam olarak credential'a bağlı.

**v2.0.** `GUN-0-ON-KOSUL.md` eklendi: kod yazılmadan önce toplanacak credential/erişim listesi,
her birinin tipik temin süresi ve o credential gelmezse hangi işin bloke olacağı.
Ayrıca paralel yürütme planı ile Gün 1 ve Gün 4 ikiye bölündü.

### B2 — Tekrarlayan ders serisi yok  *(en büyük domain boşluğu)*
**Sorun.** Müzik/sanat okulunda ders "her Çarşamba 17:00, 12 hafta" şeklindedir. v1.0 tek tek `lessons`
kaydı varsayıyor. Eksikler: seri tanımı, "bu dersi mi tüm seriyi mi taşıyorum" ayrımı, seri içi istisna,
tatil çakışmasında serinin davranışı, seri iptalinde ledger etkisi.
Gün 2'de eklenmezse Gün 4'te geriye dönük şema değişimi çıkar.

**v2.0.** `ADR-0004` + Gün 2 bloğunda `lesson_series`, `series_exceptions`, `SeriesEditScope`
(`THIS_ONLY | THIS_AND_FOLLOWING | ALL`) modeli zorunlu kılındı.

### B3 — Ödeme/finans modülü yok
**Sorun.** v1.0'da `PAYMENT` intent'i ve `finance` rolü var, karşılığında modül yok. Paket satılıyor
ama tahsilat, borç/alacak, fatura, öğretmen hakediş yok. Bir kurs yazılımının en çok kullanılan ekranı bu.

**v2.0.** Gün 4B olarak eklendi: `invoices`, `payments`, `payment_allocations`, `instructor_earnings`,
`payout_runs`. Ödeme sağlayıcı entegrasyonu (iyzico/Stripe) **kapsam dışı**, ama şema ve manuel tahsilat
(nakit/havale/POS) akışı kapsam içi — sağlayıcı sonradan adapter olarak takılır.

### B4 — Concurrency çözümü ajana bırakılmış
**Sorun.** v1.0: "database transaction/locking **veya** exclusion constraint ile engelle".
Bu belirsizlik en pahalı P0'ın çıkacağı yer. Ajan advisory lock + serializable + application-level check
karışımı yapar; 20 paralel istek testi tesadüfen geçer, 100'de çöker.

**v2.0.** `ADR-0001` kararı sabitledi: `btree_gist` extension + `EXCLUDE USING gist` constraint,
instructor / room / student için ayrı ayrı, `tstzrange` üzerinde, iptal edilmiş dersler `WHERE` ile hariç.
Uygulama katmanı kontrolü yalnızca kullanıcıya erken geri bildirim içindir; **garanti veritabanındadır**.

### B5 — Webhook senkron okunuyor
**Sorun.** v1.0 Gün 3.5 akışı (`webhook → AI → scheduling → confirmation`) tek istek gibi duruyor.
Vercel fonksiyon timeout'u ile birleşince Meta retry fırtınası ve duplicate işlem üretir.
v1.0 idempotency istiyor ama "webhook senkron AI çağırmasın" demiyor.

**v2.0.** Gün 3'te sert kural: webhook handler yalnızca **signature doğrular, ham payload'ı persist eder,
job enqueue eder ve 200 döner**. Hedef p99 < 500 ms. AI ve scheduling worker'da çalışır.

### B6 — Queue/worker altyapısı seçilmemiş
**Sorun.** `apps/worker` var ama teknoloji yok. Vercel'de uzun ömürlü worker yok.
Reminder engine, dead-letter, replay runbook — hepsi bu seçimin üstünde duruyor.

**v2.0.** `ADR-0002`: PostgreSQL tabanlı queue (pg-boss), Vercel Cron tetikleyici + ayrı worker süreci.
Harici SaaS queue bağımlılığı yok; dead-letter ve replay aynı veritabanında, aynı transaction semantiğinde.

### B7 — Service-role ile RLS bypass mimarisi tanımsız
**Sorun.** Worker ve webhook service-role kullanır, yani RLS devre dışı kalır. v1.0 "service-role kullanım
sınırı" diyor ama nasıl sınırlandığını söylemiyor. Bu, tenant leakage'ın en olası kaynağı.

**v2.0.** `ADR-0003`: service-role client'a doğrudan erişim ESLint kuralıyla yasak; tek giriş noktası
`withTenantContext(tenantId, fn)`. Sarmalayıcı `SET LOCAL app.tenant_id` yapar ve RLS policy'leri
service-role altında da bu değişkeni kontrol eder — yani service-role RLS'i tamamen bypass etmez.

### B8 — i18n yanlış blokta
**Sorun.** v1.0'da i18n Gün 4'te. Ama UI Gün 1'de yazılıyor. Sonuç: tüm ekranların geriye dönük refactor'ü.

**v2.0.** i18n Gün 1'e alındı. Gün 1'den itibaren hard-coded kullanıcı metni lint hatası.

### B9 — Seed tek tenant
**Sorun.** v1.0 seed'i yalnızca Merkür tenant'ını üretiyor. Cross-tenant DENIED testleri yazılamaz.

**v2.0.** Seed **iki** tenant üretir: `Merkür` (gerçek) ve `Test Akademi` (izolasyon testi için),
her birinde ayrı kullanıcılar, öğrenciler, paketler ve dosyalar.

### B10 — Timezone stratejisi yazılmamış
**Sorun.** v1.0 DST testi istiyor ama saklama kuralını söylemiyor. Ajan `timestamp` seçerse
DST testleri kökten çöker.

**v2.0.** `ADR-0005`: tüm zaman kolonları `timestamptz`, UTC saklanır; `tenant_settings.timezone`
(IANA, örn. `Europe/Istanbul`) tek doğruluk kaynağı; çalışma saatleri ve tatiller tenant TZ'de yorumlanır;
`Date` aritmetiği yasak, tarih hesapları TZ-aware kütüphaneyle.

### B11 — Eksik domain kavramları
v1.0'da bulunmayan, kurs işletmesinde günlük kullanılan kavramlar:
telafi (makeup) hakkı ledger event'i, paket dondurma, bekleme listesi, deneme dersi,
öğrenci/veli portalı davet-aktivasyon akışı (bug-hunt'ta test ediliyor ama hiç spec edilmemiş),
"Raporlar" menüsü (var ama tanımsız), AI için tenant başına token bütçesi,
WhatsApp conversation-based pricing ve marketing/utility/service kategori ayrımı.

**v2.0.** Ledger event listesi genişletildi (`MAKEUP_GRANTED`, `PACKAGE_FROZEN`, `PACKAGE_UNFROZEN`,
`TRIAL_LESSON`), Gün 1'e davet/aktivasyon akışı, Gün 2'ye bekleme listesi ve deneme dersi,
Gün 4'e rapor seti ve AI bütçe limiti eklendi.

### B12 — Prompt mekaniği / context riski
**Sorun.** Her blok tek dev prompt. Ajan context limitine takılıp oturum ortasında bağlamı kaybeder;
yarım kalan iş görünmez olur.

**v2.0.** `docs/PROGRESS.md` state protokolü: ajan her alt adım bitiminde tek satır ekler
(adım, durum, commit SHA, çalıştırılan test). Yeni oturum önce bu dosyayı okur.
Ayrıca her blok numaralandırılmış alt görevlere bölündü.

### B13 — "CI'ı minimumda tut" kuralı riskli
**Sorun.** Doğru bir maliyet kuralı, ama RLS ve concurrency testleri tam olarak lokalde
yanıltıcı geçen testlerdir (tek connection, tek process, sıcak cache).

**v2.0.** Kural korundu, iki istisna eklendi: **RLS izolasyon suite'i ve concurrency suite'i
her blok sonunda CI'da çalışır**, lokal sonuç kanıt sayılmaz.

---

## 3. Değiştirilmeyen ama dikkat edilmesi gerekenler

- **`REQUIRES_APPROVAL` durumu** v1.0'da tanımlı ama onay akışının kim tarafından, hangi ekrandan
  yürütüleceği yazılmamış. v2.0 Gün 2'de koordinatör onay kuyruğu olarak somutlaştırıldı.
- **Grup dersi kapasitesi** var, ama grup paketi/ücretlendirmesi belirsizdi. v2.0'da grup dersinde
  her katılımcının kendi paketinden düşüm yapılır kuralı sabitlendi.
- **KVKK** maddeleri v1.0'da doğru ama yüzeysel. Çocuk öğrenci için veli açık rızası ayrı bir
  kayıt tipi olarak Gün 1'e alındı; aydınlatma metni ve veri envanteri Gün 5 dokümantasyonunda.
