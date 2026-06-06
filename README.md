# Canvas MCP for Poke

A read-only [MCP](https://modelcontextprotocol.io/) server that pulls your Canvas LMS data — deadlines, assignments, announcements, grades, and calendar events — into [Poke](https://poke.com/) so you can ask about your coursework in plain language over text.

> Stop screenshotting your Canvas calendar. Just ask Poke *"What do I need to do today?"*

## What it does

Poke is a personal AI assistant you talk to over iMessage. It is great at email and to-do lists, but it can't see Canvas. This MCP server bridges that gap: it aggregates and cleans up Canvas API responses, then exposes them to Poke as a handful of focused tools.

Ask Poke things like:

- *"What do I need to do today on Canvas?"*
- *"What are my upcoming deadlines?"*
- *"Did any of my assignments get graded recently?"*
- *"What announcements were posted this week?"*
- *"Help me plan my week."*

**Access model:** read-only, using a personal Canvas access token scoped to your own student account. No admin or developer keys required.

## How it works

The server normalizes several Canvas endpoints into clean, minimal payloads (it strips the noisy metadata Canvas returns and converts all timestamps to UTC so deadlines filter correctly):

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/dashboard/dashboard_cards` | Active courses, in your dashboard order |
| `GET /api/v1/courses` | All enrolled courses |
| `GET /api/v1/courses/:id/assignments?include[]=submission` | Assignments with submission status |
| `GET /api/v1/courses/:id/discussion_topics?only_announcements=true` | Course announcements |
| `GET /api/v1/planner/items` | Planner feed: events, quizzes, and graded items |

Requests to the server are authenticated with a shared API key (`x-api-key` header or `Authorization: Bearer`) so only Poke can reach your data.

## Tools

| Tool | What it answers |
|------|-----------------|
| `get_today_summary` | **Best default.** Curated daily check-in: deadlines, events, recent announcements, new grades, and overdue work. |
| `get_upcoming_assignments` | Deadlines across all courses, most urgent first (optionally including overdue). |
| `get_course_assignments` | Upcoming/overdue assignments for one specific course. |
| `get_recent_announcements` | Recent announcements across courses, with optional full message body. |
| `get_recently_graded` | Assignments and quizzes graded in the last *N* days. |
| `get_week_ahead` | Everything coming up in the next week: assignments, quizzes, and calendar events. |
| `get_dashboard_cards` | Lightweight list of active dashboard courses (id + name). |
| `list_courses_raw` | Raw Canvas course list, useful for troubleshooting. |

> **Tip — filtering old courses.** Canvas often keeps past-semester courses enrolled. Most tools accept a `term_prefix` (e.g. `26FS` for Fall 2026, `26SS` for Spring 2026) to show only the current term.

## Setup

### 1. Get a Canvas access token

1. Log in to Canvas.
2. Go to **Account → Settings**.
3. Scroll to **Approved Integrations** and click **New Access Token**.
4. Copy the token immediately — Canvas won't show it again.

### 2. Run locally

```bash
git clone https://github.com/rikhinkavuru/canvas-poke-mcp
cd canvas-poke-mcp

python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# Edit .env with your Canvas URL, token, and a strong POKE_API_KEY

python src/server.py
```

The server runs at `http://localhost:8000/mcp`. To test it, run the MCP inspector in a second terminal and connect using the **Streamable HTTP** transport:

```bash
npx @modelcontextprotocol/inspector
```

### Environment variables

| Variable | Description |
|----------|-------------|
| `CANVAS_BASE_URL` | Your school's Canvas URL, e.g. `https://yourschool.instructure.com` |
| `CANVAS_ACCESS_TOKEN` | The personal access token from step 1 |
| `POKE_API_KEY` | A strong secret (32+ random chars) that Poke uses to authenticate to this server |

Generate a key with:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

## Deploy to Render

The included [`render.yaml`](render.yaml) configures a free web service.

1. Fork or push this repo to your GitHub account.
2. Create a new **Web Service** on [Render](https://render.com/) and connect the repo.
3. Add the three environment variables (`CANVAS_BASE_URL`, `CANVAS_ACCESS_TOKEN`, `POKE_API_KEY`) and deploy.

## Connect to Poke

1. Open [poke.com/settings/connections](https://poke.com/settings/connections).
2. Add your deployed server URL as an MCP connection.
3. Add your `POKE_API_KEY` so Poke can authenticate (sent as `x-api-key` or `Authorization: Bearer`).

Then just ask Poke *"What do I need to do today on Canvas?"* — or call a tool directly: *"Use the Canvas MCP connection's `get_today_summary` tool."*
