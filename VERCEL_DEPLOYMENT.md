# Vercel Deployment Guide

This guide will walk you through deploying your ChatKit application to Vercel.

## Prerequisites

1. A [Vercel account](https://vercel.com/signup)
2. Your OpenAI API key from the same org/project as your workflow
3. Your published workflow ID from [Agent Builder](https://platform.openai.com/agent-builder)
4. GitHub repository set up (already done: https://github.com/AI-MakesSense/agentkit-chat)

## Step 1: Add Domain to OpenAI Allowlist

**IMPORTANT**: Before deploying, you must add your Vercel domain to OpenAI's domain allowlist.

1. Go to [OpenAI Domain Allowlist Settings](https://platform.openai.com/settings/organization/security/domain-allowlist)
2. Click "Add Domain"
3. Add your Vercel domain:
   - If using Vercel's default domain: `your-project-name.vercel.app`
   - If using a custom domain: `yourdomain.com`
4. Click "Save"

> **Note**: You can deploy first to get your Vercel URL, then add it to the allowlist, but the chat won't work until the domain is allowlisted.

## Step 2: Deploy to Vercel

### Option A: Deploy via Vercel Dashboard (Recommended)

1. Go to [Vercel Dashboard](https://vercel.com/new)
2. Click "Import Project"
3. Select "Import Git Repository"
4. Paste your repository URL: `https://github.com/AI-MakesSense/agentkit-chat`
5. Click "Import"
6. Configure your project:
   - **Framework Preset**: Next.js (should auto-detect)
   - **Root Directory**: `./` (leave as default)
   - **Build Command**: `npm run build` (auto-filled)
   - **Output Directory**: `.next` (auto-filled)

### Option B: Deploy via Vercel CLI

```bash
# Install Vercel CLI globally
npm i -g vercel

# Navigate to project directory
cd openai-chatkit-starter-app

# Login to Vercel
vercel login

# Deploy
vercel
```

## Step 3: Configure Environment Variables

After creating the project, add your environment variables:

1. In Vercel Dashboard, go to your project
2. Click "Settings" → "Environment Variables"
3. Add the following variables:

| Variable Name | Value | Environment |
|---------------|-------|-------------|
| `OPENAI_API_KEY` | `sk-proj-...` | Production, Preview, Development |
| `NEXT_PUBLIC_CHATKIT_WORKFLOW_ID` | `wf_...` | Production, Preview, Development |

**Important Notes:**
- Make sure to select all environments (Production, Preview, Development) for each variable
- The `OPENAI_API_KEY` must be from the same organization/project as your workflow
- After adding variables, redeploy your application

### To Redeploy After Adding Variables:

1. Go to "Deployments" tab
2. Click the three dots on the latest deployment
3. Select "Redeploy"
4. Check "Use existing Build Cache"
5. Click "Redeploy"

## Step 4: Verify Organization (If Using GPT-5 or Premium Models)

If your workflow uses models requiring organization verification:

1. Go to [Organization Settings](https://platform.openai.com/settings/organization/general)
2. Click "Verify Organization"
3. Follow the verification process

## Step 5: Test Your Deployment

1. Visit your Vercel URL (e.g., `https://your-project-name.vercel.app`)
2. You should see the ChatKit interface
3. Try the starter prompts to verify the workflow connection
4. Check browser console for any errors (F12 → Console)

### Common Issues:

**"Workflow not found" error:**
- Verify your `OPENAI_API_KEY` is from the same org/project as the workflow
- Check that `NEXT_PUBLIC_CHATKIT_WORKFLOW_ID` is correct
- Ensure the workflow is published in Agent Builder

**Chat doesn't load:**
- Verify your domain is added to OpenAI's allowlist
- Check browser console for CORS or network errors
- Ensure environment variables are set correctly

**Session creation fails:**
- Check Vercel function logs (Dashboard → Deployments → Click deployment → Functions tab)
- Verify `OPENAI_API_KEY` is set and valid

## Step 6: Custom Domain (Optional)

To use a custom domain:

1. In Vercel Dashboard, go to your project
2. Click "Settings" → "Domains"
3. Click "Add"
4. Enter your domain name
5. Follow DNS configuration instructions
6. **Important**: Add your custom domain to OpenAI's domain allowlist (Step 1)

## Monitoring and Logs

### View Function Logs:
1. Go to your project in Vercel Dashboard
2. Click "Deployments"
3. Click on a deployment
4. Go to "Functions" tab
5. Click on a function to see logs

### View Analytics:
1. Go to "Analytics" tab in your project
2. View page views, performance metrics, and errors

## Environment-Specific Deployments

Vercel automatically creates deployments for:
- **Production**: Pushes to `main` branch
- **Preview**: Pull requests and other branches

You can set different environment variables for each environment in Settings → Environment Variables.

## Updating Your Deployment

To update your deployed app:

```bash
# Make changes locally
git add .
git commit -m "Your commit message"
git push origin main
```

Vercel will automatically:
1. Detect the push
2. Build your project
3. Deploy to production
4. Update your live site

## Rollback

If something goes wrong:

1. Go to "Deployments" tab
2. Find a previous working deployment
3. Click the three dots
4. Select "Promote to Production"

## Performance Optimization

Your app is already optimized for Vercel:
- ✅ Edge Runtime for API routes (fast global execution)
- ✅ Automatic CDN for static assets
- ✅ Image optimization
- ✅ Automatic HTTPS

## Security Checklist

Before going live:
- [ ] Domain added to OpenAI allowlist
- [ ] Environment variables set in Vercel (not committed to git)
- [ ] `.env` and `.env.local` in `.gitignore`
- [ ] API key has appropriate permissions
- [ ] Organization verified (if required)

## Cost Considerations

**Vercel Costs:**
- Hobby plan: Free for personal projects
- Pro plan: $20/month for production apps

**OpenAI API Costs:**
- Charged per API call/token usage
- Monitor usage at [OpenAI Usage Dashboard](https://platform.openai.com/usage)

## Support

- **Vercel Docs**: https://vercel.com/docs
- **ChatKit Docs**: http://openai.github.io/chatkit-js/
- **OpenAI Support**: https://help.openai.com/

## Next Steps

After deployment:
1. Customize the UI in `lib/config.ts`
2. Add custom analytics tracking
3. Integrate with your backend services
4. Set up monitoring and alerting
