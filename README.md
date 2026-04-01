# AI GitHub PR Review Agent

An Express-based webhook service that listens to GitHub pull request events, asks OpenAI to review the diff, posts review comments back to GitHub, and then either approves or requests changes. It can also merge a clean PR automatically when enabled.

## Features

- Listens to GitHub `pull_request` webhook events
- Verifies webhook signatures with `GITHUB_WEBHOOK_SECRET`
- Fetches pull request metadata and full diff with the GitHub API
- Sends the diff to OpenAI for structured review
- Posts inline review comments when issues are found
- Approves a PR when no critical or medium issues are found
- Requests changes when critical or medium issues are found
- Optionally merges the PR automatically after approval
- Exposes a health check route for local verification

## How It Works

1. GitHub sends a `POST` request to `/webhook`
2. The server captures the raw body and verifies the `x-hub-signature-256` header
3. The webhook payload is parsed and filtered to supported PR actions
4. The review agent fetches PR details and the full diff
5. OpenAI analyzes the diff and returns structured findings
6. The agent posts comments and decides the final review state
7. If auto-merge is enabled and the review is clean, the PR merge is attempted

## Supported Pull Request Events

The webhook currently reacts only to these pull request actions:

- `opened`
- `reopened`
- `synchronize`

That means the agent runs when:

- a new PR is opened
- a closed PR is reopened
- new commits are pushed to an open PR

## Project Structure

- [server.js](/d:/Projects%20AI%20AGENT/git_agent/server.js): Express webhook server and signature verification
- [agent.js](/d:/Projects%20AI%20AGENT/git_agent/agent.js): OpenAI-driven PR review loop
- [github.js](/d:/Projects%20AI%20AGENT/git_agent/github.js): GitHub API helpers for PR reads, comments, approvals, change requests, and merge
- [analyzer.js](/d:/Projects%20AI%20AGENT/git_agent/analyzer.js): OpenAI prompt and JSON parsing for code analysis
- [WEBHOOK_SETUP.md](/d:/Projects%20AI%20AGENT/git_agent/WEBHOOK_SETUP.md): Step-by-step webhook setup guide
- [package.json](/d:/Projects%20AI%20AGENT/git_agent/package.json): Scripts and dependencies

## Requirements

- Node.js 18 or newer recommended
- npm
- A GitHub repository you can configure webhooks for
- A GitHub fine-grained personal access token
- An OpenAI API key
- A public tunnel for local development such as `cloudflared`

## Installation

Clone the repository and install dependencies:

```powershell
npm install
```

## Environment Variables

Create a `.env` file in the project root with the following values:

```env
OPENAI_API_KEY=your_openai_api_key
GITHUB_TOKEN=your_github_token
GITHUB_WEBHOOK_SECRET=your_webhook_secret
PORT=3000
AUTO_MERGE_ON_APPROVAL=false
GITHUB_MERGE_METHOD=squash
```

### Variable Reference

- `OPENAI_API_KEY`: Used by the review agent to call OpenAI
- `GITHUB_TOKEN`: Used to read PR data and post review actions/comments
- `GITHUB_WEBHOOK_SECRET`: Used to verify GitHub webhook signatures
- `PORT`: Local server port, defaults to `3000`
- `AUTO_MERGE_ON_APPROVAL`: When `true`, merge is attempted after a clean approval
- `GITHUB_MERGE_METHOD`: Merge strategy, must be one of `merge`, `squash`, or `rebase`

## Generating `GITHUB_WEBHOOK_SECRET`

You generate this yourself. GitHub does not provide it.

PowerShell:

```powershell
$rng = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($rng)
($rng | ForEach-Object { $_.ToString("x2") }) -join ""
```

Node.js:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Use the exact same secret value in:

- `.env` as `GITHUB_WEBHOOK_SECRET`
- GitHub webhook settings as `Secret`

## Generating `GITHUB_TOKEN`

Use a fine-grained personal access token in GitHub.

Path:

1. GitHub profile picture
2. `Settings`
3. `Developer settings`
4. `Personal access tokens`
5. `Fine-grained tokens`
6. `Generate new token`

Recommended settings:

- Token name: `pr-review-agent`
- Repository access: `Only select repositories`
- Select the repository this bot should review

Repository permissions:

- `Pull requests`: `Read and write`
- `Issues`: `Read and write`
- `Contents`: `Read and write` if you want auto-merge to work

Why these are needed:

- Read PR metadata and changed files
- Read the diff
- Post inline review comments
- Approve PRs
- Request changes
- Post fallback PR comments via the Issues API
- Merge the PR when auto-merge is enabled

## Running the Server

Start the webhook server:

```powershell
npm start
```

Development mode:

```powershell
npm run dev
```

Manual PR review from the CLI:

```powershell
npm run review -- owner repo 123
```

Equivalent direct command:

```powershell
node agent.js owner repo 123
```

## Health Check

Open:

```text
http://localhost:3000/health
```

Expected response:

```json
{
  "status": "ok",
  "agent": "PR Review Agent (GPT-4o)",
  "time": "..."
}
```

## Important Note About `/webhook`

Do not open `/webhook` in a browser and expect it to work.

Why:

- Browsers send `GET /webhook`
- This app only supports `POST /webhook`

So this is expected:

```text
Cannot GET /webhook
```

## Local Webhook Setup With Cloudflare Tunnel

### 1. Start the app

```powershell
npm start
```

### 2. Start the tunnel

If `cloudflared` is on your `PATH`:

```powershell
cloudflared tunnel --url http://127.0.0.1:3000
```

If it is not on your `PATH`, run the full executable path:

```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel --url http://127.0.0.1:3000
```

Use `127.0.0.1` instead of `localhost` to avoid IPv6 `[::1]` connection issues.

You will get a public URL like:

```text
https://example-name.trycloudflare.com
```

### 3. Configure the GitHub webhook

In the target GitHub repository:

1. Open `Settings`
2. Open `Webhooks`
3. Click `Add webhook`

Use:

- Payload URL: `https://example-name.trycloudflare.com/webhook`
- Content type: `application/json`
- Secret: same as `GITHUB_WEBHOOK_SECRET`
- Events: choose `Pull requests`
- Active: enabled

Important:

- Always include `/webhook`
- If you restart `cloudflared`, the public URL usually changes
- If the tunnel URL changes, update the GitHub webhook Payload URL

## Review Decision Logic

The OpenAI analysis returns findings grouped by severity:

- `critical`
- `medium`
- `low`

Current behavior:

- If critical or medium issues are found, the agent requests changes
- If only low or no issues are found, the agent approves the PR
- If `AUTO_MERGE_ON_APPROVAL=true`, the agent also attempts to merge after approval

## Auto-Merge

To enable auto-merge:

```env
AUTO_MERGE_ON_APPROVAL=true
GITHUB_MERGE_METHOD=squash
```

Allowed merge methods:

- `merge`
- `squash`
- `rebase`

### Auto-merge Preconditions

Auto-merge can still fail even with valid code if:

- the PR is a draft
- the PR is already closed
- branch protection rules block merge
- required status checks are still pending
- required approvals are missing
- the token lacks `Contents: Read and write`

When merge fails after approval, the PR remains approved, but the merge is not completed.

## Scripts

From [package.json](/d:/Projects%20AI%20AGENT/git_agent/package.json):

- `npm start`: Runs the webhook server
- `npm run dev`: Runs the webhook server with `nodemon`
- `npm run review -- owner repo pr_number`: Runs a manual PR review from the command line

## Example `.env`

```env
OPENAI_API_KEY=sk-...
GITHUB_TOKEN=github_pat_...
GITHUB_WEBHOOK_SECRET=your_random_secret_here
PORT=3000
AUTO_MERGE_ON_APPROVAL=false
GITHUB_MERGE_METHOD=squash
```

## Troubleshooting

### `cloudflared : The term 'cloudflared' is not recognized`

Install it:

```powershell
winget install --id Cloudflare.cloudflared
```

Find it:

```powershell
where.exe cloudflared
```

If needed, run the full path:

```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel --url http://127.0.0.1:3000
```

### `Unable to reach the origin service`

Cause:

- the app is not running on port `3000`
- or the tunnel is pointed at the wrong local address

Fix:

1. Run:

```powershell
npm start
```

2. Check:

```text
http://localhost:3000/health
```

3. Restart the tunnel with:

```powershell
cloudflared tunnel --url http://127.0.0.1:3000
```

### `Invalid webhook signature`

Cause:

- secret mismatch between `.env` and GitHub webhook settings
- server not restarted after editing `.env`
- wrong content type in GitHub webhook settings

Fix:

- confirm `GITHUB_WEBHOOK_SECRET` exactly matches the GitHub webhook `Secret`
- set webhook content type to `application/json`
- restart `npm start`
- redeliver the event from GitHub `Recent Deliveries`

### `Cannot GET /webhook`

Cause:

- opening the webhook route in a browser

Fix:

- this is expected
- use `/health` for browser testing

### `No connection could be made because the target machine actively refused it`

Cause:

- nothing is listening on port `3000`

Fix:

```powershell
netstat -ano | findstr :3000
```

If nothing appears, restart the app:

```powershell
npm start
```

### GitHub Webhook Delivers But No Review Appears

Check:

- the event is `pull_request`
- the action is `opened`, `reopened`, or `synchronize`
- the `npm start` terminal is still running
- `OPENAI_API_KEY` is valid
- `GITHUB_TOKEN` is valid

## Observability

Watch the terminal running `npm start`.

That terminal will show:

- webhook receipt
- signature failures
- agent iterations
- tool calls
- review success/failure

GitHub also shows webhook delivery status under:

`Repository Settings -> Webhooks -> Recent Deliveries`

Use delivery status like this:

- `200` or `202`: webhook reached the app
- `400`: payload parse issue
- `401`: invalid signature
- `404`: wrong webhook path
- timeout / refused: server or tunnel is unavailable

## Security Notes

- Do not commit `.env`
- Rotate any exposed OpenAI or GitHub tokens immediately
- Rotate the webhook secret if it has been shared
- Use least-privilege token permissions
- Do not leave auto-merge enabled unless you are comfortable with the risk

## Recommended First-End-to-End Test

1. Create `.env`
2. Run `npm install`
3. Run `npm start`
4. Open `http://localhost:3000/health`
5. Start `cloudflared`
6. Configure GitHub webhook with the current tunnel URL
7. Open a new PR or push a commit to an existing PR
8. Watch the app terminal
9. Check GitHub `Recent Deliveries` if anything fails

## Related Documentation

- [WEBHOOK_SETUP.md](/d:/Projects%20AI%20AGENT/git_agent/WEBHOOK_SETUP.md) for the setup walkthrough
- [server.js](/d:/Projects%20AI%20AGENT/git_agent/server.js) for webhook handling
- [agent.js](/d:/Projects%20AI%20AGENT/git_agent/agent.js) for review orchestration
- [github.js](/d:/Projects%20AI%20AGENT/git_agent/github.js) for GitHub actions
- [analyzer.js](/d:/Projects%20AI%20AGENT/git_agent/analyzer.js) for OpenAI analysis
