# M4 Frontend Architecture Plan

> **Purpose:** Define the complete architecture for the new BeastInsights frontend before writing any code.
> **Last updated:** 2026-02-02

---

## 1. Overview

### 1.1 Goals
- Migrate all existing functionality from `beastinsights-old`
- Replace Power BI reports with JSON-driven dynamic rendering
- Use Tremor as primary UI library (shadcn for sidebar only)
- Create a global component layer for single-point theme changes
- Replicate exact colors and sizing from old design system

### 1.2 Tech Stack

| Layer | Technology | Notes |
|-------|------------|-------|
| Framework | Next.js 15 LTS | App Router |
| React | React 18 LTS | Stable version |
| UI Library | Tremor | Primary - cards, charts, tables |
| Navigation UI | shadcn/ui | Sidebar only |
| State | Redux Toolkit + RTK Query | Caching, async |
| HTTP | Axios | Keep existing pattern |
| Styling | Tailwind CSS | With design tokens |
| Icons | Lucide React | Same as old |

### 1.3 Repository
- **Location:** `/Users/bhautikkhunt/BK/Beast-Insights/beast-ideation/beastinsights/`
- **Status:** Empty (fresh setup)

---

## 2. Folder Structure

```
beastinsights/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Auth group (no sidebar)
│   │   ├── login/page.tsx
│   │   ├── forgot-password/page.tsx
│   │   ├── reset-password/page.tsx
│   │   └── onboard/page.tsx
│   ├── (dashboard)/              # Dashboard group (with sidebar)
│   │   ├── layout.tsx            # Sidebar + Header wrapper
│   │   ├── home/page.tsx
│   │   ├── reports/
│   │   │   ├── page.tsx          # Reports landing
│   │   │   └── [templateKey]/page.tsx  # Dynamic report route
│   │   ├── routing-insights/page.tsx
│   │   ├── routing-setup/page.tsx
│   │   ├── integration/page.tsx
│   │   ├── schedule/page.tsx
│   │   ├── users/page.tsx
│   │   ├── reporting-setup/page.tsx
│   │   └── billing/page.tsx
│   ├── (admin)/                  # Admin group
│   │   ├── layout.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── filter-config/page.tsx
│   │   ├── etl-logs/page.tsx
│   │   └── maintenance/page.tsx
│   ├── layout.tsx                # Root layout
│   ├── providers.tsx             # Redux + Theme providers
│   └── globals.css               # Design tokens
│
├── components/
│   ├── ui/                       # Global UI wrappers (TREMOR)
│   │   ├── button.tsx            # Wraps Tremor Button
│   │   ├── card.tsx              # Wraps Tremor Card
│   │   ├── input.tsx             # Wraps Tremor TextInput
│   │   ├── select.tsx            # Wraps Tremor Select
│   │   ├── badge.tsx             # Wraps Tremor Badge
│   │   ├── table.tsx             # Wraps Tremor Table
│   │   ├── tabs.tsx              # Wraps Tremor TabGroup
│   │   ├── dialog.tsx            # Wraps Tremor Dialog
│   │   ├── dropdown.tsx          # Wraps Tremor DropdownMenu
│   │   ├── checkbox.tsx          # Wraps Tremor Checkbox
│   │   ├── switch.tsx            # Wraps Tremor Switch
│   │   ├── tooltip.tsx           # Wraps Tremor Tooltip
│   │   ├── progress.tsx          # Wraps Tremor ProgressBar
│   │   ├── skeleton.tsx          # Loading skeleton
│   │   ├── avatar.tsx            # Wraps Tremor Avatar
│   │   └── index.ts              # Barrel export
│   │
│   ├── ui-shadcn/                # shadcn components (Tremor gaps)
│   │   ├── sidebar.tsx           # shadcn Sidebar
│   │   ├── collapsible.tsx       # shadcn Collapsible
│   │   ├── command.tsx           # shadcn Command (search)
│   │   └── index.ts
│   │
│   ├── charts/                   # Global chart wrappers (TREMOR)
│   │   ├── area-chart.tsx        # Wraps Tremor AreaChart
│   │   ├── bar-chart.tsx         # Wraps Tremor BarChart
│   │   ├── line-chart.tsx        # Wraps Tremor LineChart
│   │   ├── donut-chart.tsx       # Wraps Tremor DonutChart
│   │   ├── combo-chart.tsx       # Custom: Bar + Line
│   │   └── index.ts
│   │
│   ├── blocks/                   # Composed UI blocks
│   │   ├── kpi-card.tsx          # KPI card with trend
│   │   ├── metric-card.tsx       # Simple metric display
│   │   ├── trend-chart.tsx       # Chart with header + legend
│   │   ├── data-table.tsx        # Table with sort/page/filter
│   │   ├── filter-bar.tsx        # Slicer panel block
│   │   ├── date-picker.tsx       # Date range picker
│   │   ├── multi-select.tsx      # Multi-select dropdown
│   │   ├── search-input.tsx      # Search with debounce
│   │   ├── empty-state.tsx       # No data state
│   │   ├── error-state.tsx       # Error display
│   │   ├── loading-state.tsx     # Loading skeleton
│   │   └── index.ts
│   │
│   ├── templates/                # Page-level templates
│   │   ├── report-template.tsx   # JSON-driven report layout
│   │   ├── dashboard-template.tsx
│   │   ├── settings-template.tsx
│   │   ├── table-page-template.tsx
│   │   └── index.ts
│   │
│   ├── visuals/                  # Report visual renderers
│   │   ├── card-visual.tsx       # Renders card from JSON
│   │   ├── chart-visual.tsx      # Renders chart from JSON
│   │   ├── table-visual.tsx      # Renders table from JSON
│   │   ├── matrix-visual.tsx     # Cohort heatmap
│   │   ├── waterfall-visual.tsx  # P&L waterfall
│   │   ├── visual-renderer.tsx   # Switch by visual type
│   │   └── index.ts
│   │
│   ├── layout/                   # Layout components
│   │   ├── app-sidebar.tsx       # Main sidebar
│   │   ├── app-header.tsx        # Top header
│   │   ├── client-selector.tsx   # Client dropdown
│   │   ├── nav-section.tsx       # Sidebar section
│   │   ├── nav-item.tsx          # Sidebar item
│   │   └── index.ts
│   │
│   ├── auth/                     # Auth components
│   │   ├── login-form.tsx
│   │   ├── forgot-password-form.tsx
│   │   ├── reset-password-form.tsx
│   │   ├── protected-route.tsx
│   │   └── index.ts
│   │
│   └── common/                   # Shared utilities
│       ├── logo.tsx
│       ├── theme-toggle.tsx
│       ├── last-sync.tsx
│       ├── past-due-banner.tsx
│       └── index.ts
│
├── store/                        # Redux Toolkit
│   ├── index.ts                  # Store configuration
│   ├── api/                      # RTK Query APIs
│   │   ├── authApi.ts
│   │   ├── clientApi.ts
│   │   ├── reportApi.ts          # Reports navigation + templates
│   │   ├── queryApi.ts           # Query engine (batch/table)
│   │   ├── filterApi.ts
│   │   ├── billingApi.ts
│   │   └── index.ts
│   ├── slices/                   # Redux slices
│   │   ├── authSlice.ts
│   │   ├── clientSlice.ts
│   │   ├── uiSlice.ts            # Sidebar state, theme
│   │   └── index.ts
│   └── hooks.ts                  # Typed hooks
│
├── hooks/                        # Custom hooks
│   ├── use-auth.ts
│   ├── use-client.ts
│   ├── use-filters.ts
│   ├── use-toggles.ts
│   ├── use-report.ts
│   └── index.ts
│
├── lib/                          # Utilities
│   ├── axios.ts                  # Axios instance
│   ├── utils.ts                  # cn(), formatters
│   ├── constants.ts              # Routes, config
│   └── types.ts                  # TypeScript types
│
├── types/                        # Global types
│   ├── api.ts                    # API response types
│   ├── report.ts                 # Report template types
│   ├── visual.ts                 # Visual types
│   └── index.ts
│
└── public/                       # Static assets
    ├── logo.svg
    ├── favicon.ico
    └── ...
```

---

## 3. Global UI Component Layer

### 3.1 Philosophy
Every UI component used in the app goes through a wrapper. This allows:
- Single-point theme customization
- Easy swap from Tremor to another library
- Consistent prop interfaces
- Centralized styling

### 3.2 Example: Button Wrapper

```tsx
// components/ui/button.tsx
import { Button as TremorButton, ButtonProps as TremorButtonProps } from '@tremor/react';
import { cn } from '@/lib/utils';

export interface ButtonProps extends Omit<TremorButtonProps, 'variant'> {
  variant?: 'primary' | 'secondary' | 'destructive' | 'ghost' | 'link';
  size?: 'sm' | 'md' | 'lg';
}

const variantMap = {
  primary: 'primary',
  secondary: 'secondary',
  destructive: 'destructive',
  ghost: 'light',
  link: 'light',
} as const;

export function Button({ variant = 'primary', size = 'md', className, ...props }: ButtonProps) {
  return (
    <TremorButton
      variant={variantMap[variant]}
      size={size}
      className={cn(
        variant === 'link' && 'underline-offset-4 hover:underline',
        className
      )}
      {...props}
    />
  );
}
```

### 3.3 Example: Card Wrapper

```tsx
// components/ui/card.tsx
import { Card as TremorCard } from '@tremor/react';
import { cn } from '@/lib/utils';

export interface CardProps {
  children: React.ReactNode;
  className?: string;
  decoration?: 'top' | 'left' | 'bottom' | 'right';
  decorationColor?: string;
}

export function Card({ children, className, decoration, decorationColor }: CardProps) {
  return (
    <TremorCard
      decoration={decoration}
      decorationColor={decorationColor}
      className={cn('bg-card', className)}
    >
      {children}
    </TremorCard>
  );
}

export function CardHeader({ children, className }: { children: React.ReactNode; className?: string }) {
  return <div className={cn('flex flex-col space-y-1.5 p-6', className)}>{children}</div>;
}

export function CardTitle({ children, className }: { children: React.ReactNode; className?: string }) {
  return <h3 className={cn('text-h3 font-semibold leading-none tracking-tight', className)}>{children}</h3>;
}

export function CardContent({ children, className }: { children: React.ReactNode; className?: string }) {
  return <div className={cn('p-6 pt-0', className)}>{children}</div>;
}
```

---

## 4. Blocks Architecture

### 4.1 Block vs Visual

| Concept | Purpose | Data Source |
|---------|---------|-------------|
| **Block** | Reusable UI composition | Props passed in |
| **Visual** | JSON-driven report component | Query API response |

### 4.2 Example: KPI Card Block

```tsx
// components/blocks/kpi-card.tsx
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { TrendingUp, TrendingDown } from 'lucide-react';
import { cn, formatValue } from '@/lib/utils';

export interface KPICardProps {
  title: string;
  value: number;
  format: string;          // "$#,##0" | "0.0%" | "#,##0"
  trend?: number;          // Percentage change
  trendDirection?: 'up-good' | 'down-good';
  comparison?: string;     // "vs last period"
  loading?: boolean;
}

export function KPICard({
  title,
  value,
  format,
  trend,
  trendDirection = 'up-good',
  comparison,
  loading,
}: KPICardProps) {
  if (loading) return <KPICardSkeleton />;

  const isPositive = trend && trend > 0;
  const isGood = trendDirection === 'up-good' ? isPositive : !isPositive;

  return (
    <Card>
      <CardContent className="p-4">
        <p className="text-small text-muted-foreground">{title}</p>
        <p className="text-h1 font-semibold mt-1">{formatValue(value, format)}</p>
        {trend !== undefined && (
          <div className="flex items-center gap-1 mt-2">
            {isPositive ? (
              <TrendingUp className={cn('h-4 w-4', isGood ? 'text-success' : 'text-destructive')} />
            ) : (
              <TrendingDown className={cn('h-4 w-4', isGood ? 'text-success' : 'text-destructive')} />
            )}
            <Badge variant={isGood ? 'success' : 'destructive'}>
              {isPositive ? '+' : ''}{trend.toFixed(1)}%
            </Badge>
            {comparison && <span className="text-tiny text-muted-foreground">{comparison}</span>}
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

---

## 5. Visual Renderers (JSON-Driven)

### 5.1 Visual Renderer Switch

```tsx
// components/visuals/visual-renderer.tsx
import { CardVisual } from './card-visual';
import { ChartVisual } from './chart-visual';
import { TableVisual } from './table-visual';
import { MatrixVisual } from './matrix-visual';
import { WaterfallVisual } from './waterfall-visual';
import type { VisualConfig, VisualData } from '@/types/visual';

interface VisualRendererProps {
  config: VisualConfig;
  data: VisualData;
  loading?: boolean;
  error?: Error;
  onTableAction?: (action: TableAction) => void;
}

export function VisualRenderer({ config, data, loading, error, onTableAction }: VisualRendererProps) {
  if (error) return <ErrorState message={error.message} />;

  switch (config.type) {
    case 'card':
      return <CardVisual config={config} data={data} loading={loading} />;
    case 'chart':
      return <ChartVisual config={config} data={data} loading={loading} />;
    case 'table':
      return <TableVisual config={config} data={data} loading={loading} onAction={onTableAction} />;
    case 'matrix':
      return <MatrixVisual config={config} data={data} loading={loading} />;
    case 'waterfall':
      return <WaterfallVisual config={config} data={data} loading={loading} />;
    default:
      return <ErrorState message={`Unknown visual type: ${config.type}`} />;
  }
}
```

### 5.2 Card Visual (from JSON)

```tsx
// components/visuals/card-visual.tsx
import { KPICard } from '@/components/blocks/kpi-card';
import type { CardVisualConfig, VisualData } from '@/types/visual';

interface CardVisualProps {
  config: CardVisualConfig;
  data: VisualData;
  loading?: boolean;
}

export function CardVisual({ config, data, loading }: CardVisualProps) {
  // data.columns = [{key: 'revenue', label: '$ Revenue', format: '$#,##0'}]
  // data.rows = [[2324600.93]]

  return (
    <div className="grid gap-4" style={{ gridTemplateColumns: `repeat(${config.metrics.length}, 1fr)` }}>
      {config.metrics.map((metricKey, index) => {
        const column = data.columns.find(c => c.key === metricKey);
        const value = data.rows[0]?.[index] ?? 0;

        return (
          <KPICard
            key={metricKey}
            title={column?.label ?? metricKey}
            value={value}
            format={column?.format ?? '#,##0'}
            loading={loading}
          />
        );
      })}
    </div>
  );
}
```

---

## 6. Report Template Renderer

### 6.1 Report Page Component

```tsx
// app/(dashboard)/reports/[templateKey]/page.tsx
'use client';

import { useParams } from 'next/navigation';
import { useGetReportTemplateQuery } from '@/store/api/reportApi';
import { useGetBatchQueryQuery } from '@/store/api/queryApi';
import { ReportTemplate } from '@/components/templates/report-template';
import { useFilters } from '@/hooks/use-filters';
import { useToggles } from '@/hooks/use-toggles';

export default function ReportPage() {
  const { templateKey } = useParams();
  const { filters, setFilter } = useFilters();
  const { toggles, setToggle } = useToggles();

  // 1. Fetch template JSON
  const { data: template, isLoading: templateLoading } = useGetReportTemplateQuery(templateKey);

  // 2. Build visuals array from template
  const visuals = template?.layout?.sections?.flatMap(s => s.visuals) ?? [];

  // 3. Fetch data for all visuals
  const { data: queryResult, isLoading: dataLoading, refetch } = useGetBatchQueryQuery({
    templateKey,
    filters,
    toggles,
    visuals: visuals.map(v => ({
      visualId: v.visualId,
      type: v.type,
      metrics: v.metrics,
      groupBy: v.groupBy,
      limit: v.limit,
      pageSize: v.pageSize,
    })),
  }, { skip: !template });

  if (templateLoading) return <ReportSkeleton />;
  if (!template) return <NotFound />;

  return (
    <ReportTemplate
      template={template}
      data={queryResult?.data?.visuals ?? {}}
      loading={dataLoading}
      filters={filters}
      toggles={toggles}
      onFilterChange={setFilter}
      onToggleChange={setToggle}
      onRefresh={refetch}
    />
  );
}
```

### 6.2 Report Template Component

```tsx
// components/templates/report-template.tsx
import { SlicerPanel } from '@/components/blocks/filter-bar';
import { VisualRenderer } from '@/components/visuals/visual-renderer';
import type { ReportTemplate as ReportTemplateType, VisualDataMap } from '@/types/report';

interface ReportTemplateProps {
  template: ReportTemplateType;
  data: VisualDataMap;
  loading: boolean;
  filters: FilterState;
  toggles: ToggleState;
  onFilterChange: (key: string, value: any) => void;
  onToggleChange: (key: string, value: string) => void;
  onRefresh: () => void;
}

export function ReportTemplate({
  template,
  data,
  loading,
  filters,
  toggles,
  onFilterChange,
  onToggleChange,
  onRefresh,
}: ReportTemplateProps) {
  return (
    <div className="flex flex-col gap-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <h1 className="text-h1">{template.name}</h1>
        <Button variant="ghost" onClick={onRefresh}>Refresh</Button>
      </div>

      {/* Slicer Panel */}
      {template.layout.slicerPanel && (
        <SlicerPanel
          config={template.layout.slicerPanel}
          filters={filters}
          toggles={toggles}
          onFilterChange={onFilterChange}
          onToggleChange={onToggleChange}
        />
      )}

      {/* Sections */}
      {template.layout.sections.map((section) => (
        <ReportSection
          key={section.sectionId}
          section={section}
          data={data}
          loading={loading}
        />
      ))}
    </div>
  );
}

function ReportSection({ section, data, loading }) {
  return (
    <section className={cn(getSectionClassName(section.type))}>
      {section.title && <h2 className="text-h2 mb-4">{section.title}</h2>}
      <div className={cn(getGridClassName(section.type, section.columns))}>
        {section.visuals.map((visual) => (
          <VisualRenderer
            key={visual.visualId}
            config={visual}
            data={data[visual.visualId] ?? { columns: [], rows: [] }}
            loading={loading}
          />
        ))}
      </div>
    </section>
  );
}
```

---

## 7. Redux Store Architecture

### 7.1 Store Configuration

```tsx
// store/index.ts
import { configureStore, combineReducers } from '@reduxjs/toolkit';
import { setupListeners } from '@reduxjs/toolkit/query';
import { persistStore, persistReducer } from 'redux-persist';
import storage from 'redux-persist/lib/storage';

import { authApi } from './api/authApi';
import { clientApi } from './api/clientApi';
import { reportApi } from './api/reportApi';
import { queryApi } from './api/queryApi';

import authReducer from './slices/authSlice';
import clientReducer from './slices/clientSlice';
import uiReducer from './slices/uiSlice';

const persistConfig = {
  key: 'beastinsights',
  version: 1,
  storage,
  whitelist: ['auth', 'client', 'ui'],
};

const rootReducer = combineReducers({
  auth: authReducer,
  client: clientReducer,
  ui: uiReducer,
  [authApi.reducerPath]: authApi.reducer,
  [clientApi.reducerPath]: clientApi.reducer,
  [reportApi.reducerPath]: reportApi.reducer,
  [queryApi.reducerPath]: queryApi.reducer,
});

const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST', 'persist/REHYDRATE'],
      },
    }).concat(
      authApi.middleware,
      clientApi.middleware,
      reportApi.middleware,
      queryApi.middleware
    ),
});

setupListeners(store.dispatch);

export const persistor = persistStore(store);
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### 7.2 RTK Query - Report API

```tsx
// store/api/reportApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';
import type { NavigationResponse, ReportTemplate } from '@/types/report';

export const reportApi = createApi({
  reducerPath: 'reportApi',
  baseQuery: fetchBaseQuery({
    baseUrl: process.env.NEXT_PUBLIC_API_URL,
    credentials: 'include',
  }),
  tagTypes: ['Navigation', 'Template'],
  endpoints: (builder) => ({
    // Sidebar navigation
    getNavigation: builder.query<NavigationResponse, void>({
      query: () => '/api/v1/reports/navigation',
      providesTags: ['Navigation'],
    }),

    // Report template
    getReportTemplate: builder.query<ReportTemplate, string>({
      query: (templateKey) => `/api/v1/reports/templates/${templateKey}`,
      providesTags: (result, error, templateKey) => [{ type: 'Template', id: templateKey }],
    }),
  }),
});

export const { useGetNavigationQuery, useGetReportTemplateQuery } = reportApi;
```

### 7.3 RTK Query - Query API

```tsx
// store/api/queryApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';
import type { BatchQueryRequest, BatchQueryResponse, TableQueryRequest, TableQueryResponse } from '@/types/api';

export const queryApi = createApi({
  reducerPath: 'queryApi',
  baseQuery: fetchBaseQuery({
    baseUrl: process.env.NEXT_PUBLIC_API_URL,
    credentials: 'include',
  }),
  endpoints: (builder) => ({
    // Batch query - all visuals at once
    getBatchQuery: builder.query<BatchQueryResponse, BatchQueryRequest>({
      query: (body) => ({
        url: '/api/v1/query/batch',
        method: 'POST',
        body,
      }),
    }),

    // Table query - single table with sort/page/groupBy
    getTableQuery: builder.query<TableQueryResponse, TableQueryRequest>({
      query: (body) => ({
        url: '/api/v1/query/table',
        method: 'POST',
        body,
      }),
    }),
  }),
});

export const { useGetBatchQueryQuery, useGetTableQueryQuery, useLazyGetTableQueryQuery } = queryApi;
```

---

## 8. API Endpoints Required

### 8.1 New Endpoints (Backend)

| Endpoint | Purpose | Response |
|----------|---------|----------|
| `GET /api/v1/reports/navigation` | Sidebar report list | `{ sections: [...], features: {...} }` |
| `GET /api/v1/reports/templates/:key` | Full template JSON | `{ templateKey, name, layout: {...} }` |

### 8.2 Existing Endpoints (Already Built)

| Endpoint | Purpose |
|----------|---------|
| `POST /api/v1/query/batch` | Fetch all visuals data |
| `POST /api/v1/query/table` | Fetch single table data |

---

## 9. Design System Migration

### 9.1 CSS Variables (Exact from old)

The complete design system from `beastinsights-old/app/globals.css` will be copied exactly:
- Color primitives (blue, red, amber, green, teal, purple, grey)
- Semantic tokens (text, stroke, fill, background, icon)
- Light/dark mode support
- Typography scale (data-optimized, 14px default)
- Shadows, animations, transitions

### 9.2 Tremor Theme Customization

Tremor will be configured to use the existing CSS variables:

```tsx
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        tremor: {
          brand: {
            faint: 'var(--fill-brand-weak)',
            muted: 'var(--blue-light-200)',
            subtle: 'var(--blue-light-800)',
            DEFAULT: 'var(--fill-brand-strong)',
            emphasis: 'var(--blue-light-1000)',
            inverted: 'var(--text-inverse-strong)',
          },
          // ... map all Tremor colors to CSS variables
        },
      },
    },
  },
};
```

---

## 10. Migration Checklist

### Phase 1: Setup & Core
- [ ] Initialize Next.js 15 project
- [ ] Install dependencies (Tremor, Redux Toolkit, etc.)
- [ ] Copy design system (globals.css, tailwind.config)
- [ ] Set up Redux store with RTK Query
- [ ] Create global UI wrappers (components/ui/*)
- [ ] Create layout components (sidebar, header)

### Phase 2: Auth & Navigation
- [ ] Implement auth flow (login, forgot-password, reset)
- [ ] Create protected route wrapper
- [ ] Build sidebar with dynamic report navigation
- [ ] Build client selector

### Phase 3: Report Renderer
- [ ] Create visual renderers (card, chart, table)
- [ ] Create report template component
- [ ] Implement slicer panel (filters + toggles)
- [ ] Build dynamic report page ([templateKey])

### Phase 4: Feature Pages
- [ ] Migrate Home/Dashboard
- [ ] Migrate Routing Insights
- [ ] Migrate Routing Setup
- [ ] Migrate Integration
- [ ] Migrate Schedule
- [ ] Migrate Users
- [ ] Migrate Reporting Setup
- [ ] Migrate Billing

### Phase 5: Admin Pages
- [ ] Migrate Admin Dashboard
- [ ] Migrate Filter Config
- [ ] Migrate ETL Logs
- [ ] Migrate Maintenance

### Phase 6: Polish
- [ ] Error boundaries
- [ ] Loading states
- [ ] Empty states
- [ ] Responsive design
- [ ] Performance optimization

---

## 11. Questions Resolved

| Question | Answer |
|----------|--------|
| State management | Redux Toolkit + RTK Query |
| API layer | Axios + RTK Query |
| UI library | Tremor (shadcn for sidebar) |
| Component wrapping | All UI through global wrappers |
| Report rendering | JSON-driven dynamic |
| Sidebar navigation | Separate API endpoint |
| Colors/theme | Exact copy from old |

---

*M4 Frontend Architecture Plan*
*Created: 2026-02-02*
