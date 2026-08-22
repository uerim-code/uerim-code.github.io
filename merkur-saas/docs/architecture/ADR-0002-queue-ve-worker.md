# ADR-0002 — Queue PostgreSQL üzerinde, worker ayrı süreçte

**Durum:** Kabul edildi · **Bağlayıcı:** Ajan bu kararı değiştiremez.

## Karar

Asenkron iş kuyruğu **pg-boss** (PostgreSQL tabanlı) ile kurulur. Harici queue SaaS'ı kullanılmaz.

- `apps/worker` uzun ömürlü bir Node.js sürecidir (Railway / Fly.io / Render — tenant başına değil, sistem başına bir deployment).
- Vercel Cron yalnızca **tetikleyicidir**: periyodik olarak reminder üretim job'ını enqueue eder, işi kendisi yapmaz.
- Dead-letter kuyruğu, retry sayacı ve replay aynı veritabanında durur.

## Gerekçe

- Vercel serverless fonksiyonlarında uzun ömürlü worker yoktur; reminder ve AI işleri fonksiyon
  timeout'una sığmaz.
- İş kuyruğu ile domain verisi aynı veritabanında olduğu için "job enqueue + domain mutation"
  **tek transaction** içinde yapılabilir. Harici queue'da bu mümkün değildir ve
  "DB yazıldı ama job kaybolduğu" sınıfı hatalar üretir.
- Dead-letter replay'i incelemek için ayrı bir dashboard'a gerek yok; SQL yeterli.

## Sonuçlar

- Her job tipi `jobKey` ile idempotent olmak zorundadır. Reminder için:
  `lesson_id + reminder_rule_id + scheduled_for`.
- Retry politikası: exponential backoff, en fazla 5 deneme, sonra dead-letter.
- Worker `SIGTERM` aldığında çalışan job'ı bitirir, yeni job almaz (graceful shutdown).
- Worker'ın ayrı deploy'u Gün 5 deploy runbook'una dahildir; Vercel tek başına yeterli değildir.
- Kuyruk derinliği ve dead-letter sayısı Gün 5 alarm listesindedir.
