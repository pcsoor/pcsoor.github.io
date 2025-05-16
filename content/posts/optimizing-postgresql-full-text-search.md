---
title: "Optimizing PostgreSQL Full-Text Search: Indexing, Ranking, and Speed"
date: 2025-05-14T12:35:47+02:00
draft: false
ignore: true
hidden: true
summary: "Discovering full-text search possibilities with PostgreSQL"
---

Search engines have always fascinated me, especially full text search, faceting (those helpful little filters you see in web shops that are immensely helpful for users)

Lately, I've been diving into PostgreSQL's built-in full text search capabilities. I wanted to understand how far you can take it without adding Elasticsearch or a dedicated search engine. This post shares what I’ve learned so far, including performance tuning and index considerations.

To make this meaningful, I looked for a relatively big dataset. I found one with 4 million rows from one of my favorite sources of tech discussions: **Hacker News**.

### First attempt with GIN index on the fly

I start with a basic GIN index, which is the best starting point when using Postgres Full Text Search.
```sql
CREATE INDEX title_gin ON hacker_news USING GIN(to_tsvector('english', title));
```

Then I ran a search on the title field:

```sql
SELECT post_type, COUNT(*)
FROM hacker_news
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'DHH')
GROUP BY post_type
ORDER BY post_type;
```
While PostgreSQL used the index (_Bitmap Index Scan on title_gin_), the query was still slow.

The key issue: the `tsvector` was being recalculated on the fly during the query. This means PostgreSQL had to compute `to_tsvector('english', title)` for every matching row, which is CPU-intensive and inefficient.

The execution plan warns us about it too:

```
Recheck Cond: (to_tsvector('english', title) @@ '''dhh'''::tsquery)
```

### Optimising with persistent `tsvector` column

To fix this issue, I added a `tsvector` column and populated it:

```sql
-- add tsvector column
ALTER TABLE hacker_news ADD COLUMN title_tsvector tsvector;
-- populate
UPDATE hacker_news SET title_tsvector = to_tsvector('english', title);
```

Now, I can use it in my query:

```sql
SELECT post_type, count(*)
FROM hacker_news
WHERE hacker_news.title_tsvector @@ to_tsquery('english', 'DHH')
GROUP BY post_type
ORDER BY post_type;
```

### GIN's hidden trap

Before creatign the GIN index on the `title_tsvector`, I want to highlight an important GIN feature, that can hurt search performance.

PostgreSQL offers some advanced features for GIN indexes, like **Fast Update**. By default, this feature is enabled for GIN. It can make inserts and updates faster by using a _pending list_: new entries not added directly to the index, but instead go into that pending list (like a temporary queue)

Sounds nice, but here's a catch which PostgreSQL warns us about:
> _The main disadvantage of this approach is that searches must scan the list of pending entries in addition to searching the regular index, and so a large list of pending entries will slow searches significantly._

Meaning, when we run a search query using the index, PostgreSQL also has to scan the _pending list_ as well, and if we have many entries in it, this can slow down the search performance significantly over time (especially on large datasets).

The solution is to disable this fast update technique (`fastupdate = off`) for search-optimised queries:
```sql
CREATE INDEX idx_gin_title_tsvector
ON hacker_news USING GIN (title_tsvector)
WITH (fastupdate = off);
```

with these optimization techniques, I could achieve an `12 ms` execution time.

### Ranking

For a good search experience, we don't just want to find matches, we want to rank them by relevance.

To test it effectively, I started running the following query multiple times for different keywords:

```sql
SELECT title, ts_rank(hacker_news.title_tsvector, websearch_to_tsquery('english', 'postgres fts')) rank
FROM hacker_news WHERE hacker_news.title_tsvector @@ websearch_to_tsquery('english', 'postgres')
ORDER BY rank DESC limit 10;
```

| query               | Worst times (ms) | Best times (ms) |
|---------------------|------------------|-----------------|
| postgres fts        | 368              | 43              |
| programming in ruby | 337              | 24              |
| ask                 | 1981             | 324             |

When the query matches only a few rows, it performs pretty well. But searching "ask" in a hacker news dataset, things slowed down quickly.

The reason is ranking, specifically how expensive it is:

1. Finds all matches
2. Calculates `ts_rank` for all of them
3. Sorts them all
4. Returns the top 10

On large datasets, steps 2 and 3 are very expensive calculations.

PostgreSQL even warns us about this:

> _Ranking can be expensive since it requires consulting the tsvector of each matching document, which can be I/O bound and therefore slow. Unfortunately, it is almost impossible to avoid since practical queries often result in large numbers of matches._

So the slow execution time is caused by ranking: Postgres needs to call `ts_rank()` function for each of the matching rows:

One clever ranking optimization technique I came across (and really liked) is to **apply ranking only to a sample of the matches**. The idea is: if your query matched let's say 400,000 posts, you can take a sample of 10,000 and just rank that subset.

#### You might wonder: Isn't this less accurate?

Yes, slightly. But here is a tradeoff:

- If your query returns 100,000+ matches, do you really care about perfectly ranking every one?
- Or would you rather get a good enough top 10 in ~50 ms instead of ~2000 ms?

With this new approach, the query looks like this:

```sql
WITH sample_query AS (
    SELECT title, title_tsvector FROM hacker_news
    WHERE title_tsvector @@ websearch_to_tsquery('english','ask')
    LIMIT 10000)
SELECT title, ts_rank(title_tsvector, websearch_to_tsquery('english', 'ask')) rank
FROM sample_query
ORDER BY rank DESC limit 10;
```

With this, the "ask" query went **from 1981 ms to 58 ms**. That’s a huge win!

Of course, there are specialized tools out there that are built specifically for fast, relevance-based ranking. But for this exploration, I wanted to stick with PostgreSQL’s built-in full-text search and see how far it can go on its own.

### Conclusion

PostgreSQL’s full-text search is surprisingly capable, and with the right setup, like tsvector columns and GIN indexes, it performs really well even on large datasets.
