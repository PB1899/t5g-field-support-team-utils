# Redis Data Loading Visualization

## Architecture Overview

```mermaid
graph TB
    subgraph "Celery Scheduled Tasks"
        CELERY[taskmgr.cache_data]
    end

    subgraph "Cache Operations"
        GET_CASES[cache.get_cases]
        GET_CARDS[cache.get_cards]
        GET_DETAILS[cache.get_case_details]
        GET_BZ[cache.get_bz_details]
        GET_ISSUES[cache.get_issue_details]
        GET_ESC[cache.get_escalations]
        GET_STATS[cache.get_stats]
    end

    subgraph "Data Storage"
        REDIS[(Redis Cache)]
        POSTGRES[(PostgreSQL DB)]
    end

    subgraph "External APIs"
        PORTAL[Red Hat Portal API]
        JIRA[Jira API]
        BUGZILLA[Bugzilla API]
    end

    CELERY -->|"cases"| GET_CASES
    CELERY -->|"cards"| GET_CARDS
    CELERY -->|"details"| GET_DETAILS
    CELERY -->|"bugs"| GET_BZ
    CELERY -->|"issues"| GET_ISSUES
    CELERY -->|"escalations"| GET_ESC
    CELERY -->|"stats"| GET_STATS

    GET_CASES -->|fetch| PORTAL
    GET_CASES -->|write| REDIS
    GET_CASES -->|write| POSTGRES

    GET_CARDS -->|fetch| JIRA
    GET_CARDS -->|read| REDIS
    GET_CARDS -->|write| REDIS
    GET_CARDS -->|write| POSTGRES

    GET_DETAILS -->|fetch| PORTAL
    GET_DETAILS -->|read| REDIS
    GET_DETAILS -->|write| REDIS

    GET_BZ -->|fetch| BUGZILLA
    GET_BZ -->|read| REDIS
    GET_BZ -->|write| REDIS

    GET_ISSUES -->|fetch| PORTAL
    GET_ISSUES -->|fetch| JIRA
    GET_ISSUES -->|read| REDIS
    GET_ISSUES -->|write| REDIS

    GET_ESC -->|fetch| JIRA
    GET_ESC -->|write| REDIS

    GET_STATS -->|read| REDIS
    GET_STATS -->|write| REDIS
```

## Detailed Call Flow: Cases to Redis

```mermaid
graph TD
    START[cache.get_cases] --> TOKEN[libtelco5g.get_token]
    TOKEN --> HEADERS[make_headers]
    HEADERS --> FETCH[requests.get Portal API]
    FETCH --> PARSE[Parse JSON response]
    PARSE --> PG{Load to Postgres}

    PG --> SESSION[db_config.SessionLocal]
    SESSION --> PARSEDATE[parser.parse dates]
    PARSEDATE --> QUERY[session.query Case]
    QUERY --> ADDMERGE{Exists?}
    ADDMERGE -->|No| ADD[session.add]
    ADDMERGE -->|Yes| MERGE[session.merge]
    ADD --> COMMIT[session.commit]
    MERGE --> COMMIT

    COMMIT --> REDIS[libtelco5g.redis_set]
    REDIS --> REDISCONN[redis.Redis host=redis]
    REDISCONN --> MSET[mset key: cases]
    MSET --> END[Complete]
```

## Detailed Call Flow: Cards to Redis

```mermaid
graph TD
    START[cache.get_cards] --> CACHED[_get_cached_data]
    CACHED --> R1[redis_get cases]
    CACHED --> R2[redis_get bugs]
    CACHED --> R3[redis_get issues]
    CACHED --> R4[redis_get escalations]
    CACHED --> R5[redis_get details]

    START --> JCONN[libtelco5g.jira_connection]
    JCONN --> GETLIST[_get_jira_cards_list]

    GETLIST --> PROJ[get_project_id]
    GETLIST --> BOARD[get_board_id]
    GETLIST --> SPRINT[get_latest_sprint]
    GETLIST --> QUERY[search_issues]

    QUERY --> LOOP{For each card}
    LOOP --> BUILD[_build_card_data]

    BUILD --> ISSUE[_get_jira_issue_with_retry]
    BUILD --> COMM[_get_card_comments]
    BUILD --> ASSIGN[_get_assignee_info]
    BUILD --> CONTRIB[_get_contributor_info]
    BUILD --> BUG[_get_bug_info]
    BUILD --> ESC[_get_escalation_info]
    BUILD --> DETAIL[_get_case_detail_info]
    BUILD --> LABEL[_get_label_flags]

    BUILD --> LOADPG[load_jira_card_postgres]
    LOADPG --> PGSESS[SessionLocal]
    LOADPG --> PGQUERY[query JiraCard]
    LOADPG --> PGADD[add or update]
    LOADPG --> COMMLOOP{For each comment}
    COMMLOOP --> PGCOMM[add/merge JiraComment]
    PGCOMM --> COMMLOOP
    LOADPG --> PGCOMMIT[session.commit]

    LOOP --> LOOP
    LOOP --> SAVE[redis_set cards]
    SAVE --> TIMESTAMP[redis_set timestamp]
    TIMESTAMP --> END[Complete]
```

## Detailed Call Flow: Issues to Redis

```mermaid
graph TD
    START[cache.get_issue_details] --> GETCASES[redis_get cases]
    START --> SETUP[_setup_issue_processing]

    SETUP --> TOKEN[get_token]
    SETUP --> HEADERS[make_headers]
    SETUP --> JCONN[jira_connection]

    START --> LOOP{For each open case}
    LOOP --> PROCESS[_process_case_issues]

    PROCESS --> GETAPI[_get_case_issues_from_api]
    GETAPI --> FETCH[requests.get issues]

    PROCESS --> ISSUELOOP{For each issue}
    ISSUELOOP --> SINGLE[_process_single_jira_issue]

    SINGLE --> JISSUE[jira_conn.issue]
    SINGLE --> EXTRACT[_extract_jira_fields]

    EXTRACT --> QA[_extract_qa_contact]
    EXTRACT --> SEV[_extract_jira_severity]
    EXTRACT --> TYPE[_extract_jira_type]
    EXTRACT --> ASSIGNEE[_extract_assignee_email]
    EXTRACT --> FIX[_extract_fix_versions]
    EXTRACT --> PRIO[_extract_priority]
    EXTRACT --> PRIV[_extract_private_keywords]

    SINGLE --> DATE[format_date]
    ISSUELOOP --> ISSUELOOP

    LOOP --> LOOP
    LOOP --> REDIS[redis_set issues]
    REDIS --> END[Complete]
```

## Data Flow Timeline

```mermaid
gantt
    title Redis Data Loading Schedule
    dateFormat HH:mm
    axisFormat %H:%M

    section Every Hour
    Cases (every 15 min) :milestone, m1, 00:00, 0min
    Cases (every 15 min) :milestone, m2, 00:15, 0min
    Cases (every 15 min) :milestone, m3, 00:30, 0min
    Cases (every 15 min) :milestone, m4, 00:45, 0min
    Cards (every hour :21) :milestone, m5, 00:21, 0min

    section Twice Daily
    Details (:24) :milestone, m6, 00:24, 0min
    Details (:24) :milestone, m7, 12:24, 0min
    Bugs (:48) :milestone, m8, 00:48, 0min
    Bugs (:48) :milestone, m9, 12:48, 0min
    Issues (:54) :milestone, m10, 00:54, 0min
    Issues (:54) :milestone, m11, 12:54, 0min

    section Every 2 Hours
    Escalations (:37) :milestone, m12, 00:37, 0min
    Escalations (:37) :milestone, m13, 02:37, 0min

    section Daily
    Stats (04:40) :milestone, m14, 04:40, 0min
```

## Redis Keys and Data Structure

```mermaid
graph LR
    subgraph "Redis Cache Structure"
        K1[cases]
        K2[cards]
        K3[details]
        K4[case_bz]
        K5[bugs]
        K6[issues]
        K7[escalations]
        K8[stats]
        K9[timestamp]
        K10[last_choice]
        K11[refresh_id]
    end

    subgraph "Data Format"
        JSON[JSON Strings]
    end

    K1 --> JSON
    K2 --> JSON
    K3 --> JSON
    K4 --> JSON
    K5 --> JSON
    K6 --> JSON
    K7 --> JSON
    K8 --> JSON
    K9 --> JSON
    K10 --> JSON
    K11 --> JSON
```

## Dependencies Between Data Types

```mermaid
graph TD
    CASES[cases] --> CARDS[cards]
    CASES --> DETAILS[details]
    CASES --> ISSUES[issues]

    DETAILS --> CASEBZ[case_bz]
    CASEBZ --> BUGS[bugs]

    CASES --> ESCALATIONS[escalations]

    CARDS --> STATS[stats]
    CASES --> STATS
    BUGS --> STATS
    ISSUES --> STATS

    style CASES fill:#e1f5ff
    style CARDS fill:#e1f5ff
    style DETAILS fill:#ffe1e1
    style BUGS fill:#ffe1e1
    style ISSUES fill:#ffe1e1
    style ESCALATIONS fill:#e1ffe1
    style STATS fill:#fff5e1
```

## Component Interaction Sequence

```mermaid
sequenceDiagram
    participant Celery
    participant Cache
    participant LibTelco5g
    participant Redis
    participant Postgres
    participant PortalAPI
    participant JiraAPI

    Note over Celery: Scheduled Task Triggers

    Celery->>Cache: cache_data("cases")
    Cache->>LibTelco5g: get_token()
    Cache->>PortalAPI: GET /search/cases
    PortalAPI-->>Cache: cases JSON
    Cache->>Postgres: load_cases_postgres()
    Postgres-->>Cache: OK
    Cache->>LibTelco5g: redis_set("cases", data)
    LibTelco5g->>Redis: mset({cases: data})
    Redis-->>LibTelco5g: OK

    Celery->>Cache: cache_data("cards")
    Cache->>LibTelco5g: redis_get("cases")
    Redis-->>Cache: cached cases
    Cache->>JiraAPI: search_issues()
    JiraAPI-->>Cache: cards list
    loop For each card
        Cache->>JiraAPI: get issue details
        JiraAPI-->>Cache: issue data
        Cache->>Postgres: load_jira_card_postgres()
    end
    Cache->>LibTelco5g: redis_set("cards", data)
    LibTelco5g->>Redis: mset({cards: data})
```

## Function Call Depth Analysis

```mermaid
graph LR
    subgraph "Depth 0: Entry"
        D0[cache_data]
    end

    subgraph "Depth 1: Main Functions"
        D1A[get_cases]
        D1B[get_cards]
        D1C[get_issue_details]
    end

    subgraph "Depth 2: Helper Functions"
        D2A[get_token]
        D2B[_get_jira_cards_list]
        D2C[_process_case_issues]
    end

    subgraph "Depth 3: Data Processing"
        D3A[_build_card_data]
        D3B[_process_single_jira_issue]
        D3C[load_cases_postgres]
    end

    subgraph "Depth 4: Field Extraction"
        D4A[_extract_jira_fields]
        D4B[_get_assignee_info]
        D4C[session.query]
    end

    subgraph "Depth 5: Atomic Operations"
        D5A[_extract_qa_contact]
        D5B[session.add]
        D5C[redis_set]
    end

    D0 --> D1A
    D0 --> D1B
    D0 --> D1C

    D1A --> D2A
    D1B --> D2B
    D1C --> D2C

    D2B --> D3A
    D2C --> D3B
    D1A --> D3C

    D3A --> D4A
    D3B --> D4A
    D3C --> D4C

    D4A --> D5A
    D4C --> D5B
    D3C --> D5C
```
