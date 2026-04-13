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
Sesuaikan nama database & schema. Pastikan daftar tabel di publication match dengan stream yang ada di `sync-config.yaml`.

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
  products,
  transactions,
  expenses,
  customers,
  discounts,
  business_settings,
  printer_settings;
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

## Sync Streams
Sync config ada di `sync-config.yaml` (edition: 3) untuk:
- categories (tenant-scoped)
- products (tenant-scoped)
- transactions (tenant-scoped)
- expenses (tenant-scoped)
- customers (tenant-scoped)
- discounts (tenant-scoped)
- business_settings (tenant-scoped)
- printer_settings (tenant-scoped)

---

## Troubleshooting: Data ada di Postgres, tapi mobile 0 (setelah reset/recreate DB)
Gejala umum:
- Di Postgres (source DB) data produk/kategori/transaksi ada (COUNT > 0).
- Di mobile setelah logout/login atau sync ulang, data tampil 0.
- Log PowerSync mengandung error seperti:
  - `permission denied for schema public`
  - `Replication error ... permission denied for schema public`

Penyebab paling sering:
- Setelah reset DB / recreate table, privilege untuk user replication (`powersync_repl`) hilang, sehingga PowerSync tidak bisa baca schema/tabel untuk logical replication.

### 1) Konfirmasi user DB PowerSync yang dipakai
User DB yang dipakai PowerSync ada di `service.yaml`, pada `replication.connections[0].uri`.

Contoh:
- `postgresql://powersync_repl:change_me@kasirgoplus-postgres:5432/kasirgoplus`
  - user: `powersync_repl`
  - db: `kasirgoplus`

Cek cepat:
```bash
cd ~/kasirgoplus-powersync
grep -nE "replication:|postgresql://|connections:|uri:" -n service.yaml
```

### 2) Cek log PowerSync untuk memastikan error permission
```bash
cd ~/kasirgoplus-powersync
sudo docker compose logs -f --tail 200 powersync
```

### 3) Fix privilege Postgres (GRANT) untuk user replication
Jalankan di host yang punya akses ke container Postgres (contoh container: `kasirgoplus-postgres`).

Ganti `powersync_repl` dan `kasirgoplus` sesuai `service.yaml`.
```bash
sudo docker exec -it kasirgoplus-postgres psql -U postgres -d kasirgoplus -v ON_ERROR_STOP=1 -c "
GRANT CONNECT ON DATABASE kasirgoplus TO powersync_repl;
GRANT USAGE ON SCHEMA public TO powersync_repl;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO powersync_repl;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO powersync_repl;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO powersync_repl;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO powersync_repl;

ALTER ROLE powersync_repl WITH REPLICATION;
"
```

### 4) Pastikan publication untuk PowerSync masih ada
Reset DB sering menghapus publication, padahal PowerSync butuh logical replication.

```bash
sudo docker exec -it kasirgoplus-postgres psql -U postgres -d kasirgoplus -c "SELECT pubname, puballtables FROM pg_publication;"
```

Jika publication `powersync` hilang, buat ulang (pastikan tabel sesuai yang di-sync di `sync-config.yaml`):
```bash
sudo docker exec -it kasirgoplus-postgres psql -U postgres -d kasirgoplus -v ON_ERROR_STOP=1 -c "
DROP PUBLICATION IF EXISTS powersync;
CREATE PUBLICATION powersync FOR TABLE
  categories,
  products,
  transactions,
  expenses,
  customers,
  discounts,
  business_settings,
  printer_settings;
"
```

Alternatif paling simpel (lebih permisif): publication untuk semua tabel di schema `public`.
```bash
sudo docker exec -it kasirgoplus-postgres psql -U postgres -d kasirgoplus -v ON_ERROR_STOP=1 -c "
DROP PUBLICATION IF EXISTS powersync;
CREATE PUBLICATION powersync FOR ALL TABLES;
"
```

### 5) Restart PowerSync dan verifikasi
```bash
cd ~/kasirgoplus-powersync
sudo docker compose restart powersync
sudo docker compose logs -f --tail 200 powersync
```

Expected:
- Error `permission denied for schema public` hilang.
- PowerSync mulai checkpoint / replicate normal.
- Mobile setelah sync/login ulang mulai terisi data.
