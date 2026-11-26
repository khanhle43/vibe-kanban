# Deploy Vibe Kanban: Vercel + Supabase + Render

Hướng dẫn này deploy **bản web/cloud của Vibe Kanban** (`crates/remote` + `packages/remote-web`).

> Vibe Kanban gồm 2 phần:
> - **App local** (`npx vibe-kanban`): chạy coding agent (Claude Code, Codex…) trên máy của bạn. Phần này **phải chạy local**, không deploy lên cloud được.
> - **Remote/cloud** (phần được deploy ở đây): kanban dùng chung cho team, đăng nhập GitHub, đồng bộ realtime. App local có thể kết nối tới nó qua `VK_SHARED_API_BASE`.

## Kiến trúc

```
Trình duyệt ──> Vercel  (SPA packages/remote-web)
                  └─ /v1/*  rewrite ──> Render (Rust API, crates/remote)
                                          ├─ Supabase Postgres  (session pooler, IPv4)
                                          └─ ElectricSQL        (sync realtime)
                                               └─ Supabase Postgres (direct connection, logical replication)
```

Vì sao không chạy hết trên Vercel? Backend là một server Rust chạy liên tục, kèm ElectricSQL
(service Elixir giữ kết nối logical replication lâu dài). Vercel chỉ chạy serverless function
và file tĩnh, nên Vercel host frontend và proxy `/v1/*` sang backend.

| Thành phần | Nơi chạy | Cấu hình |
| --- | --- | --- |
| Frontend | Vercel project `vibe-kanban` (team `khanhle43`) | `vercel.json` |
| Database | Supabase project `vibe-kanban` (`raishyyxvxpjxslsnrcf`, Singapore) | migrations trong `crates/remote/migrations` (server tự chạy khi khởi động) |
| API | Render web service `vibe-kanban-api-khanhle43` | `render.yaml`, `crates/remote/Dockerfile.render` |
| Sync | Electric Cloud (khuyến nghị) hoặc Electric tự host | `ELECTRIC_URL`, `ELECTRIC_SECRET` |

## 1. Supabase (đã làm sẵn)

- Đã tạo project `vibe-kanban`, Postgres 17, `wal_level=logical`.
- Đã tạo database phụ rỗng tên `remote`. Database này chỉ để migration
  `20251127000000_electric_support.sql` (có lệnh `GRANT CONNECT ON DATABASE remote`)
  chạy được mà không phải sửa file migration. Dữ liệu thật nằm trong database `postgres`.
- Đã thử trước các lệnh cần quyền cao trên Supabase (`CREATE ROLE ... REPLICATION`,
  `CREATE PUBLICATION`, `REPLICA IDENTITY FULL`) và tất cả đều chạy được.

Chuỗi kết nối (Supabase Dashboard → **Connect**):

- **Session pooler** (IPv4, cho Render API):
  `postgresql://postgres.raishyyxvxpjxslsnrcf:<DB_PASSWORD>@aws-<n>-ap-southeast-1.pooler.supabase.com:5432/postgres?sslmode=require`
- **Direct connection** (IPv6, cho Electric):
  `postgresql://postgres:<DB_PASSWORD>@db.raishyyxvxpjxslsnrcf.supabase.co:5432/postgres?sslmode=require`

## 2. ElectricSQL

Electric cần **direct connection** (logical replication không đi qua pooler được), mà direct
connection của Supabase gói free **chỉ có IPv6**. Render **không có IPv6 đi ra**,
nên Electric **không thể** chạy trên Render nếu không có add-on IPv4 của Supabase.

**Cách A: Electric Cloud (khuyến nghị)**
1. Đăng ký tại <https://dashboard.electric-sql.cloud>, tạo **Source** loại Postgres.
2. Database URL: dùng chuỗi **Direct connection** ở trên (user `postgres`).
3. Lấy `source_id` và `secret`, rồi đặt trên Render:
   - `ELECTRIC_URL=https://api.electric-sql.cloud?source_id=<SOURCE_ID>`
   - `ELECTRIC_SECRET=<SECRET>`
4. Nếu Electric Cloud dùng publication tên khác `electric_publication_default`, đặt thêm
   `ELECTRIC_PUBLICATION_NAMES=<tên publication>`. Server sẽ tự đồng bộ danh sách bảng sang publication đó.

**Cách B: Tự host `electricsql/electric:1.4.13`** trên host có IPv6 (Fly.io, VPS…), hoặc trên
Render nếu bật Supabase IPv4 add-on (cần gói Pro). Biến môi trường dùng cùng giá trị như trong `crates/remote/docker-compose.yml`, chỉ thay:
`DATABASE_URL=postgresql://electric_sync:<ELECTRIC_ROLE_PASSWORD>@db.raishyyxvxpjxslsnrcf.supabase.co:5432/postgres?sslmode=require`
và đặt `ELECTRIC_SECRET` (bỏ `AUTH_MODE: insecure` khi chạy public).

## 3. GitHub OAuth App

GitHub → Settings → Developer settings → OAuth Apps → **New OAuth App**
- Homepage URL: `https://vibe-kanban-khanhle43.vercel.app`
- Authorization callback URL: `https://vibe-kanban-khanhle43.vercel.app/v1/oauth/github/callback`

Lưu lại Client ID và Client Secret.

## 4. Render (API)

1. <https://dashboard.render.com> → **New → Blueprint** → chọn repo `khanhle43/vibe-kanban`
   (branch có file `render.yaml`).
2. Điền các biến `sync: false`:
   - `SERVER_DATABASE_URL`: chuỗi Session pooler
   - `VIBEKANBAN_REMOTE_JWT_SECRET`: kết quả `openssl rand -base64 48`
   - `ELECTRIC_URL`, `ELECTRIC_SECRET`: lấy ở bước 2
   - `GITHUB_OAUTH_CLIENT_ID`, `GITHUB_OAUTH_CLIENT_SECRET`: lấy ở bước 3
3. Lần build đầu (Rust release) mất khoảng 15–30 phút. Kiểm tra:
   `https://vibe-kanban-api-khanhle43.onrender.com/v1/health`

> Nếu Render đặt cho service một URL khác (ví dụ thêm hậu tố ngẫu nhiên), hãy sửa
> `destination` trong `vercel.json` rồi redeploy Vercel.
>
> Gói free của Render sẽ "ngủ" sau 15 phút không có request, và lần gọi đầu tiên sau đó mất khoảng 1 phút.
> Nên dùng gói Starter nếu chạy thật.

## 5. Vercel (frontend, đã làm sẵn)

- Project `vibe-kanban`, domain production: <https://vibe-kanban-khanhle43.vercel.app>
- `vercel.json`: build `packages/remote-web`, rewrite `/v1/*` sang Render, các đường dẫn còn lại trả về `index.html` (SPA fallback).
- Frontend gọi API bằng đường dẫn tương đối (`window.location.origin`). Nhờ vậy cookie và OAuth chạy cùng origin, không cần cấu hình CORS.
- `crates/remote/src/routes/electric_proxy.rs` gắn header `CDN-Cache-Control: no-store` để CDN của Vercel
  không bao giờ cache dữ liệu sync của từng người dùng.

## Kết nối app local tới server này

```bash
VK_SHARED_API_BASE=https://vibe-kanban-khanhle43.vercel.app npx vibe-kanban
```
