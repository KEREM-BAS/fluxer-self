# Fluxer Dokploy Deploy Runbook

Bu rehber Fluxer'ı küçük bir arkadaş grubu için Dokploy üzerinde canlıya almak için adım adım kurulumu anlatır. Chat, voice chat ve video streaming dahil.

> ⚠️ **Upstream uyarısı:** [fluxer_docs/self-hosting/index.mdx](../fluxer_docs/self-hosting/index.mdx) maintainer'lar veri persistence katmanı refactor'ı tamamlanana kadar self-host etmemeyi öneriyor. Küçük grup + düzenli backup ile uygulanabilir; production'a critical data yazma.

## Mimari

İki ayrı Dokploy Compose uygulaması:

| App | Servisler | Domain | Portlar |
|---|---|---|---|
| **fluxer-app** | fluxer_server + valkey + nats | `chat.<yourdomain>` | 8080 (Traefik) |
| **fluxer-livekit** | livekit | `voice.<yourdomain>` | 7880 (Traefik) + 3478/udp, 7881/tcp, 50000-50100/udp (direct) |

Kritik nokta: LiveKit UDP portları Traefik'i bypass etmeli (Traefik UDP proxy yapamaz). Dokploy "Published Ports" ile host'a direkt bind edilir.

---

## 1. Önkoşullar

### VPS
- Dokploy kurulu, public IP
- **Minimum**: 2 CPU, 4 GB RAM, 30 GB disk
- Önerilen: 4 CPU, 8 GB RAM (LiveKit + monolith + Valkey + NATS birlikte)

### Firewall (VPS provider + VPS iptables/ufw)
Açılması gereken portlar:

| Port | Protokol | Kullanım |
|---|---|---|
| 22 | TCP | SSH |
| 80 | TCP | HTTP (Let's Encrypt challenge) |
| 443 | TCP | HTTPS (Traefik) |
| 3478 | UDP | LiveKit TURN/STUN |
| 7881 | TCP | LiveKit ICE-TCP fallback |
| 50000-50100 | UDP | LiveKit RTP/RTCP media |

ufw örneği:
```bash
sudo ufw allow 22/tcp
sudo ufw allow 80,443/tcp
sudo ufw allow 3478/udp
sudo ufw allow 7881/tcp
sudo ufw allow 50000:50100/udp
```

### DNS
Bir domain'in altına iki A record:
```
chat.<yourdomain>   A   <VPS_IP>
voice.<yourdomain>  A   <VPS_IP>
```

## 2. Secrets üretimi (lokalde bir kez)

Tüm secret'ları bir password manager'a kaydet. Aynı değerler birden fazla dosyada kullanılacak.

```bash
# 8 adet 64-char hex secret
for key in media_proxy admin_base admin_oauth marketing_base \
           gateway_reload sudo_secret conn_init nats_token; do
  echo "$key=$(openssl rand -hex 32)"
done

# S3 credentials
echo "S3_ACCESS=$(openssl rand -hex 16)"
echo "S3_SECRET=$(openssl rand -hex 32)"

# VAPID (web push) — bir kez node/npx ile
npx web-push generate-vapid-keys

# LiveKit API key/secret
echo "LIVEKIT_API_KEY=APIkey$(openssl rand -hex 6)"
echo "LIVEKIT_API_SECRET=$(openssl rand -hex 32)"
```

## 3. Config dosyaları hazırla

### `config/config.json` (fluxer-app için)

[config/config.production.template.json](../config/config.production.template.json) dosyasını `config/config.json` olarak kopyala ve doldur:

- Tüm `GENERATE_A_64_CHAR_HEX_SECRET` → adım 2'den hex secret'lar
- `YOUR_S3_ACCESS_KEY` / `YOUR_S3_SECRET_KEY` → S3 credentials
- `YOUR_VAPID_PUBLIC_KEY` / `YOUR_VAPID_PRIVATE_KEY` → VAPID keypair
- `YOUR_LIVEKIT_API_KEY` / `YOUR_LIVEKIT_API_SECRET` → LiveKit credentials (livekit.yaml ile aynı)
- `chat.example.com` → gerçek chat domain'i
- `voice.example.com` → gerçek voice domain'i
- `services.nats.auth_token` → NATS_AUTH_TOKEN ile aynı (adım 2'deki `nats_token`)

**Önerilen:** Arama kullanmıyorsanız `integrations.search` bloğunu silin veya `api_key`'i boş bırakın → [NullSearchProvider](../packages/api/src/SearchFactory.tsx) graceful fallback çalışır.

**Önerilen:** Email provider yoksa `integrations.email` eklemeyin (template'de yok). Signup'ta verification emaili gönderilmeyecek; ilk kullanıcıları manuel verify edin (bkz. adım 8).

### `config/livekit.yaml` (fluxer-livekit için)

[config/livekit.example.yaml](../config/livekit.example.yaml) dosyasını `config/livekit.yaml` olarak kopyala ve:
- `<replace-with-api-key>` → LIVEKIT_API_KEY
- `<replace-with-api-secret>` → LIVEKIT_API_SECRET
- `https://chat.example.com/api/webhooks/livekit` → gerçek chat domain

İki dosya da `.gitignore`'da; commit edilmezler.

## 4. Dokploy'da LiveKit app deploy et (önce bu)

1. Dokploy UI → Projects → Create Project: `fluxer-livekit`
2. Create Service → **Compose**
3. Compose source: [compose.livekit.yaml](../compose.livekit.yaml) içeriğini yapıştır
4. **Mounts** (file mount):
   - Path: `./config/livekit.yaml`
   - Content: adım 3'teki `config/livekit.yaml` içeriği
5. **Domains** → Traefik:
   - Host: `voice.<yourdomain>`
   - Container port: `7880`
   - HTTPS: enabled (Let's Encrypt)
6. **Advanced → Ports** (Published Ports — Traefik bypass):
   - `3478:3478/udp`
   - `7881:7881/tcp`
   - `50000-50100:50000-50100/udp`
7. Deploy
8. **Doğrulama**:
   - `curl -I https://voice.<yourdomain>/` → HTTP 200/404 response (LiveKit HTTP endpoint'i var)
   - `nc -vzu <VPS_IP> 3478` → `succeeded` çıktısı

## 5. Dokploy'da fluxer-app deploy et

1. Dokploy UI → Projects → Create Project: `fluxer-app`
2. Create Service → **Compose**
3. Compose source: repo kökündeki [compose.yaml](../compose.yaml) içeriğini yapıştır (NATS servisi ile birlikte güncel hali)
4. **Mounts**:
   - Path: `./config/config.json`
   - Content: adım 3'teki `config/config.json` içeriği
5. **Environment vars**:
   - `NATS_AUTH_TOKEN=<adım 2'deki nats_token>` (compose'da `${NATS_AUTH_TOKEN:?}` olarak referanslı, zorunlu)
6. **Domains** → Traefik:
   - Host: `chat.<yourdomain>`
   - Service: `fluxer_server`
   - Container port: `8080`
   - HTTPS: enabled
7. Dokploy volume binding: `valkey_data`, `nats_data`, `fluxer_data` otomatik yaratılır (persistent volume)
8. Health check: compose'daki `/_health` zaten tanımlı
9. Deploy

## 6. İlk boot doğrulama

Dokploy UI → fluxer-app → Logs → `fluxer_server`:

Beklenen log satırları:
- `KVClient connected`
- `JetStream connection established`
- `JetStream stream and consumer verified`
- `HTTP server listening on 0.0.0.0:8080`

### Sık hata: `JetStream connection failed`
Sebep: `NATS_AUTH_TOKEN` env var'ı compose'da set edilmemiş **veya** `config.json`'daki `services.nats.auth_token` ile uyuşmuyor.
Çözüm: İki yerde aynı değer mi kontrol et, app'i redeploy et.

### Sık hata: `Config not loaded`
Sebep: `config.json` file mount path yanlış.
Çözüm: Dokploy mount config'inde path `/usr/src/app/config/config.json` (Dockerfile `ENV FLUXER_CONFIG=/usr/src/app/config/config.json` bekliyor) olmalı.

## 7. Web arayüzü testi

`https://chat.<yourdomain>` → Fluxer login ekranı yüklenmeli.

Browser DevTools → Network → WS sekmesi: `wss://chat.<yourdomain>/gateway` upgrade başarılı mı?

## 8. İlk kullanıcı + manuel verify

Signup akışı → email girdiğinde verification maili gitmeyecek (email provider yok). Manuel verify:

```bash
# Dokploy VPS'te SSH ile bağlan:
docker exec -it fluxer_server sqlite3 /usr/src/app/data/db/fluxer.db
```

SQLite shell'de:
```sql
-- Kolonu doğrula (şema değişebilir):
PRAGMA table_info(users);

-- Email doğrulandı olarak işaretle:
UPDATE users SET email_verified = 1 WHERE email = 'you@example.com';

-- (Gerekirse) Admin yetkisi ver:
UPDATE users SET is_admin = 1 WHERE email = 'you@example.com';

.quit
```

Login ol, bir community/sunucu oluştur, bir text channel + bir voice channel ekle.

## 9. Canlı smoke test

Hepsi çalışmıyorsa deploy başarısız.

### Chat
- [ ] İki tarayıcıda (normal + incognito) signup + manuel verify
- [ ] Text channel'da mesaj gönder → karşı taraf anında görüyor mu
- [ ] Emoji reaction, reply, typing indicator çalışıyor mu
- [ ] Görsel upload (media proxy testi)

### Voice
- [ ] Voice channel'a iki tarayıcıdan katıl
- [ ] Karşılıklı ses duyuluyor mu
- [ ] `chrome://webrtc-internals/` → ICE state `connected`, port range 50000-50100
- [ ] **Ses yok ama bağlı** → UDP firewall kapalı: `nc -vzu <VPS_IP> 3478` ve `50005` (range içi) test

### Video
- [ ] Voice channel içinde kamera aç → karşıda görüntü
- [ ] Screen share → paylaşım çalışıyor

### TLS / Network
- [ ] `curl -I https://chat.<yourdomain>/` → 200, valid cert
- [ ] `curl -I https://voice.<yourdomain>/` → LiveKit response
- [ ] Browser console'da CORS veya mixed-content error yok

## 10. Backup (deploy sonrası hemen ayarla)

Fluxer'ın tüm durumu `fluxer_data` volume'unda:
- `data/db/fluxer.db` — SQLite tüm veritabanı (mesajlar, kullanıcılar, sunucular)
- `data/storage/` — dosya upload'ları
- `data/queue/` — worker queue state

Günlük cron ile volume snapshot:
```bash
# VPS'te:
0 3 * * * docker exec fluxer_server sqlite3 /usr/src/app/data/db/fluxer.db ".backup /usr/src/app/data/backup-$(date +\%Y\%m\%d).db"
# Daha sonra rsync/rclone ile dış storage'a kopyala.
```

## Sorun giderme

### fluxer_server sürekli restart ediyor
- `docker logs fluxer_server --tail 50` ile hatayı gör
- En sık: NATS connection fail, config.json path yanlış, secret eksik

### Voice çalışmıyor, "connection failed"
- LiveKit logs: `docker logs livekit --tail 50`
- `voice.<domain>` DNS doğru mu, `wss://` bağlantı işaretini tarayıcıda aç
- LiveKit'in public key'i config.json'daki ile aynı mı

### "NATS_AUTH_TOKEN not set" hatası
- Dokploy Environment Variables sekmesinde bu env var set edilmeli, yoksa compose başlamaz

### Image pull 403/404
- `ghcr.io/fluxerapp/fluxer-server:stable` public mi kontrol et: `docker pull ghcr.io/fluxerapp/fluxer-server:stable` lokalde çalışıyor mu
- Private ise `FLUXER_SERVER_IMAGE` env var ile kendi registry override et

## Referanslar

- [Upstream self-hosting overview](../fluxer_docs/self-hosting/index.mdx)
- [Config schema reference](../fluxer_docs/self-hosting/configuration.mdx)
- [Bilinen sorunlar](./PROJECT_STRUCTURE.md) — SORUN-01..14
- [LiveKit deployment docs](https://docs.livekit.io/home/self-hosting/deployment/)
