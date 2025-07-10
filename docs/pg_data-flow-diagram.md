# T5G PostgreSQL Data Flow and Synchronisation Process

## Data Synchronisation Flow with PostgreSQL

This diagram shows the updated data flow with PostgreSQL integration, demonstrating the hybrid Redis + PostgreSQL storage approach.

```mermaid
sequenceDiagram
    participant Portal as Red Hat Portal
    participant JIRA as JIRA Server
    participant BZ as Bugzilla
    participant Celery as Celery Beat
    participant Worker as Celery Worker
    participant Redis as Redis Cache
    participant Postgres as PostgreSQL DB
    participant Web as Flask Web App
    participant User as End User

    %% Service Health Checks
    Note over Postgres, Redis: Health checks before startup
    Postgres->>Docker: pg_isready health check
    Redis->>Docker: redis-cli ping health check
    
    %% Scheduled Data Synchronisation with Dual Write
    Note over Celery: Every 15 minutes
    Celery->>Worker: Trigger case_sync
    Worker->>Portal: Fetch support cases
    Portal-->>Worker: Cases JSON data
    
    %% Dual Write Strategy
    Worker->>Redis: Store cases (fast access)
    Worker->>Postgres: Store cases (persistence)
    Note over Worker, Postgres: SQLAlchemy ORM<br/>Cases table with<br/>composite PK
    
    Note over Celery: Every hour at :21
    Celery->>Worker: Trigger card_sync
    Worker->>JIRA: Fetch JIRA cards & comments
    JIRA-->>Worker: Cards + comments data
    
    %% Dual Storage for Comments
    Worker->>Redis: Store cards (cache)
    Worker->>Postgres: Check existing comments
    Postgres-->>Worker: Comment existence check
    Worker->>Postgres: Store new JIRA comments
    Note over Worker, Postgres: Foreign key validation<br/>to Cases table
    
    Note over Celery: Every 12 hours
    Celery->>Worker: Trigger details_sync
    Worker->>Portal: Fetch case details
    Portal-->>Worker: Extended case details
    Worker->>Redis: Cache details
    Worker->>Postgres: Update case records
    
    %% User Interaction with Hybrid Storage
    User->>Web: Request dashboard
    Web->>Redis: Fetch cached data (fast)
    Redis-->>Web: Return cached data
    
    %% Complex Queries Use PostgreSQL
    User->>Web: Request detailed report
    Web->>Postgres: Complex SQL query
    Postgres-->>Web: Relational data
    Web-->>User: Render detailed view
    
    %% Manual Refresh
    User->>Web: Click refresh button
    Web->>Worker: Trigger background refresh
    Worker->>JIRA: Fetch latest data
    JIRA-->>Worker: Updated data
    Worker->>Redis: Update cache
    Worker->>Postgres: Update persistent storage
    Worker-->>Web: Progress updates
    Web-->>User: Progress bar updates
```

## Updated Synchronisation Schedule

| Data Type | Frequency | Redis Storage | PostgreSQL Storage | Purpose |
|-----------|-----------|---------------|-------------------|---------|
| **Cases** | Every 15 minutes | ✅ JSON Cache | ✅ Cases Table | Fast access + persistence |
| **Cards** | Every hour (:21) | ✅ JSON Cache | ❌ | JIRA card status |
| **JIRA Comments** | Every hour (:21) | ✅ In cards JSON | ✅ JiraComment Table | Comment persistence + relationships |
| **Details** | Every 12 hours | ✅ JSON Cache | ✅ Cases Table Update | Extended case information |
| **Issues** | Every 12 hours | ✅ JSON Cache | ❌ | Bug tracking updates |
| **Stats** | Daily (4:40 AM) | ✅ JSON Cache | ❌ | Generate metrics |

## Hybrid Storage Architecture

```mermaid
graph TB
    subgraph "Data Sources"
        Portal["Red Hat Portal API"]
        JIRA["JIRA API"]
        BZ["Bugzilla API"]
    end

    subgraph "Processing Layer"
        Worker["Celery Worker<br/>Dual Write Logic"]
        Cache["cache.py<br/>Hybrid Caching"]
    end

    subgraph "Storage Layer"
        subgraph "Redis (Fast Access)"
            CasesCache["cases<br/>JSON"]
            CardsCache["cards<br/>JSON"]
            DetailsCache["details<br/>JSON"]
            StatsCache["stats<br/>JSON"]
        end
        
        subgraph "PostgreSQL (Persistence)"
            CasesTable[("Cases Table<br/>Structured Data")]
            CommentsTable[("Comments Table<br/>Case Comments")]
            JiraCommentsTable[("JIRA Comments<br/>JIRA Comments")]
        end
    end

    subgraph "Application Layer"
        Flask["Flask Application"]
        Models["SQLAlchemy Models"]
        DB["Database Session"]
    end

    Portal --> Worker
    JIRA --> Worker
    BZ --> Worker
    
    Worker --> Cache
    Cache --> CasesCache
    Cache --> CardsCache
    Cache --> DetailsCache
    Cache --> StatsCache
    
    Cache --> CasesTable
    Cache --> CommentsTable
    Cache --> JiraCommentsTable
    
    Flask --> CasesCache
    Flask --> CardsCache
    Flask --> CasesTable
    Flask --> JiraCommentsTable
    
    Models --> DB
    DB --> CasesTable
    DB --> CommentsTable
    DB --> JiraCommentsTable
```

## Database Schema with Relationships

```mermaid
erDiagram
    CASES {
        varchar case_number PK
        timestamp created_date PK
        varchar owner
        integer severity
        varchar account
        varchar summary
        varchar status
        timestamp last_update
        text description
        varchar product
        varchar product_version
        varchar fe_jira_card
    }
    
    COMMENTS {
        serial id PK
        varchar case_number FK
        timestamp created_date FK
        varchar author
        text comment_text
        date commented_at
    }
    
    JIRA_COMMENTS {
        varchar jira_comment_id PK
        varchar case_number FK
        timestamp created_date FK
        varchar author
        text body
        timestamp last_update_date
    }

    CASES ||--o{ COMMENTS : "case_number + created_date"
    CASES ||--o{ JIRA_COMMENTS : "case_number + created_date"
```

## Data Consistency Strategy

### Write Operations
```mermaid
graph TD
    Input[Data from External API] --> Validate{Validate Data}
    Validate -->|Valid| RedisWrite[Write to Redis]
    Validate -->|Invalid| Error[Log Error & Skip]
    
    RedisWrite --> PostgresWrite[Write to PostgreSQL]
    PostgresWrite --> CheckFK{Check Foreign Keys}
    CheckFK -->|Valid| Commit[Commit Transaction]
    CheckFK -->|Invalid| Rollback[Rollback & Log]
    
    Commit --> Success[Operation Complete]
    Rollback --> Error
    Error --> Retry{Retry Logic}
    Retry -->|Retry| Validate
    Retry -->|Max Retries| Fail[Operation Failed]
```

### Read Operations
```mermaid
graph TD
    Query[User Query] --> Type{Query Type}
    Type -->|Fast Lookup| Redis[Query Redis Cache]
    Type -->|Complex Query| Postgres[Query PostgreSQL]
    Type -->|Reporting| Postgres
    
    Redis --> CacheHit{Cache Hit?}
    CacheHit -->|Yes| Return[Return Data]
    CacheHit -->|No| Fallback[Fallback to PostgreSQL]
    
    Postgres --> SQLQuery[Execute SQL Query]
    SQLQuery --> Return
    Fallback --> SQLQuery
```

## Performance Benefits

### Dual Storage Advantages
- **Redis Benefits**: Sub-millisecond response times for frequent queries
- **PostgreSQL Benefits**: Complex queries, joins, and data integrity
- **Resilience**: Data persistence across container restarts
- **Scalability**: Optimised storage for different use cases

### Health Monitoring
- **PostgreSQL Health**: `pg_isready` checks database availability
- **Redis Health**: `redis-cli ping` verifies cache connectivity
- **Service Dependencies**: All services wait for database health checks

### Data Integrity Features
- **ACID Transactions**: PostgreSQL ensures data consistency
- **Foreign Key Constraints**: Maintains referential integrity
- **Composite Primary Keys**: Handles duplicate case numbers with different creation dates
- **Audit Trail**: Full history preserved in PostgreSQL

## Error Handling and Recovery

```mermaid
graph TD
    Task[Background Task] --> HealthCheck{Services Healthy?}
    HealthCheck -->|No| Wait[Wait for Health Check]
    HealthCheck -->|Yes| Fetch[Fetch Data from API]
    Wait --> HealthCheck
    
    Fetch --> RedisWrite{Redis Write OK?}
    RedisWrite -->|Yes| PostgresWrite{PostgreSQL Write OK?}
    RedisWrite -->|No| RetryRedis[Retry Redis Write]
    
    PostgresWrite -->|Yes| Success[Task Complete]
    PostgresWrite -->|No| RetryPostgres[Retry PostgreSQL Write]
    
    RetryRedis --> MaxRetries{Max Retries?}
    RetryPostgres --> MaxRetries
    MaxRetries -->|No| Fetch
    MaxRetries -->|Yes| Alert[Send Alert & Log]
```

## Migration Benefits

### From Redis-Only to Hybrid Model
1. **Data Durability**: Survive container restarts and crashes
2. **Query Flexibility**: SQL queries for complex reporting
3. **Data Relationships**: Proper foreign key relationships
4. **Backup Strategy**: Standard PostgreSQL backup procedures
5. **Audit Capabilities**: Complete data change history
6. **Performance**: Best of both worlds - speed + persistence

This hybrid approach maintains the original performance characteristics whilst adding robust data persistence and relationship management capabilities. 