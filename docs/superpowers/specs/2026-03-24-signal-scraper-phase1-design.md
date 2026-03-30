# Signal Scraper — Phase 1 Design Spec

## Overview

Motor de scraping real + auditorias con IA para AI Agency OS (Base44 SaaS).
Reemplaza el sistema actual donde la IA inventa datos sin acceder a la web.

## Architecture

```
Base44 App (UI + DB)  ──webhook POST──►  Railway Microservice (Python)
                                              │
                                              ├── httpx: fetch real HTML
                                              ├── BeautifulSoup4: extract data
                                              ├── Anthropic API: analyze with Claude
                                              │
Base44 App (UI + DB)  ◄──callback POST──  Railway returns structured results
```

## Microservice: signal-scraper

### Stack
- Python 3.12
- FastAPI (async web framework)
- httpx (async HTTP client for scraping)
- BeautifulSoup4 (HTML parsing)
- anthropic (Claude API for analysis)
- Pydantic (data validation)

### Endpoints

#### `POST /audit`
Receives scraping request from Base44.

Request:
```json
{
  "url": "https://www.example.com",
  "audit_id": "69beeafa3f346f5c8d2fe3ce",
  "callback_url": "https://app.base44.com/webhook/..."
}
```

Response (immediate):
```json
{
  "status": "accepted",
  "job_id": "uuid"
}
```

Then processes async and POSTs results to callback_url.

#### `GET /health`
Returns `{"status": "ok"}` for monitoring.

### Security
- `X-API-Key` header required on all requests
- Key stored as Railway environment variable
- Rate limit: 10 requests/minute
- Timeout: 30s per scrape

### File Structure
```
signal-scraper/
├── main.py            # FastAPI app, /audit and /health endpoints
├── scraper.py         # Web scraping logic (httpx + BS4)
├── analyzer.py        # Claude API call for business analysis
├── models.py          # Pydantic schemas
├── config.py          # Settings and env vars
├── requirements.txt   # Dependencies
├── Procfile           # Railway entry point
└── README.md          # Setup instructions
```

## Scraping Pipeline

### Step 1: Fetch
- GET the URL with httpx (follow redirects, 30s timeout)
- Custom User-Agent to avoid blocks
- Capture response time for speed metric

### Step 2: Extract (7 data points)
1. **Main texts**: headings, paragraphs, service descriptions, about content
2. **Contact data**: phone, email, address, social media links
3. **Tech stack**: detect CMS (WordPress, Shopify, Wix, etc.) from meta tags, scripts, headers
4. **Basic SEO**: title, meta description, H1 hierarchy, SSL, sitemap.xml presence
5. **Social media**: Instagram, Facebook, LinkedIn, Google Business links
6. **Image analysis**: count, alt text coverage percentage
7. **Load speed**: response time in ms

### Step 3: Analyze with Claude
- Send all extracted data as structured context
- Prompt generates: business summary, ideal client, services, problems, opportunities, recommended automations, suggested agents, web improvements, estimated ROI, suggested pricing, opportunity score (0-100)
- Output matches existing Auditorias entity schema

### Step 4: Callback
- POST structured results to Base44 callback_url
- Base44 updates ScrapingResults + Auditorias entities

## New Base44 Entities

### ScrapingJobs
| Field | Type | Description |
|---|---|---|
| audit_id | string | Links to Auditorias |
| url | string | URL to scrape |
| estado | enum: pendiente/scrapeando/analizando/completado/error | Job status |
| raw_html_preview | string | First 5000 chars of HTML |
| response_time_ms | number | Page load time |
| error_message | string | Error details if failed |
| created_at | datetime | |

### ScrapingResults
| Field | Type | Description |
|---|---|---|
| scraping_job_id | string | Links to ScrapingJobs |
| empresa_id | string | Links to Empresas |
| textos_principales | string | Extracted main content |
| datos_contacto | string | JSON: phone, email, address, socials |
| stack_tecnologico | string | CMS, frameworks detected |
| seo_basico | string | JSON: title, meta, H1s, SSL, sitemap |
| redes_sociales | string | JSON: social media links found |
| imagenes_analisis | string | Count, alt text coverage |
| velocidad_carga | number | Response time in ms |
| score_web | number | 0-100 overall web quality score |
| created_at | datetime | |

## Modified Audit Flow

### Current (broken)
URL → AI generates fictional analysis → Auditorias record

### New
1. User enters URL in Base44
2. Base44 creates ScrapingJob (estado: pendiente)
3. Base44 fires webhook to Railway: POST /audit
4. Railway scrapes real website
5. Railway extracts 7 data points
6. Railway sends data to Claude for analysis
7. Railway POSTs results to Base44 callback
8. Base44 updates ScrapingResults with extracted data
9. Base44 updates Auditorias with AI analysis
10. User sees complete audit with real, verifiable data

## Environment Variables (Railway)

- `API_KEY`: shared secret with Base44
- `ANTHROPIC_API_KEY`: for Claude analysis
- `PORT`: set by Railway automatically

## Cost Estimate

- Railway: free tier (500h/month)
- Anthropic: ~$0.01-0.03 per audit (Claude Haiku for analysis)
- Total: essentially free for first 100+ audits/month
