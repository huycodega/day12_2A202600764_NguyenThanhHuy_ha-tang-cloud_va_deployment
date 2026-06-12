# Day 12 Lab - Mission Answers

**Student Name:** Hieu Nguyen
**Student ID:** 2A202600680
**Date:** 2026-06-12

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`

1. **API key hardcode** — `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` — push lên GitHub là lộ key ngay
2. **Database password hardcode** — `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"` — lộ credentials DB
3. **Log ra secret** — `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` — secret xuất hiện trong log file
4. **`print()` thay vì logging** — không có level/filter, không có timestamp, gây crash khi encoding lỗi (Vietnamese text trên Windows cp1252)
5. **Không có health check** — thiếu `/health` endpoint, platform không biết restart khi crash
6. **Port cứng + sai host** — `host="localhost"` (không nhận kết nối từ ngoài container), `port=8000` cứng (Railway/Render inject PORT qua env), `reload=True` luôn bật trong production tốn CPU

### Exercise 1.2: Observation khi chạy basic version

App chạy và trả về response, nhưng trong terminal log thấy rõ:
```
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
```
Secret bị lộ hoàn toàn trong log — đây là lý do không được dùng `print()` để log trong production.

### Exercise 1.3: Comparison table

| Feature | Develop (basic) | Production (advanced) | Tại sao quan trọng? |
|---------|-----------------|----------------------|---------------------|
| Config | Hardcode trong code | `.env` + `Settings` dataclass từ env vars | Không lộ secret khi push Git, dễ thay đổi giữa dev/staging/prod |
| Health check | Không có | `GET /health` (liveness) + `GET /ready` (readiness) | Platform tự restart khi crash; load balancer biết không route traffic vào instance chưa ready |
| Logging | `print()` thô, log secret | JSON structured logging, không log secret | Dễ parse trong Datadog/Loki, có log level, có timestamp |
| Shutdown | Đột ngột (process bị kill) | `lifespan` context + SIGTERM handler | Không mất request đang xử lý, đóng connections sạch |
| Host binding | `localhost` (chỉ local) | `0.0.0.0` | Container phải bind 0.0.0.0 để nhận traffic từ bên ngoài |
| Port | Cứng 8000 | `int(os.getenv("PORT", "8000"))` | Railway/Render inject PORT tự động, nếu cứng sẽ không nhận được traffic |
| Debug reload | `reload=True` luôn | `reload=settings.debug` | Reload trong production tốn CPU, gây lag |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions

1. **Base image:** `python:3.11` — full Python distribution (~1 GB), bao gồm tất cả dev tools
2. **Working directory:** `/app`
3. **Tại sao COPY requirements.txt trước?** — Docker build theo layer cache. Nếu `requirements.txt` không thay đổi, bước `pip install` sẽ dùng lại cache từ lần build trước → build nhanh hơn đáng kể. Nếu copy code trước, mỗi lần code thay đổi sẽ phải pip install lại từ đầu.
4. **CMD vs ENTRYPOINT:**
   - `CMD` có thể bị override khi `docker run <image> <other-command>`
   - `ENTRYPOINT` không override được (luôn chạy), dùng khi muốn container hoạt động như một executable cố định
   - Kết hợp: `ENTRYPOINT ["python"]` + `CMD ["app.py"]` → python luôn chạy, nhưng script có thể override

### Exercise 2.3: Image size comparison

| Image | Disk Usage | Content Size | Ghi chú |
|-------|-----------|--------------|---------|
| `my-agent:develop` | 1.66 GB | **424 MB** | Single-stage, `python:3.11` full |
| `my-agent:production` | 236 MB | **56.6 MB** | Multi-stage, `python:3.11-slim` |

**Nhỏ hơn ~7.5 lần** nhờ:
- `python:3.11-slim` thay vì `python:3.11` full
- Stage 2 (runtime) không chứa gcc, build tools
- Non-root user (security best practice)

### Exercise 2.4: Docker Compose architecture

```
Client (browser/curl)
        │
        ▼ port 80/443
  ┌─────────────┐
  │   Nginx      │  ← Reverse proxy, rate limiting 10r/s, security headers
  └──────┬───────┘
         │ internal network
         ▼
  ┌─────────────┐
  │   agent      │  ← FastAPI AI agent (có thể scale)
  └──────┬───────┘
         │
    ┌────┴────┐
    ▼         ▼
 redis     qdrant
(cache)  (vector DB)
```

**Services:** agent + Redis + Qdrant + Nginx  
**Communication:** Tất cả qua internal Docker network, chỉ Nginx expose ra ngoài  
**Volumes:** `redis_data`, `qdrant_data` — data persistent khi restart

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway / Render deployment

**Platform:** Render  
**URL:** https://day12-ha-tang-cloud-va-deployment.onrender.com *(sau khi deploy)*

**Steps thực hiện:**
1. Push code lên GitHub (đã có fork tại `Zilexz/day12_ha-tang-cloud_va_deployment`)
2. Vào [render.com](https://render.com) → New → Blueprint
3. Connect GitHub repo → Render đọc `06-lab-complete/render.yaml`
4. Set secrets: `AGENT_API_KEY`, `JWT_SECRET`
5. Deploy → nhận public URL

**Test commands:**
```bash
curl https://<your-app>.onrender.com/health
curl -X POST https://<your-app>.onrender.com/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```

### Exercise 3.2: So sánh railway.toml và render.yaml

| | railway.toml | render.yaml |
|-|-------------|-------------|
| Builder | Dockerfile | Docker |
| Start cmd | uvicorn với `$PORT` | Tự động từ Dockerfile CMD |
| Health check | `/health` path | `/health` path |
| Env vars | Set qua `railway variables set` | Định nghĩa trong YAML, secrets set qua dashboard |
| Region | Không chỉ định | Singapore |
| Auto-deploy | Không | `autoDeploy: true` |

---

## Part 4: API Security

### Exercise 4.1: API Key authentication

**API key check ở đâu?**
```python
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def verify_api_key(api_key: str = Security(api_key_header)) -> str:
    if not api_key:
        raise HTTPException(status_code=401, ...)
    if api_key != API_KEY:
        raise HTTPException(status_code=403, ...)
    return api_key
```

**Nếu sai key:** trả về HTTP 403 Forbidden  
**Rotate key:** thay đổi env var `AGENT_API_KEY` và restart service

**Test results:**
```
# Không có key → 401
{"detail": "Missing API key. Include header: X-API-Key: <your-key>"}

# Có key đúng → 200
{"question": "hello", "answer": "..."}
```

### Exercise 4.2: JWT flow

1. Client POST `/auth/token` với username/password
2. Server verify credentials, tạo JWT với `{sub, role, iat, exp}`, ký bằng `SECRET_KEY` (HS256)
3. Client nhận token (hết hạn sau 60 phút)
4. Client gửi `Authorization: Bearer <token>` trong mỗi request
5. Server verify signature, extract user info → process request (không cần DB lookup)

**Ưu điểm:** Stateless — server không cần lưu session

### Exercise 4.3: Rate limiting

**Algorithm:** Sliding Window Counter  
**Limit:** User: 10 req/min | Admin: 100 req/min  
**Bypass cho admin:** Khác `RateLimiter` instance — `rate_limiter_admin = RateLimiter(max_requests=100, ...)`

**Test results:**
```
Request 1-9:  HTTP 200
Request 10:   HTTP 429 — "Rate limit exceeded"
              Headers: X-RateLimit-Remaining: 0, Retry-After: <seconds>

Admin requests 1-12: HTTP 200 (không bị block)
```

### Exercise 4.4: Cost guard implementation

```python
# Trong production/cost_guard.py
def check_budget(self, user_id: str) -> None:
    record = self._get_record(user_id)
    # Global budget check ($10/ngày toàn hệ thống)
    if self._global_cost >= self.global_daily_budget_usd:
        raise HTTPException(503, "Service unavailable due to budget limits")
    # Per-user budget check ($1/ngày)
    if record.total_cost_usd >= self.daily_budget_usd:
        raise HTTPException(402, {"error": "Daily budget exceeded", ...})
    # Warning khi dùng 80%
    if record.total_cost_usd >= self.daily_budget_usd * 0.8:
        logger.warning(f"User {user_id} at 80% budget")
```

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health và readiness checks

```python
@app.get("/health")
def health():
    # Liveness: agent còn sống?
    return {"status": "ok", "uptime_seconds": ..., "checks": {"memory": {...}}}

@app.get("/ready")
def ready():
    # Readiness: sẵn sàng nhận traffic?
    if not _is_ready:
        raise HTTPException(503, "Agent not ready")
    return {"ready": True, "in_flight_requests": _in_flight_requests}
```

**Test results:**
- `/health`: `{"status":"ok","uptime_seconds":11.9,"version":"1.0.0","checks":{"memory":{"status":"ok","used_percent":85.5}}}`
- `/ready`: `{"ready":true,"in_flight_requests":1}`

**Khác biệt quan trọng:**
- `/health` = liveness: process còn sống? → Platform restart nếu fail
- `/ready` = readiness: có thể nhận request? → Load balancer stop routing nếu fail

### Exercise 5.2: Graceful shutdown

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    global _is_ready
    _is_ready = True
    yield
    # Shutdown phase
    _is_ready = False
    timeout = 30
    while _in_flight_requests > 0 and elapsed < timeout:
        time.sleep(1)  # Chờ request đang xử lý hoàn thành

signal.signal(signal.SIGTERM, handle_sigterm)
```

**Quan sát:** Khi SIGTERM được gửi, server dừng nhận request mới (`_is_ready = False`), nhưng chờ request đang xử lý hoàn thành (tối đa 30s) trước khi exit.

### Exercise 5.3: Stateless design

**Anti-pattern (stateful):**
```python
conversation_history = {}  # State trong memory của instance
```
→ Khi scale 3 instances, user A gửi request 1 tới instance 1, request 2 tới instance 2 → **mất session!**

**Correct (stateless với Redis):**
```python
def save_session(session_id, data):
    _redis.setex(f"session:{session_id}", 3600, json.dumps(data))

def load_session(session_id):
    data = _redis.get(f"session:{session_id}")
    return json.loads(data) if data else {}
```
→ Bất kỳ instance nào cũng đọc được session từ Redis → scale an toàn

**Test results:**
```json
{"session_id": "e5fed0b9-...", "turn": 2, "served_by": "instance-dfd86e", "storage": "in-memory"}
{"session_id": "e5fed0b9-...", "turn": 3, "served_by": "instance-dfd86e", "storage": "in-memory"}
```
Session được giữ qua nhiều turns.

### Exercise 5.4: Load balancing

Docker Compose scale:
```bash
docker compose up --scale agent=3
```
→ Nginx round-robin giữa 3 instances. Nếu 1 instance die (healthcheck fail), Nginx tự loại ra khỏi pool, traffic chuyển sang 2 instances còn lại.

### Exercise 5.5: Stateless test

Khi Redis available: conversation history persist dù request được serve bởi instance khác.  
Khi Redis không có (fallback in-memory): history chỉ tồn tại trong 1 instance — không stateless.

---

## Tổng kết

| Concept | Key lesson |
|---------|-----------|
| Localhost vs Production | Không bao giờ hardcode secrets; dùng env vars + 12-Factor |
| Docker | Multi-stage build: image nhỏ hơn 7.5x (56MB vs 424MB) |
| Cloud deployment | Push to GitHub → Render auto-deploy từ render.yaml |
| API Security | JWT stateless auth + Sliding window rate limit + Cost guard per user |
| Scaling | Stateless design (Redis session) + Health/Ready probes + Graceful shutdown |
