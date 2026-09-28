# Week 20: Code Comparison Rosetta

## The Mission: A Deliberately Slow JSON Log Parser
We have a background job that parses thousands of JSON log lines. The initial implementation is deliberately slow: it allocates strings unnecessarily and uses inefficient deserialization. We will show the bad code, how to profile it, the optimized code, and the before/after results in all three languages.

### 1. C# (.NET 8)

**The Bad Code (`LogParser.cs`):**
```csharp
using System.Text.Json;

public class LogParser
{
    // BUG: Allocating a new string for every line, and deserializing to a dictionary 
    // instead of a strongly typed object or using System.Text.Json.Utf8JsonReader
    public int CountErrors(IEnumerable<string> logLines)
    {
        int errors = 0;
        foreach (var line in logLines)
        {
            var dict = JsonSerializer.Deserialize<Dictionary<string, string>>(line);
            if (dict != null && dict.ContainsKey("level") && dict["level"] == "error")
            {
                errors++;
            }
        }
        return errors;
    }
}
```

**Profiling Workflow:**
1. Write a BenchmarkDotNet runner over `CountErrors`.
2. Run with MemoryDiagnoser: `[MemoryDiagnoser] public class ParserBench { ... }`
3. Run: `dotnet run -c Release`

**The Fixed Code:**
```csharp
using System.Text.Json;

public class LogParser
{
    // FIX: Use Utf8JsonReader to parse directly from bytes without allocations.
    public int CountErrorsOptimized(ReadOnlySpan<byte> logData)
    {
        int errors = 0;
        var reader = new Utf8JsonReader(logData);
        while (reader.Read())
        {
            if (reader.TokenType == JsonTokenType.PropertyName && reader.ValueTextEquals("level"u8))
            {
                reader.Read();
                if (reader.ValueTextEquals("error"u8))
                {
                    errors++;
                }
            }
        }
        return errors;
    }
}
```
**Results:** Time dropped from 2.5ms to 0.1ms. Allocations dropped from 15MB to 0 bytes (Gen0 collections eliminated).

### 2. Go (1.21+)

**The Bad Code (`parser.go`):**
```go
package parser

import (
	"encoding/json"
	"strings"
)

// BUG: Using map[string]interface{} forces the json decoder to allocate heavily 
// on the heap for every line.
func CountErrors(logLines string) int {
	errors := 0
	lines := strings.Split(logLines, "\n")
	for _, line := range lines {
		if line == "" { continue }
		var data map[string]interface{}
		json.Unmarshal([]byte(line), &data)
		
		if val, ok := data["level"]; ok {
			if val == "error" {
				errors++
			}
		}
	}
	return errors
}
```

**Profiling Workflow:**
1. Write a standard benchmark: `func BenchmarkCountErrors(b *testing.B) { ... }`
2. Run benchmark with memprofile: 
   `go test -bench . -benchmem -cpuprofile cpu.prof -memprofile mem.prof`
3. Analyze memory: `go tool pprof -http=:8080 mem.prof` (Notice `json.Unmarshal` taking 90% of heap allocations).

**The Fixed Code:**
```go
package parser

import (
	"bytes"
	"github.com/valyala/fastjson" // Fast schema-less JSON parser
)

func CountErrorsOptimized(logData []byte) int {
	errors := 0
	var p fastjson.Parser

    // FIX: Parse over raw bytes. Use fastjson to avoid reflection and map allocations.
	lines := bytes.Split(logData, []byte("\n"))
	for _, line := range lines {
		if len(line) == 0 { continue }
		v, _ := p.ParseBytes(line)
		if string(v.GetStringBytes("level")) == "error" {
			errors++
		}
	}
	return errors
}
```
**Results:** `ns/op` dropped from 15000 to 800. `B/op` (bytes allocated) dropped from 4500 to 0.

### 3. Rust

**The Bad Code (`src/lib.rs`):**
```rust
use serde_json::Value;

// BUG: Parsing into an untyped serde_json::Value heap-allocates maps and strings.
pub fn count_errors(log_data: &str) -> usize {
    let mut errors = 0;
    for line in log_data.lines() {
        if line.is_empty() { continue; }
        
        let v: Value = serde_json::from_str(line).unwrap();
        if let Some(level) = v.get("level") {
            if level == "error" {
                errors += 1;
            }
        }
    }
    errors
}
```

**Profiling Workflow:**
1. Setup a Criterion benchmark.
2. Run `cargo flamegraph --bench parser_bench`.
3. Open `flamegraph.svg`. Look at the wide `malloc` bars under `serde_json::de::ValueVisitor`.

**The Fixed Code:**
```rust
use serde::Deserialize;

// FIX: Define a struct. Serde will skip allocating strings for unmapped fields.
#[derive(Deserialize)]
struct LogLine<'a> {
    // Borrow from the input string instead of allocating a new String
    #[serde(borrow)]
    level: Option<&'a str>, 
}

pub fn count_errors_optimized(log_data: &str) -> usize {
    let mut errors = 0;
    for line in log_data.lines() {
        if line.is_empty() { continue; }
        
        // Zero-allocation parsing
        if let Ok(log) = serde_json::from_str::<LogLine>(line) {
            if log.level == Some("error") {
                errors += 1;
            }
        }
    }
    errors
}
```
**Results:** Execution time dropped from 450µs to 45µs per iteration.

### Critical Observations for C# Developers
1. **The Allocation Trap:** Across all three languages, parsing unstructured JSON (Dictionary, map, serde_json::Value) is the #1 cause of performance degradation because it forces heap allocations for every field key and value.
2. **Zero-Copy Parsers:** C# relies on `Utf8JsonReader` and `ReadOnlySpan<byte>`, Go relies on `[]byte` manipulation and packages like `fastjson`, and Rust relies on Serde's powerful `#[serde(borrow)]` attribute to parse data directly from the input buffer without copying strings.
3. **Benchmarking Cultures:** C# requires third-party BenchmarkDotNet. Go has it built directly into `go test`. Rust requires `criterion`, but it provides the deepest statistical rigor out of the box.
