# Render deployment

FloCafe can run as one Render Node.js Web Service: Express serves both the
static frontend and `/api` from the same public URL. This configuration does
not replace Railway or Vercel support.

## Production requirements

Use a **paid Render Web Service** with a Persistent Disk. Render Free Web
Services have ephemeral storage and cannot attach persistent disks, so they
are not safe for this SQLite POS database.

Create one service and one disk per client. Never share a disk, SQLite file,
JWT secret, owner account, or configuration between clients.

## Dashboard settings

Connect the client repository and select its `main` branch. The repository's
[`render.yaml`](../render.yaml) supplies the following settings:

| Setting | Value |
| --- | --- |
| Service type | Web Service |
| Runtime | Node.js |
| Build command | `npm ci --include=dev --ignore-scripts && npm rebuild better-sqlite3 --build-from-source && npm run build:frontend && npm run build` |
| Start command | `node dev-server.js` |
| Health check | `/api/health` |
| Persistent disk mount path | `/data` |

Set the disk up during service creation (Advanced) or afterwards on the
service's **Disks** page. Choose a paid compute plan before attaching it.

The root package has an Electron desktop `postinstall` hook. Render deliberately
installs dependencies with lifecycle hooks disabled, then rebuilds only
`better-sqlite3` for Render's Node.js runtime. This prevents Electron download
and packaging hooks from running on the web service while retaining the native
SQLite binding needed by the API. Do not replace this with a plain
`npm ci --ignore-scripts` without the explicit `better-sqlite3` rebuild.

Set these environment variables in the service:

| Variable | Value |
| --- | --- |
| `NODE_ENV` | `production` |
| `FLO_DEV_USER_DATA` | `/data` |
| `JWT_SECRET` | unique, randomly generated secret for this client |
| `FLO_ALLOWED_ORIGINS` | the public Render URL, for example `https://pos-ioy4.onrender.com` |

Do not set `PORT`; Render supplies it. Do not set `NEXT_PUBLIC_API_URL` for a
same-origin Render deployment. The frontend will call `/api` on its own Render
origin.

At runtime, the app creates and uses `/data/flo.db` and `/data/backups`.
Production startup fails rather than silently using the project directory when
no persistent database path is configured. The disk is only mounted at runtime,
so the build must not create or copy a production database.

## First deployment and verification

1. Create a new Render service and a new empty disk for the client.
2. Add the variables above, with the exact generated Render domain in
   `FLO_ALLOWED_ORIGINS`.
3. Deploy from `main`; do not copy another client's database.
4. Open `https://YOUR-SERVICE.onrender.com/api/health`; it must report database
   status `ok`.
5. Open the root URL and complete the normal first-run owner setup.

The Cloud Services setup page finishes through `POST /api/auth/setup/initialize`.
Its cloud integration remains optional/configurable; it is not an alternate
authentication or hosting endpoint.

Render publicly routes only the service's `PORT`. FloCafe also starts its
existing KDS and Server App listeners for local/LAN compatibility; do not add
public Render domains for ports 3002 or 3003.

## Updates and backups

Pushing to the configured branch triggers Render's normal deployment. The
database and local backups remain on that client disk across deploys. Check the
service logs and `/api/health` after each update. Use FloCafe's database backup
features before restoring data; a disk snapshot restore is not a substitute for
an application-consistent SQLite backup.
