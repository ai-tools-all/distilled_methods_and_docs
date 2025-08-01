# Tracing Patterns Analysis - Zed Codebase

## Overview

Analysis of tracing implementation patterns in the Zed codebase, focusing on observability setup, instrumentation strategies, and production-ready logging practices.

## 1. Tracing Initialization & Configuration

**Location**: `crates/collab/src/main.rs:332-361`

### Pattern: Environment-Driven Flexible Configuration

```rust
pub fn init_tracing(config: &Config) -> Option<()> {
    use tracing_subscriber::layer::SubscriberExt;
    
    let filter = EnvFilter::from_str(config.rust_log.as_deref()?).log_err()?;
    
    tracing_subscriber::registry()
        .with(if config.log_json.unwrap_or(false) {
            // JSON format for production
            Box::new(
                tracing_subscriber::fmt::layer()
                    .fmt_fields(JsonFields::default())
                    .event_format(format().json().flatten_event(true).with_span_list(false))
                    .with_filter(filter)
            )
        } else {
            // Pretty format for development
            Box::new(
                tracing_subscriber::fmt::layer()
                    .event_format(format().pretty())
                    .with_filter(filter)
            )
        })
        .init();
}
```

### Key Implementation Patterns:

- **Environment-driven config**: Uses `RUST_LOG` env var via `EnvFilter`
- **Format switching**: JSON for production, pretty for development
- **Layered architecture**: Uses `registry().with()` pattern for composability
- **Error handling**: Graceful degradation with `log_err()`
- **Type erasure**: `Box<dyn Layer<_> + Send + Sync>` for dynamic layer selection

## 2. HTTP Request Tracing

**Location**: `crates/collab/src/main.rs:167-201`

### Pattern: Comprehensive Request Lifecycle Tracking

```rust
TraceLayer::new_for_http()
    .make_span_with(|request: &Request<_>| {
        let matched_path = request.extensions().get::<MatchedPath>().map(MatchedPath::as_str);
        let geoip_country_code = request.headers().typed_get::<CloudflareIpCountryHeader>()
            .map(|header| header.to_string());

        tracing::info_span!(
            "http_request",
            method = ?request.method(),
            matched_path,
            geoip_country_code,
            user_id = tracing::field::Empty,      // Filled later
            login = tracing::field::Empty,        // Filled later
            authn.jti = tracing::field::Empty,    // Filled later
            is_staff = tracing::field::Empty      // Filled later
        )
    })
    .on_response(|response: &Response<_>, latency: Duration, _: &tracing::Span| {
        let duration_ms = latency.as_micros() as f64 / 1000.;
        tracing::info!(
            duration_ms,
            status = response.status().as_u16(),
            "finished processing request"
        );
    })
```

### Key Implementation Patterns:

- **Empty fields**: Reserve fields for later population using `tracing::field::Empty`
- **Request context capture**: Extract useful metadata (path, geo, method)
- **Response timing**: Automatic latency tracking with microsecond precision
- **Structured fields**: Use field names that support querying/filtering
- **Extension extraction**: Pull additional context from request extensions

## 3. Async Task Instrumentation

**Location**: `crates/collab/src/rpc.rs:468-483`

### Pattern: Span Attachment to Async Futures

```rust
let span = info_span!("start server");
self.app_state.executor.spawn_detached(
    async move {
        tracing::info!("waiting for cleanup timeout");
        timeout.await;
        tracing::info!("cleanup timeout expired, retrieving stale rooms");
        // ... async work
    }
    .instrument(span)  // Attach span to entire async block
);
```

### Key Implementation Patterns:

- **Span creation before async**: Create span in sync context
- **Instrument attachment**: Use `.instrument(span)` for async futures
- **Descriptive span names**: Clear hierarchical naming ("start server")
- **Async context preservation**: Maintains tracing context across await points

## 4. Panic Hook Integration

**Location**: `crates/collab/src/main.rs:363-378`

### Pattern: Structured Panic Logging

```rust
std::panic::set_hook(Box::new(move |panic_info| {
    let panic_message = match panic_info.payload().downcast_ref::<&'static str>() {
        Some(message) => *message,
        None => match panic_info.payload().downcast_ref::<String>() {
            Some(message) => message.as_str(),
            None => "Box<Any>",
        },
    };
    let backtrace = std::backtrace::Backtrace::force_capture();
    let location = panic_info.location().map(|loc| format!("{}:{}", loc.file(), loc.line()));
    
    tracing::error!(
        panic = true,           // Boolean field for filtering
        ?location,              // Debug format with ?
        %panic_message,         // Display format with %
        %backtrace,             // Full backtrace
        "Server Panic"
    );
}));
```

### Key Implementation Patterns:

- **Boolean flags**: `panic = true` for easy filtering
- **Mixed field formats**: `?` for debug, `%` for display
- **Rich context**: Location, message, and full backtrace
- **Payload extraction**: Handle different panic payload types safely

## 5. Dependency Configuration

**Location**: `crates/collab/Cargo.toml`

### Pattern: Feature-Rich Tracing Setup

```toml
[dependencies]
tracing = { version = "0.1.40" }
tracing-subscriber = { 
    version = "0.3.18", 
    features = ["env-filter", "json", "registry", "tracing-log"] 
}
```

### Key Features:

- **env-filter**: Environment-based log level control (`RUST_LOG`)
- **json**: Production-ready structured logging
- **registry**: Composable subscriber architecture
- **tracing-log**: Bridge for legacy `log` crate integration

## Best Practices Identified

### 1. Configuration Management
- Always use `EnvFilter` for runtime log level control
- Support both human-readable (pretty) and machine-readable (JSON) formats
- Make configuration environment-driven rather than compile-time

### 2. Field Management
- Use `tracing::field::Empty` for fields populated later in request lifecycle
- Consistent field naming across spans for effective querying
- Mix format specifiers (`?` for debug, `%` for display) appropriately

### 3. Async Integration
- Always use `.instrument()` for async task tracing
- Create spans in synchronous context before spawning async tasks
- Maintain consistent span hierarchies across async boundaries

### 4. Error Handling
- Implement graceful degradation when tracing setup fails
- Use `.log_err()` pattern for non-critical tracing operations
- Never let tracing failures crash the application

### 5. Production Readiness
- Hook into panic handler for complete observability
- Include timing information (latency, duration) in key operations
- Capture rich context (geo-location, user info, request metadata)

### 6. Performance Considerations
- Use structured fields rather than string interpolation
- Leverage lazy evaluation of span creation
- Balance detail level with performance impact

## Anti-Patterns to Avoid

1. **String interpolation in log messages** - Use structured fields instead
2. **Synchronous tracing setup in async contexts** - Create spans before async
3. **Ignoring tracing setup failures** - Always handle gracefully
4. **Missing context in long-running operations** - Use `.instrument()`
5. **Inconsistent field naming** - Establish conventions early

## Implementation Checklist

- [ ] Environment-driven configuration with `EnvFilter`
- [ ] Dual format support (pretty/JSON) based on environment
- [ ] HTTP request tracing with timing and context
- [ ] Async task instrumentation with `.instrument()`
- [ ] Panic hook integration for error visibility
- [ ] Structured field usage over string interpolation
- [ ] Graceful degradation on tracing failures
- [ ] Consistent span naming conventions