# ChatKit Local Debug Guide

## Problem Summary

When running the OpenAI ChatKit starter app locally, the application was failing with a 404 error:

```
Workflow with id 'wf_69023ae2976c819081ef6b9e6236f2cf0e4b5e09ca3a79fa' not found.
```

Despite having:
- A valid workflow ID
- A valid API key
- Correct `.env` file configuration
- Successful direct API testing with curl

## Root Cause

The issue was caused by a **system environment variable** `OPENAI_API_KEY` that was set in the shell environment. This environment variable was overriding the `.env` and `.env.local` files, causing the application to use a different API key than the one configured in the project files.

The key indicator was in the debug logs showing `keyPrefix: 'sk-proj-LD'` when the correct key in `.env` started with `sk-proj-Bc`.

## Technical Details

### How Environment Variables Work in Next.js

Next.js loads environment variables in the following order (higher priority first):
1. **System/Shell environment variables** (highest priority)
2. `.env.local` (not committed to git)
3. `.env` (committed to git)

This means if you have an `OPENAI_API_KEY` set in your system environment, it will **always override** the values in your `.env` files.

### Why This Caused a 404 Error

OpenAI's ChatKit API associates workflows with specific organizations and projects. When using an API key from a different organization/project than where the workflow was created, the API returns a 404 "Workflow not found" error instead of a permissions error.

## Troubleshooting Playbook

### Step 1: Verify the Issue

Check if a system environment variable is overriding your `.env` files:

```bash
# Check for OPENAI_API_KEY in your environment
printenv | grep OPENAI

# Or check what Node.js sees
node -e "console.log('OPENAI_API_KEY:', process.env.OPENAI_API_KEY?.substring(0, 10))"
```

Compare the output with your `.env` file:

```bash
cat .env | grep OPENAI_API_KEY
```

If the prefixes don't match, you have an environment variable override.

### Step 2: Unset the System Environment Variable

**For the current terminal session (temporary):**

```bash
# On Linux/Mac/Git Bash
unset OPENAI_API_KEY

# On Windows Command Prompt
set OPENAI_API_KEY=

# On Windows PowerShell
Remove-Item Env:OPENAI_API_KEY
```

**To permanently remove it:**

- **Linux/Mac**: Remove it from `~/.bashrc`, `~/.zshrc`, or `~/.bash_profile`
- **Windows**:
  1. Search for "Environment Variables" in Start Menu
  2. Click "Edit the system environment variables"
  3. Click "Environment Variables" button
  4. Remove `OPENAI_API_KEY` from User or System variables

### Step 3: Use `.env.local` (Best Practice)

The `.env.local` file takes precedence over `.env` and is ignored by git. Create it with your sensitive credentials:

```bash
cp .env.example .env.local
```

Edit `.env.local` with your actual values:

```
OPENAI_API_KEY=sk-proj-YOUR-ACTUAL-KEY
NEXT_PUBLIC_CHATKIT_WORKFLOW_ID=wf_YOUR-WORKFLOW-ID
```

### Step 4: Restart the Development Server

Environment variables are loaded at startup, so you must restart:

```bash
# Kill any running servers
npx kill-port 3000

# Start the server in a clean shell
unset OPENAI_API_KEY && npm run dev
```

### Step 5: Verify the Fix

Add debug logging to `app/api/create-session/route.ts` (if needed):

```typescript
if (process.env.NODE_ENV !== "production") {
  console.info("[create-session] API key check", {
    hasKey: !!openaiApiKey,
    keyPrefix: openaiApiKey?.substring(0, 10),
  });
}
```

Check the logs when you refresh the page - the `keyPrefix` should match your `.env.local` file.

## Prevention Tips

1. **Never set `OPENAI_API_KEY` as a system environment variable** when working with multiple projects
2. **Always use `.env.local`** for local development secrets
3. **Add `.env.local` to `.gitignore`** (it should already be there)
4. **Check environment variables** before starting development:
   ```bash
   printenv | grep OPENAI
   ```
5. **Read the README warnings** - the OpenAI ChatKit README explicitly warns about this issue

## Additional Notes

### Verifying API Key and Workflow Compatibility

You can test if your API key can access the workflow directly:

```bash
curl -X POST "https://api.openai.com/v1/chatkit/sessions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR-API-KEY" \
  -H "OpenAI-Beta: chatkit_beta=v1" \
  -d '{"workflow":{"id":"YOUR-WORKFLOW-ID"},"user":"test-user"}'
```

A successful response (200) confirms the API key and workflow are compatible.

### Domain Allowlist (For Deployment Only)

Domain permissions are **only required for production deployment**, not for local development. Before deploying:

1. Go to [OpenAI Domain Allowlist](https://platform.openai.com/settings/organization/security/domain-allowlist)
2. Add your deployment domain (e.g., `yourapp.vercel.app`)

This is not related to the "workflow not found" error during local development.

## Summary

The key lesson: **System environment variables always override `.env` files in Next.js**. When troubleshooting, always check for environment variable conflicts before assuming there's an issue with your configuration files or API credentials.
