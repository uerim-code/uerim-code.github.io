# GÜN 3 — WHATSAPP + AI + KOORDİNATÖR HANDOFF

**Hedef akış:**
```
Öğrenci → WhatsApp → Webhook (200 OK, <500ms) → Queue → Worker
  → AI (niyet + varlık) → Scheduling Engine (deterministic karar)
  → Takvim → WhatsApp confirmation
```

**Ön koşul:** Gün 0 A1–A5 tamamlanmış olmalı. Değilse mock transport ile ilerle,
`BLOCKED_BY_CREDENTIAL` yaz ve gerçek entegrasyonu Gün 5'e taşı.
**Bağlayıcı:** ADR-0002 (queue), ADR-0003 (service-role).

---

## 3.1 — Meta Cloud API ve webhook  *(3A)*

Doğrudan Meta Cloud API; **BSP bağımlılığı kurma**.

Env: `WHATSAPP_PHONE_NUMBER_ID` · `WHATSAPP_BUSINESS_ACCOUNT_ID` · `WHATSAPP_ACCESS_TOKEN` ·
`WHATSAPP_VERIFY_TOKEN` · `WHATSAPP_APP_SECRET`

- `GET` verification (hub.challenge) ve `POST` webhook.
- **X-Hub-Signature-256 doğrulaması zorunlu.** Geçersiz imza → 401, hiçbir işlem yapılmaz, hiçbir kayıt açılmaz.
- WhatsApp `message.id` üzerinde unique constraint → duplicate webhook ikinci kez işlenmez.

### Webhook asenkron zorunluluğu (v1.0'da eksikti — kritik)

Webhook handler **yalnızca** şunu yapar:
1. İmza doğrula
2. Ham payload'ı `webhook_events` tablosuna yaz (idempotent, `message.id` unique)
3. İşleme job'ını enqueue et (aynı transaction — ADR-0002)
4. `200` dön

Hedef **p99 < 500 ms**. AI çağrısı, scheduling ve mesaj gönderimi **worker'da** yapılır.
Webhook içinde AI çağrılırsa Meta timeout'a düşer, retry fırtınası ve duplicate işlem üretir.
Bu kural ihlal edilemez.

## 3.2 — Mesajlaşma modeli ve inbox  *(3A şema, 3B UI)*

`conversations` · `conversation_participants` · `messages` · `message_delivery_events` ·
`conversation_assignments` · `webhook_events` · `opt_in_status`

- `direction`: `INBOUND | OUTBOUND` · `actor`: `STUDENT | GUARDIAN | AI | COORDINATOR | SYSTEM`
- **Unified inbox** *(3B)*: solda konuşma listesi, ortada thread, sağda bağlam paneli —
  öğrenci/veli, paket ve kalan ders, sonraki ders, öğretmen, açık ödev, **bakiye/borç**.
- Filtreler: AI aktif · koordinatör bekliyor · insan aktif · okunmamış · çözüldü
- Realtime: yeni mesaj geldiğinde inbox Supabase Realtime ile güncellenir.

## 3.3 — AI provider abstraction  *(3A)*

```ts
interface AIProvider {
  classifyIntent(input): Promise<IntentResult>
  extractEntities(input): Promise<EntityResult>
  generateReply(input): Promise<string>
  summarizeConversation(input): Promise<string>
}
```

- İlk provider OpenAI olabilir; provider ve model `tenant_settings` üzerinden değişir.
- API key encrypted secret/environment'ta durur, **frontend'e asla gönderilmez**.
- **Token bütçesi** (v1.0'da yoktu): tenant başına aylık limit; limit aşılırsa AI devre dışı kalır ve
  tüm konuşmalar koordinatöre düşer — sessizce başarısız olmaz, koordinatöre uyarı gider.
- Her AI çağrısı için token kullanımı, gecikme ve maliyet kaydedilir.

## 3.4 — Structured output ve intent'ler  *(3A)*

```
SCHEDULE_LESSON | RESCHEDULE_LESSON | CANCEL_LESSON
QUERY_SCHEDULE | QUERY_PACKAGE | QUERY_HOMEWORK | QUERY_BALANCE | GENERAL_INFO
PAYMENT | COMPLAINT | TEACHER_CHANGE | UNKNOWN
```

```json
{
  "intent": "RESCHEDULE_LESSON",
  "confidence": 0.96,
  "requestedDate": "2026-08-28",
  "requestedTime": "17:00",
  "lessonId": "…",
  "needsHuman": false
}
```

- **Schema validation zorunlu** (zod). Doğrulamayı geçmeyen AI çıktısı hiçbir database mutation başlatamaz.
- Doğrulama hatası → konuşma koordinatöre devredilir, kullanıcıya "bir yetkilimiz dönecek" mesajı gider.
  Sessiz düşme yok, uydurma yanıt yok.
- AI'ın ürettiği `lessonId` ve `studentId` **daima** konuşmanın sahibi öğrenciye ve tenant'a ait mi
  diye doğrulanır. AI'ın verdiği ID'ye güvenilmez.

## 3.5 — AI planlama konuşması  *(3A)*

Örnek: *"Çarşamba piyano dersimi cuma 17.00'ye alabilir miyiz?"*

```
gönderen kimliği → öğrenci çözümleme → intent sınıflama → ilgili dersi bulma
→ tarih/saat çıkarımı → scheduling engine çağrısı
```

- Uygunsa atomic reschedule + onay mesajı.
- Uygun değilse `findAlternativeSlots()` ile en fazla 3 seçenek sunulur.
- *"18 olsun"* gibi devam mesajı konuşma bağlamından çözülür ve **uygunluk yeniden kontrol edilir**.
- Ders bir seriye aitse kapsam sorulur: *"Sadece bu haftayı mı, yoksa bundan sonraki tüm dersleri mi?"*
- Aynı numaraya birden fazla öğrenci bağlıysa AI tahmin etmez, **sorar**.

## 3.6 — İnsan devri ve takeover  *(3A mantık, 3B UI)*

Konuşma durumları: `AI_ACTIVE · WAITING_FOR_USER · NEEDS_COORDINATOR · HUMAN_ACTIVE · RESOLVED · FAILED`

Otomatik koordinatöre devir: `PAYMENT`, `COMPLAINT`, `TEACHER_CHANGE`, politika istisnası,
paket uyuşmazlığı, düşük confidence, birden fazla öğrenci eşleşmesi, belirsiz ders,
backend veya AI hatası, token bütçesi aşımı.

- Koordinatör ekranı: öğrenci, konu, AI özeti, ilgili ders/paket/bakiye, önerilen yanıt.
- Koordinatör devralınca `HUMAN_ACTIVE` olur; **AI otomatik mesaj gönderemez**, yalnızca taslak önerir.
- Kuyrukta bekleyen gecikmiş AI job'ı, işlenirken konuşma `HUMAN_ACTIVE` olmuşsa **iptal edilir**.
  Bu kontrol job'ın başında değil, **gönderim anında** yapılır.

## 3.7 — WhatsApp politika ve template kuralları  *(3A)*

- 24 saatlik hizmet penceresi dışında işletme kaynaklı mesajlarda **onaylı template** kullanılır.
- Template kategorileri (`utility` / `marketing` / `service`) ve conversation-based ücretlendirme
  dikkate alınır; pazarlama içerikli duyuru `utility` template'i ile gönderilemez.
- Opt-in/opt-out kaydı saklanır; `STOP` benzeri talepler **derhal** uygulanır ve
  opt-out sonrası reminder/duyuru/AI yanıtı gönderilmez.
- Her outbound mesajda delivery status ve correlation ID tutulur.
- Rate limit, 429/5xx retry + exponential backoff, dead-letter handling (ADR-0002).

---

## Gün 3 testleri

**Unit:** imza doğrulama, template eligibility hesabı (24 saat penceresi iki tarafı),
intent şema doğrulaması, telefon → öğrenci çözümleme belirsizliği.

**Integration:**
- Webhook verification, **geçersiz imza reddi**, duplicate `message.id` idempotency
- Webhook p99 < 500 ms ve AI'ın webhook içinde çağrılmadığının kanıtı
- Gönderen → öğrenci/veli çözümleme; birden fazla eşleşmede soru sorulması
- Geçersiz/yarım AI çıktısı → **sıfır mutation**, koordinatöre devir
- Başarılı reschedule; çakışmada alternatif; ikinci mesajla seçim (bağlamdan)
- Alternatif önerildikten sonra slotun dolması → revalidation reddi
- Düşük confidence / PAYMENT / COMPLAINT / TEACHER_CHANGE → handoff
- `HUMAN_ACTIVE` iken kuyrukta bekleyen AI job'ının mesaj göndermemesi
- Opt-out sonrası hiçbir outbound mesaj gönderilmemesi
- Token bütçesi aşımında AI'ın devre dışı kalması ve koordinatöre düşmesi
- Retry, delivery status, dead-letter akışı

**Security (CI):** AI'ın ürettiği başka tenant'a ait `lessonId` ile mutation = DENIED.

**Kapı:** lint · typecheck · unit · integration · RLS (CI) · concurrency (CI) ·
rapor: `docs/test-reports/gun-3.md`

## Blok bitmezse

3.1, 3.2, 3.4, 3.6 Gün 5 E2E'sinin ön koşuludur. 3.3'ün provider değişimi ve 3.7'nin
template kategori ayrımı Gün 5'e taşınabilir. Gerçek Meta entegrasyonu credential yokluğundan
yapılamadıysa mock transport ile tüm mantık test edilir ve Gün 5'te smoke test olarak tekrarlanır.
