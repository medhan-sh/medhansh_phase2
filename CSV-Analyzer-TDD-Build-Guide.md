# CSV Analyzer — A Test-Driven Build Guide
### Go API + Redis Queue + Python Worker + Docker

*A test-driven, follow-along guide to building the CSV Analyzer from an empty pair of modules to a working, dockerized system —
written in the same red/green/refactor tradition as the Telemetry Agent guides, applied here to a request/response pipeline
instead of a streaming one.*

```
$ curl -F "file=@sales.csv" http://localhost:8080/upload
{"job_id":"3f9c1a2e-...","status":"pending"}
```

**Project:** CSV Analyzer — upload a CSV, get back cleaned stats and a chart
**Covers:** the job contract · dependency injection for Redis (done from the start, not bolted on later) · TDD-ing HTTP handlers
with `httptest` · TDD-ing pandas cleaning/stats logic with `pytest` · a job index endpoint · upload guards · retention · Docker
Compose hardening · a minimal web dashboard

---

## How to Use This Guide

Same method as the Telemetry Agent guides: **red** (write a failing test), **green** (smallest code that passes), **refactor**
(clean up with the test as a safety net). The one lesson carried over directly from that project's own mistake-and-fix history:
**Redis access is hidden behind a small interface from the first line of code that touches it**, not bolted on later. A quick
package-level `var rdb *redis.Client` is tempting — it's exactly the shortcut the Telemetry Agent guide took in an early draft of
its HTTP shipper and then had to undo. This guide skips that detour: `JobStore` exists before `handleUpload` does.

A quick, honest note on TDD here, same as before: the chart-rendering code (matplotlib output) has no meaningfully decidable
behaviour a test can pin down — "does this PNG look right" isn't something `assert` can answer. That code gets written directly,
run, and eyeballed, same as the Telemetry Agent's Dockerfile or its raw `/proc` read. Everything with real branching logic — the
JSON contract, the cleaning rules, the stats math, every HTTP handler — gets a test first.

### Prerequisites

- Go 1.22+, Python 3.11+, Docker, `git`.
- A `redis-server` you can run locally for iterating outside Docker (`brew install redis` / `apt install redis-server`), or just
  `docker run -p 6379:6379 redis:7-alpine` in a spare terminal.

### Setup

```
$ mkdir csv-analyzer && cd csv-analyzer
$ git init

$ mkdir api && cd api
$ go mod init csv-analyzer/api
$ go get github.com/google/uuid github.com/redis/go-redis/v9
$ cd ..

$ mkdir worker && cd worker
$ python -m venv .venv && source .venv/bin/activate
$ pip install pandas matplotlib redis pytest fakeredis
$ pip freeze > requirements.txt
$ cd ..

$ git add -A && git commit -m "chore: initialize Go api module and Python worker venv"
```

`fakeredis` is doing here what `respx` did for the Telemetry Agent's Gemini detector: an in-memory stand-in so the worker's tests
never need a real Redis instance running.

---

# Part I — The Go API

## Chapter 1 — The Job Contract

Every job is one small record: an ID, a status, and an optional error. Before any HTTP or Redis code exists, lock down its shape
— the same instinct as the Telemetry Agent's `SystemMetrics`, for the same reason: it's the one value every other piece of this
system agrees on.

### Red — Write the Test First

```go
// job_test.go
package main

import (
	"encoding/json"
	"testing"
)

func TestJobStatusJSON(t *testing.T) {
	j := JobStatus{JobID: "abc-123", Status: "pending"}

	got, err := json.Marshal(j)
	if err != nil {
		t.Fatalf("did not expect an error, got %v", err)
	}

	want := `{"job_id":"abc-123","status":"pending"}`
	if string(got) != want {
		t.Errorf("got %s\nwant %s", got, want)
	}
}

func TestJobStatusJSON_OmitsEmptyError(t *testing.T) {
	j := JobStatus{JobID: "abc-123", Status: "error", Error: "bad csv"}

	got, _ := json.Marshal(j)

	want := `{"job_id":"abc-123","status":"error","error":"bad csv"}`
	if string(got) != want {
		t.Errorf("got %s\nwant %s", got, want)
	}
}
```

```
$ go test ./...
./job_test.go:9:10: undefined: JobStatus
FAIL
```

### Green — Make It Pass

```go
// job.go
package main

type JobStatus struct {
	JobID  string `json:"job_id"`
	Status string `json:"status"` // pending, processing, done, error
	Error  string `json:"error,omitempty"`
}
```

```
$ go test ./...
ok      csv-analyzer/api    0.003s
```

### Checkpoint

```
$ git add -A && git commit -m "feat(api): add JobStatus with a JSON-contract test"
```

---

## Chapter 2 — A `JobStore` Interface, Before Any Redis Code Exists

This is the chapter that matters most for everything downstream, same role Chapter 0 played in the Telemetry Agent guide: name
the interface first, so every handler that needs Redis depends on a two-method contract instead of a concrete client.

### Red — Write the Test First

The test writes against an interface that doesn't exist yet, using a fake implementation that lives entirely in the test file —
no real Redis anywhere in this chapter.

```go
// jobstore_test.go
package main

import "testing"

type fakeJobStore struct {
	jobs  map[string]JobStatus
	queue []string
}

func newFakeJobStore() *fakeJobStore {
	return &fakeJobStore{jobs: map[string]JobStatus{}}
}

func (f *fakeJobStore) Save(job JobStatus) error {
	f.jobs[job.JobID] = job
	return nil
}

func (f *fakeJobStore) Get(jobID string) (JobStatus, bool, error) {
	j, ok := f.jobs[jobID]
	return j, ok, nil
}

func (f *fakeJobStore) Enqueue(jobID string) error {
	f.queue = append(f.queue, jobID)
	return nil
}

func TestFakeJobStore_RoundTrips(t *testing.T) {
	store := newFakeJobStore()
	job := JobStatus{JobID: "abc", Status: "pending"}

	if err := store.Save(job); err != nil {
		t.Fatalf("did not expect an error, got %v", err)
	}

	got, found, err := store.Get("abc")
	if err != nil {
		t.Fatalf("did not expect an error, got %v", err)
	}
	if !found {
		t.Fatal("expected to find the job, did not")
	}
	if got != job {
		t.Errorf("got %+v, want %+v", got, job)
	}
}
```

This test only proves the fake behaves sensibly — its real purpose is to force `JobStore` into existence with exactly the shape
the fake already assumes.

```
$ go test ./...
./jobstore_test.go:12:19: undefined: JobStore  # (implicitly, once handlers reference it below)
```

### Green — Define the Interface and a Real Redis-Backed Implementation

```go
// jobstore.go
package main

import (
	"context"
	"encoding/json"

	"github.com/redis/go-redis/v9"
)

// JobStore is the seam between HTTP handlers and Redis — the direct
// equivalent of the Telemetry Agent guide's MetricShipper interface.
type JobStore interface {
	Save(job JobStatus) error
	Get(jobID string) (JobStatus, bool, error)
	Enqueue(jobID string) error
}

type RedisJobStore struct {
	client *redis.Client
	ctx    context.Context
}

func NewRedisJobStore(client *redis.Client) *RedisJobStore {
	return &RedisJobStore{client: client, ctx: context.Background()}
}

func (s *RedisJobStore) Save(job JobStatus) error {
	data, err := json.Marshal(job)
	if err != nil {
		return err
	}
	return s.client.Set(s.ctx, "job:"+job.JobID, data, 0).Err()
}

func (s *RedisJobStore) Get(jobID string) (JobStatus, bool, error) {
	val, err := s.client.Get(s.ctx, "job:"+jobID).Result()
	if err == redis.Nil {
		return JobStatus{}, false, nil
	}
	if err != nil {
		return JobStatus{}, false, err
	}
	var job JobStatus
	if err := json.Unmarshal([]byte(val), &job); err != nil {
		return JobStatus{}, false, err
	}
	return job, true, nil
}

func (s *RedisJobStore) Enqueue(jobID string) error {
	return s.client.LPush(s.ctx, "jobs:queue", jobID).Err()
}
```

```
$ go test ./...
ok      csv-analyzer/api    0.004s
```

`RedisJobStore` doesn't get a unit test of its own — same reasoning as the Telemetry Agent's `GopsutilSource`: it's a thin adapter
with no branching logic of its own, and there's nothing here that a test could catch that `go-redis`'s own test suite doesn't
already cover. What *does* get tested, extensively, is every handler that depends on `JobStore` — using `fakeJobStore`, never a
real Redis connection.

### Checkpoint

```
$ go test ./...
ok      csv-analyzer/api    0.005s
$ git add -A && git commit -m "feat(api): add JobStore interface + Redis implementation

Handlers will depend on JobStore, never on *redis.Client directly —
avoids the package-level-global detour the Telemetry Agent guide
had to undo in its own Phase 3."
```

---

## Chapter 3 — The Upload Handler

`POST /upload` needs to: accept a multipart file, save it to disk under a generated job ID, write a `pending` job to the store,
and push that ID onto the queue. Four responsibilities, one handler — testable now that `JobStore` is an interface and the upload
directory is a parameter instead of a global.

### Red — Write the Test First

```go
// upload_test.go
package main

import (
	"bytes"
	"encoding/json"
	"mime/multipart"
	"net/http"
	"net/http/httptest"
	"os"
	"path/filepath"
	"testing"
)

func newMultipartUpload(t *testing.T, filename, content string) (*bytes.Buffer, string) {
	t.Helper()
	var buf bytes.Buffer
	writer := multipart.NewWriter(&buf)
	part, err := writer.CreateFormFile("file", filename)
	if err != nil {
		t.Fatalf("did not expect an error, got %v", err)
	}
	part.Write([]byte(content))
	writer.Close()
	return &buf, writer.FormDataContentType()
}

func TestHandleUpload_SavesFileAndQueuesJob(t *testing.T) {
	store := newFakeJobStore()
	uploadDir := t.TempDir()
	handler := &UploadHandler{store: store, uploadDir: uploadDir, maxBytes: 10 << 20}

	body, contentType := newMultipartUpload(t, "sales.csv", "a,b\n1,2\n")
	req := httptest.NewRequest(http.MethodPost, "/upload", body)
	req.Header.Set("Content-Type", contentType)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("got status %d, want %d, body: %s", rec.Code, http.StatusOK, rec.Body.String())
	}

	var job JobStatus
	if err := json.NewDecoder(rec.Body).Decode(&job); err != nil {
		t.Fatalf("could not decode response: %v", err)
	}
	if job.Status != "pending" {
		t.Errorf("got status %q, want %q", job.Status, "pending")
	}
	if job.JobID == "" {
		t.Error("expected a non-empty job ID")
	}

	savedPath := filepath.Join(uploadDir, job.JobID+".csv")
	if _, err := os.Stat(savedPath); err != nil {
		t.Errorf("expected the file to be saved at %s, got %v", savedPath, err)
	}

	saved, _, found := store.jobs[job.JobID], "", true
	_ = found
	if saved.Status != "pending" {
		t.Errorf("store has status %q, want %q", saved.Status, "pending")
	}
	if len(store.queue) != 1 || store.queue[0] != job.JobID {
		t.Errorf("expected the job to be enqueued, queue was %v", store.queue)
	}
}

func TestHandleUpload_RejectsMissingFile(t *testing.T) {
	store := newFakeJobStore()
	handler := &UploadHandler{store: store, uploadDir: t.TempDir(), maxBytes: 10 << 20}

	req := httptest.NewRequest(http.MethodPost, "/upload", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusBadRequest {
		t.Errorf("got status %d, want %d", rec.Code, http.StatusBadRequest)
	}
}
```

```
$ go test ./...
./upload_test.go:23:14: undefined: UploadHandler
FAIL
```

### Green — Make It Pass

```go
// upload.go
package main

import (
	"encoding/json"
	"io"
	"net/http"
	"os"
	"path/filepath"

	"github.com/google/uuid"
)

type UploadHandler struct {
	store     JobStore
	uploadDir string
	maxBytes  int64
}

func (h *UploadHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	r.Body = http.MaxBytesReader(w, r.Body, h.maxBytes)

	file, _, err := r.FormFile("file")
	if err != nil {
		http.Error(w, "missing file field", http.StatusBadRequest)
		return
	}
	defer file.Close()

	jobID := uuid.New().String()
	destPath := filepath.Join(h.uploadDir, jobID+".csv")

	dest, err := os.Create(destPath)
	if err != nil {
		http.Error(w, "failed to save file", http.StatusInternalServerError)
		return
	}
	defer dest.Close()

	if _, err := io.Copy(dest, file); err != nil {
		http.Error(w, "failed to write file (too large?)", http.StatusBadRequest)
		return
	}

	job := JobStatus{JobID: jobID, Status: "pending"}
	if err := h.store.Save(job); err != nil {
		http.Error(w, "failed to save job", http.StatusInternalServerError)
		return
	}
	if err := h.store.Enqueue(jobID); err != nil {
		http.Error(w, "failed to queue job", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(job)
}
```

```
$ go test ./...
ok      csv-analyzer/api    0.014s
```

**`http.MaxBytesReader` is the upload-size guard flagged in the earlier review** — folded in from the start here rather than
patched on later. A request body over `maxBytes` now fails the `io.Copy` with a clear `400`, instead of an unbounded read that
could exhaust memory on a large or malicious file.

### Refactor — Nothing Structural Yet

The handler is doing four things, but each is a couple of lines and the sequence — validate, save file, record job, enqueue — reads
top to bottom with no duplication. Worth revisiting only if it grows; it hasn't yet.

### Checkpoint

```
$ go test ./...
ok      csv-analyzer/api    0.014s
$ git add -A && git commit -m "feat(api): add UploadHandler with MaxBytesReader-guarded uploads

Tested entirely against a fakeJobStore and t.TempDir() — no real
Redis or filesystem side effects beyond the test's own temp dir."
```

---

## Chapter 4 — The Status Handler

`GET /status/{job_id}` reads a job back from the store. Small, but worth its own test — particularly the not-found path, which the
original handler's `filepath.Base` parsing made easy to get subtly wrong.

### Red — Write the Test First

```go
// status_test.go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHandleStatus_ReturnsAKnownJob(t *testing.T) {
	store := newFakeJobStore()
	store.Save(JobStatus{JobID: "abc", Status: "processing"})
	handler := &StatusHandler{store: store}

	req := httptest.NewRequest(http.MethodGet, "/status/abc", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("got status %d, want %d", rec.Code, http.StatusOK)
	}
}

func TestHandleStatus_UnknownJobReturns404(t *testing.T) {
	store := newFakeJobStore()
	handler := &StatusHandler{store: store}

	req := httptest.NewRequest(http.MethodGet, "/status/does-not-exist", nil)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusNotFound {
		t.Errorf("got status %d, want %d", rec.Code, http.StatusNotFound)
	}
}
```

```
$ go test ./...
./status_test.go:13:14: undefined: StatusHandler
FAIL
```

### Green — Make It Pass

```go
// status.go
package main

import (
	"encoding/json"
	"net/http"
	"strings"
)

type StatusHandler struct {
	store JobStore
}

func (h *StatusHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	jobID := strings.TrimPrefix(r.URL.Path, "/status/")
	job, found, err := h.store.Get(jobID)
	if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	if !found {
		http.Error(w, "job not found", http.StatusNotFound)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(job)
}
```

```
$ go test ./...
ok      csv-analyzer/api    0.006s
```

### Checkpoint

```
$ git add -A && git commit -m "feat(api): add StatusHandler with an explicit not-found test"
```

---

## Chapter 5 — A `/jobs` Index (Closing the README's Own TODO)

The original README listed "add a `/jobs` endpoint" as a stretch idea. It's not a stretch — without it, nothing can ever *list*
what's been uploaded, which means no dashboard, no history, no way to recover if you lose a job ID. Build it now, while the
`JobStore` interface is still small and easy to extend, rather than as an afterthought.

### Red — Write the Test First

Extend `JobStore` with one method, and extend the fake to match — this is the same "small interface, easy to fake" payoff the
Telemetry Agent guide's Chapter 0 promised.

```go
// jobstore_test.go (addition)
func (f *fakeJobStore) List(limit int) ([]JobStatus, error) {
	start := 0
	if len(f.recent) > limit {
		start = len(f.recent) - limit
	}
	var out []JobStatus
	for _, id := range f.recent[start:] {
		out = append(out, f.jobs[id])
	}
	// most recent first
	for i, j := 0, len(out)-1; i < j; i, j = i+1, j-1 {
		out[i], out[j] = out[j], out[i]
	}
	return out, nil
}
```

```go
// jobs_test.go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHandleJobs_ListsRecentJobsMostRecentFirst(t *testing.T) {
	store := newFakeJobStore()
	for _, id := range []string{"job-1", "job-2", "job-3"} {
		store.Save(JobStatus{JobID: id, Status: "done"})
		store.recent = append(store.recent, id)
	}
	handler := &JobsHandler{store: store, defaultLimit: 10}

	req := httptest.NewRequest(http.MethodGet, "/jobs", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	var jobs []JobStatus
	json.NewDecoder(rec.Body).Decode(&jobs)

	if len(jobs) != 3 {
		t.Fatalf("got %d jobs, want 3", len(jobs))
	}
	if jobs[0].JobID != "job-3" {
		t.Errorf("got most-recent job %q, want %q", jobs[0].JobID, "job-3")
	}
}
```

```
$ go test ./...
./jobstore_test.go:57:14: f.recent undefined (type *fakeJobStore has no field or method recent)
FAIL
```

### Green — Make It Pass

Add `recent []string` to `fakeJobStore`, update `Save`/`Enqueue` to track insertion order, and define the real thing:

```go
// jobstore.go (additions)
type JobStore interface {
	Save(job JobStatus) error
	Get(jobID string) (JobStatus, bool, error)
	Enqueue(jobID string) error
	List(limit int) ([]JobStatus, error) // newest first
}
```

```go
// jobstore.go — RedisJobStore additions
func (s *RedisJobStore) Save(job JobStatus) error {
	data, err := json.Marshal(job)
	if err != nil {
		return err
	}
	pipe := s.client.TxPipeline()
	pipe.Set(s.ctx, "job:"+job.JobID, data, 0)
	pipe.ZAdd(s.ctx, "jobs:index", redis.Z{Score: float64(time.Now().Unix()), Member: job.JobID})
	_, err = pipe.Exec(s.ctx)
	return err
}

func (s *RedisJobStore) List(limit int) ([]JobStatus, error) {
	ids, err := s.client.ZRevRange(s.ctx, "jobs:index", 0, int64(limit-1)).Result()
	if err != nil {
		return nil, err
	}
	jobs := make([]JobStatus, 0, len(ids))
	for _, id := range ids {
		job, found, err := s.Get(id)
		if err != nil {
			return nil, err
		}
		if found {
			jobs = append(jobs, job)
		}
	}
	return jobs, nil
}
```

A Redis **sorted set** (`ZADD`/`ZREVRANGE`), scored by upload time, is the natural fit here — it gives newest-first ordering for
free, without maintaining a separate manually-sorted list. `TxPipeline` batches the two writes (the job hash and the index entry)
into one round trip so they can't drift apart from a partial failure.

```go
// jobs.go
package main

import (
	"encoding/json"
	"net/http"
)

type JobsHandler struct {
	store        JobStore
	defaultLimit int
}

func (h *JobsHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	jobs, err := h.store.List(h.defaultLimit)
	if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(jobs)
}
```

```
$ go test ./...
ok      csv-analyzer/api    0.011s
```

### Checkpoint

```
$ go test ./...
ok      csv-analyzer/api    0.011s
$ git add -A && git commit -m "feat(api): add GET /jobs backed by a Redis sorted set

Closes the README's own 'add a /jobs endpoint' TODO — this is also
the prerequisite for any future dashboard or job history view."
```

---

## Chapter 6 — The Results Handler

`GET /results/{job_id}` returns stats JSON; `GET /results/{job_id}/chart` returns the PNG. Both read from disk, not from
`JobStore` — the worker writes results as files, so this handler needs a small `ResultsDir` it can search, injected the same way
`uploadDir` was in Chapter 3.

### Red — Write the Test First

```go
// results_test.go
package main

import (
	"net/http"
	"net/http/httptest"
	"os"
	"path/filepath"
	"testing"
)

func TestHandleResults_ReturnsStatsJSON(t *testing.T) {
	resultsDir := t.TempDir()
	os.WriteFile(filepath.Join(resultsDir, "abc.json"), []byte(`{"row_count":10}`), 0644)
	handler := &ResultsHandler{resultsDir: resultsDir}

	req := httptest.NewRequest(http.MethodGet, "/results/abc", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("got status %d, want %d", rec.Code, http.StatusOK)
	}
	if rec.Body.String() != `{"row_count":10}` {
		t.Errorf("got body %s", rec.Body.String())
	}
}

func TestHandleResults_MissingStatsReturns404WithHelpfulMessage(t *testing.T) {
	handler := &ResultsHandler{resultsDir: t.TempDir()}

	req := httptest.NewRequest(http.MethodGet, "/results/not-ready-yet", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusNotFound {
		t.Errorf("got status %d, want %d", rec.Code, http.StatusNotFound)
	}
}

func TestHandleResults_ReturnsChartPNG(t *testing.T) {
	resultsDir := t.TempDir()
	os.WriteFile(filepath.Join(resultsDir, "abc.png"), []byte("fake-png-bytes"), 0644)
	handler := &ResultsHandler{resultsDir: resultsDir}

	req := httptest.NewRequest(http.MethodGet, "/results/abc/chart", nil)
	rec := httptest.NewRecorder()
	handler.ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("got status %d, want %d", rec.Code, http.StatusOK)
	}
	if rec.Header().Get("Content-Type") != "image/png" {
		t.Errorf("got Content-Type %s, want image/png", rec.Header().Get("Content-Type"))
	}
}
```

```
$ go test ./...
./results_test.go:12:14: undefined: ResultsHandler
FAIL
```

### Green — Make It Pass

```go
// results.go
package main

import (
	"net/http"
	"os"
	"path/filepath"
	"strings"
)

type ResultsHandler struct {
	resultsDir string
}

func (h *ResultsHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	rest := strings.TrimPrefix(r.URL.Path, "/results/")
	jobID := rest
	wantChart := false
	if strings.HasSuffix(rest, "/chart") {
		jobID = strings.TrimSuffix(rest, "/chart")
		wantChart = true
	}

	if wantChart {
		chartPath := filepath.Join(h.resultsDir, jobID+".png")
		if _, err := os.Stat(chartPath); err != nil {
			http.Error(w, "chart not found (job may still be processing)", http.StatusNotFound)
			return
		}
		w.Header().Set("Content-Type", "image/png")
		http.ServeFile(w, r, chartPath)
		return
	}

	statsPath := filepath.Join(h.resultsDir, jobID+".json")
	data, err := os.ReadFile(statsPath)
	if err != nil {
		http.Error(w, "results not found (job may still be processing)", http.StatusNotFound)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.Write(data)
}
```

```
$ go test ./...
ok      csv-analyzer/api    0.008s
```

### Refactor — `strings.HasSuffix` Instead of `filepath.Base`/`filepath.Dir`

The original handler used `filepath.Base(rest) == "chart"` and `filepath.Dir(rest)` to detect the `/chart` suffix — that works, but
it's solving a string-splitting problem with a path-manipulation API built for a different job, and it's easy to misread. The
`strings.HasSuffix`/`strings.TrimSuffix` pair above says exactly what it means. Test coverage from this chapter makes swapping the
implementation like this a zero-risk change — that's the whole reason the test came first.

### Checkpoint

```
$ git add -A && git commit -m "feat(api): add ResultsHandler for stats JSON and chart PNG

Uses strings.HasSuffix/TrimSuffix instead of filepath.Base/Dir for
clearer intent; covered by tests before and after the refactor."
```

---

## Chapter 7 — Wiring `main.go`

Every handler so far takes its dependencies as struct fields, not globals — `main` is now just the place those get constructed and
handed to a router. Config comes from the environment, same as the Telemetry Agent's `LoadConfig`.

```go
// config.go
package main

import "os"

type Config struct {
	RedisAddr  string
	UploadDir  string
	ResultsDir string
	MaxUploadBytes int64
}

func LoadConfig() Config {
	return Config{
		RedisAddr:      getEnv("REDIS_ADDR", "localhost:6379"),
		UploadDir:      getEnv("UPLOAD_DIR", "/shared/uploads"),
		ResultsDir:     getEnv("RESULTS_DIR", "/shared/results"),
		MaxUploadBytes: 50 << 20, // 50 MiB
	}
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

```go
// main.go
package main

import (
	"log"
	"net/http"
	"os"

	"github.com/redis/go-redis/v9"
)

func main() {
	cfg := LoadConfig()
	os.MkdirAll(cfg.UploadDir, 0755)
	os.MkdirAll(cfg.ResultsDir, 0755)

	client := redis.NewClient(&redis.Options{Addr: cfg.RedisAddr})
	store := NewRedisJobStore(client)

	mux := http.NewServeMux()
	mux.Handle("/upload", &UploadHandler{store: store, uploadDir: cfg.UploadDir, maxBytes: cfg.MaxUploadBytes})
	mux.Handle("/status/", &StatusHandler{store: store})
	mux.Handle("/jobs", &JobsHandler{store: store, defaultLimit: 50})
	mux.Handle("/results/", &ResultsHandler{resultsDir: cfg.ResultsDir})
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("ok"))
	})

	log.Println("api listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

`main` reads the same way the Telemetry Agent's did by the end of its own guide: load config, build the one shared dependency,
hand it to every handler that needs it, start serving. Every decision inside it was already tested in isolation in an earlier
chapter.

### Checkpoint

```
$ go build ./... && go test ./...
ok      csv-analyzer/api    0.015s
$ git add -A && git commit -m "feat(api): wire handlers together in main via config + one shared JobStore"
```

---

## Chapter 8 — Retention: Deleting Old Jobs

Nothing so far ever deletes anything. `uploads/` and `results/` grow forever, and so does `jobs:index`. This is genuinely testable
logic — "is this job older than the retention window" is a pure function of two timestamps — so it gets the same red/green/refactor
treatment as everything else, using the Telemetry Agent guide's `Sleeper`-style trick: inject *time itself* so the test never has
to wait for anything.

### Red — Write the Test First

```go
// retention_test.go
package main

import (
	"testing"
	"time"
)

type fixedClock struct{ now time.Time }

func (c fixedClock) Now() time.Time { return c.now }

func TestRetentionPolicy_KeepsRecentJobsDropsOldOnes(t *testing.T) {
	now := time.Date(2026, 7, 9, 12, 0, 0, 0, time.UTC)
	clock := fixedClock{now: now}
	policy := RetentionPolicy{maxAge: 24 * time.Hour, clock: clock}

	recent := JobStatus{JobID: "recent", Status: "done"}
	old := JobStatus{JobID: "old", Status: "done"}

	store := newFakeJobStore()
	store.Save(recent)
	store.Save(old)
	store.timestamps = map[string]time.Time{
		"recent": now.Add(-1 * time.Hour),
		"old":    now.Add(-48 * time.Hour),
	}

	expired, err := policy.FindExpired(store)
	if err != nil {
		t.Fatalf("did not expect an error, got %v", err)
	}

	if len(expired) != 1 || expired[0] != "old" {
		t.Errorf("got expired %v, want [old]", expired)
	}
}
```

```
$ go test ./...
./retention_test.go:11:14: undefined: RetentionPolicy
FAIL
```

### Green — Make It Pass

```go
// retention.go
package main

import "time"

// Clock abstracts time.Now so retention logic can be tested with a
// fixed instant — the same idiom as the Telemetry Agent's Sleeper.
type Clock interface {
	Now() time.Time
}

type RealClock struct{}

func (RealClock) Now() time.Time { return time.Now() }

type RetentionPolicy struct {
	maxAge time.Duration
	clock  Clock
}

func (p RetentionPolicy) FindExpired(store JobStore) ([]string, error) {
	jobs, err := store.List(10000)
	if err != nil {
		return nil, err
	}
	var expired []string
	cutoff := p.clock.Now().Add(-p.maxAge)
	for _, j := range jobs {
		if j.CreatedAt.Before(cutoff) {
			expired = append(expired, j.JobID)
		}
	}
	return expired, nil
}
```

This introduces a `CreatedAt` field `JobStatus` doesn't have yet — add it back in Chapter 1's struct (`CreatedAt time.Time
\`json:"created_at"\``), set it in `UploadHandler` when a job is first saved, and the fake's `timestamps` map in the test above
becomes real data instead of a test-only shortcut.

```
$ go test ./...
ok      csv-analyzer/api    0.006s
```

### Refactor — Wire It Into a Periodic Sweep

```go
// main.go (addition)
go func() {
	ticker := time.NewTicker(1 * time.Hour)
	defer ticker.Stop()
	policy := RetentionPolicy{maxAge: 7 * 24 * time.Hour, clock: RealClock{}}
	for range ticker.C {
		expired, err := policy.FindExpired(store)
		if err != nil {
			log.Println("retention sweep failed:", err)
			continue
		}
		for _, jobID := range expired {
			os.Remove(filepath.Join(cfg.UploadDir, jobID+".csv"))
			os.Remove(filepath.Join(cfg.ResultsDir, jobID+".json"))
			os.Remove(filepath.Join(cfg.ResultsDir, jobID+".png"))
			store.Delete(jobID) // a fourth JobStore method, same small-interface pattern as List
		}
		log.Printf("retention: removed %d expired jobs", len(expired))
	}
}()
```

Same `time.Ticker` pattern as the Telemetry Agent's collection loop, in a background goroutine `main` never has to wait on — a
reasonable trade-off for a sweep that only needs to run roughly on schedule, not shut down with the same precision as the
request-serving path.

### Checkpoint

```
$ go test ./...
ok      csv-analyzer/api    0.009s
$ git add -A && git commit -m "feat(api): add time-injected RetentionPolicy + hourly cleanup sweep

FindExpired is tested with a fixed clock, never a real timer, same
pattern as the Telemetry Agent's Sleeper interface."
```

---

# Part II — The Python Worker

## Chapter 9 — Cleaning the DataFrame

`clean_dataframe` is pure — no I/O, no Redis, no randomness — which makes it the easiest thing in this whole project to test
thoroughly, and the most valuable to get right, since every downstream stat depends on it.

### Red — Write the Test First

```python
# test_cleaning.py
import pandas as pd
from cleaning import clean_dataframe


def test_drops_fully_empty_rows_and_columns():
    df = pd.DataFrame({
        "a": [1, 2, None],
        "b": [None, None, None],
    })
    df.loc[2] = [None, None]

    cleaned = clean_dataframe(df)

    assert "b" not in cleaned.columns
    assert len(cleaned) == 2


def test_strips_whitespace_from_string_columns_and_headers():
    df = pd.DataFrame({" name ": ["  alice  ", "bob"]})

    cleaned = clean_dataframe(df)

    assert list(cleaned.columns) == ["name"]
    assert cleaned["name"].tolist() == ["alice", "bob"]


def test_drops_exact_duplicate_rows():
    df = pd.DataFrame({"a": [1, 1, 2], "b": [9, 9, 8]})

    cleaned = clean_dataframe(df)

    assert len(cleaned) == 2


def test_coerces_mostly_numeric_string_columns():
    df = pd.DataFrame({"amount": ["10", "20", "30", "not-a-number"]})

    cleaned = clean_dataframe(df)

    assert pd.api.types.is_numeric_dtype(cleaned["amount"])


def test_leaves_genuinely_categorical_columns_alone():
    df = pd.DataFrame({"category": ["red", "blue", "green", "red"]})

    cleaned = clean_dataframe(df)

    assert cleaned["category"].dtype == object
```

```
$ pytest -v
ModuleNotFoundError: No module named 'cleaning'
```

### Green — Make It Pass

```python
# cleaning.py
import pandas as pd


def clean_dataframe(df: pd.DataFrame) -> pd.DataFrame:
    df = df.dropna(axis=0, how="all").dropna(axis=1, how="all")

    df.columns = [str(c).strip() for c in df.columns]
    for col in df.select_dtypes(include="object").columns:
        df[col] = df[col].astype(str).str.strip()

    df = df.drop_duplicates()

    for col in df.columns:
        if df[col].dtype == object:
            coerced = pd.to_numeric(df[col], errors="coerce")
            if coerced.notna().mean() > 0.9:
                df[col] = coerced

    return df
```

```
$ pytest -v
5 passed
```

### Refactor — Name the Magic Number

`0.9` (the "mostly numeric" threshold) is a real design decision hiding in a bare literal. Naming it documents the choice and makes
it a single place to tune later:

```python
# cleaning.py
NUMERIC_COERCION_THRESHOLD = 0.9  # a column converts to numeric if ≥90% of values parse as numbers


def clean_dataframe(df: pd.DataFrame) -> pd.DataFrame:
    df = df.dropna(axis=0, how="all").dropna(axis=1, how="all")

    df.columns = [str(c).strip() for c in df.columns]
    for col in df.select_dtypes(include="object").columns:
        df[col] = df[col].astype(str).str.strip()

    df = df.drop_duplicates()

    for col in df.columns:
        if df[col].dtype == object:
            coerced = pd.to_numeric(df[col], errors="coerce")
            if coerced.notna().mean() > NUMERIC_COERCION_THRESHOLD:
                df[col] = coerced

    return df
```

```
$ pytest -v
5 passed
```

### Checkpoint

```
$ git add -A && git commit -m "feat(worker): add clean_dataframe with a full pytest suite

Pure function, no I/O — five tests cover empty-row/column dropping,
whitespace stripping, dedup, and the numeric-coercion threshold."
```

---

## Chapter 10 — Computing Stats

Same shape as Chapter 9 — pure function, deterministic output, straightforward to test with small hand-built DataFrames.

### Red — Write the Test First

```python
# test_stats.py
import pandas as pd
from stats import compute_stats


def test_reports_row_and_column_counts():
    df = pd.DataFrame({"a": [1, 2, 3], "b": ["x", "y", "z"]})

    stats = compute_stats(df)

    assert stats["row_count"] == 3
    assert stats["column_count"] == 2


def test_reports_null_counts_per_column():
    df = pd.DataFrame({"a": [1, None, 3]})

    stats = compute_stats(df)

    assert stats["null_counts"]["a"] == 1


def test_numeric_summary_is_empty_when_no_numeric_columns():
    df = pd.DataFrame({"a": ["x", "y", "z"]})

    stats = compute_stats(df)

    assert stats["numeric_summary"] == {}


def test_numeric_summary_includes_mean_for_numeric_columns():
    df = pd.DataFrame({"a": [10.0, 20.0, 30.0]})

    stats = compute_stats(df)

    assert stats["numeric_summary"]["a"]["mean"] == 20.0
```

```
$ pytest -v
ModuleNotFoundError: No module named 'stats'
```

### Green — Make It Pass

```python
# stats.py
import json
import pandas as pd


def compute_stats(df: pd.DataFrame) -> dict:
    numeric_df = df.select_dtypes(include="number")
    return {
        "row_count": int(len(df)),
        "column_count": int(len(df.columns)),
        "columns": list(df.columns),
        "dtypes": {c: str(t) for c, t in df.dtypes.items()},
        "null_counts": {c: int(n) for c, n in df.isnull().sum().items()},
        "numeric_summary": json.loads(numeric_df.describe().to_json()) if not numeric_df.empty else {},
    }
```

```
$ pytest -v
9 passed
```

### Checkpoint

```
$ git add -A && git commit -m "feat(worker): add compute_stats with a pytest suite covering the empty case"
```

---

## Chapter 11 — The Chart: Written Directly, Not TDD'd

Same honest exception as the Telemetry Agent's Dockerfile and its raw `/proc` read: "does this histogram look right" isn't a
question `assert` can answer. There's no meaningful red step here — write it, run it, look at the PNG.

```python
# chart.py
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import pandas as pd


def make_chart(df: pd.DataFrame, out_path: str, column: str | None = None):
    numeric_df = df.select_dtypes(include="number")
    fig, ax = plt.subplots(figsize=(8, 5))

    if column and column in df.columns:
        target_col, is_numeric = column, column in numeric_df.columns
    elif not numeric_df.empty:
        target_col, is_numeric = numeric_df.columns[0], True
    else:
        target_col, is_numeric = df.columns[0], False

    if is_numeric:
        df[target_col].dropna().plot(kind="hist", bins=20, ax=ax, color="#4C72B0")
        ax.set_title(f"Distribution of '{target_col}'")
    else:
        df[target_col].value_counts().head(15).plot(kind="bar", ax=ax, color="#4C72B0")
        ax.set_title(f"Top values in '{target_col}'")

    ax.set_xlabel(target_col)
    fig.tight_layout()
    fig.savefig(out_path)
    plt.close(fig)
```

```
$ python -c "
import pandas as pd
from chart import make_chart
make_chart(pd.DataFrame({'age': [23,45,31,29,52,38]}), '/tmp/test.png')
"
$ open /tmp/test.png   # eyeball it
```

**What *is* testable here, and worth a quick check:** that a chart file gets written at all, and that passing an unknown `column`
falls back sanely instead of raising `KeyError`. Two lightweight tests, not a pixel-comparison test:

```python
# test_chart.py
import os
import pandas as pd
from chart import make_chart


def test_writes_a_file(tmp_path):
    df = pd.DataFrame({"age": [23, 45, 31]})
    out = tmp_path / "chart.png"

    make_chart(df, str(out))

    assert out.exists()
    assert out.stat().st_size > 0


def test_unknown_column_falls_back_without_raising(tmp_path):
    df = pd.DataFrame({"age": [23, 45, 31]})
    out = tmp_path / "chart.png"

    make_chart(df, str(out), column="does_not_exist")  # should not raise

    assert out.exists()
```

```
$ pytest -v
11 passed
```

This is also where the earlier review's "let the user pick a column" idea lands — `make_chart`'s `column` parameter — wired up to
an actual query param in Chapter 13.

### Checkpoint

```
$ git add -A && git commit -m "feat(worker): add make_chart with an optional column param

Not TDD'd in the usual sense — visual output isn't assertable — but
covered by two smoke tests: a file gets written, an unknown column
falls back instead of raising."
```

---

## Chapter 12 — The Job Loop, Tested Without a Real Redis

`process_job` and the `BRPOP` loop are where Redis and the filesystem meet the pure functions from Chapters 9–11. `fakeredis`
gives an in-memory Redis-API-compatible client, so this gets tested the same way the Telemetry Agent's Gemini call was tested
without ever touching the network — swap the client, keep the code identical to production.

### Red — Write the Test First

```python
# test_process_job.py
import json
import os
import pandas as pd
import fakeredis
from worker import process_job, set_status


def test_process_job_writes_stats_and_chart_and_marks_done(tmp_path):
    upload_dir = tmp_path / "uploads"
    results_dir = tmp_path / "results"
    upload_dir.mkdir()
    results_dir.mkdir()

    df = pd.DataFrame({"amount": [10, 20, 30, 40]})
    df.to_csv(upload_dir / "job-1.csv", index=False)

    r = fakeredis.FakeRedis(decode_responses=True)

    process_job("job-1", redis_client=r, upload_dir=str(upload_dir), results_dir=str(results_dir))

    assert (results_dir / "job-1.json").exists()
    assert (results_dir / "job-1.png").exists()

    status = json.loads(r.get("job:job-1"))
    assert status["status"] == "done"


def test_process_job_marks_error_on_bad_csv(tmp_path):
    upload_dir = tmp_path / "uploads"
    results_dir = tmp_path / "results"
    upload_dir.mkdir()
    results_dir.mkdir()
    (upload_dir / "job-2.csv").write_bytes(b"\x00\x01not,a,valid,csv\xff")

    r = fakeredis.FakeRedis(decode_responses=True)

    process_job("job-2", redis_client=r, upload_dir=str(upload_dir), results_dir=str(results_dir))

    status = json.loads(r.get("job:job-2"))
    assert status["status"] == "error"
    assert "error" in status
```

```
$ pytest -v
ImportError: cannot import name 'process_job' from 'worker' (function currently takes no redis_client param)
```

### Green — Make It Pass

```python
# worker.py
import json
import os
import time
import traceback

import redis
from cleaning import clean_dataframe
from stats import compute_stats
from chart import make_chart
import pandas as pd


def set_status(redis_client, job_id: str, status: str, error: str = ""):
    payload = {"job_id": job_id, "status": status}
    if error:
        payload["error"] = error
    redis_client.set(f"job:{job_id}", json.dumps(payload))


def process_job(job_id: str, redis_client, upload_dir: str, results_dir: str):
    set_status(redis_client, job_id, "processing")
    csv_path = os.path.join(upload_dir, f"{job_id}.csv")

    try:
        df = pd.read_csv(csv_path)
        df = clean_dataframe(df)

        stats = compute_stats(df)
        with open(os.path.join(results_dir, f"{job_id}.json"), "w") as f:
            json.dump(stats, f, indent=2)

        column = redis_client.get(f"job:{job_id}:column")  # set by the API, see Chapter 13
        make_chart(df, os.path.join(results_dir, f"{job_id}.png"), column=column)

        set_status(redis_client, job_id, "done")
        print(f"[worker] finished job {job_id}")
    except Exception as e:
        traceback.print_exc()
        set_status(redis_client, job_id, "error", str(e))


def main():
    redis_addr = os.environ.get("REDIS_ADDR", "localhost")
    redis_port = int(os.environ.get("REDIS_PORT", 6379))
    upload_dir = os.environ.get("UPLOAD_DIR", "/shared/uploads")
    results_dir = os.environ.get("RESULTS_DIR", "/shared/results")

    r = redis.Redis(host=redis_addr, port=redis_port, decode_responses=True)
    os.makedirs(results_dir, exist_ok=True)

    print("[worker] waiting for jobs...")
    while True:
        item = r.brpop("jobs:queue", timeout=5)
        if item is None:
            continue
        _, job_id = item
        print(f"[worker] picked up job {job_id}")
        process_job(job_id, redis_client=r, upload_dir=upload_dir, results_dir=results_dir)


if __name__ == "__main__":
    main()
```

```
$ pytest -v
13 passed
```

`process_job` now takes `redis_client` as a parameter instead of importing a module-level connection — the same dependency
injection idiom as every Go chapter in this guide, expressed as a plain function argument rather than a constructor, since Python
doesn't need an interface keyword for structural typing to work.

**The error path is genuinely load-bearing here.** A malformed CSV, a permissions error, a `pandas` edge case — all get caught,
logged with a full traceback for the operator, and turned into an `error` status the API can report, instead of crashing the whole
worker loop and silently leaving every job after it stuck at `processing` forever. This is the exact failure mode the original
`except Exception` in the job loop was already guarding against — moving it inside `process_job` just makes it directly testable.

### Checkpoint

```
$ pytest -v
13 passed
$ git add -A && git commit -m "feat(worker): tie clean/stats/chart together in process_job

redis_client is a parameter, not a module-level global — tested
entirely with fakeredis, including the error path on a bad CSV."
```

---

## Chapter 13 — Letting the User Pick a Chart Column, End to End

Chapter 11 gave `make_chart` a `column` parameter. Wiring a real query param to it touches both sides of the system, so it's a
good final example of how a small feature moves through this stack — and it's TDD'd on the Go side, where the branching logic is.

### Red — Write the Test First (Go side)

```go
// upload_test.go (addition)
func TestHandleUpload_StoresRequestedColumn(t *testing.T) {
	store := newFakeJobStore()
	handler := &UploadHandler{store: store, uploadDir: t.TempDir(), maxBytes: 10 << 20}

	body, contentType := newMultipartUpload(t, "sales.csv", "amount,region\n10,west\n")
	req := httptest.NewRequest(http.MethodPost, "/upload?column=amount", body)
	req.Header.Set("Content-Type", contentType)
	rec := httptest.NewRecorder()

	handler.ServeHTTP(rec, req)

	var job JobStatus
	json.NewDecoder(rec.Body).Decode(&job)

	if store.columns[job.JobID] != "amount" {
		t.Errorf("got stored column %q, want %q", store.columns[job.JobID], "amount")
	}
}
```

```
$ go test ./...
./upload_test.go:...: fakeJobStore has no field columns
FAIL
```

### Green — Make It Pass

```go
// jobstore.go — one more small JobStore method, same pattern as List/Delete
type JobStore interface {
	Save(job JobStatus) error
	Get(jobID string) (JobStatus, bool, error)
	Enqueue(jobID string) error
	List(limit int) ([]JobStatus, error)
	SaveColumn(jobID, column string) error
}

func (s *RedisJobStore) SaveColumn(jobID, column string) error {
	if column == "" {
		return nil
	}
	return s.client.Set(s.ctx, "job:"+jobID+":column", column, 0).Err()
}
```

```go
// upload.go (addition)
column := r.URL.Query().Get("column")
if err := h.store.SaveColumn(jobID, column); err != nil {
	http.Error(w, "failed to save column preference", http.StatusInternalServerError)
	return
}
```

```
$ go test ./...
ok      csv-analyzer/api    0.012s
```

The worker's `process_job` from Chapter 12 already reads `job:{job_id}:column` and passes it straight to `make_chart` — the two
sides meet at that one Redis key, with each side's own test suite covering its half independently. Nothing needed to change in
`worker.py` at all; that's the payoff of designing `make_chart`'s signature with this in mind back in Chapter 11.

### Checkpoint

```
$ go test ./... && (cd ../worker && pytest -v)
ok      csv-analyzer/api    0.012s
13 passed
$ git add -A && git commit -m "feat: let the client choose the chart column via ?column=

Go side stores the preference; Python side already knew how to use
it, from Chapter 11's make_chart signature."
```

---

# Part III — Docker and a Web Dashboard

## Chapter 14 — Hardening docker-compose.yml

Two concrete fixes from the earlier review, applied to the compose file directly — no code changes needed for either.

### Fix 1: Redis Gets a Persistent Volume

```yaml
services:
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    expose:
      - "6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  redis-data:
```

`--appendonly yes` turns on Redis's append-only-file persistence, and the named volume survives `docker compose down` (though not
`docker compose down -v`) — a job's status now outlives a container restart, matching the fact that its CSV, stats, and chart
files already did.

### Fix 2: Redis Is No Longer Published to the Host

```diff
   redis:
     image: redis:7-alpine
-    ports:
-      - "6379:6379"
+    expose:
+      - "6379"
```

`expose` documents the port to other containers on the `csv-analyzer` network without publishing it to your host machine — the
exact same fix, and the exact same class of mistake, as the Telemetry Agent guide's Chapter 13 exercise (an unauthenticated Redis
instance reachable from your host's whole network is a real, easy-to-introduce misconfiguration, not a hypothetical one).

### Checkpoint

```
$ docker compose up --build
$ docker compose down && docker compose up -d   # job history should still be there
$ redis-cli -h localhost -p 6379 ping   # from your host — should now fail to even connect
$ git add -A && git commit -m "chore(compose): persist Redis data, stop publishing it to the host"
```

---

## Chapter 15 — A Minimal Web Dashboard

Everything so far has one interface: `curl`. For a tool whose entire output is a table of stats and a PNG, that's the biggest gap
between what it does and how it feels to use. This doesn't need React — one Go route serving one HTML page, with
[htmx](https://htmx.org/) doing the polling, gets the whole loop working with no build step and almost no JavaScript.

This chapter is written directly rather than TDD'd, same honest exception as Chapter 11's chart — there's no meaningful assertion
for "does this page look right," though the route itself (does `GET /` return `200` and the right content type) is worth a
one-line test if you want it, using the same `httptest.NewRecorder` pattern as every other handler in this guide.

### Write It

```go
// dashboard.go
package main

import (
	"html/template"
	"net/http"
)

const dashboardHTML = `<!DOCTYPE html>
<html>
<head>
  <title>CSV Analyzer</title>
  <script src="https://unpkg.com/htmx.org@1.9.12"></script>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 720px; margin: 3rem auto; color: #222; }
    .job { border: 1px solid #ddd; border-radius: 8px; padding: 1rem; margin-top: 1rem; }
    .status-done { color: #2a8a4a; } .status-error { color: #c0392b; } .status-pending, .status-processing { color: #8a6d2a; }
    img { max-width: 100%; border-radius: 4px; margin-top: 0.5rem; }
    table { width: 100%; border-collapse: collapse; margin-top: 0.5rem; }
    td, th { text-align: left; padding: 4px 8px; border-bottom: 1px solid #eee; font-size: 0.9rem; }
  </style>
</head>
<body>
  <h1>CSV Analyzer</h1>

  <form hx-post="/upload" hx-target="#result" hx-encoding="multipart/form-data">
    <input type="file" name="file" accept=".csv" required>
    <button type="submit">Upload</button>
  </form>

  <div id="result"></div>

  <div id="job-poll"
       hx-get=""
       hx-trigger="load"
       hx-swap="none">
  </div>

  <h2>Recent jobs</h2>
  <div hx-get="/dashboard/jobs" hx-trigger="load, every 3s" hx-swap="innerHTML"></div>
</body>
</html>`

func (a *App) HandleDashboard(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/html")
	w.Write([]byte(dashboardHTML))
}

var jobsFragmentTmpl = template.Must(template.New("jobs").Parse(`
{{range .}}
<div class="job">
  <strong>{{.JobID}}</strong>
  — <span class="status-{{.Status}}">{{.Status}}</span>
  {{if eq .Status "done"}}
    <br><a href="/results/{{.JobID}}/chart" target="_blank">
      <img src="/results/{{.JobID}}/chart" alt="chart for {{.JobID}}">
    </a>
  {{else if eq .Status "error"}}
    <br><small>{{.Error}}</small>
  {{end}}
</div>
{{else}}
<p>No jobs yet — upload a CSV above.</p>
{{end}}
`))

func (a *App) HandleDashboardJobs(w http.ResponseWriter, r *http.Request) {
	jobs, err := a.store.List(20)
	if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "text/html")
	jobsFragmentTmpl.Execute(w, jobs)
}
```

```go
// main.go (additions)
app := &App{store: store}
mux.HandleFunc("/", app.HandleDashboard)
mux.HandleFunc("/dashboard/jobs", app.HandleDashboardJobs)
```

`hx-trigger="load, every 3s"` is the entire polling mechanism — no hand-written `fetch`/`setInterval`, no JSON parsing on the
client, no separate frontend build. The server renders HTML fragments directly from the same `JobStore` every other handler in
this guide already depends on; the browser just asks for a fresh fragment every three seconds and swaps it in.

### Checkpoint

```
$ docker compose up --build
$ open http://localhost:8080/
```

Drag a CSV onto the page, watch the job list pick it up, watch its status flip to `done`, watch the chart appear inline — the same
loop the README's four separate `curl` commands walked through, now visible in a browser with zero commands after the first.

```
$ git add -A && git commit -m "feat(api): add a minimal htmx dashboard at GET /

No build step, no JS framework — server-rendered HTML fragments
polled every 3s, backed by the same JobStore every handler uses."
```

---

## Wrapping Up

You now have a CSV Analyzer that's genuinely end-to-end, not just a curl demo: upload a file, watch it get cleaned and summarized,
see a chart, browse job history, all through a page that updates itself — backed by a test suite on both sides that never touches
a real Redis instance or the real filesystem beyond `t.TempDir()`/`tmp_path`. Concretely, you built:

- **`JobStore` as an interface from Chapter 2 onward** — every handler depends on a two-method-plus contract, never on
  `*redis.Client` directly, so the whole Go test suite runs without Redis ever being started.
- **`fakeredis` on the Python side** doing the same job for `process_job` — production and test code paths are identical except
  for which client gets passed in.
- **Pure, thoroughly-tested cleaning and stats functions**, deliberately separated from the one genuinely untestable piece (the
  chart image itself), which got two honest smoke tests instead of a fake assertion pretending to verify something it can't.
- **A time-injected `RetentionPolicy`**, tested with a fixed clock, never a real timer — the same trick as the Telemetry Agent's
  `Sleeper`.
- **Two real security/ops fixes** (Redis persistence, Redis no longer published to the host) applied directly to
  `docker-compose.yml`, each tied to a concrete before/after check rather than taken on faith.
- **A feature that touches both languages** (column selection) built end to end in one chapter, each side tested independently,
  meeting at exactly one Redis key.
- **A dashboard with no frontend build step**, server-rendered and polling, that turns "run four curl commands in order" into
  "open a browser tab."

### What's Still Genuinely Missing, Written Down Rather Than Hidden

- No auth — anyone who can reach `:8080` can upload, browse, and download every job. Fine on a private network, not fine exposed
  publicly, same caveat the original README already named.
- No rate limiting on `/upload` — nothing stops one client from queuing thousands of jobs and starving everyone else's.
- The worker processes one job at a time — a burst of uploads queues up serially. A worker pool (same bounded-concurrency pattern
  as the Telemetry Agent's Chapter 9) is the natural next step if throughput ever matters.
- Chart selection is single-column; multi-chart output (a correlation heatmap, categorical breakdowns) is still just an idea in
  the README, not a chapter.

That list isn't a defect in this guide — it's the honest table of contents for whatever you build on top of it next.
