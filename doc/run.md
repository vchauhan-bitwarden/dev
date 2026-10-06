# Bitwarden Local Development — Dev Quick Reference

## 💳 Test Payment Cards

| Card       | Number                | CVC          | Expiry          |
| ---------- | --------------------- | ------------ | --------------- |
| Visa       | `4111 1111 1111 1111` | Any 3 digits | Any future date |
| Mastercard | `5555 5555 5555 4444` | Any 3 digits | Any future date |

## 👤 Admin Roles

| Role             | Permission Key               | Test Email          |
| ---------------- | ---------------------------- | ------------------- |
| Owner            | `adminSettings:role:owner`   | `owner@localhost`   |
| Admin            | `adminSettings:role:admin`   | `admin@localhost`   |
| Customer Success | `adminSettings:role:cs`      | `cs@localhost`      |
| Billing          | `adminSettings:role:billing` | `billing@localhost` |
| Sales            | `adminSettings:role:sales`   | `sales@localhost`   |

---

# 🖥️ Server

Run each component in a **separate terminal**.

### Identity

```bash
cd ~/Bitwarden/server/src/Identity
dotnet restore
dotnet run
```

### API

```bash
cd ~/Bitwarden/server/src/Api
dotnet restore
dotnet run
```

### Admin Console

```bash
cd ~/Bitwarden/server/src/Admin
dotnet restore
npm ci
dotnet build
npm run build
dotnet run
```

---

# 🌐 Client — Web

### Normal watch mode

```bash
cd ~/Bitwarden/clients/apps/web
npm run build:bit:watch
npm run build:bit:dev:watch
```

**Watch mode:** change code → build runs automatically → updated output.

---

# 🧹 Client Clean Reinstall

Use when dependencies/build state becomes broken.

```bash
cd ~/Bitwarden/clients

rm -rf node_modules .nx apps/web/build apps/browser/build apps/cli/build

npm cache clean --force
npm cache verify
npm ci
```

Then:

```bash
cd apps/web
npm run build:oss:watch
```

---

# 🗄️ Reset Local Database

⚠️ **DESTRUCTIVE — deletes local database data.**

```bash
cd ~/Bitwarden/server/dev
```

### 1. Remove containers

```bash
docker compose --profile cloud --profile ef rm -sf \
  mssql postgres mysql mariadb redis
```

### 2. Delete database volumes

```bash
docker volume rm \
  bitwardenserver_mssql_dev_data \
  bitwardenserver_postgres_dev_data \
  bitwardenserver_mysql_dev_data \
  bitwardenserver_mariadb_dev_data \
  bitwardenserver_redis_data
```

### 3. Start fresh databases

```bash
docker compose --profile cloud --profile ef up -d \
  mssql postgres mysql mariadb redis
```

### 4. Check MSSQL

```bash
docker logs bitwardenserver-mssql-1
```

Wait until MSSQL is ready.

### 5. Restore .NET tools

```bash
cd ~/Bitwarden/server
dotnet tool restore
cd dev
```

### 6. Run migrations

```bash
pwsh ./migrate.ps1 -all
```

### 7. Optional: seed test data

```bash
pwsh ./seed.ps1
```

---

# ⚡ Daily Quick Start

### Terminal 1 — Server

```bash
cd ~/Bitwarden/server
dotnet run
```

### Terminal 2 — Identity

```bash
cd ~/Bitwarden/server/src/Identity
dotnet run
```

### Terminal 3 — API

```bash
cd ~/Bitwarden/server/src/Api
dotnet run
```

### Terminal 4 — Admin

```bash
cd ~/Bitwarden/server/src/Admin
dotnet build
npm run build
```

### Terminal 5 — Web

```bash
cd ~/Bitwarden/clients/apps/web
npm run build:bit:watch
```

---

## 🔥 Most Useful Commands

| Task                | Command                                                                                |
| ------------------- | -------------------------------------------------------------------------------------- |
| Restore .NET        | `dotnet restore`                                                                       |
| Run .NET project    | `dotnet run`                                                                           |
| Build .NET          | `dotnet build`                                                                         |
| Install Node deps   | `npm ci`                                                                               |
| Build Web watch     | `npm run build:bit:watch`                                                              |
| Build OSS watch     | `npm run build:oss:watch`                                                              |
| Restore .NET tools  | `dotnet tool restore`                                                                  |
| Run migrations      | `pwsh ./migrate.ps1 -all`                                                              |
| Seed test data      | `pwsh ./seed.ps1`                                                                      |
| Check MSSQL logs    | `docker logs bitwardenserver-mssql-1`                                                  |
| Start DB containers | `docker compose --profile cloud --profile ef up -d mssql postgres mysql mariadb redis` |

> **Tip:** Keep this file as your **daily development cheat sheet**. Use the official setup documentation when doing a fresh machine setup or troubleshooting something not covered here.

# 🔀 Git Quick Reference

### Update `main`

```bash
git fetch --all && git checkout main && git reset --hard origin/main
```

### Reset current branch to common ancestor with `main`

```bash
git reset $(git merge-base origin/main $(git rev-parse --abbrev-ref HEAD))
```

### Update current branch with `main`

```bash
current=$(git branch --show-current) && \
git fetch --all && \
git checkout main && \
git reset --hard origin/main && \
git checkout $current && \
git merge main
```

### Commit & force push

```bash
git add . && git commit -m "Update" --no-verify && git push --force-with-lease
```
