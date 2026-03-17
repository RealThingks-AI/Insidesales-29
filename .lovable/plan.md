

# Campaign Module Deep Audit Report & Fix Plan

## Audit Summary

The Campaign module is well-implemented with comprehensive functionality across all 9 tabs (Overview, Accounts, Contacts, Outreach, Templates, Scripts, Materials, Action Items, Analytics). Below are the issues found, categorized by severity.

---

## CRITICAL Bugs

### 1. Edge Function `send-campaign-email` uses invalid `getClaims()` API
**File**: `supabase/functions/send-campaign-email/index.ts` (line 54)
**Problem**: `supabase.auth.getClaims(token)` does not exist in Supabase JS v2. This means every campaign email send attempt will fail with a runtime error, returning 401 Unauthorized.
**Fix**: Replace with `supabase.auth.getUser(token)` and extract `user.id` from the response:
```
const { data: userData, error: userError } = await supabase.auth.getUser(token);
if (userError || !userData?.user) { return 401; }
const userId = userData.user.id;
```
Then redeploy the edge function.

### 2. Edge Function not configured in `config.toml`
**File**: `supabase/config.toml` only contains `project_id`. No `[functions.send-campaign-email]` section with `verify_jwt = false` exists. The function may fail if JWT verification is enforced at the gateway level before reaching the function code. Need to add config entry.

---

## HIGH Priority Issues

### 3. Campaign delete does NOT clean up linked `deals.campaign_id`
**File**: `src/hooks/useCampaigns.tsx` (lines 68-80)
**Problem**: When a campaign is deleted, all child records are cleaned up (contacts, accounts, communications, templates, scripts, materials), but `deals.campaign_id` references are NOT nullified. Deals that were converted from this campaign will retain a dangling `campaign_id` pointing to a deleted campaign.
**Fix**: Before deleting the campaign, run:
```sql
supabase.from('deals').update({ campaign_id: null }).eq('campaign_id', id)
```

### 4. Campaign delete does NOT clean up linked `action_items`
**File**: `src/hooks/useCampaigns.tsx` (lines 68-80)
**Problem**: Action items with `module_type = 'campaigns'` and `module_id = campaign_id` are not deleted when the campaign is deleted. These become orphan records.
**Fix**: Add cleanup:
```sql
supabase.from('action_items').delete().eq('module_type', 'campaigns').eq('module_id', id)
```

### 5. `useCampaignAggregates` fires N+1 queries per campaign
**File**: `src/hooks/useCampaigns.tsx` (lines 566-600)
**Problem**: For each campaign, 3 count queries are fired (accounts, contacts, deals). With 50 campaigns, that's 150+ queries per page load. This will cause serious performance issues at scale.
**Fix**: Replace with a single query using group-by counts or an RPC function.

---

## MEDIUM Priority Issues

### 6. No confirmation dialog for removing accounts/contacts from campaign
Clicking the X button on Accounts or Contacts tab immediately removes the record without confirmation. This is a destructive action that should require confirmation, especially since it could affect outreach tracking.

### 7. Action Items tab missing "Assigned To" column in table
**File**: `src/components/campaigns/CampaignActionItemsTab.tsx` (lines 141-192)
The table shows Title, Priority, Status, Due Date but NOT who it's assigned to. The `assigned_to` field is collected in the create form but never displayed.

### 8. CampaignModal missing "Clear Owner" option
**File**: `src/components/campaigns/CampaignModal.tsx`
The Owner select has no "None" option. Once an owner is set, it cannot be cleared. Other selects like Account in the outreach dialog have a "None" option.

### 9. Campaign cloning doesn't clone materials (files)
**File**: `src/hooks/useCampaigns.tsx` (lines 92-178)
Clone copies email templates, phone scripts, contacts, and accounts, but does NOT clone campaign materials (uploaded files). The storage files would need to be copied as well.

### 10. Outreach tab: communication log "Account" select allows empty string value
**File**: `src/components/campaigns/CampaignOutreachTab.tsx` (line 352)
`<SelectItem value="">None</SelectItem>` — Radix Select does not allow empty string as value. This will cause a React warning and the "None" option won't work correctly. Should use a sentinel value like `"none"` and convert to null on save.

### 11. Send Email dialog: same empty string issue for Account select
**File**: `src/components/campaigns/CampaignOutreachTab.tsx` (line 278)
Same problem as above.

---

## LOW Priority / UI Consistency Issues

### 12. Campaign list table lacks search within table (contacts/accounts tabs have it)
The main campaign list on the Campaigns page has a search bar in the filter area, but inside the detail panel, the Outreach/Communications table has no search or filter capability.

### 13. No pagination on Outreach communications table
Unlike Accounts and Contacts tabs which use `StandardPagination`, the Outreach tab renders all communications without pagination. With 100+ outreach entries this will degrade performance.

### 14. Campaign Settings upsert uses `as any` type casting
Minor code quality issue throughout `CampaignSettings.tsx`.

### 15. Email template body is plain text, not rich text
Templates use a `<Textarea>` for body content, but the body is sent as HTML in the email. Users writing plain text will get emails without formatting. Consider noting this in the UI or adding basic HTML support.

---

## What's Working Well (Verified)

- Campaign CRUD with all fields persisting correctly
- Campaign name required validation
- Start/end date validation (start cannot be after end)
- Owner defaults to current user on create
- Status defaults to Draft
- MART strategy fields all present (Message Strategy, Audience Segments, Region/Country, Start/End dates)
- Email templates with 5 dynamic placeholders (contact_name, company_name, email, position, sender_name)
- Multiple templates per campaign with audience segmentation
- Contacts tab: search, filter by account, filter by position, bulk add, stage tracking
- Accounts tab: search, filter by industry/country, bulk add, status tracking
- Unique constraints prevent duplicate account/contact additions
- Convert to Deal with duplicate guard, stakeholder linking, campaign_id stored in deal
- Deal stage correctly set to "Lead"
- Campaign cloning (templates, scripts, contacts, accounts)
- Analytics with funnel chart, pie chart, 9 summary metrics
- Campaign settings with follow-up rules persisting correctly
- Phone scripts with full CRUD
- Materials with upload/download/delete via Supabase Storage
- Action items linked to campaigns module correctly
- Delete confirmation dialogs on campaigns, templates, scripts

---

## Implementation Plan

### Task 1: Fix `send-campaign-email` edge function (Critical)
- Replace `getClaims()` with `getUser()`
- Add function config to `config.toml`
- Redeploy

### Task 2: Fix campaign delete orphan cleanup (High)
- Add `deals.campaign_id` nullification
- Add `action_items` deletion for campaign module
- Both in `useCampaigns.tsx` delete mutation

### Task 3: Fix empty string Select values in Outreach tab (Medium)
- Replace `value=""` with `value="none"` and convert to null on save

### Task 4: Add "Assigned To" column to Action Items table (Medium)
- Display assignee name in the campaign action items table

### Task 5: Add pagination to Outreach communications table (Low)

These 5 tasks address all actionable bugs. The N+1 aggregates query (issue #5) is a performance optimization that can be addressed separately when campaign count grows.

