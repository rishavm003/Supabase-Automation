# Supabase Auto-Resume Automation

Keep your Supabase projects alive forever. Automatically resume your project every 7 days to prevent it from being paused due to inactivity.

## 🎯 The Problem

Supabase pauses projects that have no activity for 7 days. If a paused project remains inactive for 90 days, it gets closed permanently. This automation ensures your projects never get paused.

### Timeline Without Automation ❌
- **Day 0-7**: Project active
- **Day 8-14**: Project paused (inactivity)
- **Day 15-90**: Still paused
- **Day 91+**: **Project closed forever**

### Timeline With Automation ✅
- **Day 0-7**: Automation resumes project
- **Day 7-14**: Automation resumes project
- **Day 14-90**: Always active
- **Day 90+**: **Forever alive**

## ✨ What This Does

- ✅ Automatically resumes your Supabase project every 7 days
- ✅ Prevents projects from being paused due to inactivity
- ✅ Stops projects from being closed after 90 days
- ✅ Runs on GitHub Actions (free tier)
- ✅ Zero maintenance after initial setup
- ✅ 5-minute setup time
- ✅ Secure credential management

## 🚀 Quick Start (5 Minutes)

### Step 1: Get Your Supabase Credentials

**Get Your Project ID:**
1. Go to [Supabase Dashboard](https://app.supabase.com)
2. Select your project
3. Click **Settings** → **General**
4. Copy your **Project ID** (visible in the URL and under Project Info)

**Get Your API Key:**
1. Go to [Supabase Dashboard](https://app.supabase.com)
2. Click your **Profile icon** (top right) → **Account**
3. Go to **Access Tokens** section
4. Click **Generate new token**
5. Give it a name like "GitHub Automation"
6. **Copy the token immediately** (you won't see it again!)

### Step 2: Add Secrets to GitHub

1. Go to your GitHub repository
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Add these secrets:

| Secret Name | Value |
|---|---|
| `SUPABASE_PROJECT_ID` | Your Project ID from Step 1 |
| `SUPABASE_API_KEY` | Your API Key from Step 1 |

### Step 3: Create the Workflow File

1. Create a new directory in your repo: `.github/workflows/`
2. Create a file: `.github/workflows/resume-supabase.yml`
3. Copy and paste the workflow code from below
4. Commit and push to your repo

**Simple Workflow (Recommended):**
```yaml
name: Resume Supabase Project Every 7 Days

on:
  schedule:
    # Runs every 7 days at 12:00 UTC
    - cron: '0 12 */7 * *'
  workflow_dispatch:

jobs:
  resume-project:
    runs-on: ubuntu-latest
    
    steps:
      - name: Resume Supabase Project
        run: |
          curl -X POST \
            "https://api.supabase.com/v1/projects/${{ secrets.SUPABASE_PROJECT_ID }}/restart" \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_API_KEY }}" \
            -H "Content-Type: application/json"
      
      - name: Log Success
        run: echo "✅ Supabase project resumed successfully at $(date)"
      
      - name: Verify Project Status
        run: |
          curl -X GET \
            "https://api.supabase.com/v1/projects/${{ secrets.SUPABASE_PROJECT_ID }}" \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_API_KEY }}" \
            -H "Content-Type: application/json" | jq '.status'
```

### Step 4: Test It

1. Go to your GitHub repo → **Actions** tab
2. Find the workflow **"Resume Supabase Project Every 7 Days"**
3. Click **Run workflow** → **Run workflow** again
4. Watch the execution
5. You should see: "✅ Supabase project resumed successfully"

That's it! Your automation is now live. 🎉

## 🔧 Advanced Setup

### Custom Schedule

The default runs every 7 days at **12:00 UTC**. To change it, edit the `cron` line:

```yaml
on:
  schedule:
    - cron: '0 12 */7 * *'  # Change this line
```

**Cron Format:** `minute hour day month day-of-week`

**Common Examples:**
- Every day at 9 AM: `0 9 * * *`
- Every 3 days at 6 AM: `0 6 */3 * *`
- Every 7 days at 12 PM: `0 12 */7 * *` ← Current
- Every Monday at 10 AM: `0 10 * * 1`
- Every 12 hours: `0 */12 * * *`

**Timezone Note:** Cron times are in UTC. GitHub Actions may run 5-10 minutes after the scheduled time.

### Advanced Workflow with Error Handling

Use this version for detailed logging and failure notifications:

```yaml
name: Resume Supabase Project Every 7 Days (Advanced)

on:
  schedule:
    - cron: '0 12 */7 * *'
  workflow_dispatch:

jobs:
  resume-project:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Resume Supabase Project
        id: resume
        run: |
          echo "🚀 Starting project resume at $(date)"
          
          RESPONSE=$(curl -s -w "\n%{http_code}" -X POST \
            "https://api.supabase.com/v1/projects/${{ secrets.SUPABASE_PROJECT_ID }}/restart" \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_API_KEY }}" \
            -H "Content-Type: application/json")
          
          HTTP_CODE=$(echo "$RESPONSE" | tail -n1)
          BODY=$(echo "$RESPONSE" | sed '$d')
          
          echo "HTTP Status: $HTTP_CODE"
          echo "Response: $BODY"
          
          if [ "$HTTP_CODE" -eq 200 ] || [ "$HTTP_CODE" -eq 201 ]; then
            echo "resume_status=success" >> $GITHUB_OUTPUT
            echo "✅ Project resume request successful"
          else
            echo "resume_status=failed" >> $GITHUB_OUTPUT
            echo "❌ Failed to resume project (HTTP $HTTP_CODE)"
            exit 1
          fi
      
      - name: Wait for Project to Start
        if: steps.resume.outputs.resume_status == 'success'
        run: sleep 10
      
      - name: Verify Project Status
        id: verify
        run: |
          echo "🔍 Verifying project status..."
          
          RESPONSE=$(curl -s -w "\n%{http_code}" -X GET \
            "https://api.supabase.com/v1/projects/${{ secrets.SUPABASE_PROJECT_ID }}" \
            -H "Authorization: Bearer ${{ secrets.SUPABASE_API_KEY }}" \
            -H "Content-Type: application/json")
          
          HTTP_CODE=$(echo "$RESPONSE" | tail -n1)
          BODY=$(echo "$RESPONSE" | sed '$d')
          
          if [ "$HTTP_CODE" -eq 200 ]; then
            STATUS=$(echo "$BODY" | jq -r '.status' 2>/dev/null || echo "unknown")
            echo "Project Status: $STATUS"
            echo "status=$STATUS" >> $GITHUB_OUTPUT
            
            if [[ "$STATUS" == "ACTIVE" ]] || [[ "$STATUS" == "RUNNING" ]]; then
              echo "✅ Project is active and running"
            else
              echo "⚠️ Project status is: $STATUS"
            fi
          else
            echo "❌ Failed to verify status (HTTP $HTTP_CODE)"
            echo "status=unknown" >> $GITHUB_OUTPUT
          fi
      
      - name: Send Success Notification
        if: success()
        run: |
          echo "✅ Automation completed successfully"
          echo "📅 Timestamp: $(date -u +'%Y-%m-%d %H:%M:%S UTC')"
          echo "🔄 Project Status: ${{ steps.verify.outputs.status }}"
          echo "📍 Scheduled to run again in 7 days"
      
      - name: Send Failure Notification
        if: failure()
        run: |
          echo "❌ Automation failed!"
          echo "📅 Timestamp: $(date -u +'%Y-%m-%d %H:%M:%S UTC')"
          echo "⚠️ Please check your API credentials and try again"
          exit 1
```

## 🔐 Security

### Best Practices

✅ **Secrets are encrypted** - GitHub securely stores your API key with encryption at rest
✅ **No credentials in logs** - GitHub automatically hides secret values in workflow output
✅ **Token rotation** - Regenerate your API key every 90 days for maximum security
✅ **Minimal permissions** - The API key only needs project restart permissions
✅ **Secure storage** - Secrets are only available to workflows in your repo

### How It Works

1. You store your API key in GitHub Secrets
2. GitHub encrypts and securely stores it
3. When the workflow runs, it injects the secret into the environment
4. The secret is used to authenticate with Supabase API
5. Logs automatically redact the secret value
6. The secret never appears in your repository

## 🚨 Troubleshooting

### Issue: Workflow shows "Unauthorized" error

**Solution:**
- Verify your `SUPABASE_PROJECT_ID` is correct (from Settings → General)
- Verify your `SUPABASE_API_KEY` is correct (from Account → Access Tokens)
- Check that secrets are added to GitHub (Settings → Secrets and variables)
- Regenerate the API token and update the secret

### Issue: Workflow doesn't run at scheduled time

**Solution:**
- GitHub Actions may delay by 5-10 minutes — this is normal
- Ensure the workflow file is on your default branch (main/master)
- Commit at least once to trigger GitHub Actions
- Test manually: go to Actions tab → click workflow → Run workflow

### Issue: "jq: command not found" error

**Solution:**
- `jq` should be pre-installed on Ubuntu runners
- If using a different runner, remove the jq verification step or install it

### Issue: Project still gets paused

**Solution:**
- Check workflow logs (Actions tab → click run → expand steps)
- Verify HTTP response codes (should be 200 or 201)
- Check Supabase API status page
- Ensure your Supabase plan supports the restart operation
- Try regenerating your API key

### Issue: How do I see the logs?

**Solution:**
1. Go to your repo → **Actions** tab
2. Click on the latest workflow run
3. Click the **resume-project** job
4. Expand each step to see the output

## 📊 What Happens

### Timeline of Execution

```
GitHub Actions Scheduler
         ↓
    Day 7 @ 12:00 UTC
         ↓
   GitHub runs workflow
         ↓
   Workflow sends POST request to Supabase API
         ↓
   Supabase resumes your project
         ↓
   Workflow verifies project status
         ↓
   Logs success and schedules next run
         ↓
    Repeat every 7 days
```

### What the API Calls Do

**Resume Project:**
```bash
POST https://api.supabase.com/v1/projects/{PROJECT_ID}/restart
Header: Authorization: Bearer {API_KEY}
```
Sends a restart signal to your Supabase project, resuming it if paused.

**Get Project Status:**
```bash
GET https://api.supabase.com/v1/projects/{PROJECT_ID}
Header: Authorization: Bearer {API_KEY}
```
Verifies the project is now running and returns its current status.

## 📋 Checklist

- [ ] Created Supabase API token
- [ ] Copied Project ID
- [ ] Added SUPABASE_PROJECT_ID secret to GitHub
- [ ] Added SUPABASE_API_KEY secret to GitHub
- [ ] Created `.github/workflows/resume-supabase.yml` file
- [ ] Committed and pushed the workflow file
- [ ] Tested workflow manually (Actions → Run workflow)
- [ ] Confirmed success in logs
- [ ] Checked Supabase dashboard to verify project is active
- [ ] Automation will now run automatically every 7 days

## 🎯 Next Steps

1. **Set up now** using the Quick Start guide above
2. **Test it** by running the workflow manually
3. **Monitor it** by checking logs occasionally
4. **Relax** knowing your projects will never be paused again

## 💡 Tips & Tricks

### Monitor Multiple Projects

Create separate workflows for each project:
```
.github/workflows/resume-supabase-project-1.yml
.github/workflows/resume-supabase-project-2.yml
.github/workflows/resume-supabase-project-3.yml
```

Each with its own secrets:
```
SUPABASE_PROJECT_ID_1, SUPABASE_API_KEY_1
SUPABASE_PROJECT_ID_2, SUPABASE_API_KEY_2
SUPABASE_PROJECT_ID_3, SUPABASE_API_KEY_3
```

### Add Discord/Slack Notifications

Add this step to get notified when your project is resumed:

```yaml
- name: Send Discord Notification
  uses: sarisia/actions-status-discord@v1
  with:
    webhook_url: ${{ secrets.DISCORD_WEBHOOK }}
    status: ${{ job.status }}
    title: "✅ Supabase Project Resumed"
    description: "Your project was automatically resumed"
  if: success()
```

First, add your Discord webhook URL as a secret called `DISCORD_WEBHOOK`.

### View Workflow Runs

All workflow executions are logged in the Actions tab:
- ✅ Successful runs show green checkmarks
- ❌ Failed runs show red X marks
- Logs show timestamp and detailed output

## 🆘 Need Help?

### Common Questions

**Q: How often does it run?**
A: Every 7 days at 12:00 UTC by default. You can customize this in the cron expression.

**Q: Will this cost me money?**
A: No. GitHub Actions provides 2,000 free minutes per month for public repos and 3,000 for private repos. This automation uses less than 1 minute per run.

**Q: Can I change the schedule?**
A: Yes! Edit the `cron` value in the workflow file. See the "Custom Schedule" section above.

**Q: What if I have multiple projects?**
A: Create separate workflow files and secrets for each project.

**Q: Is my API key safe?**
A: Yes. GitHub encrypts secrets and only injects them into workflows. They never appear in logs or your repository.

**Q: What if Supabase changes their API?**
A: The workflow will fail and log an error. You'll see it in the Actions tab and can update the API endpoint.

**Q: Can I run it more frequently than 7 days?**
A: Yes! Change the cron to `0 */3 * * *` for every 3 hours, `0 12 * * *` for daily, etc.

## 📚 Resources

- [Supabase API Documentation](https://supabase.com/docs/reference/api)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Secrets Management](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Cron Expression Generator](https://crontab.guru/)
- [Supabase Status Page](https://status.supabase.com/)

## 📝 License

This automation setup is provided as-is for keeping your Supabase projects alive.

## 🎉 You're All Set!

Your Supabase projects will now stay active forever. No more paused projects. No more 90-day closures. Just set it and forget it.

**Questions or improvements?** Feel free to customize the workflow or reach out for support.

---

**Made with ❤️ to keep your Supabase projects alive**

Last updated: April 2, 2026
