# State Contract

## Authentication state
- Web: In-memory token. Refresh via HttpOnly cookie.
- Mobile: Zustand + SecureStore.

## Server state
React Query for all API responses. Must be cleared on logout.

## Pagination State (WEB-14 Admin Users)
- **INITIAL LOAD**: Normal loading state.
- **NEXT PAGE LOAD**: Existing users remain visible. Load More becomes disabled/loading.
- **NEXT PAGE SUCCESS**: New users append. New `next_cursor` replaces old cursor.
- **NEXT PAGE ERROR**: Existing users remain visible. Pagination position remains recoverable. Documented error is shown.
