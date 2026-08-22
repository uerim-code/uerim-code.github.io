# GÜN 1 — TEMEL SAAS: DATABASE + AUTH + RBAC + UI SHELL

**Hedef:** Çalışan multi-tenant veritabanı, authentication, authorization, öğrenci/veli/öğretmen/hizmet/
oda/paket altyapısı, audit, i18n ve rol-duyarlı yönetim UI'ı.

**Paralel yürütme:** 1A (veri katmanı) ve 1B (UI katmanı) iki ajanla paralel yürütülebilir.
Şema sahibi daima 1A'dır; 1B tipleri `packages/database` üzerinden tüketir.

---

## 1.1 — Monorepo ve araç zinciri  *(1A)*

pnpm workspace + Turborepo, TypeScript strict (`strict: true`, `noUncheckedIndexedAccess: true`),
ESLint + Prettier, Vitest (unit/integration), Playwright (E2E, Gün 5'te kullanılacak).

Zorunlu lint kuralları:
- `no-restricted-imports`: `packages/database/src/service-client` yalnızca `withTenantContext` içinden (ADR-0003).
- Hard-coded kullanıcı metni yasağı (i18n zorunluluğu).
- `no-floating-promises`, `no-misused-promises`.

`packages/config`: **runtime env doğrulama** (zod). Eksik kritik secret'ta uygulama fail-fast açılmaz.
`.env.example` her anahtarın ne işe yaradığını açıklayan yorumla birlikte.

## 1.2 — Multi-tenant ve organizasyon modeli  *(1A)*

`tenants` · `tenant_settings` · `business_units` · `branches` · `rooms` · `holidays` · `branch_opening_hours`

- `tenant_settings`: `timezone` (IANA, ADR-0005), `locale`, `currency`,
  `cancellation_cutoff_hours`, `makeup_policy`, `freeze_policy`, `ai_monthly_token_budget`.
- `rooms`: `name, branch_id, capacity, room_type, supported_service_ids[], active`.

## 1.3 — Kullanıcı, rol ve yetkiler  *(1A)*

`profiles` · `memberships` · `roles` · `permissions` · `role_permissions` · `invitations`

Roller: `platform_admin`, `tenant_owner`, `tenant_admin`, `coordinator`, `secretary`,
`finance`, `instructor`, `student`, `guardian`.

- Custom rol desteği; permission'lar rolden ayrı modellenir.
- **Davet/aktivasyon akışı** (v1.0'da eksikti): `invitations` tablosu, tek kullanımlık token,
  son kullanma süresi, kabul edildiğinde `membership` oluşur. Token yeniden kullanılamaz.
- Permission resolver saf fonksiyondur ve cache'lenirse **yetki kaldırıldığında cache invalidate edilir**.

## 1.4 — Öğrenci ve veli  *(1A)*

`students` · `guardians` · `student_guardians` · `student_notes` · `student_documents` · `consents`

- `students`: `id, tenant_id, business_unit_id, branch_id, student_number, first_name, last_name,
  birth_date, phone, email, address, status, registration_date, photo_path, created_at, updated_at`.
- Telefonlar **E.164** normalize edilir. Normalization saf fonksiyondur ve unit test edilir.
- Bir veli birden fazla öğrenciyi, bir öğrenci birden fazla veliyi destekler (`relation`, `is_primary`).
- `consents`: reşit olmayan öğrenci için veli açık rızası; tip, metin sürümü, tarih, IP, geri çekme kaydı (KVKK).

## 1.5 — Öğretmen ve hizmet kataloğu  *(1A)*

`instructors` · `instructor_services` · `instructor_branches` · `instructor_working_hours` ·
`instructor_leaves` · `services` · `service_categories`

- **Generic hizmet modeli**: piyano, gitar, keman, şan, solfej, resim, seramik ve workshop
  aynı sistemde çalışır. Enstrüman/branş adı veriye gömülür, koda değil.
- `services`: `duration_minutes`, `buffer_minutes`, `capacity` (1 = bireysel, >1 = grup), `requires_room_type`.
- `instructor_working_hours`: yerel duvar saati olarak saklanır (ADR-0005).

## 1.6 — Paketler ve ledger  *(1A)*

`package_templates` · `student_packages` · `package_ledger`

Bakiye **yalnızca** `remaining_lessons` kolonuna bağlı olamaz. Immutable ledger event'leri:

```
PACKAGE_CREATED · LESSON_RESERVED · LESSON_COMPLETED · LESSON_CANCELLED
CREDIT_RETURNED · MANUAL_ADJUSTMENT · PACKAGE_EXPIRED
MAKEUP_GRANTED · PACKAGE_FROZEN · PACKAGE_UNFROZEN · TRIAL_LESSON
```

- `package_ledger` yalnızca INSERT kabul eder; UPDATE/DELETE trigger ile reddedilir.
- `remaining_lessons` **türetilmiş cache**'tir; ledger toplamıyla karşılaştıran bir doğrulama
  fonksiyonu (`verifyPackageBalance`) yazılır ve drift bulursa hata döner.

## 1.7 — Auth, RLS ve audit  *(1A)*

- Supabase Auth: email/password, password reset, session refresh; mobil istemciyle uyumlu token akışı.
- Tenant-owned **tüm** tablolarda RLS, **deny-by-default**.
- Policy'ler hem `auth.uid()` hem `current_setting('app.tenant_id', true)` kontrol eder (ADR-0003).
- `withTenantContext` ve `withPlatformAdminContext` implement edilir.
- `audit_logs`: `tenant, actor, action, entity_type, entity_id, before, after, timestamp, correlation_id`.
  Normal kullanıcı değiştiremez (RLS + trigger ile UPDATE/DELETE reddi).

## 1.8 — Seed  *(1A)*

**İki tenant** üretilir:
1. `Merkür` — business unit'lar: Merkür Müzik, Merkür Sanat. Gerçekçi demo verisi.
2. `Test Akademi` — izolasyon testleri için ayrı kullanıcılar, öğrenciler, paketler, dosyalar.

Tek tenant'lı seed cross-tenant DENIED testlerini imkânsız kılar; bu madde atlanamaz.

## 1.9 — i18n  *(1B)*  — v1.0'da Gün 4'teydi, öne alındı

`packages/shared/i18n`: anahtar tabanlı çeviri, `tr` (varsayılan) ve `en` iskeleti.
Gün 1'den itibaren hard-coded metin lint hatası. Tarih/saat/para biçimlendirmesi
tenant locale ve timezone'ına göre.

## 1.10 — Web UI shell  *(1B)*

Profesyonel, responsive SaaS shell. Role göre menü görünürlüğü:
Dashboard · Takvim · Öğrenciler · Öğretmenler · Dersler · Paketler · WhatsApp · Ödevler ·
Duyurular · Finans · Raporlar · Ayarlar.

**Menüyü gizlemek yetki değildir** — her endpoint kendi yetkisini bağımsız doğrular.

Öğrenci detay tabları: Genel · Dersler · Paketler · Devam · Ödevler · Dosyalar · Mesajlar · Notlar · Finans · Geçmiş.

Her liste ekranı: sunucu tarafı pagination, arama, filtre; boş/yükleniyor/hata durumları eksiksiz.

---

## Gün 1 testleri

**Unit:** E.164 normalization, permission resolver, ledger bakiye hesabı, `verifyPackageBalance` drift tespiti,
tenant yardımcıları, env şema doğrulaması.

**Integration:** student CRUD, guardian ilişkisi, instructor CRUD, paket atama, davet kabul akışı,
audit kaydı üretimi, `withTenantContext` olmadan sorgunun 0 satır dönmesi.

**Security (CI'da zorunlu):**
- Tenant A → Tenant B öğrenci/paket/dosya = DENIED (her tenant-owned tablo için en az bir test)
- `student` rolü → admin endpoint = DENIED
- Mass assignment: request body'ye `role`, `tenant_id`, `created_by` eklenmesi yok sayılır
- Kullanılmış davet token'ı ikinci kez kabul edilmez

**Kapı:** lint · strict typecheck · unit · integration · RLS suite (CI) · rapor: `docs/test-reports/gun-1.md`

## Blok bitmezse

Tamamlanmayan alt görevler `docs/PROGRESS.md` içinde `BLOCKED` veya `PARTIAL` olarak işaretlenir ve
Gün 2'ye taşınmaz — **1.2, 1.3, 1.6, 1.7 ve 1.8 Gün 2'nin ön koşuludur**, bunlar bitmeden Gün 2'ye geçilmez.
1.10'un eksik ekranları Gün 2'ye taşınabilir.
