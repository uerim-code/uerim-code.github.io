# GLOBAL MASTER PROMPT
> Her ajan oturumunun **başına** yapıştırılır. Günlük blok bunun ardından gelir.

Merkür Müzik ve Merkür Sanat için production-ready, gerçek multi-tenant bir SaaS geliştiriyoruz.
İlk tenant Merkür; aynı tenant altında Merkür Müzik ve Merkür Sanat business unit'ları bulunur.
**Hiçbir business logic içinde "Merkür" adını hard-code etme.** Sistem başka müzik okulları,
sanat merkezleri, kurslar ve akademiler tarafından da kullanılabilmelidir.

## 0. Oturuma başlarken zorunlu ilk adım

1. `docs/PROGRESS.md` dosyasını oku. En son tamamlanan alt görevi bul.
2. `docs/architecture/ADR-*.md` dosyalarının tamamını oku. **Bunlar bağlayıcıdır, tartışmaya kapalıdır.**
   Bir ADR'ye aykırı implementasyon bulursan düzelt; ADR'yi değiştirme.
3. `git log --oneline -20` ile son durumu doğrula. Belgeye değil, **koda** güven.
4. O günün blok dosyasını uygula.

## Temel teknoloji

pnpm + Turborepo · TypeScript strict · Next.js (App Router) + React ·
Supabase (PostgreSQL, Auth, Storage, Realtime, RLS) · Vercel (web) + ayrı worker deployment ·
Meta WhatsApp Cloud API · provider-bağımsız AI katmanı ·
ileride aynı API'yi kullanacak Expo/React Native white-label mobil istemci.

## Repository yapısı

```
apps/
  web/                 Next.js — UI + API routes + server actions
  worker/              pg-boss consumer (uzun ömürlü süreç)
packages/
  domain/              framework-bağımsız iş kuralları
  database/            şema tipleri, client factory, withTenantContext
  scheduling/          deterministic planlama motoru
  auth/                permission resolver, session yardımcıları
  whatsapp/            Cloud API client, webhook doğrulama, template yönetimi
  ai/                  provider abstraction, structured output şemaları
  finance/             fatura, tahsilat, hakediş hesabı
  notifications/       reminder kuralları, kanal abstraction
  files/               storage, MIME/magic-byte doğrulama, signed URL
  ui/                  paylaşılan React bileşenleri
  shared/              hata tipleri, sonuç tipleri, i18n
  config/              env şeması ve runtime doğrulama
supabase/
  migrations/
  seed/
docs/
  architecture/  test-reports/  phase-reports/  prompts/  PROGRESS.md
```

Scheduling, package entitlement, cancellation, completion, reminder eligibility, finansal hesaplar ve
AI intent routing kurallarını React bileşenlerine veya API route'larına gömme.
Bunlar framework-bağımsız TypeScript domain servisleridir ve React olmadan test edilebilir olmalıdır.

---

## KRİTİK GELİŞTİRME KURALLARI

### 1. Silent failure yasak
Her hata şu sözleşmeyle döner:
```ts
{ code: string; message: string; userMessage: string; correlationId: string; metadata?: Record<string, unknown> }
```
UI anlaşılır hata gösterir; teknik ayrıntı yapılandırılmış biçimde loglanır.
Boş `catch`, loglayıp başarı dönme ve optimistic UI ile yanlış başarı gösterme yasaktır.

### 2. AI doğrudan veri değiştiremez
- LLM SQL çalıştıramaz, veritabanına yazamaz.
- Rezervasyon oluşturamaz/silemez, paket hakkı değiştiremez, ödeme kaydı açamaz.
- Yalnızca şema doğrulamalı application tool çağrısı üretebilir.

```
AI → Application Service → Scheduling/Finance Engine → Authorization → Database Transaction
```

### 3. Scheduling ve finans AI tarafından yapılmaz
AI doğal dili ve niyeti çözer. Uygunluk, çakışma, entitlement ve para hesabı **deterministic**
motorlar tarafından yapılır. AI, motorun döndürdüğü `reason code` dışında gerekçe uyduramaz.

### 4. Multi-tenancy
Her tenant-owned tabloda `tenant_id` bulunur. İzolasyon frontend filtresiyle değil
PostgreSQL/Supabase RLS ile sağlanır. Service-role erişimi ADR-0003'e tabidir.
**Tenant leakage P0'dır** — bulunduğu anda diğer her iş durur.

### 5. CI kullanımını minimumda tut — iki istisna ile
Normal geliştirmede local typecheck, unit test ve ilgili integration test yeterlidir.
Tam CI yalnızca blok sonu, release candidate ve final production testi için çalışır.
**İstisna:** RLS izolasyon suite'i ve concurrency suite'i **her blok sonunda CI'da** çalışır.
Bu iki suite için lokal başarı kanıt sayılmaz (tek connection, tek process, sıcak cache yanıltır).

### 6. Kullanıcı onayı bekleme
Scope içinde güvenli karar ver ve ilerle. Gerçek credential yoksa placeholder ve açıklamalı
`.env.example` üret. Ara adımlarda "devam edeyim mi?" diye sorma.
Credential gerektiren ve sağlanmamış işi `BLOCKED_BY_CREDENTIAL` olarak raporla, kalan güvenli işi bitir.

### 7. Test başarısızsa devam etme
Analiz et → kök nedeni düzelt → testi yeniden çalıştır → sonucu logla → devam et.
Testi disable etme; `skip`, `todo`, gevşetilmiş assertion veya sahte mock sonucu ile geçmiş gösterme.
Production davranışını yalnızca testi geçirmek için değiştirme.

### 8. Hard-coded kullanıcı metni yasak
Gün 1'den itibaren tüm kullanıcıya görünen metinler i18n anahtarıdır. Başlangıç dili Türkçe.
Hard-coded string lint hatasıdır.

### 9. PROGRESS.md protokolü
Her alt görev bitiminde `docs/PROGRESS.md` dosyasına tek satır ekle:
```
| GUN-2.3 | DONE | <commit-sha> | pnpm test --filter scheduling (42 passed) | 2026-08-24 |
```
Bu, oturum context limitine takılırsa işin kaybolmamasını sağlar. Atlanamaz.

### 10. Commit disiplini
Her alt görev en az bir commit. Commit mesajı: `gun-2.3: exclusion constraint + conflict reason mapping`.
Yarım kalan iş commit edilmez; commit edilen her şey typecheck'ten geçmiş olmalıdır.
