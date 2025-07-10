# T5G Field Support Team Utils - PostgreSQL Architecture Diagram

## System Overview
This diagram illustrates the updated architecture with PostgreSQL integration, showing the hybrid Redis + PostgreSQL data storage approach for enhanced persistence and data integrity.

## Updated Architecture Diagram

```mermaid
graph TB
    %% External Data Sources
    subgraph "External APIs"
        Portal["Red Hat Customer Portal<br/>Cases & Support Data"]
        JIRA["JIRA<br/>Issue Tracking"]
        BZ["Bugzilla<br/>Bug Reports"]
        SAML["SAML Identity Provider<br/>Authentication"]
    end

    %% Core Application Layer
    subgraph "T5G Dashboard Application"
        subgraph "Flask Web Application"
            UI["UI Blueprint<br/>Web Interface"]
            API["API Blueprint<br/>REST Endpoints"]
            Auth["Authentication Layer<br/>SAML + RBAC"]
            App["Flask App Factory<br/>create_app()"]
        end

        subgraph "Business Logic"
            Core["t5gweb.py<br/>Core Functions"]
            LibT5G["libtelco5g.py<br/>Integration Logic"]
            Utils["utils.py<br/>Helper Functions"]
        end

        subgraph "Data Layer"
            Cache["cache.py<br/>Hybrid Data Caching"]
            Models["models.py<br/>SQLAlchemy Models"]
            DB["db.py<br/>Database Setup"]
            Deps["dependencies.py<br/>Dependency Injection"]
        end

        subgraph "Data Storage"
            Redis[("Redis Cache<br/>Fast Access")]
            Postgres[("PostgreSQL Database<br/>Persistent Storage")]
        end
    end

    %% Background Processing
    subgraph "Background Processing"
        Celery["Celery Worker<br/>Async Tasks"]
        Beat["Celery Beat<br/>Scheduler"]
        Flower["Flower<br/>Task Monitor"]
        Tasks["taskmgr.py<br/>Task Definitions"]
    end

    %% Frontend Components
    subgraph "Frontend"
        Templates["Jinja2 Templates<br/>UI Pages"]
        Static["Static Assets<br/>CSS/JS/Bootstrap"]
        DataTables["DataTables<br/>Interactive Tables"]
        Charts["Charts & Visualizations"]
    end

    %% Deployment
    subgraph "Deployment"
        Docker["Docker Containers"]
        Compose["docker-compose<br/>PostgreSQL + Redis + App"]
        Gunicorn["Gunicorn<br/>WSGI Server"]
        Prometheus["Prometheus<br/>Metrics"]
    end

    %% Data Flow Connections
    Portal --> Cache
    JIRA --> Cache
    BZ --> Cache
    SAML --> Auth

    %% Hybrid Data Storage
    Cache --> Redis
    Cache --> Postgres
    Models --> Postgres
    DB --> Postgres
    Deps --> DB

    Redis --> Core
    Postgres --> Core
    Core --> UI
    Core --> API

    Tasks --> Celery
    Beat --> Celery
    Celery --> Cache
    Celery --> Redis
    Celery --> Postgres
    Celery --> Portal
    Celery --> JIRA
    Celery --> BZ

    Auth --> UI
    UI --> Templates
    Templates --> Static
    Static --> DataTables
    UI --> Charts

    App --> UI
    App --> API
    App --> Auth
    LibT5G --> Cache
    Utils --> Core

    Docker --> App
    Compose --> Docker
    Gunicorn --> App
    Flower --> Celery
    Prometheus --> App

    %% Database Health Checks
    Postgres -.-> Docker
    Redis -.-> Docker

    %% Styling
    classDef external fill:#e1f5fe
    classDef core fill:#f3e5f5
    classDef data fill:#e8f5e8
    classDef storage fill:#fff3e0
    classDef background fill:#f1f8e9
    classDef frontend fill:#fce4ec
    classDef deployment fill:#f3e5f5

    class Portal,JIRA,BZ,SAML external
    class UI,API,Auth,App,Core,LibT5G,Utils core
    class Cache,Models,DB,Deps data
    class Redis,Postgres storage
    class Celery,Beat,Flower,Tasks background
    class Templates,Static,DataTables,Charts frontend
    class Docker,Compose,Gunicorn,Prometheus deployment
```

## Key Architectural Changes

### PostgreSQL Integration
- **Hybrid Storage Model**: Data stored in both Redis (caching) and PostgreSQL (persistence)
- **SQLAlchemy Models**: Structured data models for Cases, Comments, and JIRA Comments
- **Database Health Checks**: PostgreSQL health monitoring in docker-compose
- **Dependency Injection**: Proper database session management

### Data Models
```mermaid
erDiagram
    CASES {
        string case_number PK
        datetime created_date PK
        string owner
        integer severity
        string account
        string summary
        string status
        datetime last_update
        string description
        string product
        string product_version
        string fe_jira_card
    }
    
    COMMENTS {
        integer id PK
        string case_number FK
        datetime created_date FK
        string author
        text comment_text
        date commented_at
    }
    
    JIRA_COMMENTS {
        string jira_comment_id PK
        string case_number FK
        datetime created_date FK
        string author
        text body
        datetime last_update_date
    }

    CASES ||--o{ COMMENTS : "has"
    CASES ||--o{ JIRA_COMMENTS : "has"
```

### Benefits of PostgreSQL Integration

#### **Data Persistence**
- **Durable Storage**: Data survives container restarts
- **ACID Compliance**: Reliable transactions and data integrity
- **Backup & Recovery**: Standard PostgreSQL backup procedures
- **Audit Trail**: Full history of data changes

#### **Performance Optimisation**
- **Redis for Speed**: Sub-millisecond access for frequent queries
- **PostgreSQL for Complex Queries**: Relational queries and reporting
- **Dual Write Strategy**: Best of both worlds - speed and persistence

#### **Data Relationships**
- **Foreign Key Constraints**: Data integrity between tables
- **Normalised Structure**: Efficient storage and querying
- **Comment History**: Full comment tracking for cases and JIRA items

### Docker Compose Updates

```yaml
postgresql:
  image: docker.io/postgres:latest
  environment:
    - POSTGRES_USER=postgres
    - POSTGRES_PASSWORD=secret
    - POSTGRES_DB=dashboard
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s
    timeout: 5s
    retries: 5
  ports:
   - "5432:5432"
```

### Dependencies Added
- **SQLAlchemy**: ORM for database operations
- **psycopg2-binary**: PostgreSQL adapter
- **Health Checks**: Service dependency management

## How to View This Diagram

### Option 1: Mermaid Live Editor (Recommended)
1. Go to https://mermaid.live/
2. Copy the mermaid code block above
3. Paste it into the editor for instant rendering

### Option 2: VS Code
1. Install "Markdown Preview Mermaid Support" extension
2. Open this file in VS Code
3. Use Ctrl+Shift+V to preview

### Option 3: GitHub/GitLab
Push this file to view with automatic Mermaid rendering 