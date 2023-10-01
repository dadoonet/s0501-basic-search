# Elastic Daily Bytes S05E01: Search, the basics

## Welcome words

We are starting a new Season of the Elastic Daily Byte(s) which is this time related to the search power of Elastic. Every day, just before noon (European Central Time), we will explore many of the things you can do nowadays with Elastic Stack.

We will cover for example things like basic search today, but also using synonyms using the new 8.10 API, runtime fields, vector search, semantic search, Reciprocal Rank Fusion (RRF), how to connect Elastic and OpenAI...

Subscribe to the YouTube channel and join us every day during the next 3 weeks.

## Discuss forum

<!-- https://discuss.elastic.co/search?expanded=true&q=status%3Asolved%20%23elastic-stack%3Aelasticsearch -->

You are all probably very familiar with the Elastic forum named `discuss.elastic.co`. In the past we have added a plugin on it which helps people to mark a post as the answer to their question. For example, let's look at [this discussion #194052](https://discuss.elastic.co/t/194052). We can see a very nice and detailled by the way question and the post number 4 is marked as solving it.

We wrote a script which is getting all the posts that are marked as solved and we are sending that information to Elasticsearch running in Elastic Cloud (8.10.2).

If we open [DevTools](https://elastic-daily-bytes-s05.kb.us-central1.gcp.cloud.es.io:9243/app/dev_tools#/console), we can get that topic:

```json
GET bytes-discuss/_doc/194052
```

This gives:

```json
{
  "_index": "bytes-discuss",
  "_id": "194052",
  "_version": 1,
  "_seq_no": 9587,
  "_primary_term": 1,
  "found": true,
  "_source": {
    "topic": 194052,
    "title": "Streamed Logs not loaded into Kibana",
    "category_name": "Kibana",
    "question": {
      "text": "QUESTION",
      "author": {
        "username": "ben.sharp",
        "name": "",
        "avatar_template": "https://avatars.discourse-cdn.com/v4/letter/b/f08c70/{size}.png"
      },
      "date": "2019-08-06T16:09:05.638Z",
      "reads": 30
    },
    "solution": {
      "text": "SOLUTION",
      "post_number": 16,
      "author": {
        "username": "matw",
        "name": "Matthias Wilhelm",
        "avatar_template": "/user_avatar/discuss.elastic.co/matw/{size}/13913_2.png"
      },
      "date": "2019-08-12T14:51:28.673Z",
      "reads": 11
    },
    "duration": 8562
  }
}
```

## Search for all

Let's see how many docs we indexed:

```json
GET bytes-discuss/_search
```

This is actually giving us back:

```json
{
  "took": 1,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 10000,
      "relation": "gte"
    },
    "max_score": 1,
    "hits": [
      {
      }
    ]
  }
}
```

So we can see information about the number of shards we searched in, also the `took` information is useful when you want to know if the search is fast enough or not.

```json
"total": {
  "value": 10000,
  "relation": "gte"
}
```

By default, Elasticsearch only count exactly the 1st 10000 hits and after that is telling you that you have more than 10000 hits. That's an optimization when you don't really need the exact total number of matches after a significant number of hits.

If you need the exact count, you can add `?track_total_hits=true` to your request:

```json
GET bytes-discuss/_search?track_total_hits=true
```

So we have `26153` hits. Let's look at them...

## Exclude/Include fields

Something which might be handy as well, if you don't want to see the whole JSON content, you can select the fields you want to display.

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "_source": {
    "includes": [ 
      "title",
      "category_name",
      "duration",
      "question.author",
      "question.date",
      "question.reads", 
      "solution.author",
      "solution.date",
      "solution.reads"
    ]
  }
}
```

You can also simply exclude what you don't want to see:

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "_source": {
    "excludes": [ "*.text" ]
  }
}
```

## Response object

If we look a bit more at the response object, that gives us a lot of useful informations:

* `_index`: this is great when you are implementing a search engine on top of multiple source of data. That way you can know from which index the data is coming from. Here we are only searching in the `bytes-discuss` index.
* `_id`: we decided to use here the topic id as the `_id` of the document. That's super handy as you can directly open the discussion just knowing the `_id`.
* `_score`: we will cover that one from wednesday with text search. Basically, for now, I don't have any query so all documents have the same score.

## Pagination

Let's look at the list of the documents itself. We have only 10 documents here. Why this? Because Elasticsearch has a built-in pagination. By default, it returns 10 documents per page. We can change that using the `size` parameter:

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "size": 1,
  "_source": {
    "excludes": [ "*.text" ]
  }
}
```

If we want to read the next document, we can use the `from` parameter. It starts by default at `0` (so the first document):

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "from": 0,
  "size": 1,
  "_source": {
    "excludes": [ "*.text" ]
  }
}
```

But let's look at the second document:

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "from": 1,
  "size": 1,
  "_source": {
    "excludes": [ "*.text" ]
  }
}
```

With that you can paginate over the resultset. But note that there is a limit and in total, by default, Elasticsearch can not paginate after 10000 documents. So this works:

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "from": 9999,
  "size": 1,
  "_source": {
    "excludes": [ "*.text" ]
  }
}
```

But this not:

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "from": 10000, 
  "size": 1, 
  "_source": {
    "excludes": [ "*.text" ]
  }
}
```

The error message is giving you some advices:

> See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the `index.max_result_window` index level setting.

But this comes with some performance implications. Here, I don't have a lot of documents, but we can see the `took` time between `from: 1` and `from: 9999` has significantly increased. It would be even worse for millions of hits.

Although the scroll api is no longer the best practice. I'd suggest to read the [Paginate search results](https://www.elastic.co/guide/en/elasticsearch/reference/8.10/paginate-search-results.html) documentation which describes how to use "Point In Time" and "Search After" features to achieve the same in a much more efficient way. Search after will help to do deep pagination while point in time will allow you to do it in a consistent way across pages even though you are still modifying your dataset.

## Closure

Thank you for watching. Tomorrow we will be covering one of the key component of text search which is the **analysis** process. Join us at the same hour of the day! Bye!
