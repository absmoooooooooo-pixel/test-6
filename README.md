# Metal Weight Calculator

Engineering weight calculators for plates, sections, pipes, bars, cones, spheres,
flanges and dished heads.

The project is split in two:

- **Website** — static files: `index.html` at the repository root plus the pages
  and assets under `frontend/`. Hosted on GitHub Pages; no server needed to
  display it.
- **API** — `backend/`: the engineering formulas, material densities, validation
  and rate limiting. Deployed separately as a Node service.

## Project structure

```
Metal-Weight-Calculator/
├── index.html                 # HOME PAGE — served as "/" by GitHub Pages
├── manifest.webmanifest       # PWA manifest (root scope)
├── service-worker.js          # pass-through service worker (root scope)
├── robots.txt, sitemap.xml, ads.txt
├── .nojekyll                  # GitHub Pages: serve files as-is
├── .gitignore
│
├── frontend/                  # the rest of the static website
│   ├── about/index.html
│   ├── calculators/           # 24 calculators: page + own CSS/JS per folder
│   ├── assets/
│   │   ├── css/dashboard.css  # home page styles
│   │   ├── js/dashboard.js    # calculator cards and category filter
│   │   ├── js/calc-api.js     # calls the calculation API
│   │   ├── js/api-config.js   # ← API base URL, the one line to configure
│   │   └── data/dashboard-data.js
│   ├── shared/                # history, feedback, keypad, install popup…
│   ├── icons/, favicon.png, favicon.ico
│
├── backend/                   # private API: never part of the website
│   ├── server.js, app.js, config.js
│   ├── routes/, controllers/, middleware/
│   ├── calculations/          # the formulas (one module per family)
│   ├── core/                  # weight chain and validation
│   ├── materials/             # density tables
│   ├── test/                  # parity test + reference copies of old pages
│   └── .env.example
│
├── tools/                     # maintenance scripts (dashboard/SEO data)
└── DEPLOY.md                  # hosting notes
```

Every link is relative, so the site works at a domain root and in a GitHub
Pages repository sub-path.

## Hosting

- **Website:** GitHub Pages serves the root `index.html` as the home page — see
  [DEPLOY.md](DEPLOY.md).
- **API:** deployed on its own Node host. The browser finds it through
  `frontend/assets/js/api-config.js`:

  ```js
  window.MW_API_BASE = "https://api.example.com";
  ```

  Leave it empty only when one host serves both the pages and the API.

No API key or secret is stored in frontend files. The API allows the site's
origin through `ALLOWED_ORIGINS`; everything else is refused.

## How the split works

1. The page reads and validates the user's inputs exactly as before (messages
   and rules live in the page).
2. The page `POST`s the values with `frontend/assets/js/calc-api.js` to
   `<API base>/api/calculate/<calculator>`.
3. The backend validates the payload again, runs the engineering formula and
   returns the finished numbers.
4. The page renders those numbers with its existing formatting, drawings,
   history entries, print and PDF code.

Material densities come from the API (`/api/materials`), with a built-in
fallback for the common materials so the page still opens if the API is
unreachable.

The engineering formulas — areas, volumes, surface areas, blank diameters, hole
removal, weights and tonnes — exist **only** in `backend/calculations`.

### API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Service check |
| `GET` | `/api/calculators` | Slugs the API serves |
| `GET` | `/api/materials` | Density tables as JSON |
| `GET` | `/api/bootstrap.js` | Densities as a script (tooling and tests) |
| `POST` | `/api/calculate/:calculator` | Runs one calculation |

Example:

```bash
curl -X POST https://api.example.com/api/calculate/plate \
  -H "Content-Type: application/json" \
  -d '{"values":{"width":1000,"thickness":5,"length":2000},"unit":"mm","density":7.85,"quantity":3}'
```

```json
{"ok":true,"calculator":"plate","units":{"dimensions":"mm","density":"g/cm3"},
 "result":{"areaMM2":5000,"volumeMM3":10000000,"volumeM3":0.01,
           "singleWeight":78.5,"totalWeight":235.5,"totalTonnes":0.2355,
           "materialName":"Custom Material"}}
```

Errors are always JSON: `400` invalid input (with the field name), `401` missing
API key when one is configured, `403` origin not allowed, `404` unknown
calculator, `405` wrong method, `413` body too large, `429` rate limited, `500`
unexpected.

`dish-ends` takes an extra `type` (`flat`, `torispherical`, `ellipsoidal`,
`hemispherical`, `template`) and a density in **kg/m³**; every other calculator
uses **g/cm³**.

## Security

- The API process serves `/api/*` only; the website is static and contains no
  formulas.
- Inputs are validated and range-checked on the server, not just in the browser.
- CORS allows listed origins (`ALLOWED_ORIGINS`); anything else gets `403`.
- `/api/calculate` is rate limited per client IP (`RATE_LIMIT_MAX` per
  `RATE_LIMIT_WINDOW_MS`).
- Request bodies are capped (`MAX_BODY_BYTES`) and only JSON is accepted.
- Configuration comes from the environment (`backend/.env`, git-ignored).

## Tests

```bash
cd backend
npm test
```

The parity test runs every calculator twice with identical inputs — once with the
pre-restructure page (kept in `backend/test/fixtures/original` as reference
material only) and once with the migrated page calling the API — and compares
every rendered string, the calculation history payload and the result object.
The five dish head types are compared field by field, including the template
station table. It starts its own temporary API server, so no setup is needed.

Expected result: `calculator/scenario pairs compared: 71` followed by `PASS`.
