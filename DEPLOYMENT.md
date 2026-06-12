# Deployment Information

## Public URL

```
https://aware-respect-production.up.railway.app
```

## Platform

**Railway** (railway.com) — deployed via CLI (`railway up`)

## Source Code

https://github.com/Zilexz/day12_ha-tang-cloud_va_deployment  
Subdirectory: `06-lab-complete/`

## Environment Variables Set

| Variable | Value | Note |
|----------|-------|------|
| `ENVIRONMENT` | `production` | |
| `APP_VERSION` | `1.0.0` | |
| `AGENT_API_KEY` | *(secret)* | Set via railway variables |
| `JWT_SECRET` | *(secret)* | Set via railway variables |
| `DAILY_BUDGET_USD` | `5.0` | |
| `RATE_LIMIT_PER_MINUTE` | `20` | |

## Test Commands

### Health Check
```bash
curl https://aware-respect-production.up.railway.app/health
# Expected: {"status":"ok","version":"1.0.0","environment":"production",...}
```

### Authentication required (no key → 401)
```bash
curl -X POST https://aware-respect-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "hello"}'
# Expected: {"detail":"Missing API key..."}
```

### API Test (with authentication)
```bash
curl -X POST https://aware-respect-production.up.railway.app/ask \
  -H "X-API-Key: day12-secret-key-hieu" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is production deployment?"}'
# Expected: {"question":"...","answer":"...","model":"gpt-4o-mini","timestamp":"..."}
```

### Readiness check
```bash
curl https://aware-respect-production.up.railway.app/ready
# Expected: {"ready":true}
```

## Live Test Results

| Test | Expected | Result |
|------|----------|--------|
| `GET /health` | 200 + status ok | ✅ |
| `GET /ready` | 200 + ready true | ✅ |
| `POST /ask` (no key) | 401 | ✅ |
| `POST /ask` (valid key) | 200 + answer | ✅ |
| Rate limit (req 21+) | 429 | ✅ |
| JSON structured logging | logs in JSON | ✅ |
| Graceful shutdown | SIGTERM handled | ✅ |

## Screenshots

- [Railway deployment dashboard](screenshots/Dashboard.png)
- [Service running on Railway](screenshots/running.PNG)
- [Health endpoint response](screenshots/test.png)
