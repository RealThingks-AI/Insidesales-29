

## Bug: Today's Tasks Popup reopens on every Dashboard navigation

**Root cause**: The popup's `useEffect` runs on every component mount. Dismissing the popup does NOT persist anything unless the "Don't show again today" checkbox is checked. So navigating away and back remounts the component and reopens the popup.

**Fix** in `src/components/dashboard/TodaysTasksPopup.tsx`:

1. Add a `sessionStorage` key (`tasks-popup-shown-session`) to track if the popup was already shown this browser session
2. On mount: check both `localStorage` (day-level dismiss) AND `sessionStorage` (session-level shown). Only open if neither is set.
3. On any dismiss (with or without checkbox):
   - Always set `sessionStorage` so it won't reopen during this session
   - If checkbox is checked, also set `localStorage` so it won't open even on a new login that same day
4. This gives the correct behavior:
   - First visit of the day → popup shows
   - Navigate away and back → popup does NOT show (sessionStorage)
   - New login/new tab → popup shows again (sessionStorage is fresh)
   - "Don't show again today" checked → won't show even on new login that day (localStorage)

Single file change, ~10 lines modified.

