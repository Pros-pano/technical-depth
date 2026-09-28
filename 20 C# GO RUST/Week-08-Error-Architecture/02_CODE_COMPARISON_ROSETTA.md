# Week 08: Code Comparison - Configuration Loader

## The Scenario
We must build a robust `ConfigLoader`. It must read a TOML/YAML file from disk, parse required fields (`database_url`, `port`, `max_connections`), validate those fields (`port` 1-65535, `max_connections` > 0), check file permissions (warn if world-readable), and return a rich structured error chain that reads like a sentence, telling the user EXACTLY what failed and where.

## 1. C# Implementation: The Exception Hierarchy

In C#, we build custom exception classes and nest them in the `InnerException` property. The caller uses `try/catch` blocks.

```csharp
using System;
using System.IO;
using System.Text.Json; // Simulating YAML/TOML parser for brevity

public class ConfigException : Exception {
    public ConfigException(string message, Exception inner = null) : base(message, inner) {}
}
public class ConfigValidationException : ConfigException {
    public ConfigValidationException(string field, string reason) 
        : base($"Validation failed for '{field}': {reason}") {}
}

public class Config {
    public string DatabaseUrl { get; set; }
    public int Port { get; set; }
    public int MaxConnections { get; set; }
}

public class ConfigLoader {
    public Config Load(string path) {
        try {
            // 1. File Permissions (Simulated)
            var fileInfo = new FileInfo(path);
            if (!fileInfo.Exists) {
                throw new FileNotFoundException($"The configuration file was not found at {path}.");
            }
            
            // 2. Read and Parse
            string content = File.ReadAllText(path);
            Config config = null;
            try {
                config = JsonSerializer.Deserialize<Config>(content);
            } catch (JsonException ex) {
                throw new ConfigException("The configuration file contains invalid syntax.", ex);
            }

            // 3. Validation
            if (config.Port < 1 || config.Port > 65535) {
                throw new ConfigValidationException(nameof(config.Port), "Must be between 1 and 65535.");
            }
            if (config.MaxConnections <= 0) {
                throw new ConfigValidationException(nameof(config.MaxConnections), "Must be strictly positive.");
            }

            return config;

        } catch (ConfigException) {
            throw; // Rethrow our custom exceptions as-is
        } catch (Exception ex) {
            // Catch-all for IO exceptions, unauthorized access, etc.
            throw new ConfigException($"Failed to load configuration from {path}", ex);
        }
    }
}
```

## 2. Go Implementation: Explicit Value Unwrapping

Go wraps errors explicitly using `fmt.Errorf` and `%w`. We use `errors.Is` or `errors.As` at the top level to inspect the chain.

```go
package main

import (
    "encoding/json" // Simulating YAML/TOML parser
    "errors"
    "fmt"
    "os"
)

// Sentinel error for validation
var ErrValidation = errors.New("configuration validation failed")

type Config struct {
    DatabaseUrl    string `json:"database_url"`
    Port           int    `json:"port"`
    MaxConnections int    `json:"max_connections"`
}

func LoadConfig(path string) (*Config, error) {
    // 1. Read and Permissions
    info, err := os.Stat(path)
    if err != nil {
        if os.IsNotExist(err) {
            return nil, fmt.Errorf("configuration file not found at %s: %w", path, err)
        }
        return nil, fmt.Errorf("failed to access configuration file: %w", err)
    }

    if info.Mode().Perm() & 0044 != 0 {
        fmt.Println("WARN: Configuration file is world-readable!")
    }

    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("failed to read configuration file: %w", err)
    }

    // 2. Parse
    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("the configuration file contains invalid syntax: %w", err)
    }

    // 3. Validation
    if config.Port < 1 || config.Port > 65535 {
        return nil, fmt.Errorf("%w: field 'port' must be between 1 and 65535", ErrValidation)
    }
    if config.MaxConnections <= 0 {
        return nil, fmt.Errorf("%w: field 'max_connections' must be strictly positive", ErrValidation)
    }

    return &config, nil
}
```

## 3. Rust Implementation: `thiserror` + `anyhow`

Rust provides the most rigorous and performant error architecture. We use `thiserror` to define the exact failure taxonomy, and `anyhow` (optional here, but standard practice) in the caller to attach context. Notice how clean the `?` operator makes the happy path.

```rust
use std::fs;
use std::os::unix::fs::PermissionsExt; // For permission checking
use thiserror::Error;
use serde::Deserialize;

// 1. Define the Error Taxonomy
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("Failed to read configuration file at {path}")]
    Io {
        path: String,
        #[source]
        source: std::io::Error,
    },
    #[error("The configuration file contains invalid syntax")]
    Parse(#[from] serde_json::Error), // Automatically converts JSON errors via `?`
    
    #[error("Validation failed for '{field}': {reason}")]
    Validation {
        field: String,
        reason: String,
    },
}

#[derive(Deserialize, Debug)]
pub struct Config {
    database_url: String,
    port: u16,           // Type system automatically enforces 0-65535!
    max_connections: i32,
}

// 2. The Loader Logic
pub fn load_config(path: &str) -> Result<Config, ConfigError> {
    // Permissions check
    let metadata = fs::metadata(path).map_err(|source| ConfigError::Io {
        path: path.to_string(),
        source,
    })?;

    if metadata.permissions().mode() & 0o044 != 0 {
        eprintln!("WARN: Configuration file is world-readable!");
    }

    // Read file. Map IO error to our custom ConfigError::Io
    let data = fs::read_to_string(path).map_err(|source| ConfigError::Io {
        path: path.to_string(),
        source,
    })?;

    // Parse data. `?` automatically calls From<serde_json::Error> for ConfigError
    let config: Config = serde_json::from_str(&data)?;

    // Validation
    // Note: port is already 0-65535 due to u16, we only check > 0
    if config.port == 0 {
        return Err(ConfigError::Validation {
            field: "port".into(),
            reason: "Must be greater than 0".into(),
        });
    }
    
    if config.max_connections <= 0 {
        return Err(ConfigError::Validation {
            field: "max_connections".into(),
            reason: "Must be strictly positive".into(),
        });
    }

    Ok(config)
}
```

## Critical Observations for C# Developers
1. **The `?` Operator Magic**: Look at `serde_json::from_str(&data)?`. If the JSON is malformed, `serde_json` emits an error. The `?` operator halts execution, looks at our `ConfigError` enum, notices `#[from] serde_json::Error`, automatically converts the error, and returns it. It is as concise as C# exceptions but with zero hidden control flow.
2. **Type System Validation**: In C# and Go, we used `int` for the port and had to manually check `config.Port > 65535`. In Rust, we typed it as `u16`. The parser (`serde`) will automatically fail with a syntax error if the config file has `port = 70000`. Rust pushes validation into the type system.
3. **Readable Error Chains**: If the Rust loader fails because the file doesn't exist, printing the error yields: `Failed to read configuration file at config.json`. Printing the `source()` yields: `No such file or directory (os error 2)`. The chain reads perfectly like a sentence.
