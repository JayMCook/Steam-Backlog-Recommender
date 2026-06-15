# Steam Backlog Recommender

An AI-powered tool that looks at your Steam backlog — the games you own but haven't really played — and recommends what to play next based on your current mood.

**[Live Demo](https://steam-backlog-recommender.vercel.app/)** · **[API Docs](https://steam-backlog-recommender.onrender.com/docs)**

---

## How it works

1. **Fetch** — Pulls your owned games library from the Steam Web API using your Steam username or SteamID64.
2. **Filter** — Identifies your "backlog": games with under 2 hours of playtime.
3. **Enrich** — Fetches genre and description metadata for each backlog game from the Steam Store API, with local caching to avoid redundant requests.
4. **Recommend** — Sends your filtered backlog and mood description to Claude, which returns a top pick plus two runner-ups with reasoning, as structured JSON.

```
React frontend → FastAPI backend → Steam Web API  (library fetch)
                                 → Steam Store API (genres/descriptions, cached)
                                 → Anthropic API  (recommendation reasoning)
```

---

## Try it

**[steam-backlog-recommender.vercel.app](https://steam-backlog-recommender.vercel.app/)**

1. Set your Steam profile's **Game details** to **Public** (Steam → Edit Profile → Privacy Settings)
2. Enter your Steam username or SteamID64
3. Describe what you're in the mood to play
4. Get a recommendation pulled from games already in your library

**Finding your identifier:** If your Steam profile URL is `steamcommunity.com/id/yourname`, enter `yourname`. If it's `steamcommunity.com/profiles/76561198012345678`, use the number at the end.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React, Vite |
| Backend | Python, FastAPI |
| LLM | Anthropic Claude (claude-sonnet-4-6) |
| Data | Steam Web API, Steam Store API |
| Deployment | Vercel (frontend), Render (backend) |

---

## API

The backend is publicly accessible. Interactive docs at **[steam-backlog-recommender.onrender.com/docs](https://steam-backlog-recommender.onrender.com/docs)**.

```
GET  /backlog/{steam_id}   Returns the user's enriched backlog
POST /recommend            { "steam_id": "...", "mood": "..." } → recommendation JSON
```

**Note:** Render's free tier spins down after inactivity — the first request after idle may take 30–60 seconds to wake up.

---

## Design notes

**Why filter by playtime instead of a manual backlog list?** Most people don't maintain a curated backlog — their library *is* their backlog. Under 2 hours played captures "owned but never really played" without requiring any extra input.

**Why cache Store API metadata?** The Steam Store API is unofficial and rate-limited, and game metadata never changes. Caching by app ID means the first lookup for any game is the only slow one — ever — and repeat requests are instant.

**Why structured JSON from the LLM?** Asking Claude to return `{"recommendation": {...}, "runner_ups": [...]}` means the response renders directly into UI components without parsing. App IDs in the response are used to construct Steam's header image CDN URLs client-side, requiring no extra API calls for artwork.

**Why resolve vanity URLs server-side?** Steam's `GetOwnedGames` endpoint only accepts SteamID64s. The backend calls `ResolveVanityURL` first when a non-numeric identifier is provided, so users can enter their username instead of hunting down a 17-digit ID.

---

## Known limitations & future improvements

- **File-based cache** — game metadata is cached to a local JSON file. Doesn't persist across Render redeploys. Postgres or Redis would be the natural next step for production persistence.
- **Cold start latency** — Render's free tier spins down after inactivity. First request after idle takes 30–60 seconds.
- **Synchronous enrichment** — metadata is fetched sequentially on a cold cache, so first-time requests for large libraries can take up to a minute. Async fetching with a bounded worker pool would speed this up significantly.
- **Steam privacy requirement** — requires "Game details" set to Public. This is a Steam API constraint.
- **Vanity URL dependency** — username resolution only works if the user has set a custom Steam profile URL. Others must use their SteamID64.

---

## Project structure

```
backend/
  main.py            # FastAPI app and routes
  fetch_backlog.py   # Steam Web API integration, ID resolution, backlog filtering
  enrich.py          # Steam Store API integration with local caching
  recommend.py       # LLM prompt construction and recommendation logic
  requirements.txt
  cache/             # Local metadata cache (gitignored)

frontend/
  src/
    App.jsx          # Main UI component
    App.css          # Styling
  index.html         # Browser title and favicon
  public/            # Static assets
```