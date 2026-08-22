# GÜN 0 — ÖN KOŞUL BLOĞU (insan işi, kod yok)

> Bu blok atlanırsa Gün 3 ve Gün 5'in büyük kısmı `BLOCKED_BY_CREDENTIAL` döner ve 5 günlük plan tutmaz.
> Buradaki maddelerin çoğu **takvimsel süre** ister; kod hızıyla ilgisi yoktur.

## A. Kritik yol — bugün başlatılmazsa Gün 3'ü kilitler

| # | İş | Tipik süre | Kilitlediği blok |
|---|---|---|---|
| A1 | Meta Business hesabı + **business verification** | 1–5 iş günü | Gün 3 tamamı |
| A2 | WhatsApp Business Account + telefon numarası kaydı | 1 gün | Gün 3 |
| A3 | Cloud API sistem kullanıcısı + kalıcı access token | 1 saat | Gün 3 |
| A4 | İlk message template'lerinin onaya gönderilmesi | 1–2 iş günü, **red riski var** | Gün 3.7, Gün 4.4 |
| A5 | Webhook için public HTTPS callback URL (staging domain) | 1 saat | Gün 3.1 |

**A4 detayı — onaya gönderilecek minimum template seti** (hepsi `utility` kategorisi):
`lesson_reminder_24h`, `lesson_reminder_3h`, `lesson_rescheduled`, `lesson_cancelled`,
`homework_assigned`, `payment_due`, `announcement_generic`.
Türkçe ve İngilizce varyantlarını birlikte gönder. Red gelirse aynı gün revize et — kuyruk tekrar başlar.

## B. Altyapı hesapları

| # | İş | Süre |
|---|---|---|
| B1 | Supabase projesi (staging + production, ayrı) | 30 dk |
| B2 | Vercel projesi + domain + TLS | 1 saat |
| B3 | Worker deployment hesabı (Railway / Fly.io / Render) | 30 dk |
| B4 | Error tracking (Sentry veya muadili) | 30 dk |
| B5 | GitHub repo + branch protection + CI secret'ları | 30 dk |

## C. AI sağlayıcı

| # | İş | Süre |
|---|---|---|
| C1 | AI provider API key (OpenAI veya muadili) + fatura limiti | 30 dk |
| C2 | Tenant başına aylık token bütçesi kararı (varsayılan üst sınır) | karar |

## D. İş kuralı kararları — kod yazılmadan netleşmeli

Bunlar tenant-configurable olacak, ama **varsayılan değerleri** Gün 1 seed'ine girecek:

- [ ] İptal cutoff süresi (varsayılan: 24 saat öncesine kadar kredi iadesi)
- [ ] Telafi (makeup) hakkı politikası: yıl/paket başına kaç adet, geçerlilik süresi
- [ ] Paket dondurma: izin var mı, en fazla kaç gün, yılda kaç kez
- [ ] Ders süreleri ve aralarındaki buffer (varsayılan: 30/45/60 dk, 0 dk buffer)
- [ ] Şube çalışma saatleri ve resmi tatil takvimi
- [ ] Deneme dersi: ücretli/ücretsiz, paketten düşer mi
- [ ] Öğretmen hakediş modeli: ders başı sabit / yüzde / aylık maaş (üçü de desteklenecek, varsayılan seçilecek)
- [ ] Geç ödeme kuralı: ders alma engellenir mi, kaç gün sonra
- [ ] KVKK: veri saklama süresi, çocuk öğrenci için veli açık rızası metni

## E. Test verisi

- [ ] Gerçek olmayan ama gerçekçi 20 öğrenci, 5 öğretmen, 8 hizmet, 3 oda listesi
- [ ] WhatsApp testi için en az 2 gerçek telefon numarası (Meta test numarası + 1 gerçek cihaz)

---

## Çıkış kriteri

`.env.example` dosyasındaki her anahtarın karşılığı **staging için** elde edilmiş olmalı;
production değerleri Gün 5'e kadar bekleyebilir.

A1–A4 tamamlanmadan Gün 1'e başlanabilir (Gün 1 ve 2 credential istemez),
ancak **Gün 3'ün başlangıcında A1–A5 bitmiş olmalıdır**. Bitmemişse Gün 3
mock WhatsApp transport ile yürütülür ve gerçek entegrasyon Gün 5'e ertelenir —
bu durumda 5 günlük plan 6–7 güne çıkar ve bu `docs/PROGRESS.md` içinde kayda geçer.
