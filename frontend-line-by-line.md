# Frontend Line-By-Line Companion

The frontend is a Vite + React client. This guide points you to the most important sections and explains the intent of each block so you can follow the source quickly.

## src/main.tsx
| Lines | Explanation |
| --- | --- |
| 1-2 | Import React core and the concurrent `createRoot` entry. |
| 4 | Import the root `App` component. |
| 6-10 | Bootstrap the application: find the `#root` element, wrap `App` in `React.StrictMode`, and render it. |

## src/App.tsx
`App.tsx` orchestrates routing, theming, and navigation without React Router. Major sections:

| Lines | Focus |
| --- | --- |
| 1-20 | Imports React hooks, global styles, all page components, and type aliases from `api.ts`. |
| 22-58 | Declares the core role/auth types used throughout the UI. |
| 60-73 | Navigation entry shape and storage keys for auth/theme persistence. |
| 75-107 | `getInitialPath`, `loadAuthState`, and `persistAuthState` manage history + local storage. |
| 109-126 | Helper constants and `isOnboardingComplete` flag used to redirect humans during signup. |
| 128-136 | Reads the stored theme preference, defaulting to light. |
| 138-165 | Component definition: create state hooks for path, auth, theme, and nav expansion. |
| 167-180 | `navigate` updates both history and the local `path` state. |
| 182-202 | `popstate` listener keeps `path` synced with browser navigation. |
| 204-210 | Persist auth to storage whenever the state changes. |
| 212-222 | Apply theme selection to the `<html>` element and store it. |
| 224-230 | Redirect legacy `/journey` route to `/signup`. |
| 232-240 | Force humans in onboarding to stay on the onboarding route. |
| 242-258 | Login handlers for human/admin actors, orchestrating redirects. |
| 260-270 | Logout handler clears auth state. |
| 272-306 | `navEntries` memo builds the nav menu based on the active role. |
| 308-318 | `updateHumanStage` lets child components refresh onboarding progress. |
| 320-336 | Theme toggle and related UI constants. |
| 338-402 | Scroll animation effect: sets data attributes, observes DOM mutations, and triggers CSS transitions. |
| 404-463 | `content` memo: manual router that selects which page to render by inspecting `path` and role. |
| 465-490 | Collapse the nav drawer whenever the path changes. |
| 492-513 | Handlers for nav toggling, closing, and wheel interactions. |
| 515-562 | Render tree: header with title button, nav, theme toggle, main content, and footer. |
| 564-602 | `NavItem` component: renders either `<a>` or `<button>` entries and calls `navigate` without full page reloads. |
| 604-628 | `NotFound`: fallback view with CTA back home. |
| 630-664 | `AccessDenied`: explains role mismatch and offers context-aware redirect buttons. |

## src/api.ts
At 860+ lines this module centralises every REST call and shared TypeScript interface.

| Lines | Focus |
| --- | --- |
| 1-24 | `resolveApiBase` inspects environment variables and window location to find the API origin. |
| 26 | Caches the resolved base in `API_BASE`. |
| 28-360 | TypeScript interfaces mirroring backend Pydantic models (users, DNA, memories, insurance, support). |
| 362-382 | Support payload types for admin interactions. |
| 384-413 | Generic `request` helper wraps `fetch`, applies JSON headers, parses errors, and returns typed responses. |
| 415-520 | Auth, signup, and health profile payload/response types plus the functions that call `/users` and `/auth` routes. |
| 522-604 | Miniaturisation request + payment helpers. |
| 606-664 | Personality assessment and token issuance calls. |
| 666-714 | Memory API wrappers (`recordMemoryLog`, `fetchMemoryLogs`, `fetchMemoryTokens`). |
| 716-734 | Organism state + feed endpoints. |
| 736-804 | Admin overview, requests, settings, insurance, token management, and user updates. |
| 806-867 | Support session CRUD for both user and admin dashboards. |

Each exported function corresponds one-to-one with a backend endpoint; search for the name inside `api.ts` when wiring pages.

## src/pages/
Every file inside `pages/` renders a full view. The pattern repeats: import data hooks, request helper(s) from `api.ts`, fetch during `useEffect`, render cards/tables, and expose callbacks to mutate state. To inspect a specific workflow:

1. Locate the file (e.g. `pages/AdminMemories.tsx`).
2. Note the API functions it imports (they map back to sections above).
3. Follow local state declarations and event handlers—they are typically declared near the top of the component and wired directly into JSX.

## src/components/UserOverviewModal.tsx
| Lines | Explanation |
| --- | --- |
| 1-36 | Imports and prop types for the modal. |
| 38-132 | Modal component: renders user profile, memory summary, tokens, dreams, and support stats using the `UserOverview` type. Conditional blocks guard against missing data. |
| 134-164 | Helper components for rendering tokens and grids; styled via global CSS. |

## src/hooks/useAdminOverview.ts
| Lines | Explanation |
| --- | --- |
| 1-30 | Imports API helpers and React utilities. |
| 32-94 | Hook definition: fetches admin overview data on mount, exposes loading/error flags, and returns the overview snapshot plus reload function. |
| 96-118 | Normalises nested counts and ensures the hook never returns partial structures to the UI. |

## Styles & Static Assets
- `App.css`, `pages/*.css`, and `pages/admin-common.css` contain the themed layout and animation hooks referenced in `App.tsx` (e.g. `[data-scroll-fade]`).
- `public/admin/marova/hologram/` holds the ambient hologram UI embedded in the admin view.

Use this companion with a code editor: open the file, jump to the line range noted above, and read the code beside the explanation to internalise how data flows through the frontend.
