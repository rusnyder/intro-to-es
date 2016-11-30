# Introduction to Elasticsearch

If you've found yourself on this page, you're most likely taking my basic
introduction to Elasticsearch course and are trying to follow along. The
resources in this project are nothing more than the minimal set of files
and queries needed to walk through the "Chat log analysis" case study of
Elasticsearch which, for ease of use, I'll provide a brief review of here.

## Case Study: Chat Log Analysis

Given a dump of slack logs, I'd like to be able to answer the following
questions:

 * Which user are most commonly posting GIFs?
 * What have people saying about _me_?
 * How does chat activity vary over time?

## Getting Started

### Prerequisites

To follow along with this guide, you will need the following tools installed
and available on your local machine:

 * A terminal w/ basic command-line skills
 * A running Elasticsearch node (or access to one)

### 1. Clone this project
To get started, you'll first need to clone this project:

```shell
cd /path/to/your/workspace
git clone https://github.com/rusnyder/intro-to-es
```

### 2. Start up your Elasticsearch node
This guide assumes that you either already have an Elasticsearch node
running and accessible or that you are able to create one without
guidance.

### 3. Create an index template for all our indices
To make sure that our indexed data gets analyzed the way we want, we need to
make sure that we give Elasticsearch the appropriate settings/mappings for
our data prior to indexing any data.  This can be done in two ways:

 1. Create all our indices with the settings/mappings
 2. Create an index template that will apply settings/mappings as
    any new indices get created

For simplicity, we COULD go with option #1, but for the sake of completeness
and "best practices", this example will walk through #2.

**Creating our Template**

This project provides a template that matches on any indices named
`intro-to-es-*` (note the wildcard), which means that any index created
with a matching name will get created with all of the settings and mappings
from that template.  This is nice because once the template has been created,
we can just start indexing data and new indices will automatically pick up
their appropriate settings.

Let's go ahead and create the template:

_NOTE: All commands from this point assume your Elasticsearch node has
its HTTP service available at the location http://localhost:9200_

```shell
curl -XPUT http://localhost:9200/_template/intro-to-es -d @intro-to-es-template.json
```

### 4. Index some data
This project provides some sample data in the file
[`bulk-index.jsonl`](./bulk-index.jsonl) that are already in a format the
[Elasticsearch's Bulk API][es-bulk] accepts, so indexing our data is as
simple as POSTing that file to Elasticsearch:


```shell
# Delete any data from a previous run of these steps
curl -XDELETE 'http://localhost:9200/intro-to-es*'

# Index our events using the bulk API and refreshing
# to make them available for search immediately
curl -XPOST http://localhost:9200/_bulk --data-binary @bulk-index.jsonl

# Run a quick count just to see that your documents were
# indexed and that the "intro-to-es" alias was created
curl http://localhost:9200/intro-to-es/_count
```

_NOTE: cURL will strip newlines from any data provided using the `-d/--data`
option, so we must use the `--data-binary` option to preserve the newlines
which are necessary for proper interpretation of the bulk request body._

### 5. Now we're ready for some queries!
At this point, we've got our data indexed and are ready to start answering some questions!

**Question 1: Which users are most commonly posting GIFs?**

Here we just want to find the top senders using the `/gif` slack directive,
and since our analyzer only preserves slashes when they are at the beginning
of the sentence, we know that if we match the token `/gif`, then it's the
directive that we're looking for and not just the string "/gif" occurring
in the middle of a sentence.

Once we subset the events, aggregating over `from` gives us the top senders.

```javascript
POST /intro-to-es/_search
{
  "query": {
    "term": {
      "message": "/gif"
    }
  },
  "aggs": {
    "top_senders": {
      "terms": {
        "field": "from",
        "size": 0
      }
    }
  }
}
```

**Question 2: What have people saying about _me_?**

To answer this, we want to take a look at the chat messages that are most
relevant, with "relevance" here loosely being defined as "how likely is it
that this message pertains to or is about me?".  For this, we use a couple
of clauses with `boost` factors as a means of saying occurrences of certain
tokens/phrases can be treated as "more certainly me" than others.

```javascript
POST /intro-to-es/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "message": {
              "value": "@rusnyder",
              "boost": 10
            }
          }
        },
        {
          "match": {
            "message": {
              "query": "russell,snyder",
              "boost": 2
            }
          }
        },
        {
          "prefix": {
            "message": {
              "value": "russ",
              "boost": 1
            }
          }
        }
      ]
    }
  }
}
```

[//]: # (Links Section)
[es-bulk]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html
