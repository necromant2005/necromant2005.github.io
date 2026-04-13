---
title: Track JS exceptions and CSP violations by GoogleAnalytics + BigQuery
permalink: /analytics/track-js-exceptions-and-csp-violations-by-googleanalytics-bigquery/
---

# Track JS exceptions and CSP violations by GoogleAnalytics + BigQuery

If you do not track frontend errors, you lose knowledge at the exact moment customers hit friction.

A broken form, failed script, or blocked asset can silently stop a purchase, and without analytics you only see lower conversion, not the reason. That means lost customers, lost revenue, and product decisions made without the real cause.

Tracking JavaScript exceptions and CSP violations closes that gap. GA4 collects the signal, and BigQuery makes it possible to group, rank, and investigate what is breaking in production.

## Init Google Analytics

I suppose you already have something like this in your code for initialiazing GA4 tracking

```js
window.dataLayer = window.dataLayer || [];

function gtag() {
  window.dataLayer.push(arguments);
}

gtag('js', new Date());
gtag('config', 'G-XXXXXXXXXX');
```

## Exceptions

This listens for uncaught JavaScript errors and rejected promises, then sends them to GA4 as `exception` events with message, source, and line information.

```js
window.onerror = (message, source, lineno, colno, error) => {
  if (lineno === 0 && colno === 0) {
    return;
  }

  gtag('event', 'exception', {
    description: String(message),
    path: window.location.pathname,
    source: source || '',
    lineno,
    colno,
    fatal: true,
  });
};

window.addEventListener('unhandledrejection', (event) => {
  gtag('event', 'exception', {
    description: event.reason instanceof Error ? event.reason.message : String(event.reason || 'Unknown error'),
    path: window.location.pathname,
    source: 'unhandledrejection',
    fatal: false,
  });
});
```

## CSP Violations

This listens for browser `securitypolicyviolation` events and sends the blocked URL, directive, and file location to GA4 as `csp_violation`.

```js
document.addEventListener('securitypolicyviolation', (event) => {
  gtag('event', 'csp_violation', {
    document_uri: event.documentURI || '',
    blocked_uri: event.blockedURI || '',
    violated_directive: event.violatedDirective || '',
    effective_directive: event.effectiveDirective || '',
    source_file: event.sourceFile || '',
    line_number: event.lineNumber || 0,
    column_number: event.columnNumber || 0,
    disposition: event.disposition || '',
  });
});
```

## Connect GA4 To BigQuery

For the GA4 -> BigQuery connector/export flow:

1. In GA4, open `Admin`.
2. Under `Product links`, open `BigQuery Links`.
3. Create a link to your Google Cloud project.
4. Choose the correct region and property data stream.
5. Enable daily export. Enable streaming export too if you want fresher `intraday` tables.
6. After the link is active, GA4 starts writing base tables like `events_YYYYMMDD` and near-real-time tables like `events_intraday_YYYYMMDD`.

## Getting errors info

Replace placeholders before running queries:

- `your_project`
- `your_dataset`
- `20260401`
- `20260430`

Use the base `events_YYYYMMDD` tables for stable daily reporting.

Use `events_intraday_YYYYMMDD` together with the base tables when you want near-real-time numbers for the latest day.

Base + near-real-time CTE:

```sql
WITH params AS (
  SELECT
    '20260401' AS start_date,
    '20260430' AS end_date,
    'intraday_20260430' AS intraday_suffix
),
base_events AS (
  SELECT
    user_pseudo_id,
    event_timestamp,
    event_name,
    LOWER((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'page_location')) AS page_location
  FROM `your_project.your_dataset.events_*`
  CROSS JOIN params
  WHERE (
      _TABLE_SUFFIX BETWEEN start_date AND end_date
      OR _TABLE_SUFFIX = intraday_suffix
    )
    AND event_name IN ('exception', 'csp_violation')
)
```

Sample response:

<div class="table-responsive">
  <table class="table table-striped table-bordered">
    <thead>
      <tr>
        <th>user_pseudo_id</th>
        <th>event_timestamp</th>
        <th>event_name</th>
        <th>page_location</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><code>187654321.1714450001</code></td>
        <td><code>1714485123456789</code></td>
        <td><code>exception</code></td>
        <td><code>https://example.com/checkout</code></td>
      </tr>
      <tr>
        <td><code>187654321.1714450001</code></td>
        <td><code>1714485127890123</code></td>
        <td><code>csp_violation</code></td>
        <td><code>null</code></td>
      </tr>
      <tr>
        <td><code>287654321.1714450099</code></td>
        <td><code>1714485999123456</code></td>
        <td><code>exception</code></td>
        <td><code>https://example.com/catalog</code></td>
      </tr>
    </tbody>
  </table>
</div>

## Exception Analysis In BigQuery

If you send normalized `exception` events from JavaScript, you can group them in BigQuery and see which pages are actually failing.

```sql
WITH params AS (
  SELECT
    '20260401' AS start_date,
    '20260430' AS end_date,
    'intraday_20260430' AS intraday_suffix
),
exception_events AS (
  SELECT
    TIMESTAMP_MICROS(event_timestamp) AS event_ts,
    LOWER((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'page_location')) AS page_location,
    (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'description') AS description,
    (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'source') AS source,
    COALESCE(
      (SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'lineno'),
      SAFE_CAST((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'lineno') AS INT64)
    ) AS lineno,
    COALESCE(
      (SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'colno'),
      SAFE_CAST((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'colno') AS INT64)
    ) AS colno
  FROM `your_project.your_dataset.events_*`
  CROSS JOIN params
  WHERE (
      _TABLE_SUFFIX BETWEEN start_date AND end_date
      OR _TABLE_SUFFIX = intraday_suffix
    )
    AND event_name = 'exception'
),
normalized AS (
  SELECT
    event_ts,
    COALESCE(description, '') AS description,
    COALESCE(source, '') AS source,
    COALESCE(lineno, 0) AS line,
    COALESCE(colno, 0) AS col,
    REGEXP_EXTRACT(page_location, r'^https?://([^/]+)') AS host,
    COALESCE(REGEXP_EXTRACT(page_location, r'^https?://[^/]+(/[^?#]*)'), '/') AS path
  FROM exception_events
  WHERE page_location IS NOT NULL
)
SELECT
  COALESCE(REGEXP_EXTRACT(description, r'([A-Za-z]+Error)'), '') AS error_code,
  description,
  source,
  line,
  col,
  host,
  path,
  COUNT(*) AS count,
  FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%S UTC', MAX(event_ts), 'UTC') AS last_seen
FROM normalized
GROUP BY error_code, description, source, line, col, host, path
ORDER BY count DESC, last_seen DESC, host ASC, path ASC;
```

Sample response:

<div class="table-responsive">
  <table class="table table-striped table-bordered">
    <thead>
      <tr>
        <th>error_code</th>
        <th>description</th>
        <th>source</th>
        <th>line</th>
        <th>col</th>
        <th>host</th>
        <th>path</th>
        <th>count</th>
        <th>last_seen</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><code>TypeError</code></td>
        <td><code>Cannot read properties of null (reading 'value')</code></td>
        <td><code>https://example.com/assets/app.js</code></td>
        <td><code>182</code></td>
        <td><code>17</code></td>
        <td><code>example.com</code></td>
        <td><code>/checkout</code></td>
        <td><code>24</code></td>
        <td><code>2026-04-30 14:52:11 UTC</code></td>
      </tr>
      <tr>
        <td><code>SyntaxError</code></td>
        <td><code>Unexpected token '&lt;'</code></td>
        <td><code>unhandledrejection</code></td>
        <td><code>0</code></td>
        <td><code>0</code></td>
        <td><code>example.com</code></td>
        <td><code>/catalog</code></td>
        <td><code>7</code></td>
        <td><code>2026-04-30 13:08:44 UTC</code></td>
      </tr>
    </tbody>
  </table>
</div>

## CSP Violation Analysis In BigQuery

Use a separate query for `csp_violation` so policy issues do not get mixed with runtime exceptions.

```sql
WITH params AS (
  SELECT
    '20260401' AS start_date,
    '20260430' AS end_date,
    'intraday_20260430' AS intraday_suffix
),
csp_events AS (
  SELECT
    TIMESTAMP_MICROS(event_timestamp) AS event_ts,
    LOWER((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'document_uri')) AS document_uri,
    (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'blocked_uri') AS blocked_uri,
    (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'violated_directive') AS violated_directive,
    (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'effective_directive') AS effective_directive,
    (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'source_file') AS source_file,
    COALESCE(
      (SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'line_number'),
      SAFE_CAST((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'line_number') AS INT64)
    ) AS line_number,
    COALESCE(
      (SELECT ep.value.int_value FROM UNNEST(event_params) ep WHERE ep.key = 'column_number'),
      SAFE_CAST((SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'column_number') AS INT64)
    ) AS column_number,
    (SELECT ep.value.string_value FROM UNNEST(event_params) ep WHERE ep.key = 'disposition') AS disposition
  FROM `your_project.your_dataset.events_*`
  CROSS JOIN params
  WHERE (
      _TABLE_SUFFIX BETWEEN start_date AND end_date
      OR _TABLE_SUFFIX = intraday_suffix
    )
    AND event_name = 'csp_violation'
),
normalized AS (
  SELECT
    event_ts,
    COALESCE(blocked_uri, '') AS blocked_uri,
    COALESCE(violated_directive, '') AS violated_directive,
    COALESCE(effective_directive, '') AS effective_directive,
    COALESCE(source_file, '') AS source_file,
    COALESCE(line_number, 0) AS line_number,
    COALESCE(column_number, 0) AS column_number,
    COALESCE(disposition, '') AS disposition,
    REGEXP_EXTRACT(document_uri, r'^https?://([^/]+)') AS host,
    COALESCE(REGEXP_EXTRACT(document_uri, r'^https?://[^/]+(/[^?#]*)'), '/') AS path
  FROM csp_events
  WHERE document_uri IS NOT NULL
)
SELECT
  violated_directive,
  effective_directive,
  blocked_uri,
  source_file,
  line_number,
  column_number,
  disposition,
  host,
  path,
  COUNT(*) AS count,
  FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%S UTC', MAX(event_ts), 'UTC') AS last_seen
FROM normalized
GROUP BY violated_directive, effective_directive, blocked_uri, source_file, line_number, column_number, disposition, host, path
ORDER BY count DESC, last_seen DESC, host ASC, path ASC;
```

Sample response:

<div class="table-responsive">
  <table class="table table-striped table-bordered">
    <thead>
      <tr>
        <th>violated_directive</th>
        <th>effective_directive</th>
        <th>blocked_uri</th>
        <th>source_file</th>
        <th>line_number</th>
        <th>column_number</th>
        <th>disposition</th>
        <th>host</th>
        <th>path</th>
        <th>count</th>
        <th>last_seen</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><code>script-src-elem</code></td>
        <td><code>script-src-elem</code></td>
        <td><code>https://cdn.example.net/widget.js</code></td>
        <td><code>https://example.com/checkout</code></td>
        <td><code>0</code></td>
        <td><code>0</code></td>
        <td><code>enforce</code></td>
        <td><code>example.com</code></td>
        <td><code>/checkout</code></td>
        <td><code>15</code></td>
        <td><code>2026-04-30 14:55:02 UTC</code></td>
      </tr>
      <tr>
        <td><code>img-src</code></td>
        <td><code>img-src</code></td>
        <td><code>data</code></td>
        <td><code>https://example.com/catalog</code></td>
        <td><code>0</code></td>
        <td><code>0</code></td>
        <td><code>report</code></td>
        <td><code>example.com</code></td>
        <td><code>/catalog</code></td>
        <td><code>4</code></td>
        <td><code>2026-04-30 11:21:09 UTC</code></td>
      </tr>
    </tbody>
  </table>
</div>
