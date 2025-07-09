# T5G Data Flow and Synchronization Process

## Data Synchronization Flow

This diagram shows how data flows through the T5G system and the timing of various synchronization processes.

```mermaid
sequenceDiagram
    participant Portal as Red Hat Portal
    participant JIRA as JIRA Server
    participant BZ as Bugzilla
    participant Celery as Celery Beat
    participant Worker as Celery Worker
    participant Redis as Redis Cache
    participant Web as Flask Web App
    participant User as End User

    %% Scheduled Data Synchronization
    Note over Celery: Every 15 minutes
    Celery->>Worker: Trigger case_sync
    Worker->>Portal: Fetch support cases
    Portal-->>Worker: Cases JSON data
    Worker->>Redis: Store cases

    Note over Celery: Every hour at :21
    Celery->>Worker: Trigger card_sync
    Worker->>JIRA: Fetch JIRA cards
    JIRA-->>Worker: Cards data
    Worker->>Redis: Store cards

    Note over Celery: Every 12 hours
    Celery->>Worker: Trigger details_sync
    Worker->>Portal: Fetch case details
    Portal-->>Worker: Case details
    Worker->>Redis: Store details

    Note over Celery: Every 12 hours
    Celery->>Worker: Trigger bugs_sync
    Worker->>BZ: Fetch bug data
    BZ-->>Worker: Bug information
    Worker->>Redis: Store bugs

    Note over Celery: Every hour at :10
    Celery->>Worker: Trigger portal_jira_sync
    Worker->>Portal: Check for new cases
    Worker->>JIRA: Create new cards
    JIRA-->>Worker: Card creation response
    Worker->>Redis: Update cache

    %% User Interaction
    User->>Web: Request dashboard
    Web->>Redis: Fetch cached data
    Redis-->>Web: Return data
    Web-->>User: Render dashboard

    %% Manual Refresh
    User->>Web: Click refresh button
    Web->>Worker: Trigger background refresh
    Worker->>JIRA: Fetch latest data
    JIRA-->>Worker: Updated data
    Worker->>Redis: Update cache
    Worker-->>Web: Progress updates
    Web-->>User: Progress bar updates
```

## Synchronization Schedule

| Data Type | Frequency | Purpose |
|-----------|-----------|---------|
| **Cases** | Every 15 minutes | Keep support cases current |
| **Cards** | Every hour (:21) | Update JIRA ticket status |
| **Details** | Every 12 hours | Extended case information |
| **Issues** | Every 12 hours | Bug tracking updates |
| **Bugs** | Every 12 hours | Bugzilla synchronization |
| **Escalations** | Every 2 hours | High-priority monitoring |
| **Stats** | Daily (4:40 AM) | Generate metrics |
| **Portal→JIRA Sync** | Every hour (:10) | Create new JIRA cards |

## Cache Strategy

```mermaid
graph LR
    subgraph "Redis Cache Structure"
        Cases["cases<br/>Support Case Data"]
        Cards["cards<br/>JIRA Card Data"]
        Details["details<br/>Extended Case Info"]
        Issues["issues<br/>Bug Reports"]
        Bugs["bugs<br/>Bugzilla Data"]
        Escalations["escalations<br/>High Priority Cases"]
        Stats["stats<br/>Historical Metrics"]
        Users["users<br/>SAML User Data"]
    end

    subgraph "Data Sources"
        Portal["Red Hat Portal API"]
        JIRA["JIRA API"]
        BZ["Bugzilla API"]
        SAML["SAML Provider"]
    end

    Portal --> Cases
    Portal --> Details
    JIRA --> Cards
    JIRA --> Issues
    BZ --> Bugs
    Cases --> Escalations
    Cards --> Stats
    SAML --> Users
```

## Data Relationships

```mermaid
erDiagram
    CASES {
        string case_number PK
        string owner
        string severity
        string account
        string problem
        string status
        datetime createdate
        datetime last_update
        string description
        string product
        string bug FK
        array tags
        datetime closeddate
    }
    
    CARDS {
        string card_key PK
        string case_number FK
        string card_status
        datetime card_created
        string assignee
        array contributor
        array comments
        string priority
        string severity
        boolean escalated
        array labels
    }
    
    DETAILS {
        string case_number PK
        boolean crit_sit
        string group_name
        array notified_users
        datetime relief_at
        datetime resolved_at
    }
    
    BUGS {
        string bug_id PK
        string case_number FK
        string target_release
        string assignee
        datetime last_change_time
        string internal_whiteboard
        string qa_contact
        string severity
    }
    
    ESCALATIONS {
        string case_number PK
        string escalation_type
        string escalation_link
        datetime escalated_at
        string reason
    }

    CASES ||--o| CARDS : "generates"
    CASES ||--o| DETAILS : "extends"
    CASES ||--o{ BUGS : "references"
    CASES ||--o| ESCALATIONS : "escalates_to"
```

## Performance Considerations

### Caching Benefits
- **Reduced API Calls**: Cached data reduces external API load
- **Fast Response Times**: Redis provides sub-millisecond data access
- **Offline Capability**: Dashboard works even if external APIs are down
- **Concurrent Access**: Multiple users can access cached data simultaneously

### Background Processing
- **Non-blocking Updates**: Data synchronization doesn't impact user experience
- **Scalable Workers**: Multiple Celery workers can process tasks in parallel
- **Retry Logic**: Failed tasks are automatically retried with backoff
- **Progress Tracking**: Real-time progress updates for manual refreshes

### Data Consistency
- **Distributed Locking**: Prevents concurrent cache updates
- **Atomic Operations**: Redis operations ensure data integrity
- **Timestamping**: All cached data includes last update timestamps
- **Conflict Resolution**: Newer data always overwrites older data

## Error Handling

```mermaid
graph TD
    Task[Background Task] --> Check{API Available?}
    Check -->|Yes| Fetch[Fetch Data]
    Check -->|No| Retry[Retry with Backoff]
    Fetch --> Success{Success?}
    Success -->|Yes| Cache[Update Cache]
    Success -->|No| Retry
    Retry --> MaxRetries{Max Retries?}
    MaxRetries -->|No| Check
    MaxRetries -->|Yes| Alert[Send Alert]
    Cache --> Complete[Task Complete]
    Alert --> Complete
```

## Monitoring and Alerting

- **Flower Dashboard**: Real-time task monitoring
- **Prometheus Metrics**: Application performance metrics
- **Email Alerts**: Notification for high-priority issues
- **Slack Integration**: Team notifications for critical events
- **Health Checks**: Regular system health monitoring 