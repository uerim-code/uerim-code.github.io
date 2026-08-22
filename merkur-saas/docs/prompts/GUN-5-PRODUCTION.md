# GÜN 5 — PRODUCTION HARDENING + E2E + DEPLOY

**Hedef:** Sistemi demo seviyesinden production-ready seviyeye getirmek; gerçek entegrasyon
smoke testlerini, güvenlik ve gözlemlenebilirlik kontrollerini tamamlamak; geri dönüş planıyla deploy etmek.

---

## 5.1 — Config, secret ve ortamlar

- `development · test · staging · production` ayrımı.
- Runtime env doğrulama; eksik kritik secret'ta **fail-fast** (uygulama açılmaz, varsayılanla devam etmez).
- Secret'lar loglara ve client bundle'a sızmaz — build çıktısında otomatik secret taraması.
- `.env.example` açıklamalı; gerçek credential repository'de yok.

## 5.2 — Observability

- Structured log, correlation ID uçtan uca (webhook → queue → worker → DB → outbound mesaj).
- Error tracking ve temel performans metrikleri.
- Health ve **readiness** endpoint'leri ayrı: readiness kritik bağımlılık (DB, queue) kapalıyken
  yeşil dönmemelidir.
- Alarmlar: webhook hata oranı · **queue backlog ve dead-letter sayısı** · AI hata oranı ve token bütçesi ·
  rezervasyon çakışma anomalisi · reminder gönderim hatası · ödeme kaydı hataları.
- Queue retry, exponential backoff, dead-letter ve **replay runbook'u** (ADR-0002).

## 5.3 — Security hardening

- RLS **deny-by-default** doğrulaması: policy'siz tenant-owned tablo kalmadığının otomatik testi.
- Service-role kullanan her kod yolunun `withTenantContext` üzerinden gittiğinin kanıtı (ADR-0003).
- CSRF, XSS, SQL injection, SSRF, open redirect, rate limit testleri.
- Auth session rotation, brute-force koruması, tam permission matrisi testi.
- PII redaction (log, error payload, analytics); KVKK consent / retention / export / delete akışları.
- Dependency ve secret taraması; kritik/yüksek bulgu bırakılmaz.

## 5.4 — E2E senaryoları (Playwright)

1. Tenant admin giriş yapar; öğrenci, veli, öğretmen, hizmet, oda ve paket oluşturur.
2. Uygun slotta ders planlar; takvim ve entitlement doğrulanır.
3. **Haftalık seri ders** oluşturur; tatil gününün atlandığı, entitlement'ın doğru düştüğü doğrulanır.
4. Öğrenci WhatsApp'tan taşıma ister; AI şeması ve scheduling engine üzerinden ders taşınır.
5. Çakışmalı istekte alternatif sunulur, ikinci mesajla seçim tamamlanır ve slot yeniden doğrulanır.
6. Ödeme/şikâyet mesajı koordinatöre devredilir; takeover sonrası AI susar.
7. Reminder job **tek** mesaj gönderir ve delivery event kaydeder; ders iptal edilince reminder gitmez.
8. Öğretmen dersi tamamlar, ödev verir; öğrenci teslim eder; ledger ve hakediş doğru güncellenir.
9. Fatura kesilir, kısmi ödeme alınır, bakiye doğru görünür.
10. **Tenant A kullanıcısı Tenant B verisine hiçbir API/UI/storage yolundan erişemez.**

## 5.5 — Performance ve dayanıklılık

- Takvim sorguları ve kritik indeksler için `EXPLAIN ANALYZE` kontrolü; seq scan bırakılmaz.
- Scheduling concurrency load testi (ADR-0001 kabul testi, 100 paralel).
- Webhook burst testi ve retry davranışı.
- AI timeout/fallback: kullanıcıya **yanlış başarı mesajı verilmez**, yarım mutation bırakılmaz.
- Database backup, **restore tatbikatı** ve rollback migration kontrolü.

## 5.6 — Deploy ve dokümantasyon

- Supabase migration sırası ve seed güvenliği (production seed'i demo verisi içermez).
- Vercel (web) + worker deployment (ADR-0002) + domain/TLS + webhook callback doğrulaması.
- README, mimari diyagram, ADR'ler, API sözleşmesi, operations runbook, incident playbook.
- **Staging smoke testleri geçmeden production'a geçilmez.**
- Production smoke testinden sonra gözlem penceresi; rollback kriterleri **önceden** yazılır.

---

## Gün 5 test kapısı

- lint · strict typecheck · unit · integration · security · E2E — tam suite
- migration up/down ve **temiz veritabanında sıfırdan kurulum**
- production build + bundle secret taraması
- staging'de gerçek WhatsApp webhook + AI + scheduling smoke testi
- load/concurrency ve backup/restore smoke
- accessibility: klavye, focus, label, kontrast; temel responsive kontrol

Raporlar: `docs/test-reports/gun-5.md` · `docs/test-reports/release-candidate.md`

---

## Gün 5 sonrası — zorunlu

5 blok bittikten sonra **temiz bağlamlı, ayrı bir oturumda**
`docs/prompts/99-BUG-HUNTING.md` çalıştırılır.
Bu adım opsiyonel değildir; onun `GO` verdiği rapor olmadan production'a çıkılmaz.
