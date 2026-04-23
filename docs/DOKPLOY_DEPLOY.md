# Fluxer Dokploy Deploy Runbook

Bu rehber Fluxer'ı küçük bir arkadaş grubu için Dokploy üzerinde canlıya almak için adım adım kurulumu anlatır. Chat, voice chat ve video streaming dahil.

> ⚠️ **Upstream uyarısı:** [fluxer_docs/self-hosting/index.mdx](../fluxer_docs/self-hosting/index.mdx) maintainer'lar veri persistence katmanı refactor'ı tamamlanana kadar self-host etmemeyi öneriyor. Küçük grup + düzenli backup ile uygulanabilir; production'a critical data yazma.

## Bu branch'te uygulanan fix'ler

Upstream `:stable` image'daki bilinen bug'lar için source build yapılıyor. Uygulanan fix'ler:

| Fix | Dosya | Issue |
|---|---|---|
| CSP fluxerstatic.com izni | [fluxer_server/src/ServiceInitializer.tsx](../fluxer_server/src/ServiceInitializer.tsx) | [#558](https://github.com/fluxerapp/fluxer/issues/558) |
| Dockerfile packages/app referansı kaldırıldı | [fluxer_server/Dockerfile](../fluxer_server/Dockerfile) | SORUN-01 |
| app-build stage'e Rust 1.93.0 + wasm-pack | [fluxer_server/Dockerfile](../fluxer_server/Dockerfile) | SORUN-02 |
| Eksik packages/* package.json COPY'ları | [fluxer_server/Dockerfile](../fluxer_server/Dockerfile) | SORUN-05 |
| pnpm versiyonu 10.26.0 → 10.29.3 | [fluxer_server/Dockerfile](../fluxer_server/Dockerfile) | SORUN-06 |
| NATS core + jetstream split (upstream convention) | [compose.yaml](../compose.yaml), [config.json](../config/config.json) | [#559](https://github.com/fluxerapp/fluxer/issues/559) |
| BASE_DOMAIN build arg ile bundle'a gömme | [fluxer_server/Dockerfile](../fluxer_server/Dockerfile) | [#559](https://github.com/fluxerapp/fluxer/issues/559) |
| LiveKit webhook api_key alanı | [config/livekit.yaml](../config/livekit.yaml) | [#559](https://github.com/fluxerapp/fluxer/issues/559) |

## Mimari

İki ayrı Dokploy Compose uygulaması:

| App | Servisler | Domain | Portlar |
|---|---|---|---|
| **fluxer-app** | fluxer_server + valkey + nats_core + nats_jetstream | `chat.<yourdomain>` | 8080 (Traefik) |
| **fluxer-livekit** | livekit | `voice.<yourdomain>` | 7880 (Traefik) + 3478/udp, 7881/tcp, 50000-50100/udp (direct) |

## 1. Önkoşullar

### VPS
- Dokploy kurulu, public IP'li VPS
- **Minimum**: 4 CPU, 8 GB RAM, 30 GB disk (build için Rust compile ~4GB bellek)
- Source build ilk defa: ~15-30 dakika. Cache ile sonrakiler: ~2-5 dakika.

### Firewall (VPS provider + VPS iptables/ufw)

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

# VAPID (web push)
npx web-push generate-vapid-keys

# LiveKit API key/secret
echo "LIVEKIT_API_KEY=APIkey$(openssl rand -hex 6)"
echo "LIVEKIT_API_SECRET=$(openssl rand -hex 32)"
```

## 3. Config dosyaları

### `config/config.json` (fluxer-app için)
[config/config.production.template.json](../config/config.production.template.json) → `config/config.json` kopyası. Tüm placeholder'ları üretilen secret'lar ve gerçek domain ile doldur.

Dikkat edilecekler:
- `services.nats.core_url` = `nats://nats_core:4222`
- `services.nats.jetstream_url` = `nats://nats_jetstream:4223`
- `services.nats.auth_token` = compose'daki `NATS_AUTH_TOKEN` env ile aynı
- `integrations.voice.*` = LiveKit key/secret, livekit.yaml ile aynı

Dosya `.gitignore`'da; commit edilmez.

### `config/livekit.yaml` (fluxer-livekit için)
[config/livekit.example.yaml](../config/livekit.example.yaml) → `config/livekit.yaml`. API key/secret ve webhook URL'yi doldur.

## 4. Commit & Push

Source build için repo commit edilmiş olmalı, Dokploy git clone yapıp build edecek.

```bash
git checkout -b deploy  # veya refactor branch üzerinde kal
git add compose.yaml compose.livekit.yaml fluxer_server/ config/ docs/DOKPLOY_DEPLOY.md .gitignore
git commit -m "feat: Dokploy source build + #558 CSP + #559 NATS split fixes"
git push origin deploy
```

`config/config.json` ve `config/livekit.yaml` `.gitignore`'da, push'lanmaz. Bunlar Dokploy'a UI üzerinden "File Mount" ile yüklenecek.

## 5. Dokploy'da LiveKit app deploy

1. Dokploy UI → Projects → Create Project: `fluxer-livekit`
2. Create Service → **Compose**
3. Compose kaynağı: Git repo → `refactor` (veya `deploy`) branch, dosya yolu: `compose.livekit.yaml`
4. **Mounts** (file mount):
   - Path: `./config/livekit.yaml`
   - Content: lokal `config/livekit.yaml` içeriği
5. **Domains** → Traefik:
   - Host: `voice.<yourdomain>`
   - Container port: `7880`
   - HTTPS: enabled (Let's Encrypt)
6. **Advanced → Ports** (Published Ports — Traefik bypass):
   - `3478:3478/udp`
   - `7881:7881/tcp`
   - `50000-50100:50000-50100/udp`
7. Deploy
8. **Doğrulama**: `curl -I https://voice.<yourdomain>/`, `nc -vzu <VPS_IP> 3478`

## 6. Dokploy'da fluxer-app deploy (source build)

1. Dokploy UI → Projects → Create Project: `fluxer-app`
2. Create Service → **Compose**
3. Compose kaynağı: Git repo → `deploy` branch, dosya yolu: `compose.yaml`
4. **Mounts**:
   - Path: `./config/config.json`
   - Content: lokal `config/config.json` içeriği
5. **Environment vars** (kritik):
   - `NATS_AUTH_TOKEN=<secrets adım 2'de üretilen nats_token>`
   - `BASE_DOMAIN=chat.<yourdomain>`
   - `PUBLIC_SCHEME=https`
   - `PUBLIC_PORT=443`
6. **Domains** → Traefik:
   - Host: `chat.<yourdomain>`
   - Service: `fluxer_server`
   - Container port: `8080`
   - HTTPS: enabled
7. Dokploy persistent volumes: `valkey_data`, `nats_jetstream_data`, `fluxer_data`
8. Deploy — ilk build ~15-30dk (Rust compile + pnpm install + rspack build). Dokploy build logs'u izle.

### İlk build hata mesajları

- **"COPY packages/*/package.json not found"** → Yeni bir package eklendi, Dockerfile'a eklenmemiş. `ls packages/`'e karşı Dockerfile'daki COPY'ları karşılaştır.
- **"rustup: command not found"** → Dockerfile'ın app-build stage'inde rustup install satırı atlandı, Read Dockerfile.
- **"wasm-pack build failed"** → Rust version mismatch, fluxer_app/rust-toolchain.toml kontrol (1.93.0).
- **"FLUXER_CONFIG must be set"** → `BASE_DOMAIN` env var build arg olarak geçirilmedi.

## 7. İlk boot doğrulama

Dokploy UI → fluxer-app → Logs → `fluxer_server`:

Beklenen log satırları:
- `KVClient connected`
- `JetStream connection established`
- `JetStream stream and consumer verified`
- `HTTP server listening on 0.0.0.0:8080`

## 8. İlk kullanıcı + manuel verify

Signup akışı → email verification maili gitmeyecek (email provider yok). Manuel verify:

```bash
docker exec -it fluxer_server sqlite3 /usr/src/app/data/db/fluxer.db
```

```sql
PRAGMA table_info(users);
UPDATE users SET email_verified = 1 WHERE email = 'you@example.com';
UPDATE users SET is_admin = 1 WHERE email = 'you@example.com';
.quit
```

## 9. Canlı smoke test

Hepsi çalışmıyorsa deploy başarısız.

### Chat
- [ ] İki tarayıcıda signup + manuel verify
- [ ] Text channel'da mesaj gönder → realtime
- [ ] Emoji reaction, reply, typing indicator
- [ ] Görsel upload

### Voice
- [ ] Voice channel'a iki tarayıcıdan katıl
- [ ] Karşılıklı ses
- [ ] `chrome://webrtc-internals/` → ICE state `connected`

### Video
- [ ] Kamera aç, screen share

### CSP / TLS
- [ ] Browser DevTools Console: **CSP violation yok** (fluxerstatic.com istekleri başarılı olmalı)
- [ ] IBM Plex fontlar yükleniyor
- [ ] Favicon görünüyor
- [ ] `curl -I https://chat.<yourdomain>/` → 200

## 10. Known Issues (deploy sonrası iteratif)

Bu bug'lar upstream server runtime bug'ları, kod incelemesi + reprodüksiyon + upstream source dive gerektirir. Bu deploy'da fix edilmedi; canlıda test edip etkili olanlara tek tek investigation açılır:

| Issue | Tespit yöntemi | İlk workaround |
|---|---|---|
| [#582](https://github.com/fluxerapp/fluxer/issues/582) — upload 30s timeout | >40MB dosya upload dene | Dosya boyutunu sınırla, büyükler için external link |
| [#885](https://github.com/fluxerapp/fluxer/issues/885) — VC timeout stuck | AFK 1 saat kal | Moderator manuel disconnect |
| [#876](https://github.com/fluxerapp/fluxer/issues/876) — choppy VC | Birkaç kişi VC | LiveKit CPU/bandwidth monitor |
| [#870](https://github.com/fluxerapp/fluxer/issues/870) — ses yok | Mikrofon test | Browser permission check |
| [#829](https://github.com/fluxerapp/fluxer/issues/829) — canary connectivity | Uzun VC seansı | LiveKit log kontrol |
| [#775](https://github.com/fluxerapp/fluxer/issues/775) — yeni user VC'yi bozuyor | 3+ kişi join | Rejoin workaround |
| [#890](https://github.com/fluxerapp/fluxer/issues/890) — screen share kalite | Paylaşım testi | Bandwidth, simulcast kontrol |
| [#906](https://github.com/fluxerapp/fluxer/issues/906) — Linux screen share | Linux user | Firefox/Chrome pipewire |
| [#810](https://github.com/fluxerapp/fluxer/issues/810) — emoji channel name | Channel adında emoji | Emoji kullanma |

## 11. Backup

```bash
# VPS cron:
0 3 * * * docker exec fluxer_server sqlite3 /usr/src/app/data/db/fluxer.db ".backup /usr/src/app/data/backup-$(date +\%Y\%m\%d).db"
```

## Sorun giderme

### fluxer_server sürekli restart
- `docker logs fluxer_server --tail 50`
- NATS connection fail → `NATS_AUTH_TOKEN` env var compose'da + config.json'da aynı mı
- Config path yanlış → mount `/usr/src/app/config/config.json` olmalı

### Build çok uzun sürüyor
- İlk build 15-30dk normal (Rust kurulumu + full pnpm install + WASM + rspack). Dokploy BuildKit cache sonraki build'lerde kullanır.
- OOM → VPS RAM düşükse swap ekle veya daha büyük VPS

### Voice bağlantı kuruluyor ama ses yok
- UDP firewall: `nc -vzu <VPS_IP> 3478`
- LiveKit log: `docker logs livekit`
- `wss://voice.<yourdomain>` signaling OK, UDP media portları block olabilir

### Image pull/build 403
- Source build kullanıyoruz, pull yok. Git clone erişim var mı kontrol et.

## Referanslar

- [Upstream self-hosting docs](../fluxer_docs/self-hosting/index.mdx)
- [Config schema](../fluxer_docs/self-hosting/configuration.mdx)
- [Bilinen deploy sorunları](./PROJECT_STRUCTURE.md) — SORUN-01..14
- [LiveKit deployment](https://docs.livekit.io/home/self-hosting/deployment/)
