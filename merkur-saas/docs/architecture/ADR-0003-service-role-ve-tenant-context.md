# ADR-0003 — Service-role erişimi tek kapıdan, RLS service-role altında da çalışır

**Durum:** Kabul edildi · **Bağlayıcı:** Ajan bu kararı değiştiremez.

## Karar

1. Supabase **service-role** client'ına doğrudan erişim yasaktır. Tek giriş noktası:

```ts
withTenantContext(tenantId: string, fn: (db: Db) => Promise<T>): Promise<T>
```

Sarmalayıcı transaction açar, `SET LOCAL app.tenant_id = $1` çalıştırır, `fn`'i çağırır.

2. RLS policy'leri **yalnızca `auth.uid()`'e değil**, aynı zamanda
`current_setting('app.tenant_id', true)`'a bakar. Service-role bağlantısı da bu kontrole tabidir.
Yani service-role RLS'i tamamen bypass etmez; sadece kullanıcı kimliği yerine açıkça
beyan edilmiş tenant bağlamıyla çalışır.

3. `packages/database/src/service-client.ts` dosyasının dışından import edilmesi
ESLint `no-restricted-imports` kuralıyla engellenir. İhlal build'i kırar.

## Gerekçe

Webhook handler, worker, cron job ve migration script'leri kullanıcı session'ı olmadan çalışır ve
service-role kullanmak zorundadır. v1.0 belgesinde "service-role kullanım sınırı" isteniyordu ama
mekanizma tanımlı değildi — tenant leakage'ın en olası kaynağı tam olarak burasıdır:
unutulmuş tek bir `.eq('tenant_id', ...)` filtresi.

Tenant bağlamını connection seviyesine taşımak, filtreyi unutmayı **kod hatası değil, veri hatası olmaktan çıkarır**.

## Sonuçlar

- `withTenantContext` çağrılmadan yapılan sorgu 0 satır döner (policy eşleşmez), sessizce tüm tenant'ları döndürmez.
- Platform-admin işlemleri için ayrı ve açıkça adlandırılmış `withPlatformAdminContext` vardır;
  kullanımı audit log'a yazılır ve Gün 5 güvenlik incelemesinde tek tek gerekçelendirilir.
- Bug-hunt bölüm 1'in zorunlu kanıtı: service-role kullanan **her** kod yolu için bir cross-tenant DENIED testi.
