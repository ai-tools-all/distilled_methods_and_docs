# CGP Tutorial: Building Modular Systems with Context-Generic Programming

## Introduction

This tutorial teaches you how to implement CGP (Context-Generic Programming) patterns in your own Rust projects, based on patterns from the Hermes SDK. You'll build a simple but complete example that demonstrates all key concepts.

## Prerequisites

- Intermediate Rust knowledge (traits, generics, associated types)
- Familiarity with async/await
- Basic understanding of blockchain concepts (optional for this tutorial)

## Setup

Add these dependencies to your `Cargo.toml`:

```toml
[dependencies]
async-trait = "0.1"
tokio = { version = "1.0", features = ["full"] }
# For real projects, you'd also add the cgp crate
```

## Step 1: Define Your Domain Types

Start by defining the core types your system will work with:

```rust
// src/types.rs
use std::collections::HashMap;

#[derive(Debug, Clone, PartialEq)]
pub struct UserId(pub u64);

#[derive(Debug, Clone)]
pub struct User {
    pub id: UserId,
    pub name: String,
    pub email: String,
}

#[derive(Debug, Clone)]
pub struct Database {
    pub users: HashMap<UserId, User>,
}

#[derive(Debug)]
pub enum DatabaseError {
    UserNotFound(UserId),
    ValidationError(String),
    ConnectionError,
}
```

## Step 2: Define Associated Type Traits

Create traits that define what types your contexts provide:

```rust
// src/traits/types.rs
use crate::types::{User, UserId, DatabaseError};
use async_trait::async_trait;

// Type provider traits - these define what types a context has
pub trait HasUserIdType {
    type UserId;
}

pub trait HasUserType {
    type User;
}

pub trait HasErrorType {
    type Error;
}

// Convenience trait combining common requirements
pub trait HasAsyncErrorType: HasErrorType {
    fn raise_error<E>(error: E) -> Self::Error 
    where
        E: Into<Self::Error>;
}
```

## Step 3: Create Component Traits

Define the capabilities your system needs using the CGP component pattern:

```rust
// src/traits/user_repository.rs
use async_trait::async_trait;
use crate::traits::types::*;

// This would normally use #[cgp_component], but we'll simulate it
pub trait UserRepositoryProvider<Context> {
    async fn find_user(
        context: &Context,
        user_id: &Context::UserId,
    ) -> Result<Option<Context::User>, Context::Error>
    where
        Context: HasUserIdType + HasUserType + HasAsyncErrorType;

    async fn save_user(
        context: &mut Context,
        user: &Context::User,
    ) -> Result<(), Context::Error>
    where
        Context: HasUserIdType + HasUserType + HasAsyncErrorType;
}

#[async_trait]
pub trait CanFindUser: HasUserIdType + HasUserType + HasAsyncErrorType {
    async fn find_user(&self, user_id: &Self::UserId) -> Result<Option<Self::User>, Self::Error>;
}

#[async_trait]
pub trait CanSaveUser: HasUserIdType + HasUserType + HasAsyncErrorType {
    async fn save_user(&mut self, user: &Self::User) -> Result<(), Self::Error>;
}
```

## Step 4: Create Component Implementations

Implement the providers that give your components concrete behavior:

```rust
// src/impls/database_user_repository.rs
use async_trait::async_trait;
use crate::traits::{types::*, user_repository::*};
use crate::types::*;

pub struct DatabaseUserRepository;

impl<Context> UserRepositoryProvider<Context> for DatabaseUserRepository
where
    Context: HasUserIdType<UserId = UserId>
        + HasUserType<User = User>
        + HasAsyncErrorType<Error = DatabaseError>
        + HasDatabase,
{
    async fn find_user(
        context: &Context,
        user_id: &UserId,
    ) -> Result<Option<User>, DatabaseError> {
        let db = context.database();
        Ok(db.users.get(user_id).cloned())
    }

    async fn save_user(
        context: &mut Context,
        user: &User,
    ) -> Result<(), DatabaseError> {
        // Validate user data
        if user.name.is_empty() {
            return Err(DatabaseError::ValidationError("Name cannot be empty".to_string()));
        }
        
        let db = context.database_mut();
        db.users.insert(user.id.clone(), user.clone());
        Ok(())
    }
}

// Helper trait to access database
pub trait HasDatabase {
    fn database(&self) -> &Database;
    fn database_mut(&mut self) -> &mut Database;
}
```

## Step 5: Create Your Context

Build the context that ties everything together:

```rust
// src/context.rs
use crate::types::*;
use crate::traits::types::*;
use crate::traits::user_repository::*;
use crate::impls::database_user_repository::*;
use async_trait::async_trait;

pub struct UserServiceContext {
    database: Database,
}

impl UserServiceContext {
    pub fn new() -> Self {
        Self {
            database: Database {
                users: std::collections::HashMap::new(),
            }
        }
    }
}

// Implement type providers
impl HasUserIdType for UserServiceContext {
    type UserId = UserId;
}

impl HasUserType for UserServiceContext {
    type User = User;
}

impl HasErrorType for UserServiceContext {
    type Error = DatabaseError;
}

impl HasAsyncErrorType for UserServiceContext {
    fn raise_error<E>(error: E) -> Self::Error 
    where
        E: Into<Self::Error>
    {
        error.into()
    }
}

impl HasDatabase for UserServiceContext {
    fn database(&self) -> &Database {
        &self.database
    }
    
    fn database_mut(&mut self) -> &mut Database {
        &mut self.database
    }
}

// Implement capabilities using the component provider
#[async_trait]
impl CanFindUser for UserServiceContext {
    async fn find_user(&self, user_id: &Self::UserId) -> Result<Option<Self::User>, Self::Error> {
        DatabaseUserRepository::find_user(self, user_id).await
    }
}

#[async_trait]
impl CanSaveUser for UserServiceContext {
    async fn save_user(&mut self, user: &Self::User) -> Result<(), Self::Error> {
        DatabaseUserRepository::save_user(self, user).await
    }
}
```

## Step 6: Build Higher-Level Services

Create services that use your components:

```rust
// src/services/user_service.rs
use async_trait::async_trait;
use crate::traits::{types::*, user_repository::*};

pub struct UserService;

#[async_trait]
pub trait CanManageUsers: CanFindUser + CanSaveUser {
    async fn create_user(&mut self, name: String, email: String) -> Result<Self::User, Self::Error>;
    async fn get_user(&self, user_id: &Self::UserId) -> Result<Self::User, Self::Error>;
}

#[async_trait]
impl<Context> CanManageUsers for Context
where
    Context: CanFindUser + CanSaveUser + HasUserIdType + HasUserType,
    Context::User: From<(Context::UserId, String, String)>,
    Context::UserId: From<u64>,
{
    async fn create_user(&mut self, name: String, email: String) -> Result<Self::User, Self::Error> {
        // Generate a simple ID (in real code, use proper ID generation)
        let user_id = Context::UserId::from(42);
        let user = Context::User::from((user_id, name, email));
        
        self.save_user(&user).await?;
        Ok(user)
    }
    
    async fn get_user(&self, user_id: &Self::UserId) -> Result<Self::User, Self::Error> {
        self.find_user(user_id).await?
            .ok_or_else(|| Context::raise_error(DatabaseError::UserNotFound(UserId(42))))
    }
}

// Implement the conversion trait for our types
impl From<(UserId, String, String)> for User {
    fn from((id, name, email): (UserId, String, String)) -> Self {
        Self { id, name, email }
    }
}

impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}
```

## Step 7: Testing Your Components

CGP makes testing easy through trait-based mocking:

```rust
// src/mocks.rs
use std::collections::HashMap;
use async_trait::async_trait;
use crate::traits::{types::*, user_repository::*};
use crate::types::*;

pub struct MockUserContext {
    users: HashMap<UserId, User>,
    should_fail: bool,
}

impl MockUserContext {
    pub fn new() -> Self {
        Self {
            users: HashMap::new(),
            should_fail: false,
        }
    }
    
    pub fn with_failure(mut self) -> Self {
        self.should_fail = true;
        self
    }
}

impl HasUserIdType for MockUserContext {
    type UserId = UserId;
}

impl HasUserType for MockUserContext {
    type User = User;
}

impl HasErrorType for MockUserContext {
    type Error = DatabaseError;
}

impl HasAsyncErrorType for MockUserContext {
    fn raise_error<E>(error: E) -> Self::Error 
    where
        E: Into<Self::Error>
    {
        error.into()
    }
}

#[async_trait]
impl CanFindUser for MockUserContext {
    async fn find_user(&self, user_id: &Self::UserId) -> Result<Option<Self::User>, Self::Error> {
        if self.should_fail {
            return Err(DatabaseError::ConnectionError);
        }
        Ok(self.users.get(user_id).cloned())
    }
}

#[async_trait]
impl CanSaveUser for MockUserContext {
    async fn save_user(&mut self, user: &Self::User) -> Result<(), Self::Error> {
        if self.should_fail {
            return Err(DatabaseError::ConnectionError);
        }
        self.users.insert(user.id.clone(), user.clone());
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::services::user_service::CanManageUsers;

    #[tokio::test]
    async fn test_user_creation() {
        let mut context = MockUserContext::new();
        
        let user = context.create_user("Alice".to_string(), "alice@example.com".to_string()).await.unwrap();
        assert_eq!(user.name, "Alice");
        assert_eq!(user.email, "alice@example.com");
    }
    
    #[tokio::test]
    async fn test_connection_failure() {
        let mut context = MockUserContext::new().with_failure();
        
        let result = context.create_user("Bob".to_string(), "bob@example.com".to_string()).await;
        assert!(matches!(result, Err(DatabaseError::ConnectionError)));
    }
}
```

## Step 8: Advanced Patterns - Component Presets

For larger systems, create component bundles:

```rust
// src/presets.rs
use crate::context::UserServiceContext;
use crate::traits::user_repository::*;

pub trait UserServiceComponents: 
    CanFindUser + 
    CanSaveUser + 
    HasUserIdType<UserId = crate::types::UserId> + 
    HasUserType<User = crate::types::User> +
    HasErrorType<Error = crate::types::DatabaseError>
{
}

// Any context that implements the required traits automatically gets the preset
impl<T> UserServiceComponents for T 
where 
    T: CanFindUser + 
       CanSaveUser + 
       HasUserIdType<UserId = crate::types::UserId> + 
       HasUserType<User = crate::types::User> +
       HasErrorType<Error = crate::types::DatabaseError>
{
}

// Helper function to create a fully configured context
pub fn create_user_service() -> impl UserServiceComponents {
    UserServiceContext::new()
}
```

## Step 9: Example Usage

Put it all together in your main application:

```rust
// src/main.rs
mod types;
mod traits;
mod impls;
mod context;
mod services;
mod mocks;
mod presets;

use services::user_service::CanManageUsers;
use context::UserServiceContext;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut service = UserServiceContext::new();
    
    // Create a user
    let user = service.create_user("Alice".to_string(), "alice@example.com".to_string()).await?;
    println!("Created user: {:?}", user);
    
    // Retrieve the user
    let retrieved_user = service.get_user(&user.id).await?;
    println!("Retrieved user: {:?}", retrieved_user);
    
    Ok(())
}
```

## Key Takeaways

### 1. **Separation of Concerns**
- Traits define capabilities (`CanFindUser`)
- Types define data structures (`User`, `UserId`)
- Implementations provide behavior (`DatabaseUserRepository`)
- Contexts tie everything together (`UserServiceContext`)

### 2. **Composability**
- Components compose naturally through trait bounds
- Higher-level services use multiple capabilities
- Easy to create different contexts with different implementations

### 3. **Testability**
- Mock implementations through trait bounds
- No need for complex dependency injection frameworks
- Test individual components in isolation

### 4. **Type Safety**
- Associated types ensure type consistency
- Compiler catches mismatches at compile time
- Clear contracts between components

### 5. **Performance**
- Zero runtime overhead - everything compiles to static dispatch
- No vtables or dynamic allocation for component dispatch
- Inlined functions in release builds

## Advanced Techniques

### Generic Error Handling

```rust
pub trait CanRaiseSpecificError<E>: HasAsyncErrorType {
    fn raise_specific_error(error: E) -> Self::Error
    where
        E: Into<Self::Error>;
}

// Usage in implementations
impl<Context> SomeProvider<Context> for SomeImpl
where
    Context: CanRaiseSpecificError<ValidationError> + CanRaiseSpecificError<NetworkError>,
{
    async fn some_method(&self) -> Result<(), Context::Error> {
        // Can raise specific error types
        return Err(Context::raise_specific_error(ValidationError::InvalidInput));
    }
}
```

### Conditional Implementations

```rust
// Different implementations based on feature flags or types
#[cfg(feature = "database")]
impl<Context> UserRepositoryProvider<Context> for DatabaseUserRepository
where
    Context: HasDatabase + /* other bounds */
{
    // Database implementation
}

#[cfg(feature = "memory")]
impl<Context> UserRepositoryProvider<Context> for MemoryUserRepository  
where
    Context: /* different bounds */
{
    // In-memory implementation
}
```

### Component Delegation

```rust
pub struct CompositeUserService {
    // Delegate different operations to different implementations
}

impl<Context> UserRepositoryProvider<Context> for CompositeUserService
where
    Context: /* bounds */
{
    async fn find_user(&self, user_id: &Context::UserId) -> Result<Option<Context::User>, Context::Error> {
        // Could delegate to cache first, then database
        CacheRepository::find_user(context, user_id).await
            .or_else(|_| DatabaseRepository::find_user(context, user_id)).await
    }
}
```

## Conclusion

CGP patterns provide a powerful framework for building modular, testable, and maintainable Rust applications. The patterns scale from simple services to complex distributed systems while maintaining type safety and performance.

Key benefits:
- **Modularity**: Easy to swap implementations
- **Testability**: Mock any component via traits  
- **Type Safety**: Compiler-verified contracts
- **Performance**: Zero-cost abstractions
- **Maintainability**: Clear separation of concerns

Start with simple component traits and gradually compose them into more complex systems. The patterns will guide you toward clean, flexible architectures that grow gracefully with your requirements.