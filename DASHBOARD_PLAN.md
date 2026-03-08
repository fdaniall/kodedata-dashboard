# kodedata-dashboard — Project Plan

> **Repo:** `kodedata-dashboard`
> **URL:** `dashboard.kodedata.com`
> **Last updated:** 2026-03-08

---

## 1. Overview

Admin dashboard untuk operator KodeData mengelola seluruh tenant telephony service. Dashboard ini adalah internal tool — hanya diakses oleh operator KodeData, bukan end-user/tenant.

Dashboard dibangun dengan Next.js di Vercel. Next.js API routes berfungsi sebagai proxy layer ke `api.telephony.kodedata.com` — tidak ada VPS backend terpisah untuk dashboard.

---

## 2. Tech Stack

| Kebutuhan | Library | Versi | Catatan |
|-----------|---------|-------|---------|
| Framework | Next.js | **16.1.6** | App Router, Turbopack stable, React Compiler stable |
| Language | TypeScript | **5.9.3** | (TS 6.0 RC belum stable) |
| Styling | Tailwind CSS | **4.2.1** | CSS-first config, **tidak ada `tailwind.config.ts`** |
| Components | shadcn/ui | **CLI v4** | Support Tailwind v4, shadcn CLI v4 (Maret 2026) |
| Auth | NextAuth | **v5 (beta)** | Install via `next-auth@beta` — widely used in prod meski masih beta |
| ORM | Prisma | **7.4.0** | Rust-free, query engine via WebAssembly |
| DB (cloud) | Turso (`@libsql/client`) | **0.17.0** | SQLite cloud untuk Vercel (persistent filesystem) |
| Data fetching | TanStack Query | **5.90.21** | Client-side caching + auto-refetch |
| Validation | Zod | **4.3.6** | Identik dengan versi di telephony service |
| Icons | Lucide React | **0.577.0** | Tree-shakable, bundled via shadcn/ui |
| Date | date-fns | **4.1.0** | Format tanggal/duration, first-class timezone support |
| Charts | Recharts | **3.8.0** | Usage stats per tenant |
| Deployment | Vercel | — | Free tier cukup untuk internal tool |

> **Tailwind v4 — breaking change dari v3:** Tidak ada lagi `tailwind.config.ts`. Semua konfigurasi lewat CSS `@import "tailwindcss"` dan `@theme` di `globals.css`.

> **Prisma v7 — breaking change dari v6:** Engine sekarang WebAssembly (tidak ada binary terpisah). Ships sebagai ES module, kompatibel dengan Bun, Deno, dan Node.

> **NextAuth v5 masih beta** tapi sudah production-ready dan banyak dipakai. Install dengan `npm install next-auth@beta`.

---

## 3. Kenapa Ada Local DB (SQLite/Turso)?

Telephony service mengembalikan `api_key` **sekali saja** saat tenant dibuat (`POST /admin/tenants`). Setelah itu tidak bisa di-retrieve ulang. Dashboard harus menyimpan key ini untuk:

1. Menampilkan call history per-tenant (butuh `x-api-key` tenant tersebut)
2. Menampilkan active calls per-tenant

**SQLite** untuk local dev, **Turso** (SQLite cloud, free tier) untuk Vercel production — karena Vercel serverless tidak persistent filesystem.

---

## 4. Struktur Folder

```
kodedata-dashboard/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   └── login/
│   │   │       └── page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx           ← sidebar + header layout
│   │   │   ├── page.tsx             ← / redirect ke /tenants
│   │   │   ├── tenants/
│   │   │   │   ├── page.tsx         ← list semua tenant
│   │   │   │   ├── new/
│   │   │   │   │   └── page.tsx     ← form create tenant
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx     ← tenant detail
│   │   │   │       └── usage/
│   │   │   │           └── page.tsx ← usage/billing
│   │   │   ├── calls/
│   │   │   │   └── page.tsx         ← all active calls (cross-tenant)
│   │   │   └── settings/
│   │   │       └── page.tsx         ← telephony service status
│   │   ├── api/
│   │   │   ├── auth/
│   │   │   │   └── [...nextauth]/
│   │   │   │       └── route.ts     ← NextAuth handler
│   │   │   └── proxy/
│   │   │       ├── tenants/
│   │   │       │   ├── route.ts     ← GET list, POST create
│   │   │       │   └── [id]/
│   │   │       │       ├── route.ts ← GET detail, PATCH status
│   │   │       │       └── usage/
│   │   │       │           └── route.ts
│   │   │       └── calls/
│   │   │           ├── active/
│   │   │           │   └── route.ts ← per-tenant active calls
│   │   │           └── history/
│   │   │               └── route.ts ← per-tenant call history
│   │   └── layout.tsx               ← root layout (font, metadata)
│   ├── components/
│   │   ├── ui/                      ← shadcn/ui components (auto-generated)
│   │   ├── layout/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Header.tsx
│   │   │   └── PageHeader.tsx
│   │   ├── tenants/
│   │   │   ├── TenantTable.tsx
│   │   │   ├── TenantCard.tsx
│   │   │   ├── TenantStatusBadge.tsx
│   │   │   ├── CreateTenantForm.tsx
│   │   │   └── UpdateStatusDialog.tsx
│   │   ├── calls/
│   │   │   ├── ActiveCallsTable.tsx
│   │   │   ├── CallHistoryTable.tsx
│   │   │   └── CallStatusBadge.tsx
│   │   ├── usage/
│   │   │   ├── UsageStats.tsx
│   │   │   └── UsageChart.tsx
│   │   └── shared/
│   │       ├── StatusIndicator.tsx
│   │       ├── CopyButton.tsx
│   │       ├── ConfirmDialog.tsx
│   │       └── EmptyState.tsx
│   ├── lib/
│   │   ├── auth.ts                  ← NextAuth config
│   │   ├── db.ts                    ← Prisma client singleton
│   │   ├── telephony.ts             ← typed API client (fetch wrapper)
│   │   └── utils.ts                 ← cn(), formatDuration(), dll
│   ├── hooks/
│   │   ├── useTenants.ts
│   │   ├── useActiveCalls.ts
│   │   └── useCallHistory.ts
│   ├── types/
│   │   └── telephony.ts             ← type definitions semua response
│   └── middleware.ts                ← protect dashboard routes
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── .env.local                       ← local dev
├── .env.example
├── next.config.ts
└── package.json
# ↑ Tidak ada tailwind.config.ts — Tailwind v4 konfigurasi via globals.css
```

---

## 5. Database Schema (Prisma)

**File: `prisma/schema.prisma`**

```prisma
datasource db {
  provider = "sqlite"     // ganti "turso" kalau pakai Turso di production
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// Simpan API key tenant (karena telephony service tidak bisa retrieve ulang)
model TenantApiKey {
  id         String   @id @default(cuid())
  tenantId   String   @unique    // UUID dari telephony service
  tenantName String               // cache name untuk display
  apiKey     String               // enkripsi dengan ENCRYPTION_KEY env var
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
}

// Admin user (single user: operator KodeData)
model AdminUser {
  id           String   @id @default(cuid())
  email        String   @unique
  passwordHash String
  createdAt    DateTime @default(now())
}
```

> **Catatan security:** Field `apiKey` di-store di SQLite/Turso. Untuk production, enkripsi field ini dengan `AES-256-GCM` menggunakan `ENCRYPTION_KEY` dari env. Gunakan `node:crypto` built-in.

---

## 6. Environment Variables

**File: `.env.example`**

```bash
# Next.js
NEXTAUTH_SECRET=          # random string 32+ chars: openssl rand -base64 32
NEXTAUTH_URL=             # https://dashboard.kodedata.com (prod) / http://localhost:3000 (dev)

# Admin credentials (single operator login)
ADMIN_EMAIL=admin@kodedata.com
ADMIN_PASSWORD_HASH=      # bcrypt hash dari password: lihat "Setup Commands" section 13

# Telephony service
TELEPHONY_API_URL=        # https://api.telephony.kodedata.com
TELEPHONY_ADMIN_SECRET=   # sama dengan ADMIN_SECRET di telephony .env

# Database
DATABASE_URL=             # file:./dev.db (local) atau libsql://xxx.turso.io (prod)
DATABASE_AUTH_TOKEN=      # hanya untuk Turso, kosongkan untuk SQLite lokal

# Encryption untuk API keys di DB
ENCRYPTION_KEY=           # 32-char hex: openssl rand -hex 32
```

---

## 7. Authentication

**Approach:** NextAuth v5, Credentials Provider, single admin user.

**Flow:**
1. User buka `dashboard.kodedata.com` → redirect ke `/login`
2. Input email + password
3. NextAuth validates: hash input → compare dengan `ADMIN_PASSWORD_HASH` env var (bcrypt)
4. Session JWT disimpan di cookie (httpOnly, secure)
5. `middleware.ts` protect semua routes kecuali `/login` dan `/api/auth/*`

**`src/lib/auth.ts`:**

```typescript
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import bcrypt from 'bcryptjs';

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    Credentials({
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        if (
          credentials.email !== process.env.ADMIN_EMAIL ||
          !credentials.password
        ) return null;

        const match = await bcrypt.compare(
          credentials.password as string,
          process.env.ADMIN_PASSWORD_HASH!
        );
        if (!match) return null;

        return { id: '1', email: process.env.ADMIN_EMAIL, name: 'Admin' };
      },
    }),
  ],
  session: { strategy: 'jwt' },
  pages: { signIn: '/login' },
});
```

**`src/middleware.ts`:**

```typescript
import { auth } from '@/lib/auth';

export default auth((req) => {
  if (!req.auth) {
    return Response.redirect(new URL('/login', req.url));
  }
});

export const config = {
  matcher: ['/((?!login|api/auth|_next|favicon).*)'],
};
```

---

## 8. API Proxy Layer (Next.js API Routes)

Semua call ke telephony service dilakukan **server-side**. `TELEPHONY_ADMIN_SECRET` tidak pernah ke browser.

### `src/lib/telephony.ts` — Typed Client

```typescript
const BASE = process.env.TELEPHONY_API_URL!;
const ADMIN_SECRET = process.env.TELEPHONY_ADMIN_SECRET!;

async function adminFetch<T>(path: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'x-admin-secret': ADMIN_SECRET,
      ...options?.headers,
    },
  });
  if (!res.ok) {
    const err = await res.json();
    throw new Error(err.error || 'Request failed');
  }
  return res.json();
}

async function tenantFetch<T>(
  path: string,
  apiKey: string,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(`${BASE}/api/v1${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': apiKey,
      ...options?.headers,
    },
  });
  if (!res.ok) {
    const err = await res.json();
    throw new Error(err.error || 'Request failed');
  }
  return res.json();
}

export const telephony = { adminFetch, tenantFetch };
```

### Proxy Routes Detail

| Route | Method | Aksi |
|-------|--------|------|
| `/api/proxy/tenants` | GET | `adminFetch('/admin/tenants')` |
| `/api/proxy/tenants` | POST | `adminFetch('/admin/tenants', { method: 'POST', body })` → simpan `api_key` ke DB → return response |
| `/api/proxy/tenants/[id]` | GET | `adminFetch('/admin/tenants/:id')` |
| `/api/proxy/tenants/[id]` | PATCH | `adminFetch('/admin/tenants/:id', { method: 'PATCH', body })` |
| `/api/proxy/tenants/[id]/usage` | GET | `adminFetch('/admin/tenants/:id/usage?month=YYYY-MM')` |
| `/api/proxy/calls/active?tenantId=xxx` | GET | Ambil apiKey dari DB → `tenantFetch('/calls/active', apiKey)` |
| `/api/proxy/calls/history?tenantId=xxx&page=1&limit=20` | GET | Ambil apiKey dari DB → `tenantFetch('/calls/history?page=N&limit=N', apiKey)` |

> **Penting:** Saat `POST /api/proxy/tenants`, simpan `api_key` ke DB **sebelum** return response ke client. Jika DB gagal simpan → return error 500, jangan return `api_key` ke client (karena telephony tidak bisa retrieve ulang).

---

## 9. Pages & Features

### 9.1 `/login`

- Form: email + password, submit button
- Redirect ke `/tenants` jika sudah login
- Error toast jika credentials salah
- Loading state pada tombol submit

---

### 9.2 `/tenants` — Tenant List

**Data:** `GET /api/proxy/tenants`

**UI:**
- Page header: "Tenants" + tombol "+ New Tenant"
- Summary cards (top):
  - Total Tenants
  - Active Tenants
  - Suspended Tenants
- Tabel:

| Kolom | Keterangan |
|-------|-----------|
| Name | Nama tenant |
| Status | Badge: Active (hijau) / Suspended (kuning) / Inactive (abu) |
| Webhook URL | Truncated atau "—" |
| Created At | Format: `DD MMM YYYY` |
| Actions | [Detail] [Suspend/Activate] |

- Empty state jika belum ada tenant
- Click row → navigate ke `/tenants/[id]`
- "Suspend/Activate" → confirm dialog → `PATCH /api/proxy/tenants/[id]` → refresh

---

### 9.3 `/tenants/new` — Create Tenant

**Form fields:**
- `name` (required, text input)
- `webhook_url` (optional, URL input)
- `webhook_secret` (optional, hint: "Untuk verifikasi signature webhook")

**Behavior on success:**
1. `POST /api/proxy/tenants`
2. Proxy simpan `api_key` ke DB
3. Tampilkan **modal khusus**:
   - "Tenant berhasil dibuat!"
   - `API Key: [64-char hex]` dengan tombol copy
   - Warning merah bold: "Simpan API key ini sekarang. Tidak bisa ditampilkan lagi."
   - Tombol "Selesai" → redirect ke `/tenants/[id]`
4. Setelah modal ditutup → API key tidak bisa dilihat lagi dari UI (hanya di DB terenkripsi)

---

### 9.4 `/tenants/[id]` — Tenant Detail

**Data fetched:**
- `GET /api/proxy/tenants/[id]` — info tenant
- `GET /api/proxy/calls/active?tenantId=[id]` — active calls (refetch tiap 10s)
- `GET /api/proxy/calls/history?tenantId=[id]&page=1&limit=10` — recent calls

**Section 1 — Info Card:**
- Tenant ID (dengan copy button)
- Name
- Status badge + tombol "Ubah Status"
- Webhook URL
- Created At

**Section 2 — Active Calls** (dengan badge count di heading):

| Kolom | Keterangan |
|-------|-----------|
| Call ID | Truncated |
| Destination | Nomor tujuan |
| Agent | Agent ID |
| Status | Badge animasi |
| Duration | Live counter (hh:mm:ss) |
| Actions | Tombol [Hangup] dengan confirm dialog |

- Auto-refresh setiap 10 detik
- Empty state: "Tidak ada panggilan aktif"

**Section 3 — Recent Calls (10 baris):**

| Kolom | Keterangan |
|-------|-----------|
| Call ID | — |
| Destination | — |
| Status | Badge |
| Direction | Inbound / Outbound |
| Duration | mm:ss |
| Started At | Relative time + tooltip absolute |
| Hangup Cause | — |

- Link "Lihat semua →" ke `/tenants/[id]/usage`

---

### 9.5 `/tenants/[id]/usage` — Usage & History

**Data:**
- `GET /api/proxy/tenants/[id]/usage?month=YYYY-MM`
- `GET /api/proxy/calls/history?tenantId=[id]&page=N&limit=20`

**Section 1 — Month Selector:**
- `<input type="month">` atau dropdown
- Default: bulan berjalan

**Section 2 — Summary Cards:**
- Total Calls
- Answered Calls
- Total Minutes
- Answer Rate (%) = `answered / total * 100`

**Section 3 — Chart (Recharts):**
- Bar chart: jumlah calls per hari dalam bulan dipilih
- Data diambil dari `recent_calls`, di-group by `started_at` date

**Section 4 — Full Call History (paginated):**

| Kolom | Keterangan |
|-------|-----------|
| Call ID | — |
| Destination | — |
| Status | Badge |
| Direction | Inbound / Outbound |
| Duration | mm:ss |
| Started At | `DD MMM YYYY HH:mm` |
| Hangup Cause | — |

- Pagination controls: prev/next, "Page X of Y"

---

### 9.6 `/calls` — Active Calls (Cross-Tenant)

> Semua active calls dari **semua tenant** dalam satu tampilan.

**Data:** Loop semua tenantId dari DB → `Promise.allSettled()` → hit `/api/proxy/calls/active?tenantId=xxx` per tenant → merge hasil.

**UI:**
- Auto-refresh toggle (on/off, interval 10s)
- Summary cards: Total Active, Bridged, Ringing
- Tabel:

| Kolom | Keterangan |
|-------|-----------|
| Tenant | Nama tenant |
| Call ID | — |
| Destination | — |
| Agent | — |
| Status | Badge animasi |
| Duration | Live counter |
| Actions | [Hangup] dengan confirm dialog |

- Empty state: "Tidak ada panggilan aktif saat ini"

---

### 9.7 `/settings` — Service Status

**Data:** `GET https://api.telephony.kodedata.com/api/v1/status`

> Endpoint ini butuh `x-api-key`. Sementara pakai API key tenant pertama yang aktif dari DB. Idealnya minta tambahan `GET /admin/status` di telephony service (pakai `x-admin-secret` — tidak perlu tenant key).

**UI:**
- Card: **ARI Connection** — Connected (hijau + dot animasi) / Disconnected (merah)
- Card: **SIP Trunk** — Online/Offline, registration status, last checked
- Card: **Active Calls** — jumlah real-time
- List: All Endpoints + status masing-masing
- Tombol "Refresh"

---

## 10. Shared Components

### `TenantStatusBadge`
```
active    → Badge hijau
suspended → Badge kuning/orange
inactive  → Badge abu-abu
```

### `CallStatusBadge`
```
initiating  → Biru (pulse animation)
ringing     → Biru
answered    → Hijau (pulse animation)
bridged     → Hijau
ended       → Abu-abu
failed      → Merah
busy        → Orange
no_answer   → Kuning
canceled    → Abu-abu
```

### `CopyButton`
- Props: `value: string`
- Click → `navigator.clipboard.writeText(value)` → icon berubah ke checkmark selama 2 detik

### `ConfirmDialog`
- Props: `title`, `description`, `onConfirm`, `variant` ('destructive' | 'default')
- Dipakai untuk: suspend tenant, hangup call

### `LiveDuration`
- Client component, `'use client'`
- Props: `startTime: Date`
- Update elapsed time tiap detik dengan `setInterval`
- Format: `mm:ss` atau `hh:mm:ss`

### `EmptyState`
- Props: `title`, `description`, `action?` (ReactNode)

---

## 11. Hooks (TanStack Query)

```typescript
// hooks/useTenants.ts
export function useTenants() {
  return useQuery({
    queryKey: ['tenants'],
    queryFn: () => fetch('/api/proxy/tenants').then(r => r.json()),
    staleTime: 30_000,
  });
}

// hooks/useActiveCalls.ts
export function useActiveCalls(tenantId: string) {
  return useQuery({
    queryKey: ['calls', 'active', tenantId],
    queryFn: () => fetch(`/api/proxy/calls/active?tenantId=${tenantId}`).then(r => r.json()),
    refetchInterval: 10_000,
  });
}

// hooks/useAllActiveCalls.ts
export function useAllActiveCalls() {
  return useQuery({
    queryKey: ['calls', 'active', 'all'],
    queryFn: () => fetch('/api/proxy/calls/active').then(r => r.json()),
    refetchInterval: 10_000,
  });
}

// hooks/useCallHistory.ts
export function useCallHistory(tenantId: string, page: number, limit = 20) {
  return useQuery({
    queryKey: ['calls', 'history', tenantId, page],
    queryFn: () =>
      fetch(`/api/proxy/calls/history?tenantId=${tenantId}&page=${page}&limit=${limit}`)
        .then(r => r.json()),
    placeholderData: keepPreviousData,
  });
}
```

---

## 12. Type Definitions

**`src/types/telephony.ts`:**

```typescript
export type TenantStatus = 'active' | 'suspended' | 'inactive';
export type CallDirection = 'inbound' | 'outbound';
export type CallStatus =
  | 'initiating' | 'ringing' | 'answered' | 'bridged'
  | 'ended' | 'failed' | 'busy' | 'no_answer' | 'canceled';

export interface Tenant {
  id: string;
  name: string;
  status: TenantStatus;
  webhook_url: string | null;
  created_at: string;
}

export interface CreateTenantResponse {
  message: string;
  tenant: Pick<Tenant, 'id' | 'name' | 'status'>;
  api_key: string;
  warning: string;
}

export interface CallRecord {
  id: string;
  destination: string;
  caller_id: string | null;
  status: CallStatus;
  direction: CallDirection;
  started_at: string;
  answered_at: string | null;
  ended_at: string | null;
  duration: number;
  hangup_cause: string | null;
  recording_url: string | null;
  agent_id: string | null;
  bridge_id: string | null;
  tenant_id: string;
}

export interface Pagination {
  page: number;
  limit: number;
  total: number;
  pages: number;
}

export interface CallHistoryResponse {
  data: CallRecord[];
  pagination: Pagination;
}

export interface UsageSummary {
  tenant: { id: string; name: string };
  month: string;
  summary: {
    total_calls: number;
    answered_calls: number;
    total_minutes: number;
    updated_at: string | null;
  };
  recent_calls: CallRecord[];
}

export interface ServiceStatus {
  service: string;
  ready_to_call: boolean;
  trunk_info: {
    resource: string;
    state: 'online' | 'offline';
    registration: { status: string; lastChecked: string };
  } | null;
  active_calls: number;
  all_endpoints: Array<{ resource: string; state: string }>;
}
```

---

## 13. Setup Commands (Untuk Developer)

```bash
# 1. Buat project Next.js 16 baru (JANGAN clone dulu, init dulu)
npx create-next-app@latest kodedata-dashboard \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --turbopack
# Tailwind v4 sudah otomatis disetup oleh create-next-app terbaru
# Tidak ada tailwind.config.ts — konfigurasi ada di src/app/globals.css

# 2. Masuk ke folder
cd kodedata-dashboard

# 3. Install dependencies
npm install next-auth@beta                        # NextAuth v5 (masih beta tapi prod-ready)
npm install @prisma/client@latest prisma@latest   # Prisma v7
npm install @prisma/adapter-libsql @libsql/client # Turso adapter
npm install @tanstack/react-query@latest          # TanStack Query v5
npm install zod@latest                            # Zod v4
npm install date-fns@latest                       # date-fns v4
npm install recharts@latest                       # Recharts v3
npm install bcryptjs
npm install -D @types/bcryptjs

# 4. Copy env
cp .env.example .env.local
# Isi semua values di .env.local

# 5. Generate password hash untuk ADMIN_PASSWORD_HASH
node -e "const b=require('bcryptjs'); b.hash('PASSWORD_KAMU', 12).then(console.log)"
# Copy output ke ADMIN_PASSWORD_HASH di .env.local

# 6. Generate secrets
openssl rand -base64 32    # → NEXTAUTH_SECRET
openssl rand -hex 32       # → ENCRYPTION_KEY

# 7. Setup Prisma v7
npx prisma init
# Edit prisma/schema.prisma (lihat section 5)
npx prisma migrate dev --name init
npx prisma generate

# 8. Init shadcn/ui (CLI v4, support Tailwind v4)
npx shadcn@latest init
# Pilih: Default style, Zinc color (atau Neutral), Yes untuk CSS variables

# 9. Install shadcn components yang dibutuhkan
npx shadcn@latest add button card table badge dialog form input label select sonner sheet skeleton tooltip

# 10. Run dev (Turbopack aktif by default di Next.js 16)
npm run dev
# Buka http://localhost:3000
```

---

## 14. Deployment ke Vercel

```bash
# 1. Install Vercel CLI
npm i -g vercel

# 2. Deploy
vercel

# 3. Set environment variables di Vercel Dashboard:
#    NEXTAUTH_SECRET, NEXTAUTH_URL, ADMIN_EMAIL, ADMIN_PASSWORD_HASH,
#    TELEPHONY_API_URL, TELEPHONY_ADMIN_SECRET,
#    DATABASE_URL, DATABASE_AUTH_TOKEN, ENCRYPTION_KEY
```

### Setup Turso (DB untuk Vercel)

```bash
# 1. Install Turso CLI
curl -sSfL https://get.tur.so/install.sh | bash

# 2. Login
turso auth login

# 3. Buat database
turso db create kodedata-dashboard

# 4. Ambil URL → isi DATABASE_URL di Vercel
turso db show kodedata-dashboard --url
# Output contoh: libsql://kodedata-dashboard-xxx.turso.io

# 5. Buat token → isi DATABASE_AUTH_TOKEN di Vercel
turso db tokens create kodedata-dashboard

# 6. Update prisma/schema.prisma untuk Turso:
# datasource db {
#   provider  = "sqlite"
#   url       = env("DATABASE_URL")
# }
# (adapter libsql di-pass saat inisialisasi PrismaClient — lihat src/lib/db.ts)

# 7. src/lib/db.ts untuk Prisma v7 + Turso:
# import { createClient } from '@libsql/client'
# import { PrismaLibSQL } from '@prisma/adapter-libsql'
# import { PrismaClient } from '@prisma/client'
#
# const libsql = createClient({
#   url: process.env.DATABASE_URL!,
#   authToken: process.env.DATABASE_AUTH_TOKEN,
# })
# const adapter = new PrismaLibSQL(libsql)
# export const db = new PrismaClient({ adapter })
```

### Custom Domain

1. Vercel Dashboard → Project → Settings → Domains
2. Add `dashboard.kodedata.com`
3. Di DNS provider tambah CNAME: `dashboard` → `cname.vercel-dns.com`

---

## 15. Development Phases

### Phase 1 — Foundation
- [ ] Init Next.js 16 project dengan TypeScript + Tailwind v4 (`create-next-app@latest`)
- [ ] Setup shadcn/ui (init + install components)
- [ ] Setup Prisma schema + migration
- [ ] Implement NextAuth (credentials provider)
- [ ] Buat `/login` page
- [ ] Setup `middleware.ts` (route protection)
- [ ] Buat layout: sidebar + header

**Sidebar navigation items:**
- Tenants
- Active Calls
- Settings

### Phase 2 — Tenant Management

- [ ] `src/lib/telephony.ts` — `adminFetch` + `tenantFetch`
- [ ] `/api/proxy/tenants` — GET + POST (dengan DB save)
- [ ] `/api/proxy/tenants/[id]` — GET + PATCH
- [ ] `/tenants` page — list + summary cards + tabel
- [ ] `/tenants/new` page — form + modal API key display (one-time)
- [ ] `/tenants/[id]` page — info card + status update dialog

### Phase 3 — Calls & Usage

- [ ] `/api/proxy/calls/active` + `/history` proxy routes
- [ ] `/tenants/[id]` — section active calls (auto-refresh) + recent calls
- [ ] `/tenants/[id]/usage` page — summary cards + chart + paginated history
- [ ] `/calls` page — cross-tenant active calls (Promise.allSettled)

### Phase 4 — Polish

- [ ] `/settings` page — service status
- [ ] Error boundaries + toast notifications (Sonner)
- [ ] Loading skeletons untuk semua tables/cards
- [ ] Mobile responsive sidebar (Sheet component dari shadcn)
- [ ] Tambah `GET /admin/status` di telephony service (tidak perlu tenant key)

---

## 16. Catatan Penting

1. **API key storage saat create tenant:** Proxy route harus simpan `api_key` ke DB **sebelum** return response. Jika DB gagal → return 500, jangan return `api_key` ke client.

2. **Cross-tenant active calls:** Gunakan `Promise.allSettled()` bukan `Promise.all()`. Satu tenant error tidak boleh block semua.

3. **`/settings` sementara:** `GET /api/v1/status` butuh `x-api-key` tenant. Workaround: ambil API key tenant pertama yang aktif dari DB. Proper fix: tambah `GET /admin/status` di telephony (Phase 4).

4. **Rate limiting telephony:** 50 req/min per IP. Vercel serverless semua keluar dari IP Vercel. Monitor jika ada issue, atau minta whitelist IP Vercel di telephony service.

5. **Turso vs SQLite lokal:** Wajib Turso untuk Vercel deployment (filesystem tidak persistent). Untuk self-host di VPS, SQLite file biasa cukup.

6. **Session expiry:** Default NextAuth JWT session 30 hari. Sesuaikan di `auth.ts` → `session: { strategy: 'jwt', maxAge: 86400 }` (24 jam) jika diinginkan lebih ketat.
