# T5G Field Support Team Utils - Architecture Analysis

## Overview
This directory contains comprehensive documentation and architectural diagrams for the T5G Field Support Team dashboard application, a Flask-based web application used by Red Hat's Telco5G Field Support Team to manage customer support cases and JIRA tickets.

## Files in This Directory

### 📊 `t5g-architecture-diagram.md`
**Complete system architecture diagram** showing all components and their relationships:
- External API integrations (Red Hat Portal, JIRA, Bugzilla, SAML)
- Core Flask application structure
- Background processing with Celery
- Frontend components and deployment architecture
- Color-coded component categories for easy understanding

### 🔄 `t5g-data-flow-diagram.md`
**Detailed data flow and synchronization documentation** including:
- Sequence diagrams showing data synchronization timing
- Cache strategy and Redis data structure
- Entity-relationship diagrams
- Performance considerations and error handling
- Monitoring and alerting setup

## Key Application Characteristics

### 🎯 **Primary Purpose**
- **Customer Support Management**: Track and manage Red Hat Telco5G support cases
- **JIRA Integration**: Automated creation and synchronization of JIRA tickets
- **Team Dashboard**: Centralized view of support operations and metrics
- **Multi-System Integration**: Connects Red Hat Portal, JIRA, and Bugzilla

### 🏗️ **Architecture Highlights**
- **Flask Web Framework**: Modern Python web application
- **Microservices Approach**: Containerized components with Docker
- **Background Processing**: Celery for asynchronous tasks
- **Redis Caching**: High-performance data caching layer
- **SAML Authentication**: Enterprise SSO with role-based access control

### 📈 **Data Synchronization**
- **Real-time Updates**: Cases synchronized every 15 minutes
- **JIRA Cards**: Updated hourly with progress tracking
- **Background Tasks**: Scheduled synchronization with external APIs
- **Manual Refresh**: User-initiated background refresh capability

### 🚀 **Technology Stack**
- **Backend**: Flask, Celery, Redis, Python 3.x
- **Frontend**: Bootstrap, DataTables, Jinja2, JavaScript
- **Integration**: JIRA API, Bugzilla API, Red Hat Portal API
- **Deployment**: Docker, docker-compose, Gunicorn
- **Monitoring**: Prometheus, Flower, custom metrics

## Component Breakdown

### External Integrations
- **Red Hat Customer Portal**: Source of support cases and customer data
- **JIRA Server**: Issue tracking and project management
- **Bugzilla**: Bug tracking and resolution system
- **SAML Provider**: Authentication and user management

### Core Application
- **Flask Web App**: Main application with UI and API endpoints
- **Business Logic**: Core functions for data processing and presentation
- **Authentication**: SAML SSO with role-based access control
- **Caching Layer**: Redis-based data caching for performance

### Background Processing
- **Celery Workers**: Process background tasks and data synchronization
- **Celery Beat**: Schedule periodic tasks (cases, cards, stats)
- **Flower**: Task monitoring and management interface
- **Task Manager**: Defines and manages all background tasks

### Frontend Components
- **Responsive UI**: Bootstrap-based responsive design
- **Interactive Tables**: DataTables for sorting and filtering
- **Charts**: Data visualization and statistics
- **Real-time Updates**: Progress bars and status indicators

## Deployment Architecture

### Development Environment
```bash
cd dashboard
docker-compose up -d
```
- **Web Interface**: http://localhost:8080
- **Flower Monitor**: http://localhost:8000
- **Live Reload**: Changes reflected immediately
- **Fake Data**: Development mode with generated test data

### Production Components
- **Web Container**: Flask app with Gunicorn WSGI server
- **Worker Container**: Celery background task processing
- **Scheduler**: Celery Beat for periodic task scheduling
- **Cache**: Redis for data storage and message brokering
- **Monitor**: Flower for task monitoring and management

## Data Flow Summary

1. **External APIs** → **Background Tasks** → **Redis Cache**
2. **Redis Cache** → **Flask Application** → **User Interface**
3. **User Actions** → **API Endpoints** → **Background Tasks**
4. **Scheduled Tasks** → **Data Synchronization** → **Cache Updates**

## Performance Features

### Caching Strategy
- **Redis Backend**: Sub-millisecond data access
- **JSON Serialization**: Efficient data storage format
- **Distributed Locking**: Prevents concurrent update conflicts
- **Automatic Refresh**: Scheduled and manual cache updates

### Scalability
- **Horizontal Scaling**: Multiple Celery workers
- **Asynchronous Processing**: Non-blocking background tasks
- **Connection Pooling**: Efficient database connections
- **Load Balancing**: Docker container orchestration

## Security Implementation

### Authentication
- **SAML SSO**: Enterprise identity provider integration
- **Role-Based Access**: Fine-grained permission control
- **Session Management**: Secure user session handling
- **API Token Security**: Encrypted external API credentials

### Data Protection
- **Environment Variables**: Sensitive configuration management
- **HTTPS Enforcement**: Secure communication protocols
- **Input Validation**: Protection against injection attacks
- **Audit Logging**: Comprehensive activity tracking

## Monitoring and Observability

### Application Metrics
- **Prometheus Integration**: Performance metrics collection
- **Custom Metrics**: Application-specific monitoring
- **Health Checks**: System status monitoring
- **Error Tracking**: Comprehensive error logging

### Task Monitoring
- **Flower Dashboard**: Real-time task visibility
- **Task Status**: Progress tracking and error reporting
- **Queue Monitoring**: Background task queue management
- **Performance Analytics**: Task execution statistics

## Development Workflow

### Local Setup
1. Configure environment variables in `cfg/sample.env`
2. Run `docker-compose up -d` in dashboard directory
3. Access application at http://localhost:8080
4. Monitor tasks at http://localhost:8000

### Code Organization
- **Modular Design**: Clear separation of concerns
- **Blueprint Structure**: Organized Flask application structure
- **Configuration Management**: Environment-based configuration
- **Testing Framework**: Comprehensive test coverage

## Maintenance and Operations

### Regular Tasks
- **Cache Refresh**: Automated data synchronization
- **Metric Collection**: Daily statistics generation
- **Health Monitoring**: System status checks
- **Log Rotation**: Automated log management

### Troubleshooting
- **Flower Interface**: Task monitoring and debugging
- **Redis CLI**: Cache inspection and management
- **Application Logs**: Comprehensive error tracking
- **Health Endpoints**: System status verification

---

## Getting Started

To understand the application architecture:

1. **Start with**: `t5g-architecture-diagram.md` for overall system structure
2. **Deep dive into**: `t5g-data-flow-diagram.md` for synchronization details
3. **Explore**: The actual codebase with this architectural understanding

This documentation provides a complete picture of the T5G Field Support Team application's design, implementation, and operational characteristics. 