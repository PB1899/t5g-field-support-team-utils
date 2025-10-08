# Redis Data Loading Call Graph

## Overview
This document maps all functions called when data is loaded into Redis in the t5g-field-support-team-utils dashboard application.

## Entry Points (Scheduled Tasks via Celery)

The main entry point for Redis data loading is `taskmgr.cache_data()` which is scheduled via Celery periodic tasks:

```
taskmgr.cache_data() [scheduled tasks]
├── For "cases": cache.get_cases()
├── For "cards": cache.get_cards()
├── For "details": cache.get_case_details()
├── For "bugs": cache.get_bz_details()
├── For "issues": cache.get_issue_details()
└── For "escalations": cache.get_escalations()
```

## 1. Cases → Redis

**Entry Function:** `cache.get_cases(cfg)` (cache.py:18)

```
cache.get_cases(cfg)
├── libtelco5g.get_token(cfg["offline_token"])
├── utils.make_headers(token)
├── requests.get(url, headers=headers, params=payload)
├── database.load_cases_postgres(cases) [database/operations.py:14]
│   ├── db_config.SessionLocal()
│   ├── dateutil.parser.parse() [for dates]
│   ├── session.query(Case).where()
│   ├── session.add() or session.merge()
│   └── session.commit()
└── libtelco5g.redis_set("cases", json.dumps(cases)) [libtelco5g.py:551]
    └── redis.Redis(host="redis").mset({key: value})
```

## 2. Cards → Redis

**Entry Function:** `cache.get_cards(cfg, self, background)` (cache.py:110)

```
cache.get_cards(cfg, self, background)
├── _get_cached_data()
│   ├── libtelco5g.redis_get("cases")
│   ├── libtelco5g.redis_get("bugs")
│   ├── libtelco5g.redis_get("issues")
│   ├── libtelco5g.redis_get("escalations")
│   └── libtelco5g.redis_get("details")
├── libtelco5g.jira_connection(cfg) [libtelco5g.py:58]
│   └── jira.JIRA(server, token_auth)
├── _get_jira_cards_list(cfg, jira_conn)
│   ├── libtelco5g.get_project_id(jira_conn, cfg["project"])
│   ├── libtelco5g.get_board_id(jira_conn, cfg["board"])
│   ├── libtelco5g.get_latest_sprint(jira_conn, board.id, cfg["sprintname"])
│   └── _execute_jira_query_with_retry(jira_conn, jira_query, cfg, max_cards)
│       └── jira_conn.search_issues()
├── For each card: _build_card_data(card, jira_conn, cases, bugs, issues, escalations, details, time_now, cfg)
│   ├── _get_jira_issue_with_retry(jira_conn, card, cfg)
│   │   └── jira_conn.issue(issue_key)
│   ├── _get_card_comments(issue.fields.comment.comments)
│   │   └── utils.format_comment(comment)
│   ├── _get_assignee_info(issue)
│   ├── _get_contributor_info(issue)
│   ├── _get_bug_info(case_number, case_data, bugs)
│   ├── _get_escalation_info(case_number, escalations, case_issues, labels, cfg)
│   ├── _get_case_detail_info(case_number, details)
│   ├── _get_label_flags(issue.fields.labels, escalated)
│   └── database.load_jira_card_postgres(cases, case_number, card) [database/operations.py:55]
│       ├── db_config.SessionLocal()
│       ├── dateutil.parser.parse()
│       ├── session.query(JiraCard).filter_by(jira_card_id=issue.key).first()
│       ├── session.add(jira_card) [if new] or existing logic
│       ├── For each comment:
│       │   ├── utils.format_comment(comment)
│       │   ├── session.query(JiraComment).filter_by(jira_comment_id=comment.id).first()
│       │   └── session.add() or session.merge()
│       └── session.commit()
├── libtelco5g.redis_set("cards", json.dumps(jira_cards)) [libtelco5g.py:551]
└── libtelco5g.redis_set("timestamp", json.dumps(str(datetime.datetime.now())))
```

## 3. Case Details → Redis

**Entry Function:** `cache.get_case_details(cfg)` (cache.py:404)

```
cache.get_case_details(cfg)
├── libtelco5g.redis_get("cases")
├── libtelco5g.get_token(cfg["offline_token"])
├── utils.make_headers(token)
├── For each open case:
│   ├── requests.get(case_endpoint, headers=headers)
│   └── Extract: crit_sit, group_name, notified_users, relief_at, resolved_at, bugzillas
├── libtelco5g.redis_set("details", json.dumps(case_details))
└── libtelco5g.redis_set("case_bz", json.dumps(bz_dict))
```

## 4. Bugzilla Details → Redis

**Entry Function:** `cache.get_bz_details(cfg)` (cache.py:446)

```
cache.get_bz_details(cfg)
├── libtelco5g.redis_get("case_bz")
├── bugzilla.Bugzilla(bz_url, api_key=cfg["bz_key"])
├── For each bug in each case:
│   ├── bz_api.getbug(bug["bugzillaNumber"])
│   └── Extract: target_release, assignee, last_change_time, internal_whiteboard, qa_contact, severity
└── libtelco5g.redis_set("bugs", json.dumps(bz_dict))
```

## 5. Jira Issues → Redis

**Entry Function:** `cache.get_issue_details(cfg)` (cache.py:488)

```
cache.get_issue_details(cfg)
├── libtelco5g.redis_get("cases")
├── _setup_issue_processing(cfg)
│   ├── libtelco5g.get_token(cfg["offline_token"])
│   ├── utils.make_headers(token)
│   └── libtelco5g.jira_connection(cfg)
├── For each open case: _process_case_issues(case, cfg, token, headers, jira_conn)
│   ├── _get_case_issues_from_api(case, cfg, token, headers)
│   │   └── requests.get(issues_url, headers=headers)
│   └── For each issue: _process_single_jira_issue(issue, jira_conn)
│       ├── jira_conn.issue(issue["resourceKey"])
│       ├── _extract_jira_fields(bug)
│       │   ├── _extract_qa_contact(bug)
│       │   ├── _extract_jira_severity(bug)
│       │   ├── _extract_jira_type(bug)
│       │   ├── _extract_assignee_email(bug)
│       │   ├── _extract_fix_versions(bug)
│       │   ├── _extract_priority(bug)
│       │   └── _extract_private_keywords(bug)
│       └── utils.format_date(issue["lastModifiedDate"])
└── libtelco5g.redis_set("issues", json.dumps(jira_issues))
```

## 6. Escalations → Redis

**Entry Function:** `cache.get_escalations(cfg, cases)` (cache.py:73)

```
cache.get_escalations(cfg, cases)
├── libtelco5g.jira_connection(cfg)
├── libtelco5g.get_project_id(jira_conn, cfg["jira_escalations_project"])
├── jira_conn.search_issues(jira_query, 0, max_cards).iterable
├── For each escalated card:
│   ├── jira_conn.issue(card)
│   └── Extract: issue.fields.customfield_12313441 (SDFC Case Links)
└── Returns escalations list
    └── [Stored via taskmgr.cache_data()] libtelco5g.redis_set("escalations", json.dumps(escalations))
```

## 7. Stats → Redis

**Entry Function:** `cache.get_stats()` (cache.py:695)

```
cache.get_stats()
├── libtelco5g.redis_get("stats")
├── libtelco5g.generate_stats() [libtelco5g.py:586]
│   ├── libtelco5g.redis_get("cards")
│   ├── libtelco5g.redis_get("cases")
│   ├── libtelco5g.redis_get("bugs")
│   ├── libtelco5g.redis_get("issues")
│   └── Calculate statistics: by_customer, by_engineer, by_severity, by_status, etc.
└── libtelco5g.redis_set("stats", json.dumps(all_stats))
```

## Core Redis Functions

### redis_set() (libtelco5g.py:551)
```python
def redis_set(key, value):
    logging.warning("syncing {}..".format(key))
    r_cache = redis.Redis(host="redis")
    r_cache.mset({key: value})
    logging.warning("{}....synced".format(key))
```

### redis_get() (libtelco5g.py:558)
```python
def redis_get(key):
    logging.warning("fetching {}..".format(key))
    r_cache = redis.Redis(host="redis")
    try:
        data = r_cache.get(key)
    except redis.exceptions.ConnectionError:
        logging.warning("Couldn't connect to redis host, setting data to None")
        data = None
    if data is not None:
        data = json.loads(data.decode("utf-8"))
    else:
        data = {}
    logging.warning("{} ....fetched".format(key))
    return data
```

## Scheduling Information (taskmgr.py)

All Redis data loading is scheduled via Celery periodic tasks:

| Data Type | Schedule | Task Function |
|-----------|----------|---------------|
| cases | Every 15 minutes | `cache_data.s("cases")` |
| cards | Every hour at :21 | `cache_data.s("cards")` |
| details | Every 12 hours at :24 | `cache_data.s("details")` |
| bugs | Every 12 hours at :48 | `cache_data.s("bugs")` |
| issues | Every 12 hours at :54 | `cache_data.s("issues")` |
| escalations | Every 2 hours at :37 | `cache_data.s("escalations")` |
| stats | Daily at 4:40 AM | `cache_stats.s()` |

## Summary

All Redis write operations ultimately call `libtelco5g.redis_set(key, value)` which:
1. Serializes Python objects to JSON using `json.dumps()`
2. Stores the JSON string in Redis using `redis.Redis(host="redis").mset({key: value})`

All data is stored as JSON strings in Redis and deserialized when retrieved via `libtelco5g.redis_get(key)`.
