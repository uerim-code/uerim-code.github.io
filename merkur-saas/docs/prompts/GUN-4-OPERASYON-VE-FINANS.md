# GÜN 4 — DERS OPERASYONU + ÖDEV + DOSYA + BİLDİRİM + FİNANS

**Hedef:** Dersin tamamlanmasından ödev teslimine, reminder'dan tahsilata kadar
günlük operasyonun tamamı.

**Bu günün yükü en ağırdır. Paralel yürütme önerilir:**
- **4A** — Ders sonucu, devam, ödev, dosya, reminder, duyuru, portallar
- **4B** — Finans: fatura, tahsilat, borç takibi, öğretmen hakediş, raporlar

İki ajan aynı gün farklı branch'lerde çalışır; şema çakışması olmaması için 4B yalnızca
`finance_*` ve `invoice*/payment*` tablolarına dokunur.

---

# 4A — EĞİTİM OPERASYONU

## 4A.1 — Ders sonucu ve devam

Sonuç durumları: `COMPLETED · STUDENT_NO_SHOW · INSTRUCTOR_CANCELLED · STUDENT_CANCELLED · MAKEUP · POSTPONED`

- Öğretmen **yalnızca kendi yetkili olduğu dersi** sonuçlandırabilir.
- Completion + ledger değişikliği **aynı transaction** ve **idempotent**:
  aynı ders iki kez tamamlanırsa hak iki kez düşmez.
- Ders notu, bir sonraki hedef ve `attendance` kaydı oluşur.
- Öğrenci/veli yalnızca kendi kapsamındaki sonuçları görür.
- `STUDENT_NO_SHOW` politikası tenant ayarı: hak düşer / düşmez / makeup verilir.

## 4A.2 — Ödev ve teslim

`assignments` · `assignment_attachments` · `assignment_submissions` · `submission_reviews`

- Desteklenen içerik: PDF, nota, MP3, WAV, video, görsel, bağlantı.
- Öğrenci tamamlandı işaretler, ses/video yükleyebilir.
- Öğretmen değerlendirmesi: `COMPLETED · PRACTICE_AGAIN · REVISION_REQUIRED`
- Dosya türü, boyut, virüs tarama durumu ve signed URL kontrolü.

## 4A.3 — Storage güvenliği

- Tenant ve kullanıcı bazlı **private** bucket path'i. Public URL üretilmez.
- Kısa ömürlü signed URL; süre tenant ayarı, varsayılan 15 dk.
- **MIME + magic-byte** doğrulaması (uzantıya güvenilmez), boyut limitleri, zararlı dosya karantinası.
- SVG/HTML aktif içerik yüklenemez veya `Content-Disposition: attachment` ile servis edilir.
- Silme/retention kuralları ve audit kaydı.

## 4A.4 — Reminder engine

Tenant bazlı 48s / 24s / 3s / 1s kuralları.

```
idempotency key = lesson_id + reminder_rule_id + scheduled_for
```

- Ders değişince eski job **iptal edilir**, yeni job idempotent üretilir.
- Aynı reminder iki kez gönderilmez. Gönderim anında dersin hâlâ aktif olduğu **yeniden kontrol edilir**
  (iptal/taşınmış ders için stale reminder gitmez).
- Kanallar: WhatsApp template, e-posta, ileride push — kanal abstraction ile.
- Kayıt: message ID, status, attempt, error, audit.
- Timezone ve DST testleri (ADR-0005).

## 4A.5 — Duyurular ve portallar

- Duyuru tipleri: flyer, poster, etkinlik, konser, workshop, tatil, öğretmen duyurusu.
- Hedefleme: tenant · business unit · şube · hizmet · rol · seçili kullanıcılar.
- **Pazarlama içerikli duyuru WhatsApp'ta `marketing` template'i ve opt-in gerektirir** (Gün 3.7).
- Öğretmen "Bugün" görünümü: günün dersleri, tek tıkla sonuçlandırma, ödev verme.
- Öğrenci/veli portalı: sonraki ders, paket durumu, ödevler, duyurular, **bakiye/borç** (4B ile).

---

# 4B — FİNANS  *(v1.0'da yoktu)*

## 4B.1 — Fatura ve tahsilat

`invoices` · `invoice_lines` · `payments` · `payment_allocations` · `payment_methods`

- Paket satışı fatura üretir; fatura satırları hizmet/paket bazlı.
- Ödeme yöntemleri: nakit, havale/EFT, POS, (ileride online sağlayıcı adapter'ı).
- **Kısmi ödeme ve taksit** desteklenir; `payment_allocations` hangi ödemenin hangi faturaya
  ne kadar mahsup edildiğini tutar.
- Para birimi `tenant_settings.currency`; tutarlar **integer minor unit** (kuruş) olarak saklanır,
  float kullanılmaz.
- İade (`refund`) negatif ödeme olarak değil, ayrı bir kayıt tipi olarak modellenir.

**Online ödeme sağlayıcı entegrasyonu (iyzico/Stripe) bu sürümde kapsam dışıdır** —
şema ve akış sağlayıcı eklenebilecek şekilde tasarlanır, adapter arayüzü tanımlanır.

## 4B.2 — Borç takibi

- Öğrenci bazlı bakiye: fatura toplamı − mahsup edilmiş ödeme toplamı.
- Vade geçmiş borç raporu ve otomatik hatırlatma (`payment_due` template'i, Gün 0 A4).
- Tenant ayarı: geç ödemede ders alma engellenir mi, kaç gün sonra.
  Engelleme scheduling engine'e `PAYMENT_OVERDUE` reason code'u olarak bağlanır.

## 4B.3 — Öğretmen hakediş

`instructor_earning_rules` · `instructor_earnings` · `payout_runs`

Üç model de desteklenir: ders başı sabit ücret · ciro yüzdesi · aylık sabit maaş.
Hizmet ve öğretmen bazında farklı kural tanımlanabilir.

- Hakediş **ders tamamlandığında** hesaplanır ve kaydedilir (`COMPLETED` ile aynı transaction).
- `payout_runs`: dönem kapanışı, toplam, onay, ödendi işaretleme. Kapanmış dönem değiştirilemez.
- İptal/geri alma durumunda hakediş ters kaydı yazılır, satır silinmez.

## 4B.4 — Raporlar  *(v1.0'da menüde vardı, tanımsızdı)*

- Doluluk: öğretmen bazlı, oda bazlı, şube bazlı, dönemsel
- Gelir: hizmet/business unit/şube kırılımı, tahsil edilen vs tahakkuk eden
- Öğrenci: yeni kayıt, ayrılan (churn), devamsızlık oranı
- Paket: satılan, kullanılan, süresi dolan, dondurulan
- Öğretmen: verilen ders, hakediş, iptal oranı
- Tüm raporlar tenant timezone'ında gün sınırı kullanır; CSV/Excel dışa aktarım.

---

## Gün 4 testleri

**4A:**
- Ders tamamlama → attendance + ledger **tam olarak bir kez** (aynı istek iki kez gönderilir)
- Yetkisiz öğretmen sonuçlandıramaz
- Ödev oluştur/teslim et/değerlendir + tenant izolasyonu
- MIME/magic-byte/boyut reddi; uzantı-içerik uyuşmazlığı; signed URL süre dolması
- Reminder duplicate önleme; reschedule'da eski job iptali; iptal edilmiş derse reminder gitmemesi
- Timezone/DST sınırları; başarısız job retry ve dead-letter
- Duyuru hedef kitlesi izolasyonu; opt-out edilmiş kullanıcıya gitmemesi

**4B:**
- Fatura → kısmi ödeme → bakiye doğruluğu; aşırı ödeme reddi
- Aynı ödeme iki kez kaydedilirse ikinci mahsup oluşmaz (idempotency)
- Para hesaplarında float kullanılmadığının kanıtı; kuruş yuvarlama testleri
- Ders tamamlandığında hakediş **tam olarak bir kez** yazılır; iptalde ters kayıt oluşur
- Kapanmış `payout_run` değiştirilemez
- `PAYMENT_OVERDUE` engellemesi scheduling engine'de çalışır
- Rapor sorguları tenant izolasyonlu ve tenant timezone'ında

**Kapı:** lint · typecheck · unit · integration · RLS (CI) · concurrency (CI) ·
rapor: `docs/test-reports/gun-4.md`

## Blok bitmezse

4A.1, 4A.3, 4A.4 ve 4B.1 Gün 5 E2E'sinin ön koşuludur.
4B.3 (hakediş) ve 4B.4 (raporlar) gerekirse Gün 5 sonrasına taşınabilir —
bu durumda `docs/PROGRESS.md` içinde açık gerekçe, sahip ve hedef tarih ile kaydedilir,
Gün 5 exit checklist'inde "kapsam dışı" olarak raporlanır. **Sessizce atlanamaz.**
