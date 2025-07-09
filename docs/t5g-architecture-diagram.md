# T5G Field Support Team Utils - Architecture Diagram

## System Overview
This diagram illustrates the complete architecture of the T5G Field Support Team dashboard application, showing data flow, component relationships, and system integrations.

## Architecture Diagram

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
            Cache["cache.py<br/>Data Caching"]
            Redis[("Redis Cache<br/>JSON Data Store")]
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
        Compose["docker-compose<br/>Orchestration"]
        Gunicorn["Gunicorn<br/>WSGI Server"]
        Prometheus["Prometheus<br/>Metrics"]
    end

    %% Utilities
    subgraph "Utility Scripts"
        Scripts["bin/*.py<br/>CLI Tools"]
        FakeData["generate_fake_data.py"]
        Reports["Sprint Reports"]
        Watcher["Case Watcher"]
    end

    %% Data Flow Connections
    Portal --> Cache
    JIRA --> Cache
    BZ --> Cache
    SAML --> Auth

    Cache --> Redis
    Redis --> Core
    Core --> UI
    Core --> API

    Tasks --> Celery
    Beat --> Celery
    Celery --> Cache
    Celery --> Redis
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

    Scripts --> Portal
    Scripts --> JIRA
    FakeData --> Redis

    %% Styling
    classDef external fill:#e1f5fe
    classDef core fill:#f3e5f5
    classDef background fill:#e8f5e8
    classDef frontend fill:#fff3e0
    classDef deployment fill:#fce4ec
    classDef utility fill:#f1f8e9

    class Portal,JIRA,BZ,SAML external
    class UI,API,Auth,App,Core,LibT5G,Utils,Cache,Redis core
    class Celery,Beat,Flower,Tasks background
    class Templates,Static,DataTables,Charts frontend
    class Docker,Compose,Gunicorn,Prometheus deployment
    class Scripts,FakeData,Reports,Watcher utility
```

## Component Details

### External APIs (Blue)
- **Red Hat Customer Portal**: Source of support cases and customer data
- **JIRA**: Issue tracking and project management system
- **Bugzilla**: Bug tracking and resolution system
- **SAML Identity Provider**: Authentication and authorization

### Core Application (Purple)
- **Flask App Factory**: Application initialization and configuration
- **UI Blueprint**: Web interface and user-facing pages
- **API Blueprint**: REST endpoints for data access
- **Authentication Layer**: SAML SSO with role-based access control
- **Business Logic**: Core application functionality and data processing
- **Data Layer**: Caching system with Redis backend

### Background Processing (Green)
- **Celery Worker**: Asynchronous task execution
- **Celery Beat**: Scheduled task management
- **Flower**: Task monitoring and management interface
- **Task Manager**: Task definitions and scheduling logic

### Frontend (Orange)
- **Jinja2 Templates**: Server-side HTML templating
- **Static Assets**: CSS, JavaScript, and Bootstrap framework
- **DataTables**: Interactive data tables with sorting/filtering
- **Charts**: Data visualization and statistics

### Deployment (Pink)
- **Docker Containers**: Application containerization
- **docker-compose**: Development environment orchestration
- **Gunicorn**: WSGI server for production deployment
- **Prometheus**: Metrics collection and monitoring

### Utility Scripts (Light Green)
- **CLI Tools**: Command-line utilities in bin/ directory
- **Fake Data Generator**: Development and testing data creation
- **Sprint Reports**: Automated reporting tools
- **Case Watcher**: Monitoring and notification utilities

## Data Flow

1. **External APIs** feed data into the **Cache Layer** via scheduled background tasks
2. **Redis** stores cached data in JSON format for fast access
3. **Business Logic** processes cached data for presentation
4. **UI/API** serve data to users through web interface or REST endpoints
5. **Background Tasks** run periodic synchronization and maintenance operations

## Key Features

- **Real-time Data Synchronization**: Automated updates from multiple sources
- **Scalable Architecture**: Microservices approach with background processing
- **Responsive UI**: Modern web interface with interactive components
- **Comprehensive Monitoring**: Built-in metrics and task monitoring
- **Development-Friendly**: Docker-based development environment

## Deployment Architecture

The application uses a containerized architecture with:
- **Web Container**: Flask application with Gunicorn
- **Worker Container**: Celery background tasks
- **Scheduler Container**: Celery Beat for periodic tasks
- **Cache Container**: Redis for data storage
- **Monitor Container**: Flower for task monitoring

## Security

- **SAML Authentication**: Enterprise SSO integration
- **Role-Based Access Control**: Fine-grained permissions
- **API Token Management**: Secure external API access
- **Environment Configuration**: Sensitive data through environment variables 