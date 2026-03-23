KasirGo+ PowerSync (Self-Hosted)

Ini stack PowerSync Service untuk self-hosting di VPS, sesuai pedoman `kasirgoplus-mobile/notes/powersync-architecture.txt`.

Komponen:
- PowerSync Service (journeyapps/powersync-service)
- MongoDB replica set untuk bucket storage (internal PowerSync)
- Source DB: Postgres milik backend KasirGo+ (external; tidak disetup di sini)

---

## Prasyarat Postgres (source DB)
PowerSync butuh Postgres logical replication.

1) Aktifkan wal_level=logical di Postgres server (contoh bila pakai container postgres):
`postgres -c wal_level=logical`

2) Buat user repl + publication `powersync`.
Sesuaikan nama database & schema.

```sql
-- role untuk replication
CREATE ROLE powersync_repl WITH LOGIN PASSWORD 'change_me' REPLICATION;

-- beri akses baca ke tabel yang akan disync
GRANT CONNECT ON DATABASE kasirgoplus TO powersync_repl;
GRANT USAGE ON SCHEMA public TO powersync_repl;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO powersync_repl;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO powersync_repl;

-- publication untuk tabel yang direplikasi (tambah tabel lain seiring phase)
CREATE PUBLICATION powersync FOR TABLE
  categories,
  products;
```

---

## JWT (HS256) untuk PowerSync
PowerSync Service memverifikasi JWT user.

Backend KasirGo+ sign token HS256, dan PowerSync pakai kunci yang sama via JWKS `kty: oct`.

Kunci yang dibutuhkan PowerSync adalah **base64url-encoded shared secret** (bukan plain string).

Cara generate dari `JWT_SECRET` backend (Mac):

```bash
node -e "const s=process.argv[1];const b=Buffer.from(s,'utf8').toString('base64').replace(/\\+/g,'-').replace(/\\//g,'_').replace(/=+$/,'');console.log(b)" "$JWT_SECRET"
```

Audience (`aud`) dan key id (`kid`) harus match dengan backend env:
- `POWERSYNC_JWT_AUDIENCE` (default: `powersync`)
- `POWERSYNC_JWT_KID` (default: `kasirgo-hs256`)

---

## Menjalankan di VPS (docker compose)
1) Copy env template:

```bash
cp .env.example .env
```

2) Isi `.env`:
- Update `service.yaml`:
  - `replication.connections[0].uri`: koneksi Postgres source DB dengan user repl (contoh: `postgresql://powersync_repl:change_me@<db-host>:5432/kasirgoplus`)
  - `client_auth.jwks.keys[0].k`: hasil base64url dari `JWT_SECRET`
  - `client_auth.audience`: harus match `POWERSYNC_JWT_AUDIENCE` di backend
  - `client_auth.jwks.keys[0].kid`: harus match `POWERSYNC_JWT_KID` di backend

3) Start:

```bash
docker compose up -d
```

PowerSync service listen di `http://<vps-ip>:8080`.

---

## Sync Streams (Phase 1)
Sync config ada di `sync-config.yaml` (edition: 3) untuk:
- categories (tenant-scoped)
- products (tenant-scoped)
