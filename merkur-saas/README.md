# Merkür SaaS

Multi-tenant kurs / akademi / sanat merkezi yönetim platformu.
İlk tenant **Merkür**; aynı tenant altında **Merkür Müzik** ve **Merkür Sanat** business unit'ları çalışır.
Sistem başka müzik okulları, sanat merkezleri, kurslar ve akademiler tarafından da kullanılabilir —
hiçbir business logic içinde "Merkür" adı hard-code edilmez.

Bu repository şu an **kod değil, yürütme planı** içerir: bir kodlama ajanının 5 gün içinde
uçtan uca çalışan, test edilmiş ve entegrasyonları tamamlanmış bir SaaS çekirdeği üretmesi için
hazırlanmış prompt seti, mimari kararlar ve çıkış kriterleri.

---

## Bu repository'de ne var

| Yol | İçerik |
|---|---|
| `docs/00-INCELEME-VE-ONERILER.md` | v1.0 prompt setinin incelemesi: güçlü yanlar, tespit edilen 13 boşluk ve her birinin nasıl kapatıldığı |
| `docs/prompts/00-GLOBAL-MASTER-PROMPT.md` | Her oturumun başında ajana verilecek değişmez kurallar |
| `docs/prompts/GUN-0-ON-KOSUL.md` | Kod yazılmadan önce toplanması gereken credential ve erişimler |
| `docs/prompts/GUN-1..GUN-5` | Günlük yürütme blokları, kabul kriterleri ve test kapıları |
| `docs/prompts/90-PRODUCTION-EXIT-CHECKLIST.md` | Kanıt zorunlu production çıkış kontrol listesi |
| `docs/prompts/99-BUG-HUNTING.md` | 5 blok bittikten sonra temiz bağlamlı ikinci ajanın çalıştıracağı adversarial QA promptu |
| `docs/architecture/ADR-*.md` | Ajana bırakılmaması gereken 5 kritik teknik karar |
| `docs/PROGRESS.md` | Ajanın her alt adımda güncelleyeceği durum dosyası (oturum koptuğunda buradan devam edilir) |

---

## Sürüm farkı: v1.0 → v2.0

v1.0 belgesi ("Merkür SaaS 5 Günlük Uçtan Uca Kodlama Prompt Seti") sağlam bir iskeletti.
v2.0 aynı iskeleti korur ve şunları ekler:

1. **Gün 0 ön-koşul bloğu** — credential'a bağlı işlerin 3. ve 5. günü kilitlemesini engeller.
2. **Tekrarlayan ders serisi (recurring lessons)** — v1.0'ın en büyük domain boşluğu.
3. **Finans modülü** — tahsilat, borç takibi, öğretmen hakediş. v1.0'da `PAYMENT` intent'i vardı ama karşılığı yoktu.
4. **5 adet ADR** — concurrency, queue, service-role, timezone ve recurring model kararları ajana bırakılmaz.
5. **Webhook'un asenkron zorunluluğu** — "hemen 200 dön, işi queue'da yap".
6. **i18n Gün 1'e alındı** — v1.0'da Gün 4'teydi, tüm UI'ın geriye dönük refactor'ünü gerektiriyordu.
7. **İki tenant'lı seed** — cross-tenant DENIED testleri ancak böyle yazılabilir.
8. **Paralel yürütme planı** — tam kapsamın 5 güne sığması için Gün 1 ve Gün 4 ikiye bölünür.
9. **PROGRESS.md state protokolü** — ajan context limitine takıldığında iş kaybolmaz.
10. **CI istisnası** — RLS ve concurrency testleri her blokta çalışır, "CI'ı minimumda tut" kuralının dışındadır.

Detaylı gerekçeler: [`docs/00-INCELEME-VE-ONERILER.md`](docs/00-INCELEME-VE-ONERILER.md)

---

## Kapsam uyarısı (bir kez söylenir, sonra uygulanır)

Seçilen kapsam **v1.0'ın tamamı + yukarıdaki eklemeler, 5 güne sıkıştırılmış**.
Bu kapsam tek ajan ve seri yürütmeyle 5 güne sığmaz. Sığması için iki şart var:

1. **Gün 0 aynı gün içinde bitmiş olmalı.** Meta WhatsApp Business doğrulaması ve template onayı
   takvimsel süre ister (tipik 1–3 iş günü, template reddi olursa daha uzun). Bu süre kod hızından bağımsızdır.
   Gün 0 başlamadan Gün 1'e girilirse Gün 3 ve Gün 5 `BLOCKED_BY_CREDENTIAL` döner.
2. **Gün 1 ve Gün 4 paralel yürütülmeli.** Aşağıdaki plana bakın.

Bu iki şart sağlanmazsa gerçekçi süre 8–10 gündür. Plan yine de 5 gün üzerinden yazıldı;
her günün sonunda "bu blok bitmediyse ne yapılır" kuralı var.

### Paralel yürütme planı

| Gün | Ajan A | Ajan B |
|---|---|---|
| 0 | Ön-koşul toplama (insan işi) | — |
| 1 | 1A: DB, tenant, auth, RBAC, RLS, audit | 1B: UI shell, i18n, öğrenci/öğretmen/paket ekranları |
| 2 | Scheduling engine + recurring series | Takvim UI + drag-drop |
| 3 | Webhook + queue + AI provider + intent | Unified inbox + koordinatör handoff UI |
| 4 | 4A: Ders sonucu, ödev, dosya, reminder, duyuru | 4B: Finans — tahsilat, borç, hakediş, raporlar |
| 5 | Production hardening + E2E + deploy | Bug hunting (temiz bağlam, ayrı oturum) |

Ajan B her gün Ajan A'nın o günkü şemasını `main`'den çeker; şema sahibi her zaman Ajan A'dır.
Tek ajanla yürütülecekse aynı sıra korunur, süre ~2 kat uzar.

---

## Kullanım

Her gün, yeni bir ajan oturumunda:

```
1. docs/prompts/00-GLOBAL-MASTER-PROMPT.md  → oturumun başına yapıştır
2. docs/prompts/GUN-N-*.md                  → o günün bloğu
3. docs/PROGRESS.md                          → ajan okur, günün sonunda günceller
```

Ajan blok bitiminde `docs/test-reports/gun-N.md` raporunu yazar ve ara onay istemeden devam eder.
5 blok bittikten sonra **temiz bağlamlı ayrı bir oturumda** `docs/prompts/99-BUG-HUNTING.md` çalıştırılır.

---

## Teknoloji

pnpm + Turborepo · TypeScript strict · Next.js / React · Supabase (PostgreSQL, Auth, Storage, Realtime, RLS) ·
Vercel · Meta WhatsApp Cloud API · provider-bağımsız AI katmanı · ileride aynı API'yi kullanacak Expo/React Native white-label istemci.

Kritik teknik kararlar ajana bırakılmaz, `docs/architecture/` altında sabittir.
