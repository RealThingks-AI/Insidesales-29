

# CRM Full System Deep QA Report

## Phase 1 — Module Map

**Pages**: Dashboard, Accounts, Contacts, Deals, Campaigns, Action Items, Settings, Notifications, Auth, NotFound

**Core Components by Module**:
- **Dashboard**: UserDashboard, DashboardStats, TodaysTasksPopup, 8 widgets (Deals, Accounts, Contacts, ActionItems, Email, QuickActions, TodaysAgenda, RecentActivities)
- **Accounts**: AccountTable, AccountModal, AccountViewModal, AccountColumnCustomizer, AccountSearchableDropdown, BulkActionsBar, ImportExportBar
- **Contacts**: ContactTable, ContactModal, ContactColumnCustomizer, ContactSearchableDropdown, ContactsModuleHeader
- **Deals**: KanbanBoard, ListView, DealForm, DealCard, DealExpandedPanel, DealActionsDropdown, DealActionItemsModal, DealColumnCustomizer, DealsAdvancedFilter, DealsFilterPanel, LinkedContactsDialog, DealStageForm

<lov-plan>

# CRM Full System Deep QA Report

## Phase 1 — Module Map

**Pages**: Dashboard, Accounts, Contacts, Deals, Campaigns, Action Items, Settings, Notifications, Auth, NotFound

**Core Components**:
- **Dashboard**: UserDashboard, 8 widgets, TodaysTasksPopup, YearlyRevenueSummary
- **Accounts**: AccountTable, AccountModal, AccountViewModal, AccountColumnCustomizer, ImportExport
- **Contacts**: ContactTable, ContactModal, ContactColumnCustomizer, ImportExport
- **Deals**: KanbanBoard, ListView, DealForm (6 stage forms), DealExpandedPanel, DealCard, DealActionItemsModal, DealsAdvancedFilter
- **Campaigns**: CampaignList, CampaignModal, CampaignDetailPanel, 7 tabs (Accounts, Contacts, ActionItems, EmailTemplates, Materials, Outreach, PhoneScripts)
- **Action Items**: ActionItemsTable, ActionItemsKanban, ActionItemsCalendar, ActionItemModal
- **Settings**: AccountSettingsPage, AdminSettingsPage (Users, Access, Logs, System), EmailCenterPage, CampaignSettings
- **Notifications**: Full notification list with pagination
- **Security**: SecurityProvider, SecurityEnhancedApp, PermissionsContext, useAuth, useSecurityAudit

---

## Bugs Found

### CRITICAL

1. **Deal Form `modified_by` uses wrong user ID on save** (DealForm.tsx:124-125)
   - `modified_by: deal?.created_by || formData.created_by` — this sets `modified_by` to the **creator**, not the **current user**. Should be `user?.id`.
   - Same bug on lines 163-164, 200-201, 236-237 (all stage move operations).

2. **Contacts bulk delete does not clean up related records** (Contacts.tsx:52-72)
   - When contacts are bulk deleted, `deal_stakeholders`, `campaign_contacts`, and `deal` stakeholder foreign keys (`budget_owner_contact_id`, `champion_contact_id`, etc.) are NOT cleaned up.
   - Accounts page correctly cleans up deals and campaign_accounts before delete, but Contacts page does not.
   - This creates orphan references in `deal_stakeholders` and stale contact IDs in `deals`.

3. **Validation completely disabled** (deal-form/validation.ts)
   - All validation functions return `true` / empty errors. While this may be intentional, it means:
     - Deals can be created with no name (falls back to "Untitled Deal")
     - Deals can be moved to Won stage with no revenue data
     - No date logic validation exists
   - This is a data integrity risk for production.

### HIGH

4. **Dashboard queries may hit 1000-row Supabase limit** (useDashboardData.tsx:27-31)
   - Dashboard fetches all deals/accounts/contacts/action_items with `.select('stage')` etc. but does NOT paginate.
   - With 500+ deals, results will be silently truncated at 1000 rows, showing incorrect counts.
   - Fix: Use `.select('stage', { count: 'exact', head: true })` or aggregate via RPC.

5. **Deal form `handleActionButtonClick` always opens DealActionItemsModal** (DealForm.tsx:277-281)
   - The previous fix to navigate to action items for "View Tasks" was reverted — it now just opens `setActionModalOpen(true)` regardless. The button label says "Action" which is ambiguous.

6. **No signup flow** (Auth.tsx)
   - Auth page only has sign-in. No registration form exists. New users can only be created by admins via UserManagement. This is fine if intentional, but should be documented.

7. **Account status mismatch** (Accounts.tsx vs useDashboardData.tsx)
   - Filter options: `New, Working, Qualified, Inactive`
   - Dashboard counts: `New, Working, Hot, Nurture`
   - These don't match — some statuses will never appear in dashboard or filter.

### MEDIUM

8. **DealsPage fetches ALL deals into memory** (DealsPage.tsx:36-39)
   - Uses `fetchAllRecords` which loops through all pages. With 500+ deals, this loads everything into state.
   - Kanban view may be slow with large datasets. Consider server-side pagination for list view.

9. **Real-time subscription may cause duplicate entries** (DealsPage.tsx:295-296)
   - On INSERT event, `setDeals(prev => [payload.new as Deal, ...prev])` — but `handleSaveDeal` for creation does NOT manually add to state (comment says "Real-time subscription handles adding"). If real-time fires before the save completes, could cause race conditions.

10. **No error boundary around lazy-loaded settings pages** (Settings.tsx:188)
    - Uses `Suspense` but no `ErrorBoundary` wrapping lazy components. If a chunk fails to load, the entire settings page crashes.

11. **Notification "Clear All" deletes without confirmation** (Notifications.tsx:92-116)
    - `handleClearAll` immediately deletes ALL notifications without a confirmation dialog. Destructive action should require confirmation.

12. **Campaign delete doesn't clean up `campaign_email_templates` or `campaign_phone_scripts`** (useCampaigns.tsx:70-76)
    - Deletes `campaign_contacts`, `campaign_accounts`, `campaign_communications`, `campaign_materials` but misses `campaign_email_templates` and `campaign_phone_scripts`.
    - Creates orphan template/script records.

13. **Contact modal missing `account_id` field in database** 
    - Contacts table has no `account_id` column. Contact-Account linkage appears to rely on `company_name` text matching, which is fragile. There's no formal FK relationship.

14. **`useNotifications` uses manual state management instead of React Query**
    - Unlike other hooks that use `useQuery`, notifications use `useState` + manual fetching. This means no automatic cache invalidation, no stale-while-revalidate, and potential stale data.

### LOW

15. **Sidebar open state stored in localStorage but not synced across tabs**
    - Minor UX issue; opening sidebar in one tab doesn't reflect in another.

16. **Date picker inconsistency in Campaigns filter bar**
    - Campaigns page uses native `h-9` height on inputs while other pages use default height (post recent changes).

17. **`useSecureDeals` hook exists but is not used by DealsPage**
    - DealsPage directly queries Supabase. The secure wrapper hook appears unused in the main deals flow.

18. **Action Items `archived_at` field exists but archival is manual via DB function**
    - `archive_completed_action_items()` function exists but there's no cron job or trigger calling it automatically. The "Completed" toggle in UI filters by status, not `archived_at`.

19. **Error type assertion missing in DealForm** (DealForm.tsx:144)
    - `error.message` accessed without type guard: `${error.message || 'Unknown error'}` — `error` could be non-Error type.

---

## UI/UX Inconsistencies

1. **Header height**: All pages use `h-16` consistently — good.
2. **Filter bar styling**: Most pages use `bg-muted/30 border-b` — Campaigns uses `border-border` instead of `border-b` alone.
3. **Add button placement**: Dashboard has no "Add" button. Accounts/Contacts have top-right. Deals has it inside the view toggle area. Campaigns uses `size="sm"`. Inconsistent sizing.
4. **Search input width**: Accounts/Contacts/ActionItems use `max-w-[300px]`. Campaigns uses `max-w-xs` (320px). Minor but inconsistent.
5. **Select trigger styling**: Accounts/Contacts/ActionItems hide chevron with `[&>svg]:hidden`. Campaigns does not.
6. **Bulk actions bar**: Accounts shows below filters. Contacts puts delete inside dropdown menu AND has no separate bulk bar. Action Items has a proper bulk bar. Inconsistent patterns.

---

## Data Integrity Risks

1. No CASCADE delete on `deal_stakeholders` when contacts are deleted.
2. No cleanup of `campaign_email_templates`/`campaign_phone_scripts` on campaign delete.
3. Contact-Account relationship is text-based (`company_name`), not FK-based.
4. Deals `account_id` FK exists in schema but Accounts delete sets it to NULL (correct), however contacts delete does NOT null out stakeholder contact IDs in deals.

---

## Security Notes

- RLS policies are properly configured across all tables.
- Role checking uses `SECURITY DEFINER` functions — correct pattern.
- Admin checks use server-side `get_user_role()` — no client-side role storage.
- Edge function `user-admin` properly validates admin role before operations.
- One concern: `page_permissions` SELECT is open to all authenticated users (needed for access check), but this exposes the full permission matrix to all users.

---

## Performance Concerns

1. **DealsPage loads ALL deals** — will degrade with 500+ deals.
2. **Dashboard runs 7 parallel queries** without pagination — will truncate at 1000 rows.
3. **KanbanBoard re-renders all cards** on any deal update via real-time subscription.
4. **`fetchAllRecords` used for CSV export** — appropriate for export, but also used for deals page initial load.

---

## Implementation Plan (Priority Order)

### Task 1: Fix `modified_by` bug in DealForm (Critical)
- **File**: `src/components/DealForm.tsx`
- Change all instances of `modified_by: deal?.created_by || formData.created_by` to `modified_by: user?.id`
- Affects lines ~124, ~163, ~200, ~236

### Task 2: Fix contacts bulk delete orphan cleanup (Critical)
- **File**: `src/pages/Contacts.tsx`
- Before deleting contacts, clean up:
  - `deal_stakeholders` where `contact_id` IN selected IDs
  - `deals` stakeholder columns (set to null where matching)
  - `campaign_contacts` where `contact_id` IN selected IDs

### Task 3: Fix campaign delete orphan cleanup (Medium)
- **File**: `src/hooks/useCampaigns.tsx`
- Add deletion of `campaign_email_templates` and `campaign_phone_scripts` in the campaign delete mutation

### Task 4: Fix account status mismatch (High)
- Align status options between `Accounts.tsx` filter and `useDashboardData.tsx` dashboard counts

### Task 5: Fix dashboard 1000-row limit (High)
- **File**: `src/hooks/useDashboardData.tsx`
- Use count queries or RPC functions instead of fetching all rows

### Task 6: Add confirmation dialog for "Clear All" notifications (Medium)
- **File**: `src/pages/Notifications.tsx`

### Task 7: Add ErrorBoundary around lazy settings (Low)
- **File**: `src/pages/Settings.tsx`

### Task 8: Fix type assertion on error in DealForm (Low)
- **File**: `src/components/DealForm.tsx`

---

## Summary by Severity

| Severity | Count |
|----------|-------|
| Critical | 3 |
| High | 4 |
| Medium | 7 |
| Low | 5 |

**Recommended first action**: Fix the `modified_by` bug and contacts delete orphan cleanup — these are data corruption risks in production.

