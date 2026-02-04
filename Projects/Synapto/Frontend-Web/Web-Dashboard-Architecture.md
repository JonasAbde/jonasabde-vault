# Web Dashboard Architecture

**Complete guide to the Synapto Next.js web dashboard for administrators**

---

## Overview

The Synapto web dashboard is a **Next.js 16 application** with React 19 that provides:
- **Admin management** interface for both tenants
- **Real-time analytics** and reporting
- **Inquiry approval** workflow
- **Customer & booking** management
- **Settings & configuration** for tenant customization

**Access**: Web browser (desktop/tablet optimized)

---

## Architecture

### App Router Structure

**Next.js App Router** (file-system based routing):

```
frontend/src/app/
├── layout.tsx              # Root layout
├── globals.css             # Global styles
├── (auth)/                 # Authentication route group
│   └── login/
│       └── page.tsx        # Login page
└── (dashboard)/            # Protected dashboard group
    ├── layout.tsx          # Dashboard layout (sidebar + header)
    ├── page.tsx            # Dashboard home
    ├── analytics/
    │   └── page.tsx
    ├── bookings/
    │   ├── page.tsx
    │   └── [id]/
    │       └── page.tsx
    ├── customers/
    │   ├── page.tsx
    │   └── [id]/
    │       └── page.tsx
    ├── inquiries/
    │   ├── page.tsx
    │   └── [id]/
    │       └── page.tsx
    └── settings/
        └── page.tsx
```

**Route Groups** (`(auth)`, `(dashboard)`):
- Don't affect URL structure
- Share layouts within group
- Organize related routes

---

## State Management

### TanStack React Query

**Why React Query?**
- Server state management
- Automatic caching
- Background refetching
- Optimistic updates
- Request deduplication

### Setup

**Location**: `src/components/providers.tsx`

```typescript
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { useState } from 'react'

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,  // 1 minute
        cacheTime: 5 * 60 * 1000,  // 5 minutes
        refetchOnWindowFocus: false,
        retry: 1,
      },
    },
  }))

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

### Query Hooks

**Example**: Fetching inquiries

```typescript
// src/hooks/use-inquiries.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { api } from '@/lib/api'

export function useInquiries() {
  return useQuery({
    queryKey: ['inquiries'],
    queryFn: async () => {
      const response = await api.get('/inquiries')
      return response.data
    },
  })
}

export function useInquiry(id: string) {
  return useQuery({
    queryKey: ['inquiries', id],
    queryFn: async () => {
      const response = await api.get(`/inquiries/${id}`)
      return response.data
    },
    enabled: !!id,  // Only fetch if ID exists
  })
}

export function useUpdateInquiry() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: async ({ id, data }: { id: string; data: any }) => {
      const response = await api.put(`/inquiries/${id}`, data)
      return response.data
    },
    onSuccess: (data, variables) => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: ['inquiries'] })
      queryClient.invalidateQueries({ queryKey: ['inquiries', variables.id] })
    },
  })
}
```

### Using in Components

```typescript
'use client'

import { useInquiries, useUpdateInquiry } from '@/hooks/use-inquiries'

export default function InquiriesPage() {
  const { data: inquiries, isLoading, error } = useInquiries()
  const updateInquiry = useUpdateInquiry()
  
  if (isLoading) return <LoadingSpinner />
  if (error) return <ErrorMessage error={error} />
  
  const handleApprove = async (id: string) => {
    await updateInquiry.mutateAsync({
      id,
      data: { status: 'approved' }
    })
  }
  
  return (
    <div>
      {inquiries.map(inquiry => (
        <InquiryCard 
          key={inquiry.id} 
          inquiry={inquiry}
          onApprove={handleApprove}
        />
      ))}
    </div>
  )
}
```

---

## API Client

### Axios Instance

**Location**: `src/lib/api.ts`

```typescript
import axios from 'axios'

class ApiClient {
  private client = axios.create({
    baseURL: process.env.NEXT_PUBLIC_API_URL + '/api',
    timeout: 30000,
  })
  
  constructor() {
    // Request interceptor - add auth headers
    this.client.interceptors.request.use(
      (config) => {
        const apiKey = localStorage.getItem('apiKey')
        if (apiKey) {
          config.headers['X-API-Key'] = apiKey
        }
        return config
      },
      (error) => Promise.reject(error)
    )
    
    // Response interceptor - handle auth errors
    this.client.interceptors.response.use(
      (response) => response,
      (error) => {
        if (error.response?.status === 401) {
          // Clear auth and redirect to login
          localStorage.removeItem('apiKey')
          localStorage.removeItem('tenantId')
          window.location.href = '/login'
        }
        return Promise.reject(error)
      }
    )
  }
  
  // Convenience methods
  get<T = any>(url: string, config = {}) {
    return this.client.get<T>(url, config)
  }
  
  post<T = any>(url: string, data?: any, config = {}) {
    return this.client.post<T>(url, data, config)
  }
  
  put<T = any>(url: string, data?: any, config = {}) {
    return this.client.put<T>(url, data, config)
  }
  
  delete<T = any>(url: string, config = {}) {
    return this.client.delete<T>(url, config)
  }
}

export const api = new ApiClient()
```

### Environment Variables

**`.env.local`**:
```bash
NEXT_PUBLIC_API_URL=http://localhost:8000
```

**`.env.production`**:
```bash
NEXT_PUBLIC_API_URL=https://api.synapto.dk
```

---

## Authentication

### API Key Authentication

**Login Flow**:
1. User enters API key
2. Validate key via `GET /tenants/me`
3. Store key + tenant info in localStorage
4. Redirect to dashboard

**Login Component**:

```typescript
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { api } from '@/lib/api'

export default function LoginPage() {
  const [apiKey, setApiKey] = useState('')
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState('')
  const router = useRouter()
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError('')
    
    try {
      // Validate API key
      const response = await api.get('/tenants/me', {
        headers: { 'X-API-Key': apiKey }
      })
      
      // Store credentials
      localStorage.setItem('apiKey', apiKey)
      localStorage.setItem('tenantId', response.data.id)
      localStorage.setItem('tenantName', response.data.name)
      
      // Redirect to dashboard
      router.push('/dashboard')
    } catch (err) {
      setError('Invalid API key')
    } finally {
      setLoading(false)
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="password"
        value={apiKey}
        onChange={(e) => setApiKey(e.target.value)}
        placeholder="Enter API Key"
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Logging in...' : 'Login'}
      </button>
      {error && <p className="error">{error}</p>}
    </form>
  )
}
```

### Auth Guard

**Protect dashboard routes**:

```typescript
// src/components/auth-guard.tsx
'use client'

import { useEffect } from 'react'
import { useRouter } from 'next/navigation'

export function AuthGuard({ children }: { children: React.ReactNode }) {
  const router = useRouter()
  
  useEffect(() => {
    const apiKey = localStorage.getItem('apiKey')
    if (!apiKey) {
      router.push('/login')
    }
  }, [router])
  
  return <>{children}</>
}
```

**Use in dashboard layout**:

```typescript
// src/app/(dashboard)/layout.tsx
import { AuthGuard } from '@/components/auth-guard'
import { Sidebar } from '@/components/sidebar'
import { Header } from '@/components/header'

export default function DashboardLayout({ children }) {
  return (
    <AuthGuard>
      <div className="flex h-screen">
        <Sidebar />
        <div className="flex-1 flex flex-col">
          <Header />
          <main className="flex-1 overflow-y-auto p-6">
            {children}
          </main>
        </div>
      </div>
    </AuthGuard>
  )
}
```

---

## UI Components

### Reusable Components

**Location**: `src/components/`

#### Sidebar

```typescript
// src/components/sidebar.tsx
'use client'

import Link from 'next/link'
import { usePathname } from 'next/navigation'
import { 
  HomeIcon, 
  InboxIcon, 
  UsersIcon, 
  CalendarIcon,
  ChartBarIcon,
  SettingsIcon 
} from 'lucide-react'

const navItems = [
  { href: '/dashboard', icon: HomeIcon, label: 'Dashboard' },
  { href: '/inquiries', icon: InboxIcon, label: 'Inquiries' },
  { href: '/customers', icon: UsersIcon, label: 'Customers' },
  { href: '/bookings', icon: CalendarIcon, label: 'Bookings' },
  { href: '/analytics', icon: ChartBarIcon, label: 'Analytics' },
  { href: '/settings', icon: SettingsIcon, label: 'Settings' },
]

export function Sidebar() {
  const pathname = usePathname()
  const tenantName = localStorage.getItem('tenantName')
  
  return (
    <aside className="w-64 bg-gray-900 text-white p-4">
      <div className="mb-8">
        <h1 className="text-2xl font-bold">Synapto</h1>
        <p className="text-sm text-gray-400">{tenantName}</p>
      </div>
      
      <nav>
        {navItems.map((item) => {
          const Icon = item.icon
          const isActive = pathname === item.href
          
          return (
            <Link
              key={item.href}
              href={item.href}
              className={`flex items-center gap-3 px-4 py-2 rounded-lg mb-1 ${
                isActive 
                  ? 'bg-blue-600' 
                  : 'hover:bg-gray-800'
              }`}
            >
              <Icon className="w-5 h-5" />
              <span>{item.label}</span>
            </Link>
          )
        })}
      </nav>
    </aside>
  )
}
```

#### Header

```typescript
// src/components/header.tsx
'use client'

import { BellIcon, UserCircleIcon } from 'lucide-react'

export function Header() {
  const handleLogout = () => {
    localStorage.clear()
    window.location.href = '/login'
  }
  
  return (
    <header className="h-16 border-b border-gray-200 px-6 flex items-center justify-between">
      <h2 className="text-xl font-semibold">Dashboard</h2>
      
      <div className="flex items-center gap-4">
        <button className="relative p-2 hover:bg-gray-100 rounded-lg">
          <BellIcon className="w-5 h-5" />
          <span className="absolute top-1 right-1 w-2 h-2 bg-red-500 rounded-full" />
        </button>
        
        <button 
          onClick={handleLogout}
          className="flex items-center gap-2 hover:bg-gray-100 px-3 py-2 rounded-lg"
        >
          <UserCircleIcon className="w-5 h-5" />
          <span>Logout</span>
        </button>
      </div>
    </header>
  )
}
```

---

## Styling

### Tailwind CSS

**Configuration**: `tailwind.config.ts`

```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a',
        },
      },
    },
  },
  plugins: [],
}
export default config
```

**Usage**:
```tsx
<div className="flex items-center gap-4 p-6 bg-white rounded-lg shadow-md">
  <h3 className="text-lg font-semibold text-gray-900">Title</h3>
  <button className="px-4 py-2 bg-primary-500 text-white rounded-lg hover:bg-primary-600">
    Action
  </button>
</div>
```

---

## Charts & Analytics

### Recharts

**Example**: Inquiry stats chart

```typescript
'use client'

import { 
  LineChart, 
  Line, 
  XAxis, 
  YAxis, 
  CartesianGrid, 
  Tooltip, 
  ResponsiveContainer 
} from 'recharts'

interface AnalyticsData {
  date: string
  inquiries: number
  bookings: number
}

export function InquiryChart({ data }: { data: AnalyticsData[] }) {
  return (
    <ResponsiveContainer width="100%" height={300}>
      <LineChart data={data}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="date" />
        <YAxis />
        <Tooltip />
        <Line 
          type="monotone" 
          dataKey="inquiries" 
          stroke="#3b82f6" 
          strokeWidth={2}
        />
        <Line 
          type="monotone" 
          dataKey="bookings" 
          stroke="#10b981" 
          strokeWidth={2}
        />
      </LineChart>
    </ResponsiveContainer>
  )
}
```

---

## Build & Deployment

### Development

```bash
# Install dependencies
npm install

# Run dev server
npm run dev

# Open http://localhost:3000
```

### Production Build

```bash
# Build for production
npm run build

# Start production server
npm start
```

### Docker

**Dockerfile** (multi-stage build):

```dockerfile
# Build stage
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-slim AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000
CMD ["node", "server.js"]
```

**Build & run**:
```bash
docker build -t synapto-frontend .
docker run -p 3000:3000 -e NEXT_PUBLIC_API_URL=http://api:8000 synapto-frontend
```

---

## Related Documentation
- [[API-Documentation]] - Backend API reference
- [[Frontend-Web/Components]] - Component library
- [[Frontend-Web/State-Management]] - React Query patterns
- [[Deployment/Deployment-Guide]] - Production deployment

#frontend #nextjs #react #web #dashboard
