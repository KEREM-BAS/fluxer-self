# Fluxer Self-Host — Proje Yapısı Analiz Raporu

**Kaynak:** `refactor` branch'i, commit `bfa4bfe9`
**Analiz tarihi:** 2026-04-23

---

## BÖLÜM 1 — Repo Üst Düzey Haritası

### 1.1 Root dosya/klasör envanteri

| Öğe | Tip | Ne işe yarar |
|---|---|---|
| `.devcontainer/` | dir | VS Code devcontainer tanımı (Dockerfile + docker-compose + bootstrap scriptleri). |
| `.dockerignore` | file | Root bağlamında Docker build'inin görmezden geleceği dosya listesi. |
| `.editorconfig` | file | Editör için satır sonu ve indent ayarları. |
| `.envrc` | file | direnv hook'u — devenv shell'i otomatik yükler. |
| `.github/` | dir | 23 workflow (ci, deploy-*, release-*, migrate-cassandra, sync-*), issue/PR template'leri. |
| `.gitattributes` | file | Git attribute tanımı (içerik minimal). |
| `.gitignore` | file | Kapsamlı ignore listesi; `/config/config.json`, `dist`, `_build`, `generated`, `fluxer.env`, `secrets.env` dahil. |
| `.gitmodules` | file | **Boş** (0 byte). fluxer_static eski submodule olarak vendor edilmiş (commit `bfa4bfe9`). |
| `.ignore` | file | ripgrep/ag için ek ignore paternleri. |
| `.npmrc` | file | Sadece `update-notifier=false`. |
| `.nvmrc` | file | Node `24`. |
| `.prettierignore` | file | Prettier'in dokunmayacağı dosyalar. |
| `.tool-versions` | file | **Boş dosya** (asdf için ayrılmış ama hiç entry yok). |
| `.vscode/` | dir | Debug launch.json, editor settings, önerilen extensions. |
| `CODE_OF_CONDUCT.md` | file | Topluluk davranış kuralları. |
| `CONTRIBUTING.md` | file | Katkı rehberi. |
| `LICENSE` | file | GNU AGPL v3. |
| `LICENSING.md` | file | Ticari lisanslama ve CLA notları. |
| `README.md` | file | Tanıtım + self-hosting linki (self-host bölümü "TBD" diyor). |
| `SECURITY.md` | file | Vulnerability raporlama adresi. |
| `biome.json` | file | Biome (lint+format) konfigürasyonu. |
| `compose.yaml` | file | Üst seviye Docker Compose — `fluxer_server`, `valkey`, `meilisearch`/`elasticsearch`, `livekit`. Profile ile opsiyonel servisler. |
| `config/` | dir | `config.schema.json`, `config.dev.template.json`, `config.production.template.json`, `config.test.json`, `livekit.example.yaml`. |
| `dev/` | dir | Dev ortam için data ve Caddyfile. |
| `devenv.lock` / `devenv.nix` / `devenv.yaml` | file | devenv (Nix tabanlı) dev ortamı tanımı. Tüm dev process'leri burada. |
| `flake.lock` | file | Nix flake lock. |
| `fluxer_admin/` | dir | Admin paneli (Hono server). |
| `fluxer_api/` | dir | API servisi + Cassandra migration scriptleri. |
| `fluxer_app/` | dir | Ana web istemcisi (React + rspack + Lingui + **Rust/WASM** via `crates/libfluxcore`). |
| `fluxer_app_proxy/` | dir | App statik varlıklarını servis eden proxy. |
| `fluxer_desktop/` | dir | Electron desktop app (workspace'te değil, `!fluxer_desktop`). |
| `fluxer_devops/` | dir | Compose parçaları: cassandra, valkey, nats, signoz, caddy, livekitctl, ghost_blog, weblate, clamav, nginx, turborepo_cache. |
| `fluxer_docs/` | dir | Mintlify docs (workspace'te değil, `!fluxer_docs`). |
| `fluxer_gateway/` | dir | **Erlang/OTP 28** WebSocket gateway (cowboy + enats). |
| `fluxer_integration/` | dir | Integration test harness. |
| `fluxer_marketing/` | dir | Marketing sitesi (Hono). |
| `fluxer_media_proxy/` | dir | Medya proxy servisi; `data/model.onnx` (NSFW detection) içerir. |
| `fluxer_relay/` | dir | Erlang relay (federasyon tarafı). |
| `fluxer_relay_directory/` | dir | Relay directory servisi (Hono + zod + OpenAPI). |
| `fluxer_server/` | dir | **Self-host için tekil umbrella servis** — admin/api/app_proxy/media_proxy/s3'u tek process'te toplar. |
| `fluxer_static/` | dir | Avatar, emoji (~4025 dosya), font, badge, libs vs. statik varlıklar. Vendor edilmiş (artık submodule değil). |
| `knip.json` | file | Knip (dead code) konfigürasyonu. |
| `media/` | dir | README için logo, ekran görüntüsü. |
| `package.json` | file | Root workspace; turbo/biome/vitest devDeps. `packageManager: pnpm@10.29.3`. |
| `packages/` | dir | 48 paket (`admin`, `api`, `cache`, `config`, vb.). |
| `patches/` | dir | `@phosphor-icons/react` için pnpm patch. |
| `pnpm-lock.yaml` | file | 729 951 byte (root lockfile). |
| `pnpm-workspace.yaml` | file | Workspace tanımı + catalog (paylaşılan paket versiyonları) + allowBuilds. |
| `scripts/` | dir | `run_dev.sh`, `dev_bootstrap.sh`, `dev_gateway.sh`, `dev_fluxer_app.sh`, `dev_css_watch.sh`, `watch_css.sh`, `dev_process_entry.sh`, `ci/` (Python CI iş akışları). |
| `tsconfig.json` | file | Root TS config (workspace root için). |
| `tsconfigs/` | dir | Paylaşılan TS config base'leri. |
| `turbo.json` | file | Turbo task grafik tanımı; `globalEnv: ["FLUXER_CONFIG"]`. |

### 1.2 Monorepo türü

**pnpm workspaces + Turborepo.**

Kanıt:
- `pnpm-workspace.yaml:1-15` — workspace tanımı (`packages/*`, `fluxer_admin`, `fluxer_api`, `fluxer_app`, `fluxer_app_proxy`, `fluxer_gateway`, `fluxer_integration`, `fluxer_marketing`, `fluxer_media_proxy`, `fluxer_relay_directory`, `fluxer_server`; `!fluxer_docs`, `!fluxer_desktop` hariç).
- `package.json:35` — `"turbo": "^2.8.3"`.
- `turbo.json:1` — Turborepo schema referansı.

### 1.3 Package manager

**pnpm 10.29.3** (root) — `package.json:37`.

**UYARI:** Tüm production Dockerfile'ları `pnpm@10.26.0` kullanıyor (ör. [fluxer_server/Dockerfile:10](fluxer_server/Dockerfile), [fluxer_admin/Dockerfile:10](fluxer_admin/Dockerfile), [fluxer_api/Dockerfile:10](fluxer_api/Dockerfile)). Lockfile 10.29.3 ile üretilmişse, 10.26.0 `--frozen-lockfile` reddedebilir.

### 1.4 Node.js versiyonu

- `.nvmrc:1` → `24`
- `.tool-versions` → **boş** (entry yok)
- `fluxer_server/package.json:38-40` → `"engines": { "node": ">=24.0.0" }`
- Tüm Dockerfile base image'ları: `node:24-bookworm-slim` veya `node:24-trixie-slim`.

### 1.5 Diğer runtime gereksinimleri

| Runtime | Nerede | Kanıt |
|---|---|---|
| **Rust 1.93.0** (wasm32-unknown-unknown target) | `fluxer_app/crates/libfluxcore` | [fluxer_app/rust-toolchain.toml:1-2](fluxer_app/rust-toolchain.toml), [fluxer_app/crates/libfluxcore/Cargo.toml](fluxer_app/crates/libfluxcore/Cargo.toml), `devenv.nix:131-133` |
| **Erlang/OTP 28** | `fluxer_gateway`, `fluxer_relay` | [fluxer_gateway/Dockerfile:1](fluxer_gateway/Dockerfile), [fluxer_relay/Dockerfile:1](fluxer_relay/Dockerfile), `rebar.config` |
| **Python 3** (CI scripts) | `scripts/ci/workflows/*.py` | [.github/workflows/ci.yaml:25](.github/workflows/ci.yaml) |
| **Go 1.24** | `fluxer_devops/livekitctl` | `devenv.nix:130` |
| **wasm-pack 0.14.0** | fluxer_app WASM build | `pnpm-workspace.yaml:201`, `fluxer_app/package.json:38` |

`go.mod`, `pyproject.toml`: sadece `fluxer_devops/livekitctl`, `scripts/ci/`. Root'ta yok.

---

## BÖLÜM 2 — Paket/Modül Envanteri

`packages/` altında 48 alt klasör. Rol, `package.json` açıklaması veya kod girişinden alındı; açıklama yoksa "BULUNAMADI" işaretlendi.

| Paket Adı | Path | Rol | Dil/Framework | Deps |
|---|---|---|---|---|
| `@fluxer/admin` | `packages/admin` | Admin paneli (Hono app + React). | TS/Hono/React | 40+ |
| `@fluxer/api` | `packages/api` | Ana API layer; 90+ alt domain (guild, channel, message, voice, auth, oauth2, webhook, csam, ...). | TS/Hono | 43 (ana) |
| `@fluxer/app_proxy` | `packages/app_proxy` | SPA + statik varlık proxy katmanı (CSP, CDN). | TS/Hono | - |
| `@fluxer/cache` | `packages/cache` | In-memory cache primitif'leri. | TS | - |
| `@fluxer/captcha` | `packages/captcha` | hCaptcha/Turnstile doğrulama sarmalayıcı. | TS | - |
| `@fluxer/cassandra` | `packages/cassandra` | Cassandra driver wrapper + query builder. | TS | - |
| `@fluxer/config` | `packages/config` | Config loader (JSON→Zod), schema bundler. | TS/ajv/zod | - |
| `@fluxer/constants` | `packages/constants` | Sabitler (OAUTH2_APPLICATION_ID vb.). | TS | - |
| `@fluxer/date_utils` | `packages/date_utils` | Tarih yardımcıları. | TS | - |
| `@fluxer/elasticsearch_search` | `packages/elasticsearch_search` | ES tabanlı search adapter'ları (user/guild/message/audit/report). | TS | - |
| `@fluxer/email` | `packages/email` | E-posta gönderim + template'ler + i18n. | TS/nodemailer | - |
| `@fluxer/errors` | `packages/errors` | Tipli hata sınıfları + i18n. | TS | - |
| `@fluxer/geo_utils` | `packages/geo_utils` | Coğrafi mesafe hesaplamaları. | TS | - |
| `@fluxer/geoip` | `packages/geoip` | GeoIP veritabanı (maxmind). | TS | - |
| `@fluxer/hono` | `packages/hono` | Hono middleware + server factory. | TS/Hono | - |
| `@fluxer/hono_types` | `packages/hono_types` | Ortak Hono env tipleri. | TS | - |
| `@fluxer/http_client` | `packages/http_client` | Tipli HTTP client soyutlaması. | TS | - |
| `@fluxer/i18n` | `packages/i18n` | i18n type generator + lingui yardımcıları. | TS | - |
| `@fluxer/initialization` | `packages/initialization` | Servis init orkestrasyonu. | TS | - |
| `@fluxer/ip_utils` | `packages/ip_utils` | IP parse ve CIDR kontrolü. | TS | - |
| `@fluxer/kv_client` | `packages/kv_client` | Valkey/Redis client (ioredis) + pipeline + subscription. | TS/ioredis | - |
| `@fluxer/limits` | `packages/limits` | Kullanım limitleri (upload, guild count, vs.). | TS | - |
| `@fluxer/list_utils` | `packages/list_utils` | List işleme yardımcıları. | TS | - |
| `@fluxer/locale` | `packages/locale` | Locale çözümleme. | TS | - |
| `@fluxer/logger` | `packages/logger` | pino sarmalayıcı. | TS/pino | - |
| `@fluxer/markdown_parser` | `packages/markdown_parser` | Mesaj markdown parser. | TS | - |
| `@fluxer/marketing` | `packages/marketing` | Marketing sitesi kaynak + i18n. | TS/React | - |
| `@fluxer/media_proxy` | `packages/media_proxy` | Medya proxy Hono app. | TS/Hono | - |
| `@fluxer/media_proxy_utils` | `packages/media_proxy_utils` | Medya işleme yardımcıları. | TS | - |
| `@fluxer/meilisearch_search` | `packages/meilisearch_search` | Meilisearch adapter'ları (user/guild/message/audit/report). | TS/meilisearch | - |
| `@fluxer/mime_utils` | `packages/mime_utils` | MIME tespit yardımcıları. | TS | - |
| `@fluxer/nats` | `packages/nats` | JetStreamConnectionManager. | TS/nats.js | - |
| `@fluxer/number_utils` | `packages/number_utils` | Sayı formatlama. | TS | - |
| `@fluxer/oauth2` | `packages/oauth2` | OAuth2 server yardımcıları. | TS | - |
| `@fluxer/openapi` | `packages/openapi` | OpenAPI doc generator. | TS | - |
| `@fluxer/queue` | `packages/queue` | Queue servisi. | TS | - |
| `@fluxer/rate_limit` | `packages/rate_limit` | Rate limit middleware. | TS | - |
| `@fluxer/s3` | `packages/s3` | S3 adapter (aws-sdk v3) + yerel S3 server. | TS/aws-sdk | - |
| `@fluxer/schema` | `packages/schema` | Domain zod schemas (channel, message, guild, relay, ...). | TS/zod | - |
| `@fluxer/sentry` | `packages/sentry` | Sentry init. | TS | - |
| `@fluxer/sms` | `packages/sms` | SMS gönderim. | TS | - |
| `@fluxer/snowflake` | `packages/snowflake` | Snowflake ID generator. | TS | - |
| `@fluxer/telemetry` | `packages/telemetry` | OpenTelemetry sarmalayıcı. | TS/OTel | - |
| `@fluxer/time` | `packages/time` | Zaman abstraction. | TS | - |
| `@fluxer/ui` | `packages/ui` | Paylaşılan React UI primitive'leri. | TS/React | - |
| `@fluxer/validation` | `packages/validation` | Input validation (validator.js). | TS | - |
| `@fluxer/virus_scan` | `packages/virus_scan` | ClamAV entegrasyonu. | TS | - |
| `@fluxer/worker` | `packages/worker` | Worker task framework. | TS | - |

---

## BÖLÜM 3 — Servis ve Entry Point'ler

### 3.1 Deploy edilebilir servisler

| Servis | Entry point | Default port | Build | Start |
|---|---|---|---|---|
| **fluxer_server** (monolith) | [fluxer_server/src/startServer.tsx](fluxer_server/src/startServer.tsx) | `8080` (env `FLUXER_SERVER_PORT`, [fluxer_server/Dockerfile:182](fluxer_server/Dockerfile); config default `8772` [server.json:15](packages/config/src/schema/defs/services/server.json)) | `tsx` runtime (no build), typecheck ile doğrulanır | `tsx src/startServer.tsx` ([fluxer_server/package.json:10](fluxer_server/package.json)) |
| **fluxer_api** | [fluxer_api/src/AppEntrypoint.tsx](fluxer_api/src/AppEntrypoint.tsx) | `8080` ([fluxer_api/Dockerfile:87](fluxer_api/Dockerfile)) | — (tsx runtime) | `tsx src/AppEntrypoint.tsx` ([fluxer_api/package.json:7](fluxer_api/package.json)) |
| **fluxer_api** (worker) | [fluxer_api/src/WorkerEntrypoint.tsx](fluxer_api/src/WorkerEntrypoint.tsx) | yok | — | `tsx src/WorkerEntrypoint.tsx` |
| **fluxer_admin** | `fluxer_admin/src/index.tsx` | `8080` (env `FLUXER_ADMIN_PORT`, [fluxer_admin/Dockerfile:75](fluxer_admin/Dockerfile)) | `pnpm build:css` | `pnpm start` |
| **fluxer_app_proxy** | [fluxer_app_proxy/src/index.tsx](fluxer_app_proxy/src/index.tsx) | `8080` (env `PORT`, [fluxer_app_proxy/Dockerfile:65](fluxer_app_proxy/Dockerfile)) | — | `tsx src/index.tsx` ([fluxer_app_proxy/package.json:7](fluxer_app_proxy/package.json)) |
| **fluxer_marketing** | `fluxer_marketing/src/index.tsx` | `8080` (env `FLUXER_MARKETING_PORT`, [fluxer_marketing/Dockerfile:75](fluxer_marketing/Dockerfile)) | `pnpm build:css` | `pnpm start` |
| **fluxer_media_proxy** | `fluxer_media_proxy/src/index.tsx` | `8080` (env `PORT`, [fluxer_media_proxy/Dockerfile:79](fluxer_media_proxy/Dockerfile)) | — | `pnpm start` |
| **fluxer_gateway** (Erlang) | `fluxer_gateway/src/fluxer_gateway.app.src` + `gateway/*` | `8080`, `8081` exposed ([fluxer_gateway/Dockerfile:46](fluxer_gateway/Dockerfile)); config default `8771` [gateway.json:11](packages/config/src/schema/defs/services/gateway.json) | `rebar3 as prod release` | `/opt/fluxer_gateway/bin/docker_entrypoint.sh` |
| **fluxer_relay** (Erlang) | `fluxer_relay/src/` | `8080`, `8081` exposed ([fluxer_relay/Dockerfile:40](fluxer_relay/Dockerfile)) | `rebar3 as prod release` | `/opt/fluxer_relay/bin/fluxer_relay foreground` |
| **fluxer_relay_directory** | `fluxer_relay_directory/src/index.tsx` | `8080` ([fluxer_relay_directory/Dockerfile:53](fluxer_relay_directory/Dockerfile)) | — | `tsx src/index.tsx` |

**Not:** `fluxer_server` healthcheck'i `/_health` endpoint'inde ([fluxer_server/Dockerfile:196-197](fluxer_server/Dockerfile)). HealthCheck şeması [fluxer_server/src/HealthCheck.tsx:31-45](fluxer_server/src/HealthCheck.tsx): `kv`, `s3`, `jetstream`, `mediaProxy`, `admin`, `api`, `app` servis sağlık durumları.

### 3.2 Frontend

- **Framework:** React 19.2.4 + rspack 1.7.5 ([fluxer_app/package.json:118-119](fluxer_app/package.json))
- **Build output:** `fluxer_app/dist/` ([fluxer_app/package.json:18](fluxer_app/package.json) — `rspack build --mode production`, sonra `build-sw.mjs`)
- **Tip:** **SPA** (rspack ile tek sayfa + service worker; `fluxer_app/src/service_worker/Register.tsx`)
- **Backend iletişimi:**
  - REST: `@fluxer/api` HTTP routes üstünden
  - WebSocket: `fluxer_gateway` ile (Erlang) — mesaj/presence/typing/voice state
  - WebPush: VAPID (`fluxer_gateway/src/push/push_sender.erl`)

### 3.3 WebAssembly/Rust

Evet, [fluxer_app/crates/libfluxcore/Cargo.toml](fluxer_app/crates/libfluxcore/Cargo.toml):
- `crate-type = ["cdylib"]`
- `wasm-bindgen 0.2`, `image 0.25.9` (jpeg/png/webp/avif), `gif 0.13`, `png 0.17`, `ruzstd 0.7`
- Build komutu: `wasm-pack build --target web --out-dir ../../pkgs/libfluxcore --release` ([fluxer_app/package.json:38](fluxer_app/package.json))
- `pnpm build` önce `wasm:codegen` çalıştırır ([fluxer_app/package.json:18](fluxer_app/package.json)).

---

## BÖLÜM 4 — Dockerfile Analizi

Toplam 13 Dockerfile bulundu (node_modules hariç):

### 4.1 — `fluxer_server/Dockerfile` (self-host hedefi)

- **Base image(ler):**
  - [line 6](fluxer_server/Dockerfile): `node:24-trixie-slim` AS base
  - [line 80](fluxer_server/Dockerfile): `erlang:28-slim` AS gateway-build
  - [line 108](fluxer_server/Dockerfile): `FROM deps AS app-build`
  - [line 117](fluxer_server/Dockerfile): `node:24-trixie-slim` (final production)
- **Multi-stage:** 5 stage — `base` → `deps` → `build`, paralel `gateway-build`, paralel `app-build`, son `production`. Gateway ve fluxer_app paralel derlenip final stage'de birleşiyor.
- **COPY satırları — paket doğrulaması ([fluxer_server/Dockerfile:26-57](fluxer_server/Dockerfile)):**

| Dockerfile COPY | Diskte var mı? |
|---|---|
| `packages/admin` | ✅ |
| `packages/api` | ✅ |
| `packages/app` | ❌ **YOK — Dockerfile bozuk** |
| `packages/cache` | ✅ |
| `packages/captcha` | ✅ |
| `packages/cassandra` | ✅ |
| `packages/config` | ✅ |
| `packages/constants` | ✅ |
| `packages/email` | ✅ |
| `packages/errors` | ✅ |
| `packages/hono` | ✅ |
| `packages/hono_types` | ✅ |
| `packages/initialization` | ✅ |
| `packages/ip_utils` | ✅ |
| `packages/logger` | ✅ |
| `packages/marketing` | ✅ |
| `packages/media_proxy` | ✅ |
| `packages/oauth2` | ✅ |
| `packages/queue` | ✅ |
| `packages/rate_limit` | ✅ |
| `packages/s3` | ✅ |
| `packages/sentry` | ✅ |
| `packages/sms` | ✅ |
| `packages/snowflake` | ✅ |
| `packages/telemetry` | ✅ |
| `packages/ui` | ✅ |
| `packages/validation` | ✅ |
| `packages/virus_scan` | ✅ |
| `packages/worker` | ✅ |
| `packages/http_client` | ✅ |
| `packages/schema` | ✅ |

**Diskte var ama Dockerfile'da COPY edilmeyen paketler (17):** `date_utils`, `elasticsearch_search`, `geo_utils`, `geoip`, `i18n`, `kv_client`, `limits`, `list_utils`, `locale`, `markdown_parser`, `media_proxy_utils`, `meilisearch_search`, `mime_utils`, `nats`, `number_utils`, `openapi`, `time`. Bunların bir kısmı (`kv_client`, `nats`) [fluxer_server/package.json:24,27](fluxer_server/package.json) içinde doğrudan bağımlılık; `@fluxer/api` transitive olarak neredeyse tümünü kullanıyor.

- **Eksik sistem bağımlılıkları:**
  - `deps` stage (line 12-17): `curl, python3, make, g++` — ✅ (native build için yeterli)
  - `app-build` stage (line 108-115): Sadece `deps` base'i kullanıyor. **Rust toolchain veya wasm-pack YOK.** Buna rağmen `pnpm build` → `wasm:codegen` → `wasm-pack build` çalıştırılıyor. **BUG.**
  - `gateway-build` stage (line 86-95): `git, curl, make, gcc, g++, libc6-dev, ca-certificates, gettext-base` — ✅
- **Build command:**
  - `deps`: `pnpm install --frozen-lockfile` (line 60), `pnpm approve-builds msgpackr-extract@3.0.3 @parcel/watcher@2.5.6` (line 62), `pnpm rebuild msgpackr-extract @parcel/watcher` (line 64)
  - `build`: `pnpm --filter @fluxer/config generate` (line 71), `pnpm --filter @fluxer/marketing build:css` (line 74), `cd fluxer_server && pnpm typecheck` (line 78)
  - `gateway-build`: `envsubst` ile sys.config + vm.args üret, `rebar3 as prod release` (line 104-106)
  - `app-build`: `cd fluxer_app && pnpm build` (line 115) — **Rust'sız çalışmaz**
- **Final stage'de çalışan komut:** `ENTRYPOINT ["pnpm", "start"]` (line 199) — `fluxer_server/package.json:10`'daki `tsx src/startServer.tsx`'i çalıştırır.
- **EXPOSE:** `8080` (line 176)
- **INCLUDE_NSFW_ML:** `false` default (line 164). `true` yapılırsa `fluxer_media_proxy/data/model.onnx` kopyalanır. Dosya repoda var.
- **Mount noktaları:** `/usr/src/app/data/{storage,db}`, `/opt/data`, `/data/{s3,sqlite,queue}` (line 153-162) — root:root'a chown ediliyor, fakat final image'da `USER` değişmiyor → **container root olarak çalışır**.

### 4.2 — `fluxer_server/Dockerfile.dev`

Sadece gateway derler (build stage) ve Node base'e kopyalar (line 43). Bu dev container'ında kaynak kod bind mount ediliyor, bu yüzden TS kaynakları COPY'lenmiyor (fluxer_integration/docker/compose.yaml'da kullanılıyor).

### 4.3 — `fluxer_admin/Dockerfile`

- Base: `node:24-bookworm-slim`
- Multi-stage: `base` → `deps` → `build` → `prod-deps` → final (5)
- **COPY doğrulaması:** Tüm `packages/` bulk COPY ediliyor (line 16, 36) — P0 problem yok.
- `RUN pnpm build:css` (line 30) — `packages/admin/public/static/app.css` üretir (bu klasör kaynakta yok; `build:css` oluşturur; .gitignore'da [line 84](/.gitignore)).
- Final: `USER nobody`, `EXPOSE 8080`, `CMD ["pnpm", "start"]` ✅

### 4.4 — `fluxer_api/Dockerfile`

- Base: `node:24-bookworm-slim`
- Multi-stage: `base` → `generator` → `deps` → final (4)
- Sistem bağımlılıkları: `build-essential, gcc, libssl-dev, pkg-config, openssl, libvips-dev, libsqlite3-dev, python3` — ✅ (sharp + argon2 + sqlite için)
- Final runtime: `libimage-exiftool-perl, libvips, libsqlite3-0` — ✅
- `USER nobody`, `EXPOSE 8080`, `CMD ["pnpm", "start"]` ✅

### 4.5 — `fluxer_api/scripts/Dockerfile.cassandra-migrate`

- [line 7](fluxer_api/scripts/Dockerfile.cassandra-migrate): `package.json`'ı `echo` ile inline oluşturuyor — minimal. `cassandra-driver@4.8.0, tsx@4.21.0` bağımlılıkları sabit.
- **UYARI:** `fluxer_api/tsconfig.json` kopyalanıyor (line 11) ama bu tsconfig `tsconfigs/base.json`'u extend ediyor olabilir — tsconfig'in bağımsız çalışıp çalışmayacağını manuel doğrulayın.

### 4.6 — `fluxer_app_proxy/Dockerfile`

- `FROM node:24-bookworm-slim` — `base` → `generator` → `deps` → final
- **CRITICAL:** [line 57](fluxer_app_proxy/Dockerfile): `COPY fluxer_app/dist/ ./assets/`
  - Diskte `fluxer_app/dist/` **YOK** (ls: "No such file or directory"). Gitignore'da [line 43-44](/.gitignore) `!fluxer_app/dist/` override var ama dizin yok.
  - Bu Dockerfile **ancak fluxer_app önce build edilmişse** çalışır. Tek başına `docker build` edilince fail eder.
- `USER nobody`, healthcheck `/_health`, `CMD ["pnpm", "start"]`

### 4.7 — `fluxer_gateway/Dockerfile`

- `FROM erlang:28-slim` iki stage (build + final)
- [line 25](fluxer_gateway/Dockerfile): `envsubst < config/sys.config.template > config/sys.config` (vm.args.template envsubst edilmiyor — rebar `vm.args.src`'i direkt kullanıyor; [rebar.config:24](fluxer_gateway/rebar.config)).
- Sistem: `git, curl, make, gcc, g++, libc6-dev, gettext-base, ca-certificates` ✅
- `USER fluxer`, `EXPOSE 8080 8081`, `ENTRYPOINT ["/opt/fluxer_gateway/bin/docker_entrypoint.sh"]`
- **Bağımsız inşa ediliyor** — fluxer_server/Dockerfile içindeki gateway-build ile mantık aynı.

### 4.8 — `fluxer_marketing/Dockerfile`

- Standart node:24-bookworm-slim multi-stage. Tüm packages/ bulk COPY. `USER nobody`, `EXPOSE 8080`, `CMD ["pnpm", "start"]`.

### 4.9 — `fluxer_media_proxy/Dockerfile`

- Multi-stage generator + deps + final
- Sistem: `build-essential, gcc, g++, make, python3, pkg-config, libssl-dev, libvips-dev` (build); `ffmpeg, libvips, libgomp1, libatomic1` (final)
- [line 73](fluxer_media_proxy/Dockerfile): `cp data/model.onnx /opt/data/model.onnx` — NSFW modeli her zaman kopyalanıyor (fluxer_server Dockerfile'dan farklı olarak flag yok).
- `USER nobody`, `EXPOSE 8080`, `CMD ["pnpm", "start"]`

### 4.10 — `fluxer_relay/Dockerfile`

- Erlang 28 iki stage. `rebar3 as prod release`, `USER fluxer`, `EXPOSE 8080 8081`, `ENTRYPOINT ["/opt/fluxer_relay/bin/fluxer_relay", "foreground"]`.

### 4.11 — `fluxer_relay_directory/Dockerfile`

- `FROM node:24-bookworm-slim` → deps → final
- **BUG:** [line 11-14](fluxer_relay_directory/Dockerfile) sadece `packages/hono`, `packages/schema` COPY ediyor. [fluxer_relay_directory/package.json:14-17](fluxer_relay_directory/package.json) ise `@fluxer/config` ve `@fluxer/openapi`'ye bağımlı. `pnpm install --frozen-lockfile --prod --filter fluxer-relay-directory...` çalıştırılsa da, workspace paket dizinleri yoksa kurulum fail eder veya bağımlılık eksik kurulur.

### 4.12 — `.devcontainer/Dockerfile`

Debian-13 base üstüne Erlang 28 manuel kopyalanıyor + rebar3 + caddy + process-compose + uv. Go/Rust/Python/Node devcontainer feature'ları ile geliyor. Üretim değil.

### 4.13 — `fluxer_devops/cassandra/Dockerfile.backup`

`FROM cassandra:5.0` üstüne `age` + `awscli` ekleyip saatlik backup script'i çalıştıran sidecar. Self-host için isteğe bağlı.

---

## BÖLÜM 5 — Config ve Secret Akışı

### 5.1 Config dosyası path'i

**Zorunlu env var:** `FLUXER_CONFIG`. Loader:

- [packages/config/src/ConfigLoader.tsx:27](packages/config/src/ConfigLoader.tsx) — `const DEFAULT_CONFIG_PATHS = [process.env['FLUXER_CONFIG']].filter(...)`
- [packages/config/src/ConfigLoader.tsx:37](packages/config/src/ConfigLoader.tsx) — `throw new Error('FLUXER_CONFIG must be set to a JSON config path.')`
- Erlang tarafı: [fluxer_gateway/src/gateway/fluxer_gateway_config.erl:27-30](fluxer_gateway/src/gateway/fluxer_gateway_config.erl), [fluxer_relay/src/relay/fluxer_relay_config.erl:27-29](fluxer_relay/src/relay/fluxer_relay_config.erl) — aynı env var.

**Beklenen path örnekleri:**
- Compose: `/usr/src/app/config/config.json` ([compose.yaml:28](compose.yaml))
- CI deploy: `/etc/fluxer/config.json` (`scripts/ci/workflows/deploy_*.py`)

### 5.2 Config schema

- Ana schema: [packages/config/src/schema/root.json](packages/config/src/schema/root.json) (JSON Schema draft 2020-12)
- Zorunlu alanlar ([root.json:6](packages/config/src/schema/root.json)): `env`, `domain`, `database`, `services`, `auth`
- `deployment_mode=microservices` ise ek zorunlu: `internal`, `services.{s3,queue,media_proxy,admin,marketing,api,app_proxy,gateway,server}.port` ve `services.app_proxy`.
- Çalışma zamanında Zod ile parse: [packages/config/src/ConfigLoader.tsx:51](packages/config/src/ConfigLoader.tsx) — `MasterConfigSchema.parse(merged)` ([MasterZodSchema.generated.tsx](packages/config/src/MasterZodSchema.generated.tsx) — `generate` komutuyla üretilen).
- Bundled JSON Schema: [packages/config/src/ConfigSchema.json](packages/config/src/ConfigSchema.json) (52 KB, generate edilen).
- $defs: `packages/config/src/schema/defs/` altında — `auth.json`, `database.json`, `services/*.json`, `integrations/*.json`.

### 5.3 Template config

**Template dosyası:** [config/config.production.template.json](config/config.production.template.json)

Placeholder'lar:

| Alan | Kullanım | Üretim komutu |
|---|---|---|
| `domain.base_domain` | Public URL türetmede temel; [packages/config/src/EndpointDerivation.tsx](packages/config/src/EndpointDerivation.tsx). | Manuel (ör. `chat.example.com`). |
| `s3.access_key_id`, `s3.secret_access_key` | S3 auth ([ServiceInitializer.tsx:98-99](fluxer_server/src/ServiceInitializer.tsx)). | Depolama sağlayıcıdan al; yerel S3 için `fluxer_server` içinde oluşturulur. |
| `services.media_proxy.secret_key` | Medya imzalama anahtarı (64 char hex). | `openssl rand -hex 32` |
| `services.admin.secret_key_base` | Admin session secret. | `openssl rand -hex 32` |
| `services.admin.oauth_client_secret` | Admin OAuth2 client secret. | `openssl rand -hex 32` |
| `services.marketing.secret_key_base` | Marketing cookie secret. | `openssl rand -hex 32` |
| `services.gateway.admin_reload_secret` | Gateway reload endpoint auth (`gateway.json:13-16`). | `openssl rand -hex 32` |
| `services.gateway.media_proxy_endpoint` | Internal media proxy URL. | Deployment'a göre manuel (`http://fluxer_server:8080/media` gibi). |
| `services.nats.auth_token` | NATS token ([ServiceInitializer: JetStreamConnectionManager](packages/api/src/Config.tsx:154)). | `openssl rand -hex 32` veya NATS cluster config'e göre. |
| `auth.sudo_mode_secret` | Sudo mode token HMAC'ı. | `openssl rand -hex 32` |
| `auth.connection_initiation_secret` | Gateway connection token (gateway ↔ api). | `openssl rand -hex 32` |
| `auth.vapid.public_key`, `auth.vapid.private_key` | Web push ([fluxer_gateway/src/push/push_sender.erl](fluxer_gateway/src/push/push_sender.erl)). | `npx web-push generate-vapid-keys` veya benzer VAPID üretici. |
| `integrations.search.api_key` | Meilisearch master veya ES API key. | Search backend'den al. |

Template üst başlık ([config.production.template.json:1-8](config/config.production.template.json)):
```json
{
  "$schema": "../packages/config/src/ConfigSchema.json",
  "env": "production",
  "domain": { "base_domain": "chat.example.com", "public_scheme": "https", "public_port": 443 },
  ...
}
```

### 5.4 Çalışma zamanı env var'ları (unique `process.env.X`)

Repo genelinde bulunan tekil referanslar (node_modules hariç):

- `FLUXER_CONFIG` — **zorunlu.**
- `FLUXER_CONFIG__*` — override mekanizması ([ConfigLoader.tsx:47](packages/config/src/ConfigLoader.tsx), örnek: `FLUXER_CONFIG__DATABASE__HOST=db.example.com` → `database.host`).
- `NODE_ENV` — `production`/`development`/`test` ([fluxer_server/Dockerfile:179](fluxer_server/Dockerfile)).
- `FLUXER_SERVER_HOST`, `FLUXER_SERVER_PORT` ([Dockerfile:181-182](fluxer_server/Dockerfile)).
- `FLUXER_GATEWAY_HOST`, `FLUXER_GATEWAY_PORT` ([Dockerfile:183-184](fluxer_server/Dockerfile)).
- `DATABASE_BACKEND`, `SQLITE_PATH`, `STORAGE_ROOT`, `SEARCH_BACKEND`, `FLUXER_SERVER_STATIC_DIR` ([Dockerfile:185-189](fluxer_server/Dockerfile)).
- `FLUXER_ADMIN_PORT`, `FLUXER_MARKETING_PORT`, `PORT`.
- `BUILD_SHA`, `BUILD_NUMBER`, `BUILD_TIMESTAMP`, `RELEASE_CHANNEL` — OCI image etiketleri.
- `FLUXER_DATABASE` — devenv'de `sqlite` seti ([devenv.nix:7](devenv.nix)).
- `FLUXER_APP_DEV_PORT` — dev server portu.
- `FORCE_COLOR`, `COREPACK_HOME`, `HOME`, `NODE_OPTIONS` (`--max-old-space-size=2048` api için).
- `FLUXER_GATEWAY_NODE_FLAG`, `FLUXER_GATEWAY_NODE_NAME`, `LOGGER_LEVEL` — Erlang gateway için ([vm.args.src](fluxer_gateway/config/vm.args.src)).
- Dev/test: `FLUXER_API_URL`, `FLUXER_GATEWAY_URL`, `FLUXER_WEBAPP_ORIGIN`, `FLUXER_TEST_TOKEN`, `FLUXER_AUTO_I18N`, `OPENROUTER_API_KEY`, `FLUXER_GATEWAY_NO_SHELL`.
- Desktop build only (self-host için ilgisiz): `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`, `APPLE_TEAM_ID`, `BUILD_CHANNEL`.

**Özel override mekanizması:** `FLUXER_CONFIG__NESTED__KEY=value` formatı `env.nested.key` JSON path'ini override eder ([EnvironmentOverrides.tsx](packages/config/src/config_loader/EnvironmentOverrides.tsx)). Örnek: `FLUXER_CONFIG__SERVICES__MEDIA_PROXY__STATIC_MODE=true` ([deploy_static_proxy.py:35](scripts/ci/workflows/deploy_static_proxy.py)).

---

## BÖLÜM 6 — Veri Katmanı

### 6.1 Database backend'leri

**Enum:** `"cassandra"` veya `"sqlite"` ([packages/config/src/schema/defs/database.json:42](packages/config/src/schema/defs/database.json)).

Switch noktaları:
- [packages/api/src/Config.tsx:120-121](packages/api/src/Config.tsx) — `if (master.database.backend === 'cassandra' && !cassandraSource) throw...`
- [packages/api/src/Config.tsx:137-138](packages/api/src/Config.tsx) — API config'e `{ backend, sqlitePath }` aktarılır.
- [packages/api/src/database/SqliteKV.tsx:420](packages/api/src/database/SqliteKV.tsx) — SQLite path resolve.
- [fluxer_server/src/Instrument.tsx:28](fluxer_server/src/Instrument.tsx) — Telemetry'de cassandra flag.

`fluxer_server` imajı default olarak SQLite ([Dockerfile:185-186](fluxer_server/Dockerfile): `DATABASE_BACKEND=sqlite`, `SQLITE_PATH=/usr/src/app/data/db/fluxer.db`).

### 6.2 ORM / query builder

- Raw SQL + driver: `cassandra-driver@4.8.0` (`packages/cassandra`), native `sqlite` (better-sqlite3 veya node:sqlite — `libsqlite3-dev` Dockerfile'da kuruluyor, [fluxer_api/Dockerfile:30](fluxer_api/Dockerfile)).
- **Migration:** `fluxer_api/scripts/CassandraMigrate.tsx` — CLI aracı. SQLite için explicit migration klasörü **BULUNAMADI**; `packages/api/src/database/` içinde SqliteKV ve benzeri sınıflar muhtemelen `CREATE TABLE IF NOT EXISTS` kullanıyor (doğrulanmadı).
- Prisma/Drizzle/Knex: **yok.**

### 6.3 Schema tanımı

- Domain schema'ları Zod olarak: `packages/schema/src/domains/` — `channel`, `message`, `guild`, `relay`, `instance`, `presence` vb.
- Model klasörleri: `packages/api/src/*/` (40+ domain — `auth, guild, channel, message, voice, oauth2, oauth, emoji, sticker, invite, bot, attachment, webhook, donation, federation, ...`).
- `fluxer_docs/schemas/events/` — 46 event JSON Schema (MESSAGE_CREATE, GUILD_UPDATE, VOICE_STATE_UPDATE, ...).
- Tablo/collection listesi: Dokümanda numaralandırılmış liste **BULUNAMADI**. Migration dosyalarından çıkarmak gerekir.

### 6.4 KV store (Valkey/Redis)

- Client: `packages/kv_client` → `ioredis@5.8.1` ([pnpm-workspace.yaml:135](pnpm-workspace.yaml))
- Config: `internal.kv: "redis://valkey:6379/0"` ([config.production.template.json:14](config/config.production.template.json)), `internal.kv_mode: "standalone"|"cluster"`.
- Kullanım: cache, rate limit, presence ephemeral state, KVPipeline. Pub/sub için `KVSubscription.tsx`.
- Compose: `valkey/valkey:8.0.6-alpine` container, `--appendonly yes --save 60 1` ([compose.yaml:9-14](compose.yaml)).

### 6.5 Search

**Hem Meilisearch hem Elasticsearch desteklenir** ([search.json:10-12](packages/config/src/schema/defs/integrations/search.json), enum `["meilisearch","elasticsearch"]`, default `meilisearch`).

Adapter'lar:
- Meilisearch: `packages/meilisearch_search/src/adapters/` — `MeilisearchUserAdapter`, `MeilisearchGuildAdapter`, `MeilisearchGuildMemberAdapter`, `MeilisearchMessageAdapter`, `MeilisearchAuditLogAdapter`, `MeilisearchReportAdapter`, `MeilisearchIndexAdapter`.
- Elasticsearch: `packages/elasticsearch_search/src/adapters/` — aynı set.

Compose: `meilisearch:v1.14` ve `elasticsearch:8.19.11`, ikisi de `profiles: ['search']` altında ([compose.yaml:46-86](compose.yaml)).

### 6.6 Object storage (S3)

- SDK: `@aws-sdk/client-s3@3.985.0`, `@aws-sdk/s3-request-presigner`.
- Bucket haritası ([database.json:55-90](packages/config/src/schema/defs/database.json)):
  - `cdn` (default `"fluxer"`), `uploads`, `downloads`, `reports`, `harvests`, `static`.
- Config okuma: [fluxer_server/src/ServiceInitializer.tsx:87-100](fluxer_server/src/ServiceInitializer.tsx) — `requireValue(config.s3, 's3')`, `config.services.s3?.data_dir` (yerel S3 gateway).
- Yerel S3 sunucusu: `packages/s3/src/App.tsx` (aws-sdk uyumlu endpoint'i fluxer_server içinde serve eder — default endpoint `http://127.0.0.1:8080/s3` [config.production.template.json:20](config/config.production.template.json)).
- Yüklenen tipler: avatar, attachment, emoji, sticker, report harvest, CDN bucket'ına statik site çıktıları (bucket isimlerinden çıkarıldı).

---

## BÖLÜM 7 — Gateway ve Realtime

### 7.1 WebSocket Gateway

**Ayrı bir servis.** Dil: **Erlang/OTP 28** ([fluxer_gateway/Dockerfile:1](fluxer_gateway/Dockerfile), [rebar.config:3](fluxer_gateway/rebar.config) — `cowboy 2.14.2`, `enats 1.2.0`, `jose 1.11.10`).

`fluxer_server` imajı içinde *de* başlatılıyor ([fluxer_server/src/index.tsx:79-85](fluxer_server/src/index.tsx) → `createGatewayProcessManager`). Yani monolith modda Node process bir alt-process olarak Erlang release'i çalıştırıyor.

### 7.2 Port

- Config default `8771` ([gateway.json:11](packages/config/src/schema/defs/services/gateway.json))
- Dockerfile EXPOSE: `8080 8081` ([fluxer_gateway/Dockerfile:46](fluxer_gateway/Dockerfile))
- fluxer_server içinde embed edildiğinde: `FLUXER_GATEWAY_PORT=8082` ([fluxer_server/Dockerfile:184](fluxer_server/Dockerfile))

### 7.3 Entry point

`fluxer_gateway/src/fluxer_gateway.app.src` → OTP uygulaması. Alt modüller: `gateway/`, `guild/`, `presence/`, `session/`, `push/`, `call/`, `telemetry/`, `utils/`.

Config yükleme: [fluxer_gateway/src/gateway/fluxer_gateway_config.erl:26-111](fluxer_gateway/src/gateway/fluxer_gateway_config.erl) — JSON'ı kendi başına parse eder, `services.gateway` + `services.nats` + `auth.vapid` alanlarını okur.

### 7.4 Gateway ↔ HTTP server iletişimi

**NATS (core + JetStream)** primary. Config: `services.nats.core_url`, `services.nats.jetstream_url`, `services.nats.auth_token` ([fluxer_gateway_config.erl:53-54](fluxer_gateway/src/gateway/fluxer_gateway_config.erl), [packages/api/src/Config.tsx:151-155](packages/api/src/Config.tsx)).

Gateway'den API'ye HTTP RPC (push dispatch, guild settings fetch) için ayrıca: `gateway_http_*` config alanları ([fluxer_gateway_config.erl:74-89](fluxer_gateway/src/gateway/fluxer_gateway_config.erl)).

### 7.5 Push notification (VAPID + service worker)

- VAPID config: `auth.vapid.public_key`, `auth.vapid.private_key`, `auth.vapid.email` ([auth.json](packages/config/src/schema/defs/auth.json)).
- Sender (Erlang): [fluxer_gateway/src/push/push_sender.erl](fluxer_gateway/src/push/push_sender.erl) + `push_utils.erl`, `push_subscriptions.erl`, `push_dispatcher.erl`, `push_eligibility.erl`, `push_notification.erl`.
- Service worker (istemci): [fluxer_app/src/service_worker/Register.tsx](fluxer_app/src/service_worker/Register.tsx), [Worker.tsx](fluxer_app/src/service_worker/Worker.tsx). Build script: [fluxer_app/scripts/build-sw.mjs](fluxer_app/scripts/build-sw.mjs).

---

## BÖLÜM 8 — Voice/Video (LiveKit)

### 8.1 Config alanları

`integrations.voice` şeması ([voice.json:1-70](packages/config/src/schema/defs/integrations/voice.json)):
- `enabled` (bool)
- `api_key`, `api_secret` (enabled + default_region varsa zorunlu — `if/then` block [line 59-68](packages/config/src/schema/defs/integrations/voice.json))
- `url` (WS signal endpoint, örn. `ws://livekit:7880`)
- `webhook_url` (örn. `http://<host>/api/webhooks/livekit`)
- `default_region` → `{id, name, emoji, latitude, longitude}` (varsa instance açılışında bir region + server oluşturulur)

### 8.2 Kod kullanımı

- SDK: `livekit-server-sdk@2.15.0` (`packages/api/package.json:64`), `livekit-client@2.17.1` (fluxer_app).
- Service: [packages/api/src/infrastructure/LiveKitService.tsx](packages/api/src/infrastructure/LiveKitService.tsx) — `AccessToken`, `RoomServiceClient`.
- Disabled fallback: [packages/api/src/infrastructure/DisabledLiveKitService.tsx](packages/api/src/infrastructure/DisabledLiveKitService.tsx) — `voice.enabled=false` ise bu stub devreye giriyor.
- Webhook: [packages/api/src/webhook/WebhookController.tsx:471](packages/api/src/webhook/WebhookController.tsx) — `app.post('/webhooks/livekit', ...)`. Route [packages/api/src/middleware/RequireXForwardedForMiddleware.tsx:33](packages/api/src/middleware/RequireXForwardedForMiddleware.tsx) listesinde.
- Reconciliation worker: [packages/api/src/voice/VoiceReconciliationWorker.tsx](packages/api/src/voice/VoiceReconciliationWorker.tsx).

### 8.3 Text-only deploy

**Evet, mümkün.** `DisabledLiveKitService.tsx` stub'ı var; `voice.enabled=false` ise `LiveKitService` yerine bu injecte edilir. Config schema'da `enabled` default `false` ([voice.json:10](packages/config/src/schema/defs/integrations/voice.json)).

Ancak `voice.default_region` verildiği ve `enabled=true` yapıldığı an `api_key`/`api_secret` zorunlu olur — karşılanmazsa Zod parse fail eder.

LiveKit compose servisi `profiles: ['voice']` altında ([compose.yaml:91](compose.yaml)); profile aktive edilmediği sürece başlamaz.

---

## BÖLÜM 9 — Compose / Deploy Artifacts

### 9.1 Root `compose.yaml`

4 ana servis ([compose.yaml](compose.yaml)):

| Servis | Image | Default port | Profile |
|---|---|---|---|
| `valkey` | `valkey/valkey:8.0.6-alpine` | dahili | yok |
| `fluxer_server` | `${FLUXER_SERVER_IMAGE:-ghcr.io/fluxerapp/fluxer-server:stable}` | `8080` (env `FLUXER_HTTP_PORT`) | yok |
| `meilisearch` | `getmeili/meilisearch:v1.14` | `7700` | `search` |
| `elasticsearch` | `elasticsearch:8.19.11` | `9200` | `search` |
| `livekit` | `livekit/livekit-server:v1.9.11` | `7880`, `7881`, `3478/udp`, `50000-50100/udp` | `voice` |

**Private image uyarısı:** [compose.yaml:23](compose.yaml) `ghcr.io/fluxerapp/fluxer-server:stable` → GHCR image'ı public olmayabilir. Self-host için ya image erişimi onaylanmalı ya da `FLUXER_SERVER_IMAGE` override'ı ile kendi inşa ettiğiniz image kullanılmalı. (Access testi için `docker pull ghcr.io/fluxerapp/fluxer-server:stable` doğrulanmadı.)

Volume mount: `./config → /usr/src/app/config:ro`. Config dosyası `./config/config.json` olarak beklenir.

### 9.2 Kubernetes / Helm

**Yok.** Find sonuçlarında `*.yaml` k8s manifest veya Helm chart bulunamadı.

### 9.3 CI/CD (`.github/workflows/`)

23 workflow:

| Workflow | Amaç |
|---|---|
| `ci.yaml` | PR tetikli; typecheck + test (`blacksmith-8vcpu-ubuntu-2404` runner, Turborepo remote cache `turborepo.fluxer.dev`). Python `scripts/ci/workflows/ci.py` çalıştırıyor. |
| `build-desktop.yaml` | Electron desktop build. |
| `channel-vars.yaml` | Release channel değişken çıkaran reusable workflow. |
| `deploy-admin.yaml`, `deploy-api.yaml`, `deploy-app.yaml`, `deploy-gateway.yaml`, `deploy-marketing.yaml`, `deploy-media-proxy.yaml`, `deploy-relay-directory.yaml`, `deploy-relay.yaml`, `deploy-static-proxy.yaml` | Servis başına deploy (`scripts/ci/workflows/deploy_*.py`). Compose dosyası üreterek `/etc/fluxer/config.json` referansı ile uzak sunucuda çalıştırıyor. |
| `migrate-cassandra.yaml` | Cassandra migration runner. |
| `promote-canary-to-main.yaml` | Canary → main promotion. |
| `release-livekitctl.yaml`, `release-relay-directory.yaml`, `release-relay.yaml`, `release-server.yaml` | GHCR/release artifact yayını. |
| `restart-gateway.yaml` | Gateway yeniden başlatma runbook'u. |
| `sync-desktop.yaml`, `sync-static.yaml` | Bağımlı repo sync'i. |
| `test-cassandra-backup.yaml` | Backup sidecar testi. |
| `update-word-lists.yaml` | Word list güncelleme. |

CI runner: `blacksmith-8vcpu-ubuntu-2404` (Blacksmith.sh). Turborepo cache: `turborepo.fluxer.dev` (external). `TURBO_TOKEN` secret gerekli.

### 9.4 `devenv.nix` / `flake.nix`

[devenv.nix:117-146](devenv.nix) paketler: `nodejs_24, pnpm, erlang_28, rebar3, valkey, meilisearch, nats-server, ffmpeg, exiftool, caddy, livekit, mailpit, go_1_24, rust-bin.stable."1.93.0" (wasm32-unknown-unknown), jq, gettext, lsof, iproute2, python3, pkg-config, gcc, gnumake, sqlite, openssl, curl, uv`.

Process manager: **process-compose**. Tanımlı process'ler: `caddy`, `css_watch`, `fluxer_app`, `fluxer_gateway`, `fluxer_server`, `livekit`, `mailpit`, `meilisearch`, `valkey`, `marketing_dev`, `nats_core` (port 4222), `nats_jetstream` (port 4223).

Bootstrap task: `fluxer:bootstrap` → `scripts/dev_bootstrap.sh` ([devenv.nix:148-164](devenv.nix)).

Cassandra migration task'ları: `cassandra:mig:{create,check,status,up}` ([devenv.nix:166-203](devenv.nix)).

---

## BÖLÜM 10 — BULUNAN SORUNLAR VE BUGFIX LİSTESİ

### SORUN-01: `packages/app` diskte yok, fluxer_server/Dockerfile COPY bekliyor
- **Dosya:** [fluxer_server/Dockerfile:28](fluxer_server/Dockerfile)
- **Tanım:** `COPY packages/app/package.json ./packages/app/` satırı var, fakat `packages/` altında `app` klasörü mevcut değil. `ls packages/` çıktısında `admin, api, app_proxy, cache, captcha, cassandra, config, constants, date_utils, ...` var ama `app` yok.
- **Etki:** Docker `COPY` bulunmayan bir kaynak için hata veriyor → **build hiç başlamaz.**
- **Önerilen fix:** Satırı kaldır. fluxer_server'ın fluxer_app'e bağımlılığı zaten build stage'inde ayrı app-build olarak handle ediliyor.
- **Öncelik:** **P0**

### SORUN-02: `fluxer_server/Dockerfile` app-build stage'inde Rust/wasm-pack yok
- **Dosya:** [fluxer_server/Dockerfile:108-115](fluxer_server/Dockerfile)
- **Tanım:** `app-build` stage'i `FROM deps` ile geliyor; deps base'inde sadece `curl, python3, make, g++` kurulu (line 12-17). Buna rağmen `RUN cd fluxer_app && pnpm build` çalıştırılıyor. `fluxer_app/package.json:18` build script'i `pnpm wasm:codegen && ...`; `wasm:codegen` ise `wasm-pack build --target web --out-dir ../../pkgs/libfluxcore --release`. Rust 1.93.0 toolchain ([rust-toolchain.toml](fluxer_app/rust-toolchain.toml)) ve `wasm-pack` gerekiyor.
- **Etki:** `pnpm build` Rust çağrısında fail eder → **image üretilemez.**
- **Önerilen fix:** `app-build` için ayrı base ekle:
  ```dockerfile
  FROM deps AS app-build
  RUN apt-get update && apt-get install -y --no-install-recommends \
      curl ca-certificates build-essential pkg-config && \
      rm -rf /var/lib/apt/lists/*
  RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | \
      sh -s -- -y --default-toolchain 1.93.0 --target wasm32-unknown-unknown
  ENV PATH=/root/.cargo/bin:$PATH
  RUN curl -fsSL https://rustwasm.github.io/wasm-pack/installer/init.sh | sh
  ```
  Ya da multi-stage'de fluxer_app'i host üstünde build edip `fluxer_app/dist/` bind'leyin.
- **Öncelik:** **P0**

### SORUN-03: `fluxer_app_proxy/Dockerfile` `fluxer_app/dist/` olmadan build fail eder
- **Dosya:** [fluxer_app_proxy/Dockerfile:57](fluxer_app_proxy/Dockerfile)
- **Tanım:** `COPY fluxer_app/dist/ ./assets/`. `fluxer_app/dist/` **gitignore'da değil** (override `!fluxer_app/dist` ile .dockerignore'da da whitelist), fakat repoda fiziksel olarak yok (`.gitignore:52` `**/dist` global ignore).
- **Etki:** `docker build fluxer_app_proxy/` bağımsız çalışmaz. Önce `pnpm --filter fluxer_app build` yapılmış olmalı.
- **Önerilen fix:** Dockerfile içine app build stage'i ekleyin (fluxer_server Dockerfile'daki app-build pattern'i gibi), Rust kurulumu dahil.
- **Öncelik:** **P0**

### SORUN-04: `fluxer_relay_directory/Dockerfile` bağımlı paketleri COPY etmiyor
- **Dosya:** [fluxer_relay_directory/Dockerfile:11-14](fluxer_relay_directory/Dockerfile)
- **Tanım:** Sadece `packages/hono` ve `packages/schema` kopyalanıyor. Fakat [fluxer_relay_directory/package.json:14-17](fluxer_relay_directory/package.json) `@fluxer/config` ve `@fluxer/openapi`'ye bağımlı. `packages/config` pnpm tarafından `@fluxer/constants`, `ajv`, `zod`'u çeker; `openapi` ise `@fluxer/schema`, `@fluxer/api`'ye bağlı olabilir.
- **Etki:** `pnpm install --filter fluxer-relay-directory...` fail edebilir (workspace sym-link kurulumda package.json okumayı dener). En iyi senaryoda eksik dep warning ile geçer, runtime'da import fail eder.
- **Önerilen fix:** Tüm `packages/*/package.json`'ları kopyala (en kolay: `COPY packages/ ./packages/`).
- **Öncelik:** **P0**

### SORUN-05: `fluxer_server/Dockerfile` 17 workspace paketini deps stage'inde eksik bırakıyor
- **Dosya:** [fluxer_server/Dockerfile:26-58](fluxer_server/Dockerfile)
- **Tanım:** deps stage'i tek tek `COPY packages/<name>/package.json` ediyor; fakat `date_utils, elasticsearch_search, geo_utils, geoip, i18n, kv_client, limits, list_utils, locale, markdown_parser, media_proxy_utils, meilisearch_search, mime_utils, nats, number_utils, openapi, time` listede yok. `fluxer_server/package.json` `@fluxer/kv_client` (satır 24) ve `@fluxer/nats` (satır 27) ile doğrudan, `@fluxer/api` aracılığıyla tüm diğerlerine transitive bağımlı.
- **Etki:** `pnpm install --frozen-lockfile` workspace resolution'da fail edebilir (pnpm lockfile'da bulunmayan package.json'a karşılık gelen workspace paketlerini bulamaz). En iyi senaryoda warning, kötü senaryoda kurulum fail.
- **Önerilen fix:** Deps stage'inde tek tek COPY yerine `COPY packages/*/package.json ./packages/` kalıbı kullan veya:
  ```dockerfile
  COPY packages/ ./packages/
  RUN find packages -mindepth 2 -maxdepth 2 ! -name 'package.json' -exec rm -rf {} +
  ```
- **Öncelik:** **P0**

### SORUN-06: pnpm versiyon uyumsuzluğu (root `10.29.3` vs Dockerfile `10.26.0`)
- **Dosya:** [package.json:37](package.json), [fluxer_server/Dockerfile:10](fluxer_server/Dockerfile), [fluxer_admin/Dockerfile:10](fluxer_admin/Dockerfile), [fluxer_api/Dockerfile:10](fluxer_api/Dockerfile) vs.
- **Tanım:** Root `packageManager: pnpm@10.29.3`; tüm Dockerfile'lar `corepack prepare pnpm@10.26.0 --activate`. Lockfile 10.29.3 formatında üretildi ise 10.26.0 `--frozen-lockfile` reject edebilir.
- **Etki:** CI / local install farklılığı. Özellikle minor versiyon farkı bile lockfile format uyuşmazlığına yol açabilir.
- **Önerilen fix:** Tüm Dockerfile'lardaki `pnpm@10.26.0` → `pnpm@10.29.3`. Veya root'ta minor pin yerine major pin kullan.
- **Öncelik:** **P1**

### SORUN-07: `fluxer_server` production image'ı container root olarak çalışıyor
- **Dosya:** [fluxer_server/Dockerfile:117-199](fluxer_server/Dockerfile) (final stage)
- **Tanım:** `USER` direktifi yok. Diğer servislerin hepsi `USER nobody` kullanıyor (`fluxer_admin/Dockerfile:81`, `fluxer_api/Dockerfile:85`, vs.), ama monolith imaj root.
- **Etki:** **Güvenlik:** container escape senaryolarında yüzey büyük; `/data/*` root chown'lu.
- **Önerilen fix:** Non-root user ekleyin:
  ```dockerfile
  RUN useradd -r -u 1001 -s /sbin/nologin -d /usr/src/app fluxer && \
      chown -R fluxer:fluxer /usr/src/app /data /opt/data
  USER fluxer
  ```
- **Öncelik:** **P1**

### SORUN-08: `compose.yaml` private olabilir GHCR image'ına pin
- **Dosya:** [compose.yaml:23](compose.yaml)
- **Tanım:** `ghcr.io/fluxerapp/fluxer-server:stable`. README "TBD" diyor self-hosting için. Tag açıkça public mi doğrulanmadı.
- **Etki:** Dokployda `docker-compose up` → image pull 403/404 verebilir.
- **Önerilen fix:** Ya image public'e açıldığını doğrulayın, ya da kendi build+push pipeline'ı kurarak `FLUXER_SERVER_IMAGE` override edin:
  ```bash
  FLUXER_SERVER_IMAGE=registry.example.com/fluxer-server:0.0.1 docker compose up -d
  ```
- **Öncelik:** **P0** (deploy etmeden önce erişim doğrulanmalı)

### SORUN-09: `.tool-versions` boş
- **Dosya:** [.tool-versions](.tool-versions) (1 satır, içerik yok)
- **Tanım:** asdf user'ları için tool versiyonu bulamayacak.
- **Etki:** Sadece asdf kullanıcılarını etkiler (devenv asıl kanal). Düşük öncelik.
- **Önerilen fix:** Ya dosyayı kaldır ya da `nodejs 24` + `erlang 28` + `rust 1.93.0` + `python 3.12` + `go 1.24` ekle.
- **Öncelik:** **P2**

### SORUN-10: `fluxer_app/pnpm-lock.yaml` duplicate (nested lockfile)
- **Dosya:** [fluxer_app/pnpm-lock.yaml](fluxer_app/pnpm-lock.yaml) — 14 934 satır; root [pnpm-lock.yaml](pnpm-lock.yaml) — 20 511 satır.
- **Tanım:** pnpm workspace'lerde sadece **root** lockfile olması gerekir. Nested lockfile divergence riski yaratır. `fluxer_app/package.json` workspace member.
- **Etki:** Dev ortamda `pnpm install` fluxer_app klasöründe çalıştırılırsa nested lockfile'ı güncelleyebilir, root lockfile'dan sapabilir.
- **Önerilen fix:** `rm fluxer_app/pnpm-lock.yaml` ve `.gitignore`'a `fluxer_app/pnpm-lock.yaml` ekle.
- **Öncelik:** **P2**

### SORUN-11: `fluxer_gateway/config/vm.args.template` ve `vm.args.src` aynı içerik
- **Dosya:** [fluxer_gateway/config/vm.args.template](fluxer_gateway/config/vm.args.template) ve [vm.args.src](fluxer_gateway/config/vm.args.src)
- **Tanım:** İkisinin içeriği bire bir aynı. Rebar `{vm_args_src, "./config/vm.args.src"}` ([rebar.config:24](fluxer_gateway/rebar.config)) kullanıyor. fluxer_server Dockerfile'daki [line 105](fluxer_server/Dockerfile) vm.args.template'i envsubst'layıp vm.args'a yazıyor (ama rebar bunu okumuyor); kullanılmıyor.
- **Etki:** Gereksiz dosya; bakım maliyeti. Envsubst edilen `vm.args` rebar tarafından göz ardı ediliyor olabilir — LOGGER_LEVEL env var'ı `vm.args.src` için `relx` tarafından resolve ediliyor.
- **Önerilen fix:** Duplicate dosyalardan birini kaldır. Eğer relx `vm.args.src`'i runtime env olarak ${VAR} interpolasyonu yapıyorsa, envsubst adımını tamamen kaldır.
- **Öncelik:** **P2**

### SORUN-12: `config.production.template.json` `integrations.voice` ve `email` alanlarını içermiyor
- **Dosya:** [config/config.production.template.json](config/config.production.template.json)
- **Tanım:** Template'de sadece `search` var. Ancak dev template ([config.dev.template.json:68-108](config/config.dev.template.json)) `email, gif, klipy, tenor, voice, search` alanlarını içeriyor. Self-host'un production'a deploy ederken email/voice/etc. gerekliyse ek örnekler lazım.
- **Etki:** Self-host'un email provider'ı yapılandırmadan deploy edenler "email gönderilmedi" sessiz hatalarıyla karşılaşır.
- **Önerilen fix:** Production template'i dev template formatına genişletin, tüm placeholder'ları açıklayıcı commentlerle ekleyin.
- **Öncelik:** **P1**

### SORUN-13: `fluxer_server` final image'ı config zorunluluğunu runtime'da check ediyor
- **Dosya:** [packages/config/src/ConfigLoader.tsx:37](packages/config/src/ConfigLoader.tsx)
- **Tanım:** `FLUXER_CONFIG` env olmadan container boot olamaz (`throw new Error('FLUXER_CONFIG must be set...')`).
- **Etki:** Security değil, UX. Hata okunabilir fakat deploy'dan önce varsayılan config path'i beklemek daha iyi olabilir.
- **Önerilen fix:** Container start'ta `[ -z "$FLUXER_CONFIG" ] && echo "FLUXER_CONFIG not set; set to /usr/src/app/config/config.json" && export FLUXER_CONFIG=/usr/src/app/config/config.json` wrapper. VEYA Dockerfile'a `ENV FLUXER_CONFIG=/usr/src/app/config/config.json` ekle.
- **Öncelik:** **P2**

### SORUN-14: `.gitignore` olumlu — `config/config.json` ignore'lu, ama Dockerfile `./config → /usr/src/app/config:ro` bind ediyor
- **Dosya:** [.gitignore:72](.gitignore), [compose.yaml:36](compose.yaml)
- **Tanım:** `config/config.json` gitignore'da (güvenli). Compose root'taki `./config/`'u mount ediyor. Operatör `config/config.json` oluşturmalı; ve yanlışlıkla `git add -A` ile secret'ı commit etmeme garantisi var. İyi pattern.
- **Etki:** Doğrudan sorun yok; ama `config.json` olmadan boot fail (bkz. SORUN-13).
- **Öncelik:** bilgi amaçlı, aksiyon gerekmiyor.

---

**P0 özet (deploy engelleyici):** 6 sorun — SORUN-01, 02, 03, 04, 05, 08
**P1 özet (ilk hafta):** 3 sorun — SORUN-06, 07, 12
**P2 özet (nice-to-have):** 4 sorun — SORUN-09, 10, 11, 13

---

## BÖLÜM 11 — Deploy İçin Kritik Sorular

- [ ] `ghcr.io/fluxerapp/fluxer-server:stable` image'ı public mi? `docker pull` testi yapıldı mı? Private ise kendi build pipeline'ım nasıl olacak?
- [ ] Dockerfile P0 sorunları (SORUN-01..05) upstream'de bilinen issue mu? `fluxer_server/Dockerfile` CI'da gerçekten başarılı build ediyor mu, yoksa bu sadece "WIP" durumda mı? (CI workflow'larında `release-server.yaml` var fakat ne yaptığını doğrulamadım.)
- [ ] `fluxer_app/dist/` dev ortamında build ediliyor da Docker image'ı o bind mount'tan mi alıyor? Yani CI pipeline önce host üstünde fluxer_app build edip sonra fluxer_server image'ını build ediyor olabilir mi? (Multi-stage içindeki app-build'in gerçekten çalıştığına dair net bir başarı raporu yok — deneme build'i yapılmadı.)
- [ ] SQLite path kalıcılığı: `/usr/src/app/data/db/fluxer.db` default, fakat compose'da `fluxer_data` named volume `/usr/src/app/data`'ya mount. Tüm data (db + storage + queue) tek volume mı yoksa ayrı volume'lere mi ayrılmalı?
- [ ] Gateway `services.gateway.admin_reload_secret` runtime'da hangi endpoint tarafından kullanılıyor? (gateway/fluxer_gateway_config.erl'de okunuyor ama reload endpoint'inin rate limit/authz davranışı dokümanlamadı.)
- [ ] `internal.media_proxy` config alanı Erlang gateway tarafından `media_proxy_endpoint` olarak okunuyor ([fluxer_gateway_config.erl:90](fluxer_gateway/src/gateway/fluxer_gateway_config.erl)) ama compose.yaml'da sadece fluxer_server internal, media_proxy endpoint'i nasıl expose edilecek? `http://127.0.0.1:8080/media` mı kullanılacak (fluxer_server içinde aynı port)?
- [ ] fluxer_app'in rspack config'i ([rspack.config.mjs:62](fluxer_app/rspack.config.mjs)) `FLUXER_CONFIG` env var'ı bekliyor — fluxer_server/Dockerfile app-build stage'inde FLUXER_CONFIG set edilmiyor. Dev config mi production config mi kullanılmalı, nasıl?
- [ ] Relay (federasyon) gerçekten self-host için opsiyonel mı? fluxer_server imajı `fluxer_relay` veya `fluxer_relay_directory`'yi bundle ediyor mu? (Compose'da bu servisler yok; ayrı imaj olmalı ama entegrasyon belirsiz.)
- [ ] NATS (core + JetStream) zorunlu mu? Compose'da NATS container'ı **yok** fakat config şeması ve servis kodu NATS bekler. Monolith fluxer_server içinde NATS embed mi (enats via Erlang?) yoksa dış bağımlılık mı?
- [ ] Worker/queue init: [fluxer_server/src/index.tsx:87-122](fluxer_server/src/index.tsx) `jsConnectionManager` varsa cron + worker başlatıyor. NATS bağlantısı olmazsa worker çalışmaz — cron job'ları (attachment expiration, discovery sync, vs.) nasıl başlatılacak?
- [ ] MeiliSearch default master key'i zorunlu ([compose.yaml:53](compose.yaml) `MEILI_MASTER_KEY:?Set MEILI_MASTER_KEY`). Config'de de `integrations.search.api_key` zorunlu ([search.json:34](packages/config/src/schema/defs/integrations/search.json) `required: ["url", "api_key"]`). İki yerde eşit olmalı, dokümantasyonu yok.
- [ ] fluxer_server Dockerfile final stage'inde `/opt/data/model.onnx` var (NSFW modeli); ama [fluxer_server/src/ServiceInitializer.tsx](fluxer_server/src/ServiceInitializer.tsx) veya packages/media_proxy kodu bu dosyayı hangi yoldan okuyor? `integrations.csam` veya `integrations.photo_dna` ile mi bağlantılı? INCLUDE_NSFW_ML=false olursa hangi özellik kırılır?
- [ ] Admin paneli için ilk sudo kullanıcı nasıl oluşturulur? `services.admin.secret_key_base` var ama bootstrap user akışı **BULUNAMADI**.
- [ ] Stripe, hCaptcha, Turnstile, Klipy, Tenor entegrasyonları için zorunlu alanlar schema'da `required` değil mi? Hangileri no-op fallback ile gelebilir?
- [ ] Compose'da `fluxer_server` healthcheck `curl` ile `/_health`'e vuruyor ama Dockerfile final base image'da curl kurulu (line 136-139) ✅. Ama **fluxer_app_proxy Dockerfile** final'inde de curl kuruluyor ([fluxer_app_proxy/Dockerfile:41](fluxer_app_proxy/Dockerfile)) ✅.
- [ ] Backup stratejisi: SQLite dosyası + S3 yerel data + Valkey AOF + Meilisearch data için dokuman? `fluxer_devops/cassandra/Dockerfile.backup` sadece Cassandra için.

---

## EK — Config Alan-Kod Eşleştirmeleri (hızlı referans)

| Config alanı | Okuma noktası |
|---|---|
| `FLUXER_CONFIG` (env) | [packages/config/src/ConfigLoader.tsx:27](packages/config/src/ConfigLoader.tsx), [fluxer_gateway_config.erl:27](fluxer_gateway/src/gateway/fluxer_gateway_config.erl), [fluxer_relay_config.erl:27](fluxer_relay/src/relay/fluxer_relay_config.erl) |
| `env` | `MasterConfigSchema.parse()` → `Config.env` (tüm servislerde) |
| `domain.base_domain` | [packages/config/src/EndpointDerivation.tsx](packages/config/src/EndpointDerivation.tsx) |
| `database.backend` | [packages/api/src/Config.tsx:120,137](packages/api/src/Config.tsx), [fluxer_server/src/Instrument.tsx:28](fluxer_server/src/Instrument.tsx) |
| `database.sqlite_path` | [packages/api/src/database/SqliteKV.tsx:420](packages/api/src/database/SqliteKV.tsx), [packages/api/src/infrastructure/DirectS3ExpirationManager.tsx:37](packages/api/src/infrastructure/DirectS3ExpirationManager.tsx) |
| `internal.kv` | [fluxer_server/src/ServiceInitializer.tsx:78](fluxer_server/src/ServiceInitializer.tsx) (`new KVClient({url})`) |
| `s3.access_key_id`, `s3.secret_access_key` | [fluxer_server/src/ServiceInitializer.tsx:98-99](fluxer_server/src/ServiceInitializer.tsx) |
| `services.server.port`/`host` | [fluxer_server/src/Config.tsx:31-32](fluxer_server/src/Config.tsx) |
| `services.gateway.*` | [fluxer_gateway/src/gateway/fluxer_gateway_config.erl:45-98](fluxer_gateway/src/gateway/fluxer_gateway_config.erl) |
| `services.nats.core_url`/`jetstream_url`/`auth_token` | [packages/api/src/Config.tsx:151-155](packages/api/src/Config.tsx), [fluxer_gateway_config.erl:53-54](fluxer_gateway/src/gateway/fluxer_gateway_config.erl) |
| `auth.vapid.*` | [fluxer_gateway_config.erl:91-93](fluxer_gateway/src/gateway/fluxer_gateway_config.erl), push sender'da |
| `integrations.voice.*` | [packages/api/src/infrastructure/LiveKitService.tsx:27-](packages/api/src/infrastructure/LiveKitService.tsx), [WebhookController.tsx:471](packages/api/src/webhook/WebhookController.tsx) |
| `integrations.search.engine/url/api_key` | [packages/api/src/SearchFactory.tsx](packages/api/src/SearchFactory.tsx) |
