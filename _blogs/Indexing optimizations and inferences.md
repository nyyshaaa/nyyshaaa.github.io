---
title: "Indexing optimizations and inferences"
date: 2025-06-18 11:00
---


The costs are measured in units of disk page fetches; CPU effort cost are also included in cost they are converted into disk-page units using some fairly arbitrary fudge factors.

1.0 = 1 sequential disk page fetch
Random page fetch = ~4.0 (default is random_page_cost = 4.0)
CPU tuple processing cost = ~0.01 to 0.1 (depending on configuration)
Example: Let’s say there are 5 tuples on 5 separate disk pages that needs to be retrieved and they are sequential on disk .

Index scan by default assumes each qualifying tuple’s disk page fetch is random . There is no reliable mechanism to track exact disk layout .

So for HDD/(or even SSD) random I/O would cost more.

Index scan cost / disk page fetch= 4*seq. scan cost / disk page fetch.

For 5 tuples on 5 sequential pages excluding tuple processing cost

seq scan cost=5.0

index scan cost=20.0

But that’s just to give a glimpse of 1 page fetch cost for random I/O vs Sequential I/O

Total rows — 10_001_269

Distributions of rows per created_at

“2024–10–18” — (6_524_966)

“2024–08–10” — (800_001)

“2024–11–21” — (1_025_262)

Remaining dates have approximately 25_000 rows associated with them

EXPLAIN 
(ANALYZE,BUFFERS,VERBOSE)
SELECT short_code, visit_count 
FROM url_shortener
WHERE created_at >= '2024-11-01' 
AND created_at <  '2024-11-08'

Index on created_at column .

No limits are used so seq. scan while filtering would be very very high ,so it makes sense to do scan index(B+Tree) page and then fetch required pages from disk.

With a limit of 1000 / (or as small as 10)

Cost to yield 30_000 rows with different scan plans , which one to choose ?

Index scan/Bitmap Index scan + Bitmap Heap scan / Seq. Scan

Case of high selectivity(small fraction of total rows to be retrieved)

So index scan will do fine .


EXPLAIN 
(ANALYZE,BUFFERS,VERBOSE)
SELECT short_code, visit_count 
FROM url_shortener
WHERE created_at >= '2024-11-21' 
AND created_at <  '2024-11-25'

“2024–11–21” — (1_025_262 approx. 1/10 for whole table) contains around 50X more rows of any unique created_at

So index scan would cause high random I/O’s , Query planner will choose Bitmap index scan and then heap scan . First just retrieve required index key+TID(block+tupleoffset) by scanning index page ,then sort pages on disk as in same order as retrieved TID’s , and then fetch required pages, to minimize random I/O’s

With a limit of 100_000 (or 10)

Why it changes to seq. scan ?

Need fetch 100_000 rows → Bitmap scan start up cost is even 14_619

As anyway planner would stop after limited no of rows as per required rows in limit , seq. scan might be better than index scan which can cause more random I/O s


Index Only scans —

When all the columns used in query for filtering, group by , order by , projection are included in the index (composite index) then index only scan .

Index on (created_at,visit_count,short_code)

EXPLAIN
(BUFFERS,VERBOSE)
SELECT short_code, visit_count 
FROM url_shortener
WHERE created_at >= '2024–11–21' 
AND created_at < '2024–11–25'

EXPLAIN 
(ANALYZE,BUFFERS,VERBOSE)
SELECT *
FROM url_shortener
WHERE created_at = '2024-11-23'
ORDER BY visit_count DESC

Here projecting all columns so index only scan cannot be utilised .

Index only scan implies reading only the composite index pages .

Index scan and using individual index of created_at have been chosen

If there are two separate indexes here for e.g. Individual index on created_at , Composite index on (created_at,visit_count,short_code) which one will be preferred when doing index scan ?

Index scan via individual index plus extra sort by visit_count on created_at shows ~20_000 lower cost . But better cost is not extremely accurate and straightforward metric . For sorting planner can underestimate .

So composite index scan with no sorting is better than individual index plus extra sorting. We can check it by running this query again and again few times and check execution times and consider some average of them to decide.

EXPLAIN 
(ANALYZE,BUFFERS,VERBOSE)

SELECT *
FROM url_shortener
WHERE created_at = '2024-11-22' 
ORDER BY visit_count


DROP INDEX IF EXISTS url_shortener_created_at ;
