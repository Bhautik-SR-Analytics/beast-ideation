# Beast Insights — GA4 Tracking & Explorer Setup Guide

## Architecture Overview

```
Browser (Next.js)
  └─ window.dataLayer.push({ event, params })
       └─ Web GTM (GTM-5MRFCFCK)
            └─ GA4 Event Tags → https://analytics.beastinsights.com
                 └─ Nginx reverse proxy
                      └─ Server GTM (GTM-T368B2GR) Docker :8079
                           └─ GA4 Forward → G-CYRM1F0HEG
```

- **Ad blocker bypass**: All requests route through `analytics.beastinsights.com`
- **Dedup guard**: 300ms window in `trackEvent()` prevents React Strict Mode double-fires
- **Base params**: Every event auto-includes `user_id`, `user_role`, `client_id` from localStorage

---

## Event Reference

### 1. `page_view` (Auto — GA4 Config Tag)

Fired automatically by the GA4 Config tag on every navigation. **Do not push manually.**

| Parameter | Type | Source |
|-----------|------|--------|
| `page_location` | string | Auto (GA4) |
| `page_title` | string | Auto (GA4) |
| `page_referrer` | string | Auto (GA4) |

---

### 2. `auth_action`

**File**: `store/slices/authSlice.js`

| Parameter | Type | Values |
|-----------|------|--------|
| `action` | string | `sign_in`, `sign_out`, `sign_in_failed` |
| `user_id` | string | User ID (sign_in only) |
| `username` | string | Username (sign_in only) |
| `user_role` | string | Role (sign_in only) |
| `client_id` | string | Client ID (sign_in only) |

**GA4 Explorer use**: User acquisition, login failure rate.

---

### 3. `report_viewed`

**File**: `app/(user-type)/(user)/reports/[templateKey]/page.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `report_key` | string | `sales-report`, `approval-rate` |
| `report_name` | string | `Sales Report` |
| `report_type` | string | `analytics` |

**GA4 Explorer use**: Most viewed reports, report popularity by user/client.

---

### 4. `report_tab_switch`

**File**: `app/(user-type)/(user)/reports/[templateKey]/page.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `report_key` | string | `sales-report` |
| `action` | string | Tab name switched to |
| `previous_tab` | string | Previous tab name |

---

### 5. `filter_changed`

**Files**: `ReportFilterBar.tsx`, `DateRangePicker.tsx`, `MoreFiltersPopover.tsx`, `GroupBySelector.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `report_key` | string | `sales-report` |
| `filter_type` | string | `primary`, `advanced`, `date_range`, `toggle`, `group_by` |
| `filter_key` | string | `sales_type`, `date_preset`, `date_basis` |
| `action` | string | `select`, `deselect`, `preset_change`, `custom_range` |

**GA4 Explorer use**: Most used filters, filter engagement per report.

---

### 6. `filter_cleared`

**File**: `MoreFiltersPopover.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `report_key` | string | `approval-rate` |
| `filter_key` | string | Specific filter key (clear_one) or omitted (clear_all) |
| `action` | string | `clear_one`, `clear_all` |

---

### 7. `table_interacted`

**File**: `components/ui-tremor/tables/data-table.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `report_key` | string | `sales-report` |
| `visual_id` | string | `sales_table` |
| `action` | string | `sort` |
| `column` | string | Column key sorted |
| `direction` | string | `asc`, `desc` |

**GA4 Explorer use**: Most sorted columns, table engagement.

---

### 8. `engagement`

**File**: `components/analytics/AnalyticsProvider.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `action` | string | `dwell_time` |
| `duration_ms` | number | `45000` |
| `report_key` | string | Extracted from URL path |
| `scroll_percent` | number | (not yet implemented) |
| `visual_id` | string | (not yet implemented) |

**GA4 Explorer use**: Average time on reports, engagement quality.

---

### 9. `sidebar_action`

**File**: `components/sidebar/AppSidebar.jsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `action` | string | `expand`, `collapse`, `navigate` |
| (varies) | any | Additional context params |

---

### 10. `client_switch`

**File**: `components/sidebar/AppSidebar.jsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `action` | string | `switch` |
| `client_id` | string | New client ID |

---

### 11. `theme_switch`

**File**: `components/common/ThemeSwitch.jsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `action` | string | `light`, `dark` |

---

### 12. `bookmark_action`

**File**: `components/reports/ViewSelector.tsx`, `components/builder/ReportBuilder.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `action` | string | `create`, `apply`, `update`, `delete`, `set_default` |
| `report_key` | string | Template key or report ID |
| `bookmark_name` | string | View name |

---

### 13. `builder_action`

**File**: `components/builder/ReportBuilder.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `action` | string | `save`, `create` |
| `report_key` | string | Report ID |
| `visual_count` | number | Number of visuals |
| `builder_type` | string | (not yet used) |
| `visual_type` | string | (not yet used) |

---

### 14. `error_occurred`

**File**: `app/(user-type)/(user)/reports/[templateKey]/page.tsx`

| Parameter | Type | Example |
|-----------|------|---------|
| `error_type` | string | `report_load_error`, `api_error` |
| `error_message` | string | Error description |
| `report_key` | string | Report that errored |

---

### 15–19. Defined but Not Yet Implemented

These functions exist in `lib/analytics.ts` but are not yet called anywhere:

| Event | Function | Intended Use |
|-------|----------|--------------|
| `navigation` | `trackNavigation()` | Page/route navigation actions |
| `chart_interacted` | `trackChartInteracted()` | Chart hover, click, zoom |
| `kpi_interacted` | `trackKPIInteracted()` | KPI card click/expand |
| `visual_interacted` | `trackVisualInteracted()` | Generic visual interaction |
| `search` | `trackSearch()` | Search bar usage |
| `export_action` | `trackExportAction()` | CSV/PDF exports |
| `admin_action` | `trackAdminAction()` | Admin panel actions |

---

## GA4 Custom Dimensions & Metrics Registration

Register these in **GA4 Admin → Custom Definitions** to make them available in Explorer.

### Custom Dimensions (Event-scoped)

| Name | Parameter Name | Scope |
|------|---------------|-------|
| Action | `action` | Event |
| Report Key | `report_key` | Event |
| Report Name | `report_name` | Event |
| Report Type | `report_type` | Event |
| Filter Type | `filter_type` | Event |
| Filter Key | `filter_key` | Event |
| Visual ID | `visual_id` | Event |
| Chart Type | `chart_type` | Event |
| KPI Variant | `kpi_variant` | Event |
| Visual Type | `visual_type` | Event |
| Builder Type | `builder_type` | Event |
| Column | `column` | Event |
| Direction | `direction` | Event |
| Previous Tab | `previous_tab` | Event |
| Bookmark Name | `bookmark_name` | Event |
| Search Term | `search_term` | Event |
| Search Location | `search_location` | Event |
| Format | `format` | Event |
| Error Type | `error_type` | Event |
| Error Message | `error_message` | Event |

### Custom Dimensions (User-scoped)

| Name | Parameter Name | Scope |
|------|---------------|-------|
| User Role | `user_role` | User |
| Client ID | `client_id` | User |

### Custom Metrics

| Name | Parameter Name | Unit |
|------|---------------|------|
| Duration (ms) | `duration_ms` | Milliseconds |
| Scroll Percent | `scroll_percent` | Standard |
| Visual Count | `visual_count` | Standard |
| Results Count | `results_count` | Standard |

---

## GA4 Explorer Reports

### Report 1: Report Popularity

See which reports are viewed most and by whom.

**Setup**:
- Technique: **Free form**
- Dimensions: `report_key`, `report_name`, `user_role`, `client_id`
- Metrics: `Event count`, `Total users`
- Filter: Event name = `report_viewed`
- Rows: `report_key`
- Values: `Event count`, `Total users`
- Sort: Event count descending

**Insights**: Which reports drive the most engagement? Are certain clients using specific reports more?

---

### Report 2: Filter Usage Analysis

Understand how users interact with filters.

**Setup**:
- Technique: **Free form**
- Dimensions: `report_key`, `filter_type`, `filter_key`, `action`
- Metrics: `Event count`
- Filter: Event name = `filter_changed`
- Rows: `report_key`, `filter_key`
- Values: `Event count`

**Insights**: Which filters are used most? Are users clearing filters frequently (sign of confusion)?

---

### Report 3: User Engagement / Dwell Time

How long users spend on each report.

**Setup**:
- Technique: **Free form**
- Dimensions: `report_key`
- Metrics: `duration_ms` (average)
- Filter: Event name = `engagement`, Action = `dwell_time`
- Rows: `report_key`
- Values: Average `duration_ms`

**Insights**: High dwell time = valuable report OR confusing report. Cross-reference with filter usage.

---

### Report 4: Table Interaction Patterns

Which table columns users sort most.

**Setup**:
- Technique: **Free form**
- Dimensions: `report_key`, `visual_id`, `column`, `direction`
- Metrics: `Event count`
- Filter: Event name = `table_interacted`
- Rows: `visual_id`, `column`
- Values: `Event count`

**Insights**: Most-sorted columns indicate what data users care about most.

---

### Report 5: Error Monitoring

Track application errors.

**Setup**:
- Technique: **Free form**
- Dimensions: `error_type`, `error_message`, `report_key`
- Metrics: `Event count`
- Filter: Event name = `error_occurred`
- Rows: `error_type`, `report_key`
- Values: `Event count`
- Sort: Event count descending

**Insights**: Which reports are breaking? Recurring error patterns?

---

### Report 6: Builder Activity

Track custom report builder usage.

**Setup**:
- Technique: **Free form**
- Dimensions: `action`, `report_key`, `visual_count`
- Metrics: `Event count`
- Filter: Event name = `builder_action`
- Rows: `action`
- Values: `Event count`, Average `visual_count`

**Insights**: How many custom reports are being created vs saved? Average complexity?

---

### Report 7: Bookmark / Saved View Usage

Track how users use saved views.

**Setup**:
- Technique: **Free form**
- Dimensions: `action`, `report_key`, `bookmark_name`
- Metrics: `Event count`
- Filter: Event name = `bookmark_action`
- Rows: `action`, `report_key`
- Values: `Event count`

**Insights**: Are users saving and reusing views? Which reports benefit most from saved views?

---

### Report 8: Auth & Session Analysis

Login patterns and failures.

**Setup**:
- Technique: **Free form**
- Dimensions: `action`, `user_role`, `client_id`
- Metrics: `Event count`, `Total users`
- Filter: Event name = `auth_action`
- Rows: `action`
- Values: `Event count`

**Insights**: Login failure rate, active users by role, multi-client users.

---

### Report 9: Client Usage Comparison

Compare engagement across clients.

**Setup**:
- Technique: **Free form**
- Dimensions: `client_id`, `report_key`
- Metrics: `Event count`, `Total users`
- Filter: Event name = `report_viewed`
- Rows: `client_id`
- Columns: `report_key`
- Values: `Event count`

**Insights**: Which clients are most active? Feature adoption by client.

---

### Report 10: User Journey / Funnel

Track the path from login to report interaction.

**Setup**:
- Technique: **Funnel exploration**
- Steps:
  1. `auth_action` (action = sign_in)
  2. `report_viewed`
  3. `filter_changed`
  4. `table_interacted` OR `engagement` (dwell_time > 30s)

**Insights**: Drop-off between steps shows where users lose interest or get stuck.

---

## Implementation Status

| Event | Status | Files |
|-------|--------|-------|
| `page_view` | Active (GA4 auto) | GA4 Config Tag |
| `auth_action` | Active | `authSlice.js` |
| `report_viewed` | Active | `[templateKey]/page.tsx` |
| `report_tab_switch` | Active | `[templateKey]/page.tsx` |
| `filter_changed` | Active | `ReportFilterBar.tsx`, `DateRangePicker.tsx`, `MoreFiltersPopover.tsx`, `GroupBySelector.tsx` |
| `filter_cleared` | Active | `MoreFiltersPopover.tsx` |
| `table_interacted` | Active | `data-table.tsx` |
| `engagement` | Active | `AnalyticsProvider.tsx` |
| `sidebar_action` | Active | `AppSidebar.jsx` |
| `client_switch` | Active | `AppSidebar.jsx` |
| `theme_switch` | Active | `ThemeSwitch.jsx` |
| `bookmark_action` | Active | `ViewSelector.tsx`, `ReportBuilder.tsx` |
| `builder_action` | Active | `ReportBuilder.tsx` |
| `error_occurred` | Active | `[templateKey]/page.tsx` |
| `navigation` | Defined | Not yet called |
| `chart_interacted` | Defined | Not yet called |
| `kpi_interacted` | Defined | Not yet called |
| `visual_interacted` | Defined | Not yet called |
| `search` | Defined | Not yet called |
| `export_action` | Defined | Not yet called |
| `admin_action` | Defined | Not yet called |

---

## GTM Container Reference

### Web Container (GTM-5MRFCFCK)

| Component | Count | Purpose |
|-----------|-------|---------|
| GA4 Config Tag | 1 | Sends page_view + config to sGTM |
| GA4 Event Tags | 26 | One per custom event type |
| Custom Event Triggers | 26 | Match `event` name from dataLayer |
| DataLayer Variables | 59 | Extract event parameters |

### Server Container (GTM-T368B2GR)

| Component | Count | Purpose |
|-----------|-------|---------|
| GA4 Client | 1 | Receives hits from web container |
| GA4 Tag | 1 | Forwards all events to GA4 (G-CYRM1F0HEG) |
| Event Data Variables | 26 | Pass-through event parameters |
| All Events Trigger | 1 | `.*` regex matches all event names |

---

## Nginx Proxy Paths

| Path | Proxies To | Purpose |
|------|-----------|---------|
| `/` | sGTM Docker :8079 | Main server container |
| `/gtm.js` | googletagmanager.com | GTM loader (rewritten URLs) |
| `/gtag/js` | googletagmanager.com | gtag.js library |
| `/a` | googletagmanager.com | Analytics collection |
| `/td` | googletagmanager.com | Tag delivery |
| `/gtm/` | Preview Docker :8078 | GTM preview mode |

---

## Quick Debugging Checklist

1. **No events in GA4 Realtime?**
   - Check browser DevTools → Network → filter `analytics.beastinsights.com`
   - Verify GTM Preview mode shows tags firing
   - Check server container logs: `docker logs sgtm-main`

2. **Duplicate events?**
   - Dedup guard handles React Strict Mode (300ms window)
   - Verify only ONE trigger per event tag in Web GTM
   - GA4 Config tag auto-sends `page_view` — don't push it manually

3. **Missing custom dimensions in Explorer?**
   - Register in GA4 Admin → Custom Definitions first
   - Takes up to 24 hours to appear in Explorer
   - Realtime report shows params immediately

4. **Events blocked by ad blocker?**
   - All requests should go through `analytics.beastinsights.com`
   - Check Nginx `sub_filter` is rewriting URLs in `gtm.js`
   - Verify no direct `googletagmanager.com` requests in Network tab
