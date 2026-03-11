# Server-Side GTM — Manual Setup Steps

The JSON import (`SERVER_GTM_CONTAINER.json`) covers **variables** and **triggers**.
The GA4 Client and GA4 Tag **cannot be imported** in server-side GTM because they
depend on community templates that must be installed first. Follow these steps after
importing the JSON.

---

## Step 1: Import the JSON

1. Open your **Server** container in GTM
2. **Admin** (top nav) → **Import Container**
3. Choose `SERVER_GTM_CONTAINER.json`
4. Select **Merge** → **Rename conflicting tags, triggers, and variables**
5. Click **Confirm**

This imports: 26 Event Data variables + 2 triggers + 3 folders.

---

## Step 2: Create GA4 Client (Manual)

The GA4 Client parses incoming GA4 requests from your web container. It should
already exist by default in new server containers — verify it's there:

1. Go to **Clients** (left sidebar)
2. If `GA4` client already exists, click to verify settings:
   - **Default GA4 paths**: ON
   - **Compress server response**: ON
3. If it doesn't exist:
   - Click **New** → **Client Configuration** → Select **GA4**
   - **Name**: `GA4 Client`
   - **Default GA4 paths**: ON
   - **Compress server response**: ON
   - **Priority**: `10`
   - Click **Save**

---

## Step 3: Create GA4 Tag (Manual)

This tag forwards all events from the GA4 Client to Google Analytics.

1. Go to **Tags** → Click **New**
2. Click **Tag Configuration** → Select **Google Analytics: GA4**
3. Configure:
   - **Name**: `GA4 - Forward All Events`
   - **Measurement ID**: `G-XXXXXXXXXX` ← replace with your actual ID
   - Leave everything else as default (the tag auto-forwards all event data)
4. **Triggering**: Click the trigger area → Select **All Events** (the `ALWAYS` trigger imported from the JSON — or use the built-in "All Pages" if present)
5. Click **Save**

> The imported Event Data variables (`ED - client_id`, `ED - report_key`, etc.)
> are automatically available in the event payload since the GA4 Client passes
> them through. You only need to reference them explicitly if you want to add
> server-side transformations or enrichments.

---

## Step 4: (Optional) Add Event Parameters to GA4 Tag

If you want to explicitly map parameters (for filtering or transformation on the
server before forwarding to GA4):

1. Edit the `GA4 - Forward All Events` tag
2. Under **Event Parameters** → **Add Row** for each:

| Parameter Name | Value (Variable) |
|---------------|------------------|
| `client_id` | `{{ED - client_id}}` |
| `user_role` | `{{ED - user_role}}` |
| `report_key` | `{{ED - report_key}}` |
| `report_name` | `{{ED - report_name}}` |
| `report_type` | `{{ED - report_type}}` |
| `visual_type` | `{{ED - visual_type}}` |
| `visual_id` | `{{ED - visual_id}}` |
| `action` | `{{ED - action}}` |
| `filter_type` | `{{ED - filter_type}}` |
| `filter_key` | `{{ED - filter_key}}` |
| `chart_type` | `{{ED - chart_type}}` |
| `builder_type` | `{{ED - builder_type}}` |
| `duration_ms` | `{{ED - duration_ms}}` |
| `error_type` | `{{ED - error_type}}` |

> Note: If you don't add these explicitly, the GA4 tag still forwards them
> automatically from the incoming request. Explicit mapping is only needed if
> you want to rename, filter, or transform parameters on the server side.

---

## Step 5: Publish

1. Click **Submit** (top right)
2. **Version Name**: `v1.0 - GA4 forwarding + Beast Insights variables`
3. Click **Publish**

---

## Step 6: Verify

1. Click **Preview** in the server container
2. Open your app and trigger some events
3. In the sGTM preview panel, verify:
   - GA4 Client claims the request
   - GA4 Tag fires and forwards to Google
   - Event Data variables resolve correctly
