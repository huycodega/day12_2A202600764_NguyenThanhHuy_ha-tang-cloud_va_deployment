# Deployment Information

## Public URL

```
https://aware-respect-production.up.railway.app
```

## Platform

**Railway** (railway.com) — $5 free credit, deploy từ CLI

## Source Code

https://github.com/Zilexz/day12_ha-tang-cloud_va_deployment  
Subdirectory: `06-lab-complete/`

## How to Deploy (Render)

1. Vào [render.com](https://render.com) → Sign in với GitHub
2. New → **Blueprint**
3. Connect repo: `Zilexz/day12_ha-tang-cloud_va_deployment`
4. Root directory: `06-lab-complete`
5. Render tự đọc `render.yaml`
6. Set secrets trong dashboard:
   - `AGENT_API_KEY` = any strong random string
   - `JWT_SECRET` = any strong random string
7. Click Deploy → đợi ~3 phút
8. Copy URL từ dashboard

## Environment Variables Set

| Variable | Value | Note |
|----------|-------|------|
| `ENVIRONMENT` | `production` | từ render.yaml |
| `APP_VERSION` | `1.0.0` | từ render.yaml |
| `AGENT_API_KEY` | *(generated)* | Set trong Render dashboard |
| `JWT_SECRET` | *(generated)* | Set trong Render dashboard |
| `DAILY_BUDGET_USD` | `5.0` | từ render.yaml |
| `RATE_LIMIT_PER_MINUTE` | `20` | từ render.yaml |

## Test Commands

### Health Check
```bash
curl https://ai-agent-production-aware-respect-production.up.railway.app/.onrender.com/health
# Expected: {"status":"ok","version":"1.0.0","environment":"production",...}
```

### Authentication required (no key → 401)
```bash
curl -X POST https://ai-agent-production-aware-respect-production.up.railway.app/.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "hello"}'
# Expected: {"detail":"Invalid or missing API key..."}
```

### API Test (with authentication)
```bash
curl -X POST https://ai-agent-production-aware-respect-production.up.railway.app/.onrender.com/ask \
  -H "X-API-Key: YOUR_AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is production deployment?"}'
# Expected: {"question":"...","answer":"...","model":"gpt-4o-mini","timestamp":"..."}
```

### Rate limiting test
```bash
for i in {1..22}; do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" \
    -X POST https://ai-agent-production-aware-respect-production.up.railway.app/.onrender.com/ask \
    -H "X-API-Key: YOUR_KEY" \
    -H "Content-Type: application/json" \
    -d '{"question":"test"}'
done
# Expected: 1-20 → 200, 21-22 → 429
```

### Readiness check
```bash
curl https://ai-agent-production-aware-respect-production.up.railway.app/.onrender.com/ready
# Expected: {"ready":true}
```

## Local Test Results (before cloud deploy)

All endpoints verified locally on port 8030:

| Test | Expected | Result |
|------|----------|--------|
| `GET /health` | 200 + status ok | ✅ |
| `GET /ready` | 200 + ready true | ✅ |
| `POST /ask` (no key) | 401 | ✅ |
| `POST /ask` (wrong key) | 401 | ✅ |
| `POST /ask` (valid key) | 200 + answer | ✅ |
| Rate limit (req 21+) | 429 | ✅ (limit=20/min) |
| JSON structured logging | logs in JSON | ✅ |
| Graceful shutdown | SIGTERM handled | ✅ |

## Screenshots

*(Add screenshots after Render deployment)*

- `screenshots/render-dashboard.png` — Render deployment dashboard
- `screenshots/health-response.png` — `/health` endpoint response
- `screenshots/ask-response.png` — `/ask` endpoint with API key
