---
name: rust-patterns
description: Rust patterns, ownership best practices, and production-ready code guidelines
---

# Rust Patterns Skill

## Overview

This skill provides guidelines for writing safe, efficient, and idiomatic Rust code, covering ownership, lifetimes, error handling, and async patterns for production systems with Actix/Tokio.

## Core Principles

### 1. Ownership and Borrowing

```rust
// DO: Use ownership transfer for large data
fn process_data(data: Vec<u8>) -> ProcessedData {
    // Takes ownership, no copy
    ProcessedData::from(data)
}

// DO: Borrow for read-only access
fn print_info(data: &[u8]) {
    // Immutable borrow
    println!("{:?}", data);
}

// DO: Mutable borrow for modification
fn update_buffer(buf: &mut Vec<u8>) {
    buf.push(0);
}

// DON'T: Create multiple mutable references
let mut data = vec![1, 2, 3];
let ref1 = &mut data;
let ref2 = &mut data; // ERROR!
```

### 2. Lifetimes

```rust
// DO: Explicit lifetimes for function signatures
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// DO: Struct lifetimes
struct Parser<'a> {
    input: &'a str,
}

impl<'a> Parser<'a> {
    fn new(input: &'a str) -> Self {
        Self { input }
    }
    
    fn parse(&self) -> Result<&str, Error> {
        Ok(&self.input[..5])
    }
}

// DO: 'static for owned data
const CONFIG: Config = Config {
    timeout: 30,
};
```

### 3. Error Handling

```rust
// DO: Use Result for recoverable errors
fn read_config(path: &str) -> Result<Config, io::Error> {
    let content = fs::read_to_string(path)?;
    let config: Config = serde_json::from_str(&content)?;
    Ok(config)
}

// DO: Custom error types
#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("io error: {0}")]
    Io(#[from] io::Error),
    
    #[error("parse error: {0}")]
    Parse(#[from] serde_json::Error),
    
    #[error("validation failed: {message}")]
    Validation { message: String },
    
    #[error("not found: {resource}")]
    NotFound { resource: String },
}

type Result<T> = std::result::Result<T, AppError>;

// DO: Use ? operator for early returns
fn process_file(path: &str) -> Result<ProcessedData> {
    let content = fs::read_to_string(path)?; // Early return on error
    let data = parse_content(&content)?;
    validate_data(&data)?;
    Ok(data)
}
```

### 4. Option and Result Combinators

```rust
// DO: Use combinators instead of match where clear
let value = config.get("timeout")
    .and_then(|v| v.parse::<u64>().ok())
    .unwrap_or(30);

// DO: map for transformations
let upper = maybe_string.map(|s| s.to_uppercase());

// DO: ok_or for converting Option to Result
let port = env::var("PORT")
    .ok()
    .and_then(|p| p.parse().ok())
    .ok_or(AppError::Validation {
        message: "PORT must be a valid number".into(),
    })?;
```

### 5. Smart Pointers

```rust
// DO: Box for recursive types
enum Node {
    Value(i32),
    Cons(i32, Box<Node>),
}

// DO: Rc for shared ownership (single-threaded)
use std::rc::Rc;
let shared_data: Rc<Vec<u8>> = Rc::new(vec![1, 2, 3]);
let clone1 = Rc::clone(&shared_data);
let clone2 = Rc::clone(&shared_data);

// DO: Arc for thread-safe shared ownership
use std::sync::Arc;
let data: Arc<Vec<u8>> = Arc::new(vec![1, 2, 3]);

// DO: Mutex/RwLock for interior mutability
use std::sync::{Arc, Mutex};
let counter = Arc::new(Mutex::new(0));
let counter_clone = Arc::clone(&counter);
std::thread::spawn(move || {
    let mut num = counter_clone.lock().unwrap();
    *num += 1;
});
```

## Async Patterns (Tokio)

```rust
// DO: Use async/await
async fn fetch_data(url: &str) -> Result<Vec<u8>, reqwest::Error> {
    let response = reqwest::get(url).await?;
    let bytes = response.bytes().await?;
    Ok(bytes.to_vec())
}

// DO: Concurrent execution with join!
use futures::join;

async fn fetch_multiple() -> Result<(Vec<u8>, Vec<u8>), Error> {
    let (result1, result2) = join!(
        fetch_data("https://api1.example.com"),
        fetch_data("https://api2.example.com"),
    );
    Ok((result1?, result2?))
}

// DO: Spawn tasks for parallel work
async fn process_batch(items: Vec<Item>) -> Vec<Result<ProcessedItem, Error>> {
    let handles: Vec<_> = items
        .into_iter()
        .map(|item| {
            tokio::spawn(async move {
                process_item(item).await
            })
        })
        .collect();
    
    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// DO: Use channels for communication
use tokio::sync::mpsc;

async fn worker(mut rx: mpsc::Receiver<Job>) {
    while let Some(job) = rx.recv().await {
        process_job(job).await;
    }
}
```

## Actix Web Patterns

```rust
// DO: Application state with Arc
struct AppState {
    db: Arc<dyn Database>,
    config: Arc<Config>,
}

async fn handler(data: web::Data<AppState>) -> impl Responder {
    let result = data.db.query().await;
    HttpResponse::Ok().json(result)
}

// DO: Extractors for request data
#[derive(Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

async fn create_user(
    data: web::Data<AppState>,
    req: web::Json<CreateUserRequest>,
) -> Result<HttpResponse, Error> {
    let user = data.db.create_user(req.into_inner()).await?;
    Ok(HttpResponse::Created().json(user))
}

// DO: Custom middleware
fn auth_middleware() -> impl Transform<ServiceRequest, Response = ServiceResponse, Error = Error> {
    fn transform<S>(service: S) -> AuthMiddleware<S> {
        AuthMiddleware { service }
    }
    
    middleware::from_fn(|req, srv| async move {
        if !is_authenticated(&req) {
            return Err(ErrorUnauthorized("Unauthorized"));
        }
        srv.call(req).await
    })
}
```

## Structs and Enums

```rust
// DO: Builder pattern for complex structs
#[derive(Default)]
struct ConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    workers: Option<usize>,
}

impl ConfigBuilder {
    fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }
    
    fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }
    
    fn build(self) -> Config {
        Config {
            host: self.host.unwrap_or_else(|| "localhost".into()),
            port: self.port.unwrap_or(8080),
            workers: self.workers.unwrap_or(4),
        }
    }
}

// DO: Use enums for state machines
enum ConnectionState {
    Disconnected,
    Connecting { since: Instant },
    Connected { socket: TcpStream },
    Error { reason: String },
}

impl ConnectionState {
    fn is_connected(&self) -> bool {
        matches!(self, ConnectionState::Connected { .. })
    }
}
```

## Testing

```rust
// DO: Unit tests in the same file
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_async_function() {
        let result = async_function().await;
        assert!(result.is_ok());
    }
    
    #[test]
    fn test_with_fixture() {
        let fixture = create_test_fixture();
        let result = process(fixture);
        assert_eq!(result, expected);
    }
}

// DO: Mock with traits
trait Database {
    async fn query(&self) -> Result<Vec<Row>, Error>;
}

struct MockDb {
    results: Vec<Row>,
}

#[async_trait]
impl Database for MockDb {
    async fn query(&self) -> Result<Vec<Row>, Error> {
        Ok(self.results.clone())
    }
}
```

## When to Use

Use this skill when:
- Writing or reviewing Rust code
- Handling complex ownership/lifetime scenarios
- Implementing async/await code
- Building web services with Actix
- Writing concurrent/parallel code
- Designing Rust APIs
- Refactoring for performance
