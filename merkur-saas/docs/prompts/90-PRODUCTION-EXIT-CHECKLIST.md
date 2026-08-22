# PRODUCTION EXIT CHECKLIST

> Her madde **kanıtla** işaretlenir: komut, test adı, request/response, DB sonucu veya ekran görüntüsü.
> "Muhtemelen çalışıyor", "test mevcut", "kod doğru görünüyor" kanıt değildir.

## Kod ve kalite
- [ ] Lint temiz
- [ ] TypeScript strict typecheck temiz
- [ ] Unit, integration, E2E ve security testleri geçiyor
- [ ] `skip`/`todo` ile saklanmış kritik test yok
- [ ] Production build alınabiliyor
- [ ] Bilinen kritik/yüksek hata yok; orta/düşükler kayıtlı, sahipli ve tarihli

## Veri, tenant ve yetki
- [ ] Tüm tenant-owned tablolarda `tenant_id` ve RLS var; policy'siz tablo yok
- [ ] Cross-tenant API/UI/storage testleri DENIED — her tablo için en az bir test
- [ ] Service-role kullanan her kod yolu `withTenantContext` üzerinden (ADR-0003)
- [ ] Rol/permission matrisi test edildi; yetki kaldırıldığında cache invalidate oluyor
- [ ] Audit log değiştirilemez ve kritik işlemleri kapsıyor
- [ ] Migration temiz kurulumda ve mevcut veride doğrulandı
- [ ] Backup restore tatbikatı tamamlandı

## Scheduling ve paket
- [ ] Exclusion constraint'ler kurulu (ADR-0001); instructor, room, student için ayrı ayrı
- [ ] 100 eş zamanlı rezervasyon testinde 1 başarı / 99 conflict, DB'de tam 1 ders
- [ ] Reschedule rollback mevcut dersi koruyor
- [ ] İptal politikası tenant-configurable; cutoff tenant timezone'ında hesaplanıyor
- [ ] Ledger idempotent ve immutable; ders tamamlama iki kez hak düşürmüyor
- [ ] `verifyPackageBalance` drift bulmuyor
- [ ] Seri ders: tatil atlama, `THIS_ONLY`, `THIS_AND_FOLLOWING`, `ALL` testli (ADR-0004)
- [ ] Timezone/DST/tatil/çalışma saati/buffer/grup kapasitesi testli (ADR-0005)
- [ ] Alternatif slot seçiminde revalidation yapılıyor

## WhatsApp ve AI
- [ ] Meta webhook doğrulaması ve signature kontrolü çalışıyor
- [ ] **Webhook senkron AI çağırmıyor**; p99 < 500 ms ölçüldü
- [ ] Duplicate message idempotent
- [ ] 24 saat penceresi ve template eligibility doğru; kategori ayrımı uygulanıyor
- [ ] Opt-in/opt-out kayıtlı ve uygulanıyor
- [ ] AI structured output schema validation zorunlu
- [ ] AI hiçbir doğrudan database mutation yapamıyor
- [ ] AI'ın ürettiği ID'ler tenant ve sahiplik açısından doğrulanıyor
- [ ] Düşük confidence ve hassas konular insan devrine gidiyor
- [ ] `HUMAN_ACTIVE` durumunda AI otomatik mesaj gönderemiyor (gecikmiş job dahil)
- [ ] Token bütçesi aşımında AI devre dışı kalıp koordinatöre düşüyor
- [ ] Retry, rate limit, delivery status ve dead-letter akışı testli

## Dosya, bildirim ve gizlilik
- [ ] Private storage ve signed URL süreleri doğru
- [ ] MIME/magic-byte/boyut/zararlı dosya kontrolleri var
- [ ] Reminder duplicate önleme ve iptal/taşıma davranışı testli
- [ ] PII loglarda maskeli
- [ ] KVKK consent, retention, export ve deletion akışları dokümante; çocuk için veli rızası kayıtlı

## Finans
- [ ] Tutarlar minor unit (kuruş) integer; float yok
- [ ] Kısmi ödeme, mahsup ve iade doğru; aşırı ödeme reddediliyor
- [ ] Ödeme kaydı idempotent
- [ ] Hakediş ders tamamlamada tam bir kez yazılıyor; iptalde ters kayıt
- [ ] Kapanmış payout run değiştirilemiyor
- [ ] Raporlar tenant izolasyonlu ve tenant timezone'ında

## Operasyon ve yayın
- [ ] Env doğrulama ve secret yönetimi tamam; eksik secret'ta fail-fast
- [ ] Health ve readiness endpoint'leri doğru davranıyor (bağımlılık kapalıyken readiness kırmızı)
- [ ] Log, error tracking, metrik ve alarmlar aktif
- [ ] Worker ayrı deployment olarak çalışıyor ve graceful shutdown yapıyor
- [ ] Staging uçtan uca smoke testi geçti
- [ ] Runbook, incident playbook ve rollback prosedürü hazır
- [ ] Production deploy ve son smoke test başarılı
- [ ] Release candidate raporu ve bilinen eksikler kaydedildi
- [ ] `docs/test-reports/aggressive-bug-hunt.md` verdict = GO veya CONDITIONAL GO

---

## Kodlama ajanına son talimat

Blokları sırayla uygula. Her blok sonunda **yalnızca ilgili testleri** çalıştır
(RLS ve concurrency suite'leri hariç — onlar her blokta CI'da çalışır),
başarısızlıkları kök nedenine kadar düzelt ve test raporunu yaz. Ara onay isteme.

Bu checklist'in tüm zorunlu maddeleri **kanıtla** karşılanmadan işi tamamlanmış sayma.
Gerçek credential gerektiren fakat sağlanmamış adımları açıkça `BLOCKED_BY_CREDENTIAL` olarak
raporla ve kalan tüm güvenli işleri bitir.
