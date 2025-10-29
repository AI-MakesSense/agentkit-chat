# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an OpenAI ChatKit starter application built with Next.js 15.5, React 19, and TypeScript. It provides a minimal integration with OpenAI's hosted workflows via the ChatKit web component, allowing users to interact with AI agents built in Agent Builder.

## Commands

### Development
```bash
npm install                    # Install dependencies
npm run dev                    # Start dev server (http://localhost:3000)
npm run build                  # Create production build
npm start                      # Start production server
npm run lint                   # Run ESLint
```

### Environment Setup
```bash
cp .env.example .env.local     # Create local environment file
unset OPENAI_API_KEY           # Clear system env var (Linux/Mac)
set OPENAI_API_KEY=            # Clear system env var (Windows)
```

## Architecture Overview

### Server-Client Split
- **Server Components**: `layout.tsx` (ChatKit script injection), `app/api/create-session/route.ts` (session creation)
- **Client Components**: `App.tsx` (theme orchestration), `ChatKitPanel.tsx` (ChatKit integration)
- **Edge Runtime**: API routes use Next.js Edge Runtime for fast, globally distributed execution

### Key Data Flow
```
User loads page
→ layout.tsx injects ChatKit script from CDN
→ App.tsx initializes theme from localStorage/system preference
→ ChatKitPanel calls getClientSecret()
→ POST /api/create-session (server-side)
  → Validates OPENAI_API_KEY and WORKFLOW_ID
  → Creates session via OpenAI ChatKit API
  → Returns client_secret + sets HttpOnly cookie with user ID
→ ChatKit component renders with session credentials
```

### Session Management
- User IDs are generated via `crypto.randomUUID()` and persisted in HttpOnly cookies (30-day expiry)
- Sessions are created by calling `POST https://api.openai.com/v1/chatkit/sessions` with:
  - `Authorization: Bearer OPENAI_API_KEY`
  - `OpenAI-Beta: chatkit_beta=v1`
  - Body: `{ workflow: { id }, user, chatkit_configuration }`
- The `client_secret` returned is passed to the ChatKit web component for authentication

## Configuration System

### Centralized Config (`lib/config.ts`)
All UI customization points are in this single file:
- `STARTER_PROMPTS`: Quick-start conversation prompts
- `GREETING`: Welcome message
- `PLACEHOLDER_INPUT`: Chat input placeholder
- `getThemeConfig(theme)`: ChatKit theme options (colors, radius, typography)

### Environment Variables
**Required:**
- `OPENAI_API_KEY` (server-side): Must be from same org/project as workflow
- `NEXT_PUBLIC_CHATKIT_WORKFLOW_ID` (public): Workflow ID from Agent Builder (starts with `wf_`)

**Optional:**
- `CHATKIT_API_BASE`: Custom ChatKit API endpoint (defaults to `https://api.openai.com`)

**Critical**: System environment variables override `.env.local` files. Always `unset OPENAI_API_KEY` before running `npm run dev` to avoid using wrong credentials.

### Theme System
- Theme preference stored in localStorage as `"color-scheme-preference"`
- Syncs with system preference via `matchMedia("(prefers-color-scheme: dark)")`
- Uses `useSyncExternalStore` for proper SSR hydration
- DOM updates: `data-color-scheme` attribute, `.dark` class, `style.colorScheme`

## Component Architecture

### ChatKitPanel.tsx (Main Integration)
This component handles:
- **Script Loading**: Waits for `chatkit-script-loaded` event or checks `window.customElements.get("openai-chatkit")`
- **Session Initialization**: Calls `getClientSecret()` to create sessions
- **Error States**: Tracks `script`, `session`, and `integration` errors independently
- **Tool Invocations**: Handles `onClientTool` callbacks for:
  - `switch_theme`: Requests theme change from parent
  - `record_fact`: Saves facts with deduplication via `processedFacts` Set
- **Lifecycle Management**: Uses `isMountedRef` to prevent state updates after unmount
- **Reset Capability**: `widgetInstanceKey` forces ChatKit re-instantiation on reset

### Key Patterns

**Mounted Ref Pattern:**
```typescript
const isMountedRef = useRef(true);
useEffect(() => () => { isMountedRef.current = false; }, []);
// Always check isMountedRef.current before setState
```

**Error Hierarchy:**
```typescript
const blockingError = errors.script ?? errors.session;
// Script errors block everything
// Session errors block chat but allow script retry
// Integration errors are non-blocking (handled by ChatKit UI)
```

**Browser Guards:**
```typescript
const isBrowser = typeof window !== "undefined";
// Prevents SSR errors when accessing browser APIs
```

## API Route Details

### `app/api/create-session/route.ts`
- **Runtime**: Edge (globally distributed)
- **Method**: POST only (returns 405 for GET)
- **Cookie Management**: Reads/writes `chatkit_session_id` cookie
- **User ID Resolution**: Cookie value or `crypto.randomUUID()`
- **Error Extraction**: Recursively searches nested error objects for messages
- **Development Logging**: Logs request/response details when `NODE_ENV !== "production"`

**Request Body Structure:**
```typescript
{
  workflow?: { id?: string | null };
  workflowId?: string | null;  // Alternative field
  chatkit_configuration?: {
    file_upload?: { enabled?: boolean };
  };
}
```

**Critical Validation:**
1. Checks `OPENAI_API_KEY` is set
2. Resolves workflow ID from body or `NEXT_PUBLIC_CHATKIT_WORKFLOW_ID`
3. Validates workflow ID is not empty or placeholder

## Customization Points

### UI Customization
1. **Prompts & Text**: Edit `lib/config.ts` → `STARTER_PROMPTS`, `GREETING`, `PLACEHOLDER_INPUT`
2. **Theme**: Edit `getThemeConfig()` in `lib/config.ts` (see https://chatkit.studio/playground)
3. **Container Styling**: Edit Tailwind classes in `ChatKitPanel.tsx:347` (`h-[90vh]`, `rounded-2xl`, etc.)

### Event Handlers
Override these in `App.tsx`:
- `onWidgetAction`: Handle fact saving or custom tool invocations
- `onResponseEnd`: Track conversation completions (analytics)
- `onThemeRequest`: Respond to theme change requests from ChatKit

### Advanced Customization
- **Session Logic**: Modify `app/api/create-session/route.ts` for custom user ID strategies
- **Error Handling**: Update `ErrorOverlay.tsx` for custom error UI
- **Tool Invocations**: Add new tool handlers in `ChatKitPanel.tsx` `onClientTool` callback

## Deployment Requirements

1. **Domain Allowlist**: Add deployment domain to [OpenAI Domain Allowlist](https://platform.openai.com/settings/organization/security/domain-allowlist)
2. **Organization Verification**: Required for models like GPT-5 at [Org Settings](https://platform.openai.com/settings/organization/general)
3. **Environment Variables**: Set `OPENAI_API_KEY` and `NEXT_PUBLIC_CHATKIT_WORKFLOW_ID` in hosting platform
4. **Build Command**: `npm run build`
5. **Start Command**: `npm start`

## Critical Troubleshooting

### "Workflow not found" Error (404)
**Root Cause**: API key from different org/project than workflow
**Solution**:
1. Verify workflow ID is correct from Agent Builder
2. Check API key is from same org/project as workflow
3. Run `printenv | grep OPENAI` to check for system env variable override
4. If system var exists, `unset OPENAI_API_KEY` and restart server

### Script Loading Timeout
**Root Cause**: ChatKit script blocked or CDN unreachable
**Solution**: Check browser console for script loading errors, verify CDN URL in `layout.tsx`

### Session Creation Fails
**Root Cause**: Missing or invalid environment variables
**Solution**: Verify `.env.local` exists with valid `OPENAI_API_KEY` and `NEXT_PUBLIC_CHATKIT_WORKFLOW_ID`

## Important Notes

- The ChatKit web component is loaded from OpenAI's CDN, not bundled with the app
- Session creation happens server-side to protect `OPENAI_API_KEY`
- User IDs persist across sessions via cookies for conversation history
- Theme changes are applied immediately but persisted on next render cycle
- Development mode includes detailed logging that's stripped in production
- File uploads are enabled by default (see `ChatKitPanel.tsx:277`)

## References

- [ChatKit JavaScript Docs](http://openai.github.io/chatkit-js/)
- [ChatKit Theme Playground](https://chatkit.studio/playground)
- [Agent Builder](https://platform.openai.com/agent-builder)
- [Advanced Examples](https://github.com/openai/openai-chatkit-advanced-samples)
