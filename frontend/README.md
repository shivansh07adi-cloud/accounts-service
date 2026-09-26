# Accounts Service — frontend

A single self-contained `index.html` (no build step, no framework) that talks directly to the live [Accounts Service API](../README.md) over `fetch()`. It's a small operations-console style UI: a form to create a record, and a table to list, edit, and delete records — all writing to the real Postgres database behind the API.

## Run it locally

No install needed — just open the file, or serve it:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

The **API base URL** field at the top defaults to the deployed Render URL. Change it to `http://localhost:8080`-style addresses (whatever port your local Accounts Service is running on) to point it at a local backend instead.

## Deploy to Vercel

1. Push this repo to GitHub (if you haven't already).
2. Go to [vercel.com/new](https://vercel.com/new), import the repo.
3. Set **Root Directory** to `frontend` (this folder) in the import settings.
4. Framework preset: **Other** (it's a static file, no build command needed).
5. Deploy. Vercel gives you a `https://your-project.vercel.app` URL serving this page.

No environment variables are needed — the API base URL is set in the page itself (editable by anyone who opens it, since it's just a plain input field).
