# WhatsApp Clone - Technical Architecture Overview

## Executive Summary

This document provides a comprehensive technical architecture overview of the WhatsApp Clone application, a full-stack real-time messaging system built with modern web technologies.

### Technology Stack

#### Backend
- **Runtime**: Node.js with TypeScript
- **API Framework**: Apollo GraphQL Server v2.13.1
- **Web Framework**: Express.js v4.17.1
- **Database**: PostgreSQL v8.2.1
- **Database Client**: node-postgres (pg)
- **Authentication**: JWT (jsonwebtoken) with bcrypt password hashing
- **GraphQL Modules**: @graphql-modules/core v0.7.17 for modular GraphQL architecture
- **Real-time Communication**: GraphQL Subscriptions with graphql-postgres-subscriptions
- **Data Loading**: DataLoader v2.0.0 for batching and caching
- **External APIs**: Apollo DataSource REST for Unsplash API integration

#### Frontend
- **UI Framework**: React v17.0.2 with TypeScript
- **GraphQL Client**: Apollo Client v4.0.8
- **UI Components**: Material-UI v4.12.4
- **Styling**: Styled-Components v6.1.19
- **Routing**: React Router DOM v5.3.4
- **Real-time**: GraphQL Subscriptions via graphql-ws v6.0.6

#### Development & Testing
- **Testing**: Jest with ts-jest
- **Code Generation**: GraphQL Code Generator
- **Build Tool**: TypeScript Compiler, React Scripts
- **Load Testing**: Artillery

---

## System Architecture

### High-Level System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WebApp[React Web Application]
        ApolloClient[Apollo Client]
    end
    
    subgraph "API Gateway Layer"
        Express[Express.js Server]
        ApolloServer[Apollo GraphQL Server]
        WSServer[WebSocket Server]
    end
    
    subgraph "Business Logic Layer"
        UsersModule[Users Module]
        ChatsModule[Chats Module]
        CommonModule[Common Module]
        AuthProvider[Auth Provider]
        UsersProvider[Users Provider]
        ChatsProvider[Chats Provider]
        PubSubProvider[PubSub Provider]
    end
    
    subgraph "Data Layer"
        PostgreSQL[(PostgreSQL Database)]
        DataLoader[DataLoader Cache]
    end
    
    subgraph "External Services"
        UnsplashAPI[Unsplash API]
    end
    
    WebApp --> ApolloClient
    ApolloClient -->|GraphQL Queries/Mutations| ApolloServer
    ApolloClient -->|GraphQL Subscriptions| WSServer
    
    Express --> ApolloServer
    Express --> WSServer
    
    ApolloServer --> UsersModule
    ApolloServer --> ChatsModule
    WSServer --> PubSubProvider
    
    UsersModule --> AuthProvider
    UsersModule --> UsersProvider
    UsersModule --> CommonModule
    
    ChatsModule --> ChatsProvider
    ChatsModule --> UsersProvider
    ChatsModule --> PubSubProvider
    ChatsModule --> CommonModule
    
    AuthProvider --> UsersProvider
    UsersProvider --> DataLoader
    ChatsProvider --> DataLoader
    ChatsProvider --> PubSubProvider
    
    DataLoader --> PostgreSQL
    UsersProvider --> PostgreSQL
    ChatsProvider --> PostgreSQL
    
    ChatsModule --> UnsplashAPI
    
    style WebApp fill:#e1f5ff
    style PostgreSQL fill:#ffebee
    style UnsplashAPI fill:#fff3e0
```

---

## Component Architecture

### GraphQL Module Architecture

The application follows a modular GraphQL architecture pattern using `@graphql-modules/core`:

```mermaid
graph TB
    subgraph "GraphQL Schema"
        RootSchema[Root Schema]
    end
    
    subgraph "Users Module"
        UserTypes[User Types]
        UserResolvers[User Resolvers]
        AuthProvider[Auth Provider]
        UsersProvider[Users Provider]
    end
    
    subgraph "Chats Module"
        ChatTypes[Chat/Message Types]
        ChatResolvers[Chat Resolvers]
        ChatsProvider[Chats Provider]
    end
    
    subgraph "Common Module"
        CommonTypes[Common Types]
        DatabaseProvider[Database Provider]
        PubSubProvider[PubSub Provider]
    end
    
    RootSchema --> UserTypes
    RootSchema --> ChatTypes
    RootSchema --> CommonTypes
    
    UserResolvers --> AuthProvider
    UserResolvers --> UsersProvider
    
    ChatResolvers --> ChatsProvider
    ChatResolvers --> UsersProvider
    ChatResolvers --> PubSubProvider
    
    AuthProvider --> UsersProvider
    ChatsProvider --> DatabaseProvider
    UsersProvider --> DatabaseProvider
    
    UserTypes -.import.-> CommonTypes
    ChatTypes -.import.-> CommonTypes
    ChatTypes -.import.-> UserTypes
    
    style RootSchema fill:#e8f5e9
    style CommonTypes fill:#fff9c4
```

### Provider Dependency Injection

```mermaid
graph LR
    subgraph "Session Scope Providers"
        Auth[Auth Provider]
        Users[Users Provider]
        Chats[Chats Provider]
    end
    
    subgraph "Application Scope Providers"
        Database[Database Provider]
        PubSub[PubSub Provider]
    end
    
    subgraph "External Data Sources"
        Unsplash[Unsplash API DataSource]
    end
    
    Auth -->|@Inject| Users
    Chats -->|@Inject| Database
    Chats -->|@Inject| PubSub
    Users -->|@Inject| Database
    
    Resolvers[GraphQL Resolvers] -->|injector.get| Auth
    Resolvers -->|injector.get| Users
    Resolvers -->|injector.get| Chats
    Resolvers -->|dataSources| Unsplash
    
    style Auth fill:#ffcdd2
    style Users fill:#c5cae9
    style Chats fill:#b2dfdb
```

---

## Data Architecture

### Database Schema

```mermaid
erDiagram
    USERS ||--o{ CHATS_USERS : participates
    USERS ||--o{ MESSAGES : sends
    CHATS ||--o{ CHATS_USERS : has
    CHATS ||--o{ MESSAGES : contains
    
    USERS {
        serial id PK
        varchar username UK
        varchar name
        varchar password
        varchar picture
    }
    
    CHATS {
        serial id PK
    }
    
    CHATS_USERS {
        integer chat_id FK
        integer user_id FK
    }
    
    MESSAGES {
        serial id PK
        varchar content
        timestamp created_at
        integer chat_id FK
        integer sender_user_id FK
    }
```

### GraphQL Type System

```mermaid
graph TB
    subgraph "Core Types"
        User[User]
        Chat[Chat]
        Message[Message]
    end
    
    subgraph "Query Operations"
        Q1[me: User]
        Q2[users: User]
        Q3[chats: Chat]
        Q4[chat: Chat]
    end
    
    subgraph "Mutation Operations"
        M1[signIn: User]
        M2[signUp: User]
        M3[addMessage: Message]
        M4[addChat: Chat]
        M5[removeChat: ID]
    end
    
    subgraph "Subscription Operations"
        S1[messageAdded: Message]
        S2[chatAdded: Chat]
        S3[chatRemoved: ID]
    end
    
    subgraph "Scalar Types"
        DateTime[DateTime]
        URL[URL]
    end
    
    Chat --> Message
    Chat --> User
    Message --> Chat
    Message --> User
    
    Q1 --> User
    Q2 --> User
    Q3 --> Chat
    Q4 --> Chat
    
    M1 --> User
    M2 --> User
    M3 --> Message
    M4 --> Chat
    M5 --> Chat
    
    S1 --> Message
    S2 --> Chat
    S3 --> Chat
    
    Message --> DateTime
    User --> URL
    Chat --> URL
    
    style User fill:#e1bee7
    style Chat fill:#c5e1a5
    style Message fill:#ffcc80
```

---

## Data Flow Architecture

### Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant ApolloServer
    participant AuthProvider
    participant UsersProvider
    participant Database
    participant JWT
    
    Client->>ApolloServer: signIn(username, password)
    ApolloServer->>AuthProvider: signIn({username, password})
    AuthProvider->>UsersProvider: findByUsername(username)
    UsersProvider->>Database: SELECT * FROM users WHERE username = ?
    Database-->>UsersProvider: User record
    UsersProvider-->>AuthProvider: User object
    AuthProvider->>AuthProvider: bcrypt.compare(password, user.password)
    alt Password matches
        AuthProvider->>JWT: jwt.sign(username, secret)
        JWT-->>AuthProvider: authToken
        AuthProvider->>Client: Set-Cookie: authToken
        AuthProvider-->>ApolloServer: User object
        ApolloServer-->>Client: User
    else Password doesn't match
        AuthProvider-->>ApolloServer: Error: password is incorrect
        ApolloServer-->>Client: Error
    end
```

### Message Creation Flow

```mermaid
sequenceDiagram
    participant Client
    participant ApolloServer
    participant ChatsProvider
    participant Database
    participant PubSub
    participant Subscribers
    
    Client->>ApolloServer: addMessage(chatId, content)
    ApolloServer->>ChatsProvider: addMessage({chatId, content, userId})
    ChatsProvider->>Database: INSERT INTO messages (content, chat_id, sender_user_id)
    Database-->>ChatsProvider: Message record
    ChatsProvider->>PubSub: publish('messageAdded', message)
    PubSub-->>Subscribers: messageAdded event
    ChatsProvider-->>ApolloServer: Message object
    ApolloServer-->>Client: Message
    
    Note over Subscribers: All subscribed clients<br/>in the same chat receive<br/>the new message
```

### Query with DataLoader Caching

```mermaid
sequenceDiagram
    participant Client
    participant Resolver
    participant ChatsProvider
    participant DataLoader
    participant Database
    
    Client->>Resolver: getChat(chatId: "1")
    Resolver->>ChatsProvider: findChatByUser({chatId: "1", userId})
    ChatsProvider->>DataLoader: load({chatId: "1", userId})
    
    alt Cache hit
        DataLoader-->>ChatsProvider: Cached chat
    else Cache miss
        DataLoader->>Database: SELECT * FROM chats WHERE id = 1
        Database-->>DataLoader: Chat record
        DataLoader->>DataLoader: Store in cache
        DataLoader-->>ChatsProvider: Chat
    end
    
    ChatsProvider-->>Resolver: Chat object
    Resolver->>ChatsProvider: lastMessage(chatId)
    ChatsProvider->>DataLoader: load({chatId: "1"})
    DataLoader-->>ChatsProvider: Cached chat
    Resolver-->>Client: Fully resolved Chat
```

### Real-time Subscription Flow

```mermaid
sequenceDiagram
    participant ClientA
    participant ClientB
    participant WSServer
    participant PubSub
    participant PostgreSQL
    
    ClientA->>WSServer: subscribe { messageAdded }
    WSServer->>PubSub: asyncIterator('messageAdded')
    PubSub->>PostgreSQL: LISTEN messageAdded
    
    ClientB->>WSServer: addMessage mutation
    WSServer->>PubSub: publish('messageAdded', message)
    PubSub->>PostgreSQL: NOTIFY messageAdded
    PostgreSQL-->>PubSub: Notification
    PubSub-->>WSServer: Event data
    WSServer->>WSServer: withFilter(check participant)
    WSServer-->>ClientA: { messageAdded: Message }
    
    Note over ClientA,ClientB: Real-time message delivery<br/>with participant filtering
```

---

## Integration Architecture

### External API Integration

```mermaid
graph TB
    subgraph "Apollo Server"
        Resolvers[Chat Resolvers]
        DataSources[Data Sources]
    end
    
    subgraph "REST Data Source"
        UnsplashAPI[Unsplash API DataSource]
        Cache[RESTDataSource Cache]
    end
    
    subgraph "External Service"
        Unsplash[Unsplash API]
    end
    
    Resolvers -->|context.dataSources| DataSources
    DataSources --> UnsplashAPI
    UnsplashAPI --> Cache
    Cache -->|GET /photos/random| Unsplash
    
    Unsplash -->|Random Photo URL| Cache
    Cache -->|Cached Response| UnsplashAPI
    UnsplashAPI -->|Photo URL| Resolvers
    
    style UnsplashAPI fill:#fff9c4
    style Cache fill:#c5e1a5
    style Unsplash fill:#ffccbc
```

### Authentication & Authorization Flow

```mermaid
graph TB
    Request[HTTP Request] -->|Cookie: authToken| Middleware
    Middleware -->|Parse Cookie| CookieParser
    CookieParser --> Context[GraphQL Context]
    
    Context --> Resolver[GraphQL Resolver]
    Resolver -->|injector.get| AuthProvider
    
    AuthProvider -->|req.cookies.authToken| JWT[JWT Verify]
    JWT -->|username| UsersProvider
    UsersProvider -->|findByUsername| Database[(PostgreSQL)]
    Database -->|User| UsersProvider
    UsersProvider -->|User| AuthProvider
    AuthProvider -->|currentUser| Resolver
    
    Resolver -->|Authorized Operation| BusinessLogic[Business Logic]
    
    style AuthProvider fill:#ffcdd2
    style JWT fill:#c5cae9
    style Resolver fill:#b2dfdb
```

---

## Performance Architecture

### DataLoader Batching & Caching

```mermaid
graph TB
    subgraph "Single Request Cycle"
        R1[Resolver 1: Get Chat 1]
        R2[Resolver 2: Get Chat 2]
        R3[Resolver 3: Get Chat 1]
        R4[Resolver 4: Get User 5]
    end
    
    subgraph "DataLoader Layer"
        ChatLoader[Chat DataLoader]
        UserLoader[User DataLoader]
        Cache[Request-scoped Cache]
    end
    
    subgraph "Database Layer"
        DB[(PostgreSQL)]
    end
    
    R1 -->|load chatId:1| ChatLoader
    R2 -->|load chatId:2| ChatLoader
    R3 -->|load chatId:1| ChatLoader
    R4 -->|load userId:5| UserLoader
    
    ChatLoader -->|Batch: SELECT * WHERE id IN (1,2)| DB
    UserLoader -->|Batch: SELECT * WHERE id = 5| DB
    
    ChatLoader --> Cache
    R3 -.->|Cache Hit| Cache
    
    DB -->|Results| ChatLoader
    DB -->|Results| UserLoader
    
    ChatLoader --> R1
    ChatLoader --> R2
    Cache --> R3
    UserLoader --> R4
    
    style Cache fill:#c8e6c9
    style ChatLoader fill:#fff9c4
    style UserLoader fill:#ffccbc
```

### Session-based Provider Scoping

```mermaid
graph TB
    subgraph "Request 1 - User A"
        Session1[GraphQL Session]
        Auth1[Auth Provider Instance]
        Chats1[Chats Provider Instance]
        CurrentUser1[currentUser: User A]
    end
    
    subgraph "Request 2 - User B"
        Session2[GraphQL Session]
        Auth2[Auth Provider Instance]
        Chats2[Chats Provider Instance]
        CurrentUser2[currentUser: User B]
    end
    
    subgraph "Application Scope"
        Database[Database Provider - Singleton]
        PubSub[PubSub Provider - Singleton]
    end
    
    Session1 --> Auth1
    Session1 --> Chats1
    Auth1 --> CurrentUser1
    
    Session2 --> Auth2
    Session2 --> Chats2
    Auth2 --> CurrentUser2
    
    Auth1 --> Database
    Auth2 --> Database
    Chats1 --> Database
    Chats2 --> Database
    
    Chats1 --> PubSub
    Chats2 --> PubSub
    
    style Auth1 fill:#ffcdd2
    style Auth2 fill:#ffcdd2
    style Database fill:#e1f5fe
    style PubSub fill:#fff9c4
```

---

## Security Architecture

### Authentication Layer

```mermaid
graph TB
    subgraph "Client Side"
        Browser[Browser]
        Cookies[HTTP-only Cookie]
    end
    
    subgraph "Server Side"
        Express[Express Server]
        CookieParser[Cookie Parser Middleware]
        JWT[JWT Verification]
        Auth[Auth Provider]
    end
    
    subgraph "Security Measures"
        Bcrypt[Bcrypt Password Hashing]
        SecretKey[JWT Secret Key]
        Expiration[Token Expiration]
    end
    
    Browser -->|authToken Cookie| Express
    Express --> CookieParser
    CookieParser --> JWT
    JWT -->|Verify with| SecretKey
    JWT --> Auth
    
    Auth -->|Password Validation| Bcrypt
    Auth -->|Token Generation| SecretKey
    Auth -->|Set Max Age| Expiration
    
    Bcrypt -->|Hashed Password| Database[(PostgreSQL)]
    
    style Bcrypt fill:#ffcdd2
    style SecretKey fill:#fff9c4
    style JWT fill:#c5e1a5
```

### Authorization Strategy

```mermaid
graph TB
    Request[GraphQL Request] --> Resolver
    
    Resolver -->|1. Get Current User| Auth[Auth Provider]
    Auth -->|Verify JWT| JWT
    JWT -->|Return User| Auth
    
    Resolver -->|2. Check Authorization| AuthLogic{Authorized?}
    
    AuthLogic -->|Yes| Execute[Execute Operation]
    AuthLogic -->|No| Reject[Return null/empty]
    
    Execute --> Validate{Resource<br/>belongs to user?}
    
    Validate -->|Yes - Chat Participant| Allow[Allow Access]
    Validate -->|No| Deny[Deny Access]
    
    Allow --> Database[(PostgreSQL)]
    Database --> Response[Return Data]
    
    Deny --> EmptyResponse[Return null]
    Reject --> EmptyResponse
    
    style Auth fill:#ffcdd2
    style AuthLogic fill:#fff9c4
    style Validate fill:#c5e1a5
```

---

## Deployment Architecture

### Production Environment

```mermaid
graph TB
    subgraph "Client Deployment"
        CDN[CDN / Static Hosting]
        ReactBuild[React Build Artifacts]
    end
    
    subgraph "Server Deployment"
        NodeServer[Node.js Server Process]
        ExpressApp[Express Application]
        ApolloServer[Apollo GraphQL Server]
    end
    
    subgraph "Database Layer"
        PostgreSQL[(PostgreSQL Database)]
        Migrations[Database Migrations]
    end
    
    subgraph "Monitoring & Logging"
        Logs[Application Logs]
        Metrics[Performance Metrics]
    end
    
    ReactBuild --> CDN
    Users[Users] -->|HTTPS| CDN
    Users -->|GraphQL API| NodeServer
    
    NodeServer --> ExpressApp
    ExpressApp --> ApolloServer
    
    ApolloServer --> PostgreSQL
    Migrations --> PostgreSQL
    
    NodeServer --> Logs
    NodeServer --> Metrics
    
    style CDN fill:#e1f5fe
    style PostgreSQL fill:#ffebee
    style NodeServer fill:#c8e6c9
```

---

## Technology Decisions & Rationale

### Backend Architecture Decisions

1. **GraphQL over REST**
   - Single endpoint for all data operations
   - Strong typing with schema
   - Efficient data fetching (no over/under-fetching)
   - Real-time subscriptions support
   - Code generation for type safety

2. **GraphQL Modules Architecture**
   - Separation of concerns
   - Dependency injection for testability
   - Independent feature modules
   - Shared common functionality
   - Session-scoped providers for request isolation

3. **PostgreSQL Database**
   - ACID compliance for message consistency
   - Strong relational model for chat relationships
   - Native support for GraphQL subscriptions (LISTEN/NOTIFY)
   - Mature ecosystem and tooling

4. **DataLoader Pattern**
   - N+1 query problem mitigation
   - Request-scoped caching
   - Automatic batching of database queries
   - Performance optimization

5. **JWT Authentication**
   - Stateless authentication
   - Scalability (no server-side session storage)
   - Cross-domain support
   - Standard protocol

### Frontend Architecture Decisions

1. **React with Hooks**
   - Modern, functional component approach
   - Simplified state management
   - Better code reusability
   - Active ecosystem

2. **Apollo Client**
   - Integrated GraphQL client
   - Intelligent caching
   - Optimistic UI updates
   - Real-time subscription support
   - Normalized cache

3. **Material-UI + Styled-Components**
   - Professional UI components
   - Customizable theming
   - CSS-in-JS for component encapsulation
   - TypeScript support

4. **TypeScript**
   - Type safety across the stack
   - Better IDE support
   - Reduced runtime errors
   - GraphQL code generation integration

---

## Scalability Considerations

### Current Architecture Limitations

1. **Single Server Instance**
   - No horizontal scaling
   - Single point of failure
   - Limited by single machine resources

2. **In-Memory PubSub**
   - Not distributed across instances
   - Requires sticky sessions or external PubSub

3. **Session-based DataLoader**
   - Cache limited to request scope
   - No cross-request caching

### Scalability Improvements

```mermaid
graph TB
    subgraph "Scalable Architecture"
        LB[Load Balancer]
        S1[Server Instance 1]
        S2[Server Instance 2]
        S3[Server Instance 3]
        
        Redis[(Redis PubSub)]
        PGPool[PostgreSQL Connection Pool]
        PGMaster[(PostgreSQL Primary)]
        PGReplica1[(PostgreSQL Replica 1)]
        PGReplica2[(PostgreSQL Replica 2)]
        
        Cache[Redis Cache]
    end
    
    LB --> S1
    LB --> S2
    LB --> S3
    
    S1 --> Redis
    S2 --> Redis
    S3 --> Redis
    
    S1 --> PGPool
    S2 --> PGPool
    S3 --> PGPool
    
    S1 --> Cache
    S2 --> Cache
    S3 --> Cache
    
    PGPool --> PGMaster
    PGPool --> PGReplica1
    PGPool --> PGReplica2
    
    PGMaster -.replication.-> PGReplica1
    PGMaster -.replication.-> PGReplica2
    
    style Redis fill:#ffcdd2
    style Cache fill:#c5e1a5
    style LB fill:#fff9c4
```

---

## System Invariants & Constraints

### Data Integrity Invariants

1. **User Uniqueness**: Each username must be unique across the system
2. **Message Ownership**: Every message must belong to exactly one chat and have exactly one sender
3. **Chat Participation**: A chat must have at least 2 participants (enforced at application level)
4. **Authentication Requirement**: All mutations (except signIn/signUp) require authenticated user
5. **Authorization**: Users can only access chats they participate in

### Performance Constraints

1. **Message Pagination**: Messages fetched with limit to prevent large payloads
2. **DataLoader Batching**: Database queries batched per request cycle
3. **Connection Pooling**: PostgreSQL connections managed via pool

### Security Constraints

1. **Password Storage**: Passwords stored as bcrypt hashes, never plaintext
2. **JWT Expiration**: Authentication tokens have configurable expiration
3. **HTTP-only Cookies**: Auth tokens stored in HTTP-only cookies to prevent XSS
4. **Input Validation**: Username, password, and name fields validated for length and format

---

## Future Architecture Enhancements

1. **Microservices Migration**
   - Separate auth, chat, and user services
   - Independent scaling and deployment
   - Service mesh for inter-service communication

2. **Event Sourcing**
   - Immutable event log
   - Replay capabilities
   - Audit trail

3. **CQRS Pattern**
   - Separate read and write models
   - Optimized read replicas
   - Command and query segregation

4. **Distributed Caching**
   - Redis for distributed cache
   - Cache invalidation strategies
   - Read-through caching

5. **Message Queue Integration**
   - Asynchronous processing
   - Background jobs
   - Reliable message delivery

---

## Glossary

- **Apollo Server**: GraphQL server implementation for Node.js
- **DataLoader**: Utility for batching and caching data fetches
- **GraphQL Modules**: Library for creating modular GraphQL schemas
- **JWT (JSON Web Token)**: Standard for stateless authentication
- **PubSub**: Publish-subscribe pattern for real-time updates
- **Provider**: Dependency injection service in GraphQL Modules
- **Resolver**: Function that resolves a GraphQL field value
- **Subscription**: GraphQL operation type for real-time data streaming
- **WebSocket**: Protocol for bidirectional real-time communication

---

*Document Version: 1.0*  
*Last Updated: 2025-11-11*  
*Architecture Review Date: 2025-11-11*
