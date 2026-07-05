# Python Worker — Ingestion & AI-Backed Detection (v1)
### The FastAPI counterpart to the Go Agent guide — same TDD rhythm, a swappable detector

*A test-driven, follow-along guide to building the Python side of the system: pytest + `TestClient`, red/green/refactor, one
behaviour at a time — written so the "detector" powering `/telemetry` can be a Gemini Flash-Lite API call today and a real
anomaly-detection algorithm later, without touching anything else.*

```
$ pytest -v
collected 9 items
test_contract.py::test_accepts_go_agent_shaped_payload PASSED
test_detector.py::test_stub_detector_flags_high_cpu PASSED
...
```

**Project:** Distributed Telemetry & Anomaly Detector — Python worker
**Covers:** the JSON contract with the Go agent · an in-memory metric store · a `Detector` protocol (dependency injection, same
idiom as the Go guide's `MetricShipper`) · a Gemini Flash-Lite–backed detector, tested without ever calling the real API · health
checks · config from the environment · wiring into `docker-compose`
**Explicitly out of scope for v1:** real anomaly detection. The `GeminiDetector` built here is a deliberate placeholder — see
"Where This Leaves You" at the end for exactly how it gets replaced later.

---

## How to Use This Guide

Same method as the Go guide: **red** (write a failing test), **green** (smallest code that passes), **refactor** (clean up with the
test as a safety net). `pytest` and `httpx.TestClient` stand in for `go test` and `httptest`. The one design decision worth stating
up front, because it shapes every chapter below:

> **The AI call is hidden behind an interface from the very first line of code that uses it — same as the Go agent's `MetricShipper`
> was hidden behind an interface before a real HTTP client ever touched it.** `/telemetry` never calls Gemini directly. It calls a
> `Detector`, and `GeminiDetector` is just today's implementation of that `Detector`. When you build real anomaly detection later,
> it's a new class with a `check` method — nothing in `main.py` or its tests changes.

### Prerequisites

- Python 3.11+, `pip`, and a virtualenv tool of your choice.
- A Gemini API key (from Google AI Studio) — **only needed to run the real service; every test in this guide runs without one.**
- The Go agent's `SystemMetrics` contract fresh in mind: `{"timestamp": "...", "cpu_usage": 42.5, "mem_usage": 63.1}`.

### Setup

```
$ mkdir worker && cd worker
$ python -m venv .venv && source .venv/bin/activate
$ pip install fastapi "uvicorn[standard]" httpx pydantic pydantic-settings pytest pytest-asyncio respx
$ pip freeze > requirements.txt
$ git init && git add -A && git commit -m "chore: initialize FastAPI worker project"
```

- `httpx` — the async HTTP client we'll use to call Gemini.
- `respx` — mocks `httpx` calls in tests, the same role `httptest.NewServer` played on the Go side, but at the mocking-library
  level instead of a real loopback server (more on that trade-off in Part 4).
- `pydantic-settings` — typed config from environment variables, the Python equivalent of the Go guide's `LoadConfig`.

---

## Part 1 — Locking Down the Contract With the Go Agent

The Go guide's Chapter 5 wrote a test proving `SystemMetrics` encodes as `snake_case` JSON, specifically so this side would match
it. Let's prove that from the Python side too — a contract only holds if *both* ends test it.

### Red — Write the Test First

```python
# test_contract.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_accepts_go_agent_shaped_payload():
    payload = {
        "timestamp": "2026-07-02T12:00:00Z",
        "cpu_usage": 42.5,
        "mem_usage": 63.1,
    }

    response = client.post("/telemetry", json=payload)

    assert response.status_code == 202
```

```
$ pytest -v
ImportError: cannot import name 'app' from 'main' (main.py does not exist)
```

### Green — Make It Pass

```python
# models.py
from datetime import datetime
from pydantic import BaseModel


class SystemMetrics(BaseModel):
    timestamp: datetime
    cpu_usage: float
    mem_usage: float
```

```python
# main.py
from fastapi import FastAPI, status
from models import SystemMetrics

app = FastAPI()


@app.post("/telemetry", status_code=status.HTTP_202_ACCEPTED)
async def ingest(sample: SystemMetrics):
    return {"status": "accepted"}
```

```
$ pytest -v
test_contract.py::test_accepts_go_agent_shaped_payload PASSED
```

Field names are the whole trick here: pydantic matches JSON keys to attribute names by default, so `cpu_usage` in the payload maps
straight onto `cpu_usage: float` with zero configuration — which is exactly why the Go side's `json:"cpu_usage"` tag mattered so much
in Chapter 5. Get either side's naming wrong and this test is the one that catches it.

### Refactor — Reject What Doesn't Match, on Purpose

Right now an extra, unexpected field is silently ignored — same permissive default `encoding/json` has on the Go side, and the same
trade-off discussed in the Go guide's security chapter (15.7): permissive is right for forward-compatibility, wrong the moment you
want to know if a client is sending something you don't expect. For a payload this small and well-owned (you control both ends),
tightening it now costs one line and catches typos early:

```python
# models.py
from datetime import datetime
from pydantic import BaseModel, ConfigDict


class SystemMetrics(BaseModel):
    model_config = ConfigDict(extra="forbid")

    timestamp: datetime
    cpu_usage: float
    mem_usage: float
```

```python
# test_contract.py
def test_rejects_unexpected_fields():
    payload = {
        "timestamp": "2026-07-02T12:00:00Z",
        "cpu_usage": 42.5,
        "mem_usage": 63.1,
        "admin_override": True,  # not part of the contract
    }

    response = client.post("/telemetry", json=payload)

    assert response.status_code == 422
```

```
$ pytest -v
test_contract.py::test_accepts_go_agent_shaped_payload PASSED
test_contract.py::test_rejects_unexpected_fields PASSED
```

### Checkpoint

```
$ pytest -v
2 passed
$ git add -A && git commit -m "feat(contract): SystemMetrics pydantic model matching the Go agent's JSON

extra='forbid' rejects unexpected fields early, mirroring the
DisallowUnknownFields discussion from the Go guide's security chapter."
```

---

## Part 2 — An In-Memory Metric Store

Before anything can be checked for "is this anomalous," something needs to remember what came before it. A fixed-size rolling
window — the Python equivalent of the Go agent's planned `MetricStore` — is all v1 needs: no database, no persistence yet, just
enough history in memory to give a detector context.

### Red — Write the Test First

```python
# test_store.py
from datetime import datetime
from models import SystemMetrics
from store import MetricStore


def make_sample(cpu: float) -> SystemMetrics:
    return SystemMetrics(timestamp=datetime.now(), cpu_usage=cpu, mem_usage=50.0)


def test_store_keeps_the_most_recent_n_samples():
    store = MetricStore(max_size=3)

    for cpu in [10.0, 20.0, 30.0, 40.0]:
        store.add(make_sample(cpu))

    history = store.history()

    assert len(history) == 3
    assert [s.cpu_usage for s in history] == [20.0, 30.0, 40.0]


def test_empty_store_has_no_history():
    store = MetricStore(max_size=3)

    assert store.history() == []
```

```
$ pytest -v
ModuleNotFoundError: No module named 'store'
```

### Green — Make It Pass

```python
# store.py
from collections import deque
from models import SystemMetrics


class MetricStore:
    def __init__(self, max_size: int = 60):
        self._samples: deque[SystemMetrics] = deque(maxlen=max_size)

    def add(self, sample: SystemMetrics) -> None:
        self._samples.append(sample)

    def history(self) -> list[SystemMetrics]:
        return list(self._samples)
```

```
$ pytest -v
2 passed
```

`deque(maxlen=...)` does the ring-buffer eviction for free — the oldest sample silently drops off once the window is full. No
manual index arithmetic, no off-by-one risk.

### Refactor — Wire It Into `/telemetry`

```python
# main.py
from fastapi import FastAPI, status
from models import SystemMetrics
from store import MetricStore

app = FastAPI()
store = MetricStore(max_size=60)  # 5 minutes of history at a 5s collection interval


@app.post("/telemetry", status_code=status.HTTP_202_ACCEPTED)
async def ingest(sample: SystemMetrics):
    store.add(sample)
    return {"status": "accepted"}
```

> **A module-level `store` is a shortcut, not a design decision — flag it now.** This is the same trade-off the Go guide called out
> for its Phase 3 package-level variable: a global is quick, but it's shared mutable state that makes tests awkward to parallelize
> and hides a dependency instead of naming it. FastAPI's own answer is `Depends()` — Part 5 switches to it once the detector needs
> the same treatment, and cleans this up at the same time.

### Checkpoint

```
$ pytest -v
4 passed
$ git add -A && git commit -m "feat(store): add a bounded in-memory history via collections.deque"
```

---

## Part 3 — The `Detector` Protocol: the Swap Point for Later

This is the chapter that matters most for your plan — everything after it (Gemini today, real anomaly detection later) hangs off
this one interface. Python's structural typing (`typing.Protocol`) is the direct equivalent of Go's implicit interface
satisfaction: any class with a matching `check` method satisfies `Detector`, without ever writing `class Foo(Detector)`.

### Red — Write the Test First

```python
# test_detector.py
from datetime import datetime
from models import SystemMetrics
from detector import StubDetector, DetectionResult


def make_sample(cpu: float) -> SystemMetrics:
    return SystemMetrics(timestamp=datetime.now(), cpu_usage=cpu, mem_usage=50.0)


async def test_stub_detector_returns_a_configured_result():
    detector = StubDetector(result=DetectionResult(is_anomaly=True, reason="test fixture"))

    result = await detector.check(make_sample(99.0), history=[])

    assert result.is_anomaly is True
    assert result.reason == "test fixture"
```

```
$ pytest -v
ModuleNotFoundError: No module named 'detector'
```

### Green — Make It Pass

```python
# detector.py
from typing import Protocol
from pydantic import BaseModel
from models import SystemMetrics


class DetectionResult(BaseModel):
    is_anomaly: bool
    reason: str


class Detector(Protocol):
    async def check(
        self, sample: SystemMetrics, history: list[SystemMetrics]
    ) -> DetectionResult: ...


class StubDetector:
    """A canned detector for tests — the Python equivalent of the Go
    guide's StubMetricsSource / stubFlakyShipper. Never calls a real API."""

    def __init__(self, result: DetectionResult):
        self._result = result

    async def check(
        self, sample: SystemMetrics, history: list[SystemMetrics]
    ) -> DetectionResult:
        return self._result
```

```
$ pytest -v
5 passed
```

### Refactor — Nothing to Do Yet

Same as the Go guide's Phase 1 — there's no duplication or unclear naming yet to clean up. The value of this chapter isn't the code
you just wrote, it's the shape it locks in: `async def check(sample, history) -> DetectionResult` is now the contract every future
detector — `GeminiDetector` in Part 4, a real statistical or ML-based one later — has to satisfy, and every consumer of a `Detector`
(the `/telemetry` endpoint, and any test) only ever depends on that shape, never on which implementation is behind it.

### Checkpoint

```
$ pytest -v
5 passed
$ git add -A && git commit -m "feat(detector): define the Detector protocol + a StubDetector for tests

This is the seam GeminiDetector plugs into next, and the seam a real
anomaly-detection implementation replaces it at later — nothing else
in the codebase needs to know which one is behind it."
```

---

## Part 4 — A Gemini Flash-Lite–Backed Detector, Tested Without Ever Calling Gemini

This is the Python equivalent of the Go guide's Phase 3 — the moment real network I/O enters the picture. The same principle
applies: **the test never talks to the real service.** On the Go side, `httptest.NewServer` stood up a real (loopback) HTTP server
so `Shipper` could be tested against real `net/http` machinery. Here, we mock `httpx` directly with `respx`, which intercepts
outgoing requests before they hit the network at all — a slightly different mechanism (no real socket involved), but the same goal:
fast, deterministic, no API key or network required to run the suite.

> **Model name and endpoint may drift — verify against current docs before wiring this to production.** Google's Gemini API surface
> (model IDs, request/response shape) changes over time; treat `GEMINI_MODEL` below as a placeholder to confirm against
> [Google AI Studio](https://aistudio.google.com) / the current Gemini API docs when you actually run this against the real
> service, not as a guaranteed-correct value.

### Red — Write the Test First

```python
# test_gemini_detector.py
import json
import httpx
import respx
from datetime import datetime
from models import SystemMetrics
from detector import GeminiDetector


def make_sample(cpu: float) -> SystemMetrics:
    return SystemMetrics(timestamp=datetime.now(), cpu_usage=cpu, mem_usage=50.0)


def gemini_response(text: str) -> dict:
    # Shape of a real Gemini generateContent response, trimmed to what we read.
    return {"candidates": [{"content": {"parts": [{"text": text}]}}]}


@respx.mock
async def test_gemini_detector_parses_an_anomaly_flag():
    route = respx.post(url__regex=r".*generativelanguage\.googleapis\.com.*").mock(
        return_value=httpx.Response(
            200,
            json=gemini_response(
                json.dumps({"is_anomaly": True, "reason": "CPU spiked well above recent baseline"})
            ),
        )
    )
    detector = GeminiDetector(api_key="test-key", model="gemini-flash-lite-latest")

    result = await detector.check(make_sample(97.0), history=[make_sample(20.0), make_sample(22.0)])

    assert route.called
    assert result.is_anomaly is True
    assert "spiked" in result.reason


@respx.mock
async def test_gemini_detector_fails_open_on_api_error():
    respx.post(url__regex=r".*generativelanguage\.googleapis\.com.*").mock(
        return_value=httpx.Response(503)
    )
    detector = GeminiDetector(api_key="test-key", model="gemini-flash-lite-latest")

    result = await detector.check(make_sample(97.0), history=[])

    assert result.is_anomaly is False
    assert "unavailable" in result.reason.lower()
```

**Why "fails open," not "raises":** if the AI API is down, slow, or rate-limited, the whole point of this detector is that
ingestion shouldn't crash or block on it. This mirrors the Go guide's `resp.StatusCode >= 300` handling in `Shipper` — a downstream
failure becomes a typed result the caller can act on, not an unhandled exception that takes `/telemetry` down with it.

```
$ pytest -v
ImportError: cannot import name 'GeminiDetector' from 'detector'
```

### Green — Make It Pass

```python
# detector.py (additions)
import httpx


class GeminiDetector:
    """Calls Gemini Flash-Lite to classify a sample as anomalous.
    A deliberate v1 placeholder for real anomaly detection — see the
    Detector protocol above for the interface it satisfies."""

    def __init__(self, api_key: str, model: str, timeout: float = 5.0):
        self._api_key = api_key
        self._model = model
        self._timeout = timeout
        self._url = (
            f"https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent"
        )

    async def check(
        self, sample: SystemMetrics, history: list[SystemMetrics]
    ) -> DetectionResult:
        prompt = self._build_prompt(sample, history)
        body = {"contents": [{"parts": [{"text": prompt}]}]}

        try:
            async with httpx.AsyncClient(timeout=self._timeout) as client:
                resp = await client.post(
                    self._url,
                    params={"key": self._api_key},
                    json=body,
                )
                resp.raise_for_status()
        except httpx.HTTPError:
            return DetectionResult(is_anomaly=False, reason="detector unavailable, failing open")

        return self._parse(resp.json())

    def _build_prompt(self, sample: SystemMetrics, history: list[SystemMetrics]) -> str:
        recent = ", ".join(f"{s.cpu_usage:.1f}%" for s in history[-5:]) or "no prior data"
        return (
            "You monitor server CPU usage. Recent readings: "
            f"{recent}. Current reading: {sample.cpu_usage:.1f}%. "
            'Reply with ONLY a JSON object: {"is_anomaly": bool, "reason": "one short sentence"}.'
        )

    def _parse(self, payload: dict) -> DetectionResult:
        try:
            import json as _json

            text = payload["candidates"][0]["content"]["parts"][0]["text"]
            data = _json.loads(text)
            return DetectionResult(is_anomaly=bool(data["is_anomaly"]), reason=str(data["reason"]))
        except (KeyError, IndexError, ValueError):
            return DetectionResult(is_anomaly=False, reason="could not parse detector response")
```

```
$ pytest -v
7 passed
```

### Refactor — a Parsing Test of Its Own

The `_parse` method's except clause is currently only exercised indirectly. Give it a direct test, the same way the Go guide gave
`newTelemetryRequest` its own coverage after extracting it:

```python
# test_gemini_detector.py
async def test_gemini_detector_fails_open_on_unparseable_response():
    detector = GeminiDetector(api_key="test-key", model="gemini-flash-lite-latest")

    result = detector._parse({"candidates": [{"content": {"parts": [{"text": "not json"}]}}]})

    assert result.is_anomaly is False
    assert "could not parse" in result.reason
```

> **A prompt is not a schema.** Asking the model to "reply with ONLY a JSON object" is a request, not a guarantee — a Gemini update,
> an unusual input, or plain model variance can still return prose, markdown-fenced JSON, or a slightly different shape. The
> `try`/`except` above treats any parse failure as "no signal," never as a crash, which is the load-bearing safety property of this
> whole detector. If you find yourself relying on this parser for anything more than a v1 placeholder, structured output modes
> (JSON schema constraints, if the API you're using supports them) are worth checking for, rather than tightening the regex-style
> parsing by hand.

### Checkpoint

```
$ pytest -v
8 passed
$ git add -A && git commit -m "feat(detector): add GeminiDetector, tested entirely via respx mocks

Fails open on any HTTP error or unparseable response so a flaky or
slow AI API can never take down metric ingestion."
```

---

## Part 5 — Wiring the Detector Into `/telemetry` With `Depends`

Part 2 flagged the module-level `store` as a shortcut. Now that there's a second thing (`detector`) that needs the same treatment,
it's worth fixing both at once using FastAPI's dependency injection — the same idiom as the rest of this guide, expressed in
FastAPI's own vocabulary.

### Red — Write the Test First

```python
# test_ingest.py
from datetime import datetime
from fastapi.testclient import TestClient
from main import app, get_detector, get_store
from detector import StubDetector, DetectionResult
from store import MetricStore


def test_ingest_calls_the_injected_detector_and_records_history():
    spy_store = MetricStore(max_size=10)
    stub_detector = StubDetector(result=DetectionResult(is_anomaly=True, reason="stub"))

    app.dependency_overrides[get_store] = lambda: spy_store
    app.dependency_overrides[get_detector] = lambda: stub_detector
    client = TestClient(app)

    response = client.post(
        "/telemetry",
        json={"timestamp": "2026-07-02T12:00:00Z", "cpu_usage": 42.5, "mem_usage": 63.1},
    )

    app.dependency_overrides.clear()

    assert response.status_code == 202
    assert response.json()["is_anomaly"] is True
    assert len(spy_store.history()) == 1
```

**Why `dependency_overrides` instead of monkeypatching the module-level globals:** this is FastAPI's built-in seam for exactly this
situation — swap a real dependency for a test double per-test, without any global state leaking between tests the way Part 2's
module-level `store` would. It's the direct Python-web-framework equivalent of Go's constructor injection: the endpoint asks for a
`Detector` and a `MetricStore` by type, and doesn't know or care whether it's getting the real one or a stub.

```
$ pytest -v
ImportError: cannot import name 'get_detector' from 'main'
```

### Green — Make It Pass

```python
# main.py
from fastapi import FastAPI, Depends, status
from models import SystemMetrics
from store import MetricStore
from detector import Detector, GeminiDetector
from config import get_settings

app = FastAPI()

_store = MetricStore(max_size=60)
_detector: Detector = GeminiDetector(
    api_key=get_settings().gemini_api_key,
    model=get_settings().gemini_model,
)


def get_store() -> MetricStore:
    return _store


def get_detector() -> Detector:
    return _detector


@app.post("/telemetry", status_code=status.HTTP_202_ACCEPTED)
async def ingest(
    sample: SystemMetrics,
    store: MetricStore = Depends(get_store),
    detector: Detector = Depends(get_detector),
):
    history = store.history()
    store.add(sample)
    result = await detector.check(sample, history)
    return {"status": "accepted", "is_anomaly": result.is_anomaly, "reason": result.reason}
```

(`config.py` and `get_settings()` land in Part 7 — for now, assume it returns an object with `gemini_api_key`/`gemini_model`
attributes; the tests above never construct the real `GeminiDetector`, so this doesn't block Part 5's test from passing once
`config.py` exists with sane defaults.)

```
$ pytest -v
9 passed
```

### Refactor — Nothing Structural, One Naming Check

`get_store`/`get_detector` read clearly as "the thing `Depends()` calls," which is the FastAPI convention — no further extraction
needed. Worth double-checking one thing before moving on: `history = store.history()` is read *before* `store.add(sample)`, so the
detector sees "everything before this sample" as context, not including the sample itself twice. That ordering is deliberate;
swapping it would let the current reading silently pollute its own baseline.

### Checkpoint

```
$ pytest -v
9 passed
$ git add -A && git commit -m "feat(ingest): wire detector + store into /telemetry via FastAPI Depends

Uses dependency_overrides in tests instead of a module-level global,
closing the gap flagged back in Part 2."
```

---

## Part 6 — A Health Endpoint

`docker-compose.yml` already expects `curl -f http://localhost:8000/health` (see the Go guide's Chapter 13). A minimal endpoint
satisfies it — no dependency on the detector or Gemini being reachable, since the worker being "up" and Gemini being "reachable"
are two different failure modes you don't want conflated into one status code.

### Red — Write the Test First

```python
# test_health.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_health_returns_ok():
    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

```
$ pytest -v
FAILED test_health.py::test_health_returns_ok - assert 404 == 200
```

### Green — Make It Pass

```python
# main.py (addition)
@app.get("/health")
async def health():
    return {"status": "ok"}
```

```
$ pytest -v
10 passed
```

### Checkpoint

```
$ git add -A && git commit -m "feat(health): add /health for docker-compose's healthcheck"
```

---

## Part 7 — Configuration From the Environment

Same job as the Go guide's Chapter 11, `pydantic-settings` in place of a hand-written `LoadConfig`. The one thing worth extra care
here: **the API key is a secret, and this is the first place it enters the system.**

### Red — Write the Test First

```python
# test_config.py
from config import get_settings


def test_defaults_are_sensible(monkeypatch):
    monkeypatch.setenv("GEMINI_API_KEY", "test-key")
    get_settings.cache_clear()

    settings = get_settings()

    assert settings.gemini_model == "gemini-flash-lite-latest"
    assert settings.detector_timeout_seconds == 5.0


def test_reads_overrides_from_environment(monkeypatch):
    monkeypatch.setenv("GEMINI_API_KEY", "test-key")
    monkeypatch.setenv("GEMINI_MODEL", "gemini-flash-lite-preview")
    get_settings.cache_clear()

    settings = get_settings()

    assert settings.gemini_model == "gemini-flash-lite-preview"


def test_missing_api_key_raises(monkeypatch):
    monkeypatch.delenv("GEMINI_API_KEY", raising=False)
    get_settings.cache_clear()

    try:
        get_settings()
        assert False, "expected a validation error for a missing API key"
    except Exception:
        pass
```

`monkeypatch.setenv`/`delenv` are pytest's built-in equivalent of the Go guide's `t.Setenv` — scoped to the test, cleaned up
automatically afterward.

```
$ pytest -v
ModuleNotFoundError: No module named 'config'
```

### Green — Make It Pass

```python
# config.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    gemini_api_key: str
    gemini_model: str = "gemini-flash-lite-latest"
    detector_timeout_seconds: float = 5.0


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`gemini_api_key: str` with no default makes it a required field — pydantic raises a `ValidationError` if it's missing, which is
exactly what `test_missing_api_key_raises` checks for. `@lru_cache` means `get_settings()` reads the environment once and reuses the
result, the same "load once, pass it around" shape as the Go guide's `Config` built once in `main` and threaded through
constructors.

```
$ pytest -v
13 passed
```

### Refactor — Never Let the Key Reach a Log Line

This is the Python equivalent of the Go guide's Section 15.3 (redact secrets before they can be logged). `Settings` objects are easy
to accidentally `print()` or log whole during debugging — add a `__repr__` override so that mistake can't leak the key:

```python
# config.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    gemini_api_key: str
    gemini_model: str = "gemini-flash-lite-latest"
    detector_timeout_seconds: float = 5.0

    def __repr__(self) -> str:
        return f"Settings(gemini_model={self.gemini_model!r}, gemini_api_key='***redacted***')"
```

```python
# test_config.py
def test_repr_never_includes_the_api_key(monkeypatch):
    monkeypatch.setenv("GEMINI_API_KEY", "super-secret-value")
    get_settings.cache_clear()

    assert "super-secret-value" not in repr(get_settings())
```

### Checkpoint

```
$ pytest -v
14 passed
$ git add -A && git commit -m "feat(config): typed settings via pydantic-settings, required API key

__repr__ override keeps the key out of any accidental log/print of
the Settings object."
```

---

## Part 8 — Wiring Into docker-compose

### Dockerfile

```dockerfile
# worker/Dockerfile
FROM python:3.12-slim AS build
WORKDIR /src
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.12-slim
RUN useradd --uid 65534 --no-create-home nobody
COPY --from=build /root/.local /home/nobody/.local
WORKDIR /app
COPY . .
ENV PATH=/home/nobody/.local/bin:$PATH
USER nobody
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

A non-root `USER`, matching the same hardening step the Go guide applied to its own container in Chapter 15.5 — worth doing on both
sides, not just the one that happened to get a security chapter first.

### docker-compose.yml (worker service, updated)

```yaml
  python-worker:
    build: ./worker
    environment:
      GEMINI_API_KEY: ${GEMINI_API_KEY}   # from your shell or a .env file — never committed
      GEMINI_MODEL: gemini-flash-lite-latest
    expose:
      - "8000"
    networks:
      - telemetry-net
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"]
      interval: 30s
      timeout: 3s
      retries: 3
```

`${GEMINI_API_KEY}` pulled from the shell environment (or an untracked `.env` file next to `docker-compose.yml`, picked up
automatically by Compose) keeps the key out of the image and out of git — the same "config from environment, never hardcoded"
principle as the Go guide's Chapter 11, applied to the one value in this whole worker that actually needs to be a secret.

```
$ echo "GEMINI_API_KEY=your-real-key-here" > .env
$ echo ".env" >> .gitignore
$ docker compose up --build
```

### Checkpoint

```
$ docker compose up --build
$ curl -X POST http://localhost:8000/telemetry \
    -H "Content-Type: application/json" \
    -d '{"timestamp":"2026-07-02T12:00:00Z","cpu_usage":91.0,"mem_usage":40.0}'
{"status":"accepted","is_anomaly":true,"reason":"CPU usage is unusually high relative to recent readings"}
```

```
$ git add -A && git commit -m "chore(docker): containerize the worker, non-root user, key via env"
```

---

## Where This Leaves You

The worker ingests exactly what the Go agent sends, remembers a rolling window of recent samples, and asks Gemini Flash-Lite
whether the newest one looks off — failing open, never crashing ingestion, if the API is slow, down, or returns something
unparseable. That's a real, demoable, end-to-end loop: agent → worker → an AI opinion, wired through `docker-compose`, entirely
covered by tests that never make a real network call.

It is also, honestly, exactly as much of a "real" anomaly detector as the earlier critique warned about: a prompt asking a language
model to eyeball five numbers and guess. No calibration, no false-positive rate you can actually measure, no memory of what "normal"
means beyond whatever fits in the prompt. That's fine — it was never meant to be more than a placeholder with the right shape, and
the whole point of Part 3 was making sure the shape is the part that lasts.

### The Swap, When You're Ready

Replacing `GeminiDetector` with real anomaly detection (a rolling z-score, later something learned) is, by design, small:

1. Write a new class — `ZScoreDetector`, say — with an `async def check(self, sample, history) -> DetectionResult` method. It
   doesn't need to be async internally (no I/O), but keeping the signature identical means Part 5's `Depends()` wiring doesn't
   change at all.
2. Give it its own red/green/refactor cycle and its own test file, exactly like `StubDetector` and `GeminiDetector` got here.
3. Change one line in `main.py` — what `_detector` is constructed as — and optionally keep `GeminiDetector` around behind a
   `DETECTOR_BACKEND` config flag, so you can compare the two side by side on the same live traffic before fully cutting over.

Nothing in `store.py`, `main.py`'s route signature, or the contract tests from Part 1 needs to know the difference. That's the
payoff of Part 3, not a coincidence.

