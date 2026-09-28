# Week 20: Hands-On Lab Exercise

## The Performance Hunt Lab
You are given a web server written in Go (or Rust) that processes incoming telemetry payloads. It is currently failing performance SLA (target: 10,000 req/sec at < 50ms p99 latency). Your job is to use a load generator (k6), capture a profile, find the three bottlenecks, fix them, and prove it.

---

### Step 1: The Load Test Baseline (k6)
First, we establish our baseline metrics. 
1. Install [k6](https://k6.io/).
2. Start the naive Go telemetry server: `go run main.go`
3. Run this `loadtest.js` script:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 200 }, // Ramp up to 200 virtual users
    { duration: '1m', target: 200 },  // Hold at 200
    { duration: '10s', target: 0 },   // Ramp down
  ],
};

const payload = JSON.stringify({
  device_id: "dev-99234",
  metric: "cpu_temp",
  value: 45.2,
  timestamp: new Date().toISOString(),
});

export default function () {
  const params = { headers: { 'Content-Type': 'application/json' } };
  const res = http.post('http://localhost:8080/ingest', payload, params);
  
  check(res, {
    'is status 200': (r) => r.status === 200,
  });
}
```
Run it: `k6 run loadtest.js`. Note the `http_req_duration` (p95 and p99).

---

### Step 2: Profiling Instructions

While k6 is running the 1-minute hold phase, you must capture the profile.

**If fixing the Go server:**
```bash
# Capture a 30-second CPU profile
go tool pprof -http=:8081 http://localhost:8080/debug/pprof/profile?seconds=30

# Capture a heap (memory allocation) profile
go tool pprof -http=:8082 http://localhost:8080/debug/pprof/heap
```

**If fixing the Rust server:**
```bash
# Assuming the server is running natively on Linux
cargo flamegraph --bin telemetry_server
# Open the resulting flamegraph.svg in your browser
```

---

### Step 3: Find and Fix the Three Bottlenecks

Look closely at the Flame Graph (or pprof web UI). You are hunting for three specific anti-patterns deliberately hidden in the codebase:

1. **The Hash Collision / Bad Key:** The code is validating the `device_id` by loading a massive JSON file from disk *on every single request*. 
   * *Fix:* Load it into memory once at startup using a `sync.Map` (Go) or `Arc<HashMap>` (Rust).
2. **The Logger Lock:** The code uses a global standard logger writing to a file, causing severe Mutex lock contention (you will see `sync.Mutex.Lock` or `pthread_mutex_lock` extremely wide on the flame graph).
   * *Fix:* Switch to an asynchronous logger or disable debug logging in the hot path.
3. **The Unbounded Allocation:** The JSON deserialization is unmarshaling into a generic `map[string]interface{}` (Go) or `serde_json::Value` (Rust), trashing the heap and causing GC thrashing (you will see `runtime.gcBgMarkWorker` taking CPU time).
   * *Fix:* Bind the JSON to a strictly defined struct (`struct TelemetryPayload { ... }`).

---

### Step 4: Verification

Re-run the exact same k6 script. Fill out this table in your team channel:

| Metric | Before | After |
| :--- | :--- | :--- |
| **Requests per Second** | e.g., 400 | e.g., 12,000 |
| **p95 Latency** | e.g., 450ms | e.g., 8ms |
| **p99 Latency** | e.g., 800ms | e.g., 15ms |
| **CPU Utilization** | e.g., 100% | e.g., 30% |

---

### Friday: Mob Review & Discussion

Project the pprof web UI and the Flame Graphs on the screen.

**Discussion Questions:**
1. What was the visual signature of the I/O bottleneck (reading the file on every request) on the flame graph? Did it show up as CPU time, or did you have to look at lock/wait time?
2. When we fixed the JSON allocation issue, how did it affect the GC pause times? (Hint: Check `GODEBUG=gctrace=1` output).
3. (For Rust teams) How did the `Arc<HashMap>` perform compared to a `Mutex<HashMap>` for the device dictionary? Why?
4. How would you tune `GOMEMLIMIT` on this Go service if it was being deployed to a Kubernetes pod with a 256MiB limit?

**Sign-off Checklist (Each member must answer):**
1. [ ] Can you read the difference between "Flat" (Exclusive) and "Cum" (Inclusive) time in pprof/PerfView?
2. [ ] Do you understand why reading from disk per-request ruins web server throughput?
3. [ ] Can you identify a locking contention issue on a Flame Graph?
4. [ ] Have you successfully run a k6 load test and understood the p95 and p99 metrics?
5. [ ] Can you explain what `GOMEMLIMIT` does in Go 1.19+?
6. [ ] Can you explain why decoding generic JSON heavily burdens the garbage collector/heap allocator?
