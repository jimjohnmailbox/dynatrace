Mastering Dynatrace Query Language (DQL) – Grok Expert Session
November 23, 2025

Role:
You are an expert Dynatrace tutor, specializing in Dynatrace Query Language (DQL) and advanced observability analytics. You are teaching an experienced software developer who wants deep knowledge, practical real-world usage, and advanced querying techniques.

Task:
Provide a detailed, structured, and example-driven explanation of Dynatrace Query Language (DQL).

Expert-Level Dynatrace Query Language (DQL) Learning Module

1. Introduction to DQL
Dynatrace Query Language (DQL) is the native, purpose-built query language for Dynatrace Grail – the causational, schema-on-read data lakehouse that stores all observability, security, and business data in its raw form.

Key characteristics that experienced developers appreciate:
• Unified analytics across logs, metrics, traces/spans, business events, topology, security events, and synthetic data – all in one language.
• Schema-on-read – no ETL or upfront schema required; structure is imposed at query time.
• Strongly typed at execution (types are inferred or explicitly cast).
• Pipeline architecture (| pipe) that streams data left-to-right – extremely efficient at scale.
• Heavily inspired by Kusto Query Language (KQL) – if you know KQL you are already 80 % there.
• Native support for parsing unstructured text, cross-dataset joins, time-series generation, statistical functions, and topology traversal.

DQL is the only language you need in modern Dynatrace (Notebooks, Dashboards, Workflows, Security Investigator, Log Observability, Metrics API, etc.).

2. Core Syntax & Query Pipeline Flow
Every valid DQL query follows this structure:

fetch <dataset> [, from:<timestamp>, to:<timestamp>, scanLimitGBytes:<num>]
| command1 ...
| command2 ...
| ...

• Data flows left → right.
• Each command receives a table (records + fields) and outputs a table.
• Early filtering / field projection = massive performance wins (Grail charges by scanned bytes).
• Default timeframe = selected timeframe in UI (or last 2 h).
• Use explicit timeframe when automating: from:now()-15m or from:timestamp("2025-11-01T00:00:00Z")

Most common datasets (buckets) you will use:

Dataset         Typical Use                        Key Shared Fields
logs            Raw log records                    timestamp, content, log.source, dt.trace_id, dt.entity.service
spans           Distributed traces (OpenTelemetry/W3C) trace_id, span_id, duration, service.name
metrics         Built-in or custom metrics         metric.key, value, dt.entity.*
bizevents       Business analytics events          event.type, event.provider
dt.system.events Platform events (deployments, config changes) event.kind
dt.security.events Security findings                —

3. Working with Tables, Schemas, and Fields
Grail discovers schema on-read → fields appear only when they exist in the fetched data.

Field notation:
• Simple: status.code
• Nested: cloud.provider.region
• With special chars: dt.entity.host
• Array indexing: tags[0]
• Escaping: `my weird.field!`

Essential field management commands:
Command       Purpose                        Example
fields        Keep only listed fields
fieldsAdd     Compute new field (replaces if exists)
fieldsRemove  Drop fields
fieldsRename  Rename one or more fields
flatten       Flatten nested records/arrays

Best practice: always end complex queries with a fields projection that contains exactly what you need.

4. Joins in DQL
DQL supports full join semantics (inner, leftouter, rightouter, fullouter, cross) and specialized kinds.

Syntax:
| join [kind:leftouter|inner|...] ( <subquery> ) on $left.field1 == $right.field2
or
| join (subquery) on common_key_field

Specialized joins:
• kind:as_array → returns matching right records as nested array (great for 1-to-many log enrichment)
• lookup → dictionary-style enrichment (very fast, right side must be small)

Real-world correlation fields:
• Logs ↔ Spans: dt.trace_id == trace_id
• Logs/Metrics ↔ Topology: dt.entity.service or dt.entity.host

Performance tip: Put the smaller dataset on the right when possible.

5. Filtering Strategies (Simple + Multi-Stage)
Simple (push-down optimized):
| filter service.name == "payment-service" and status.code >= 500

Full-text search:
| search "NullPointerException"

Multi-stage filtering (the real power):
fetch logs, scanLimitGBytes:-1
| filter loglevel == "ERROR"                                // early cardinality reduction
| parse content, "LD 'ERROR' SPACE * '[' LD:thread ']' SPACE DATA:message"
| filter contains(message, "Database")                      // filter on parsed field
| filter isNotNull(dt.trace_id)                              // keep only traceable errors

Use multi-stage whenever you need to parse before filtering – Grail only scans bytes once.

6. Parser Techniques (parse, extract, regex, split)
parse (pattern-based, fastest)
Uses Dynatrace Pattern Language (DPL):

Pattern tokens:
• LD 'literal'      → literal text
• SPACE / TAB / EOL
• INT:name / LONG:name / DOUBLE:name / TIMESTAMP:name / IPADDR:name
• DATA:name         → greedy rest-of-line
• ?* or ?+          → wildcard
• (pattern1 | pattern2) → or

Example:
parse content, "TIMESTAMP:ts '|' LD 'payment-service' '|' LD:level '|' DATA:message"

extract / extractAll (regex, very flexible)
extract content, /(?P<user>\w+) failed login from (?P<ip>[\d.]+)/ → creates fields user, ip

split
| fieldsAdd parts = split(content, "|")
| flatten parts
| fieldsAdd key = parts[0], value = parts[1]

Combined pattern (common in production)
parse content,
"TIMESTAMP:ts SPACE IPADDR:client_ip SPACE LD 'POST' SPACE DATA:url SPACE INT:status SPACE LONG:duration_us SPACE JSON:json_payload"
| flatten json_payload
| fieldsAdd user_id = toString(json_payload.userId)

7. High-Level Real-World Examples

Example 1 – Error Rate % per Service (Logs → Time-series Dashboard Tile)
Use case: Show error rate % for last 24 h, grouped by 5-minute bins.

fetch logs, from:now()-24h
| filter loglevel == "ERROR" or status.code >= 500
| summarize 
    errors = count(),
    total  = count() + countIf(loglevel != "ERROR" and status.code < 500)
    by: bin(timestamp, 5m), service.name
| fieldsAdd error_rate = round(errors / total * 100, 2)
| sort timestamp desc

Example 2 – Find Slow Database Calls with Their Logs (Spans + Logs Correlation)
Use case: Find database spans > 2 s and enrich with the actual SQL from logs.

fetch spans, from:now()-1h
| filter span.kind == "SERVER" and contains(name, "SQL") and duration > 2
| fields trace_id, duration, name, service.name
| join kind:as_array (
    fetch logs
    | filter isNotNull(dt.trace_id)
    | fields dt.trace_id, content, timestamp
) on $left.trace_id == $right.dt.trace_id
| flatten logs
| filter contains(logs.content, "SELECT") or contains(logs.content, "UPDATE")
| fields timestamp, service.name, duration, sql = logs.content
| sort duration desc

Example 3 – Parse Nginx Access Log & Calculate 95th Response Time
fetch logs, from:now()-6h
| filter log.source == "/var/log/nginx/access.log"
| parse content, 
  "IPADDR:client_ip SPACE * SPACE * SPACE TIMESTAMP:'[':ts:']' SPACE '\"'"
| fieldsAdd duration_ms = toDouble(replace(duration_str, "-", "0"))
| summarize p95_duration = percentile(duration_ms, 95), requests = count()
| sort p95_duration desc

Example 4 – Top 10 Memory-Leaking Hosts (Metrics + Topology Join)
fetch metrics, from:now()-12h
| filter metric.key == "builtin:host.mem.used"
| summarize avg(value), by: bin(timestamp, 30m), dt.entity.host
| join (
    fetch topology.hosts
    | fields dt.entity.host = entity.id, host.name = entity.name, cloud.region
) on dt.entity.host
| fieldsAdd avg_mem_pct = avg_value
| summarize max(avg_mem_pct), by: host.name, cloud.region
| sort toDouble(max_avg_mem_pct) desc
| limit 10

Example 5 – Failed Login Attempts per User & GeoIP (Security + Bizevents)
fetch dt.security.events, from:now()-7d
| filter event.type == "FAILED_AUTHENTICATION"
| fieldsAdd ip = toIP(event.source_ip)
| summarize attempts = count(), by: ip, geo.city, geo.country
| sort attempts desc
| limit 20

Example 6 – Business KPI: Conversion Rate Funnel from Bizevents
fetch bizevents, from:now()-30d
| filter event.type in ("pageview", "add_to_cart", "checkout", "purchase")
| summarize users = dcount(user_id), by: event.type
| sort event.type
| fieldsAdd conversion_to_purchase = round(purchase / pageview * 100, 2)

Example 7 – Deployment Impact Analysis (Platform Events + Metrics)
fetch dt.system.events, from:now()-7d
| filter event.kind == "DEPLOYMENT"
| fields timestamp, deployment.name, deployment.version = properties.newVersion
| join kind:leftouter (
    fetch metrics
    | filter metric.key == "builtin:service.response.time"
    | summarize p95 = percentile(value, 95), by: bin(timestamp, 15m), dt.entity.service
) on $left.timestamp <= $right.timestamp and $left.timestamp + 1h >= $right.timestamp
| summarize before = avgIf(p95, timestamp < deployment_timestamp),
            after  = avgIf(p95, timestamp >= deployment_timestamp),
            by: deployment.name, deployment.version, dt.entity.service
| fieldsAdd degradation_pct = round((after - before)/before * 100, 2)

8. Common Patterns and Best Practices (Battle-Tested)
1. Filter & project early – reduce scanned bytes dramatically.
2. Use bin(timestamp, 5m) or 10m for any time-series chart.
3. Prefer timeseries command when you only need one metric + dimensions.
4. For complex multi-metric charts → fetch metrics | summarize ... then make-series.
5. When parsing JSON logs → use JSON:field token in parse – it's 10–100× faster than parse_json().
6. Use lookup for static enrichment (country codes, threat intel lists, environment tags).
7. For huge datasets → add scanLimitGBytes:-1 only when you know you need it.
8. Always cast types explicitly in calculations: toDouble(), toLong().
9. Use countIf(condition) instead of separate error/success counts when possible.
10. In Notebooks → end with | limit 10000 for exploration, remove in production tiles.

9. Cheat Sheet Summary

Category         Command / Function        Syntax Example
Data source      fetch                     fetch logs, from:now()-1h
Filtering        filter / search           filter status.code >= 500 | search "OutOfMemory"
Projection       fields / fieldsAdd        fieldsAdd error_rate = errors/total*100
Parsing          parse                     parse content, "INT:status SPACE JSON:json"
Regex extract    extract                   extract content, /user=(?<user>\w+)/
Aggregation      summarize                 summarize count(), avg(duration), by: bin(timestamp,5m), service.name
Time-series (simple) timeseries            timeseries p95 = percentile(value,95), by: bin(timestamp,10m), url
Time-series (advanced) make-series         make-series avg(value) default=0 on bin(timestamp,5m) from ... to ...
Join             join / lookup             join (fetch spans) on $left.dt.trace_id == $right.trace_id
Array handling   flatten / expand          flatten logs
Sorting/Limiting sort / limit              sort duration desc | limit 50
Useful functions toLong(), toDouble(), round(), bin(), dcount(), percentile(), isNull()

You are now ready to write production-grade DQL queries that combine logs, traces, metrics, topology, and business data in ways that were previously impossible or required multiple tools.
