---
layout: post
title: "Case study: Cloudflare Workers with Azure Functions"
date: 2020-05-11 11:00
author: scooletz
permalink: /2020/05/11/case-study-cloudflare-workers-with-azure-functions/
image: /img/2020/04/edge-workers.png
categories: ["serverless", "edge", "Cloudflare", "Azure", "Azure Functions"]
tags: ["serverless", "edge", "Cloudflare", "Azure", "Azure Functions"]
whitebackgroundimage: true
---

This is a follow up post to [migrating my blog](/2020/04/21/blog-refined/) and learning about [Cloudflare Workers capabilities](/2020/05/04/edge-with-workers/). Having a playground provided by an ability to intercept all the network traffic coming to my blog, led me to thinking about an experiment with gathering statistics. What if instead of using Google or Facebook tracking pixel I could try out to build a simple thing for this? What if I could test various approaches for this, trying to stress them a little bit?

### Some numbers

Just to give you some rough estimations what we'll be dealing with, the traffic on my blog is around `100000` requests a year. The frequency depends on the topic and promotion of a specific entry. On average , let's assume that it's `300` visits a day. This excludes the bots, crawlers and getting static content. This is just number representing real accesses to the content.

### What's needed

As always, with any feature, there's a lot of things that should be considered. Especially in the distributed world, the following aspects should be taken into consideration:

1. basic numbers, like number of requests per day, possible peaks
1. latency requirements
1. cost of storage, both for storing it and accessing it
1. SLA of all the components (remember about [TOP 1 method to lower SLA of your service](/2018/11/28/top-1-method-to-lower-sla-of-your-apps/))
1. failure scenarios (what happens if)
1. my knowledge about technology

Let's try provide some values for them, again it's a rough sketch:

| Aspect |      Value      |
|----------|----------------|
| Average requests per day | 300 |
| Peak requests per day | 2000 |
| Peak requests per hour | 500 |
| Latency | not higher that 10ms |
| Cost of storage | should be cheap and enable some kind of querying |
| SLA | gathering should fail without interrupting blog and may fail|

### Foundation

To make it possible to intercept and log anything, I needed a worker. As I briefly described in [Edge with Workers](/2020/05/04/edge-with-workers/), the first step is to provide routing, that will make it sure that no requests for static assets are logged. The following set was created, to ensure that more specific routes have no worker. At the same time, a generic one would capture all the requests to all the pages. Unfortunately, `Routes` does not accept a greedy operator `*` in the middle of a route.

1. `https://blog.scooletz.com/img*` - empty
1. `https://blog.scooletz.com/assets*` - empty
1. `https://blog.scooletz.com/*` - MyWorker

The second part was the worker itself.

```javascript
const bots = ["googlebot(at)googlebot.com",
    "bingbot(at)microsoft.com"
    ];

addEventListener('fetch', e => {
  var request = e.request;
  var date = new Date(Date.now());
  var dt = date.toISOString().split('T')[0];

  var payload = {
    date: dt,
    url: request.url,
    headers: headersToObject(request.headers)
  }

  var url = request.url.toString();
  if (url.endsWith(".php")){
    e.respondWith(new Response("Nope", {status: 404}));
  }
  else {
    var requestor = request.headers.get("from");
    if (!requestor || !bots.includes(requestor)){
      e.waitUntil(log(payload));
    }
  }
})
```

You can can probably notice huge `Nope` for bots trying to log in to Wordpress admin with `.php`. A follow up filter, removes two major bots hitting the stats as well. Again `e.waitUntil` is used to register the `log(payload)` promise to be continued after the response is served. This means, that the overhead of the statistics requests won't impact the response time of the blog.

What's in the `log(payload)` then?

### Idea 1: Cloudflare Key Value all the way

The first idea for building this, was based on Cloudflare Key Value store that is accessible from workers. It's a paid feature (minimum `5$/month`) but let's you do a million writes and 10 million reads. I know, that it's mentioned in [use cases](https://developers.cloudflare.com/workers/reference/storage/use-cases/) that it should be used for read-mostly traffic, but still I wanted to give it a try.

I created a separate keyspace and used a similar approach for generating keys, but combining the date with a randomly generated bytes, using powerful `Crypto` API.

```javascript
var date = new Date(Date.now());
var dt = date.toISOString().split('T')[0]; // "2020-05-11"
var key = dt + "-" + (await generateId());
```

The prefix for the key was quite important, as every keyspace can be queried with a prefix query. With this a proper key design, you can easily gather `all the data starting with: 2020-05-11`.
The write itself was not breaching any rules, as it was always a new key being written with a new value. The value was serialized with JSON.

The second part related to this was the query and aggregation. I started with an Azure Function, based on timer, that every single day queried the store for the data from the day before. And then I hit the wall.

First, the execution of the function was pretty lengthy. The reason for this was, that currently to get all the values for the specific day, you need to:

1. query the kv store with `prefix:` ([API link](https://api.cloudflare.com/#workers-kv-namespace-list-a-namespace-s-keys))
1. for each retrieved key, retrieve the value separately ([API link](https://api.cloudflare.com/#workers-kv-namespace-read-key-value-pair))

Even with some degree of parallelism, using a throttled `Task.WhenAny` I hit another wall, breaching API number of calls. Every request was being counted toward the limit of calls per second. This made me thing again about this approach. The foundations were ok, the logging part was failing a bit, when being queried outside of the worker.

One option would be to use another worker to combine the data. I wanted to have them gathered in Azure Storage though, so it'd require some push anyway. Then I thought that maybe using storage directly from logger could be a solution?

### Idea 2: Azure Storage with Queues, SAS Tokens and more

The other idea was based more on Azure Storage. What if I could just write to storage from a worker? Fortunately, it's possible and you need no intermediate function for this. To do it you need to generate a [SAS token](/2018/03/19/serverless-more-secure-sas-tokens/), that enabled accessing an Azure Storage service and do something with it. The permissions are pretty granular and in this case, all that was required was:

1. an azure storage queue
1. the ability to add a message to it

Worker doesn't need to be able to process data or to delete the queue. It just needs to be able to add new logs in there! This changed `log(payload)` to the following:

```javascript
async function log(entry) {
  const value = JSON.stringify(entry);
  const utf8 = new TextEncoder().encode(value);
  const encoded = btoa(String.fromCharCode.apply(null, utf8));

  // post to azure storage queues
  await fetch(QueueUrl, {
    method: 'POST',
    headers: {},
    body: "<QueueMessage><MessageText>" + encoded + "</MessageText></QueueMessage>"
  });
}
```

where `QueueUrl` was an url with attached `SAS Token`.

The other part was a function that would react to this writes. It's as simple as this, appending the payload to a blob that gathers all the data from the specific date. Again, using an `AppendBlob` prevents race conditions and enables to gather all the statistics in one place.

```csharp
[FunctionName("Analytics-Processor")]
public async Task Process([QueueTrigger("analytics")] string message,
    [Blob("analytics")] CloudBlobContainer container)
{
    // some processing omitted here
    var blob = container.GetAppendBlobReference($"{date}.json");
    await blob.CreateIfNotExists();

    await using var writer = new StringWriter();
    using var json = new JsonTextWriter(writer);

    data.WriteTo(json);
    writer.Write(',');

    await blob.AppendTextAsync(writer.ToString());
}
```

Grouping by date is a first level aggregation that can be (and will be) leveraged later on. Can we alter it a bit?

### Idea 3: Azure Storage with a direct AppendBlob access

The question that can be raised, what's the reason for the functions in there. The only one was some data sanitization. Let's sum up the costs though of a single log entry

1. write to a queue
1. function execution
1. check if a blob exists
1. append to a blob

Could we make it faster, better, cheaper? What if we enable the worker to append to the blob directly? Is it possible?

It is and to make this happen and to keep it as secure as possible we'll need to use a fine grained SAS token for blobs. It should be created on the container level and full url will consist of the following parts:

1. address of the container
1. name of the blob
1. `?comp=appendblock` which shows the kind of operation
1. `st=date1&se=date2` which gives a range of dates when SAS token is valid
1. `&sp=a` which enables `Add` operation, which is equal to an append to an existing blob
1. `sr=c` which shows that the token has been created for the container
1. `sig=asdfsdafdsfdsfsd` which provides the signature

with this, once the appender knows the name of the file it can construct a valid `url` that will enable them to perform `PUT`. With this, the list of operations is as follows:

1. append to a blob

The SLA, latency are the same. The worker calls a single endpoint in Azure Storage. From the pricing point of view and complexity, this is so much cleaner. There's one mistake though. Can you spot it?

The mistake is that the blobs cannot be created by worker, and actually there should not be. For this, you can easily create a `TimerTrigger` based function that will create blobs up-front. This can be run once a day and create blobs, let's say for the following 4 days. This means, that again, another call for ensuring that the blob was created, is skipped.

### The difference

The first idea based on using cloudflare database was a bit abusive, leveraging a database that is meant to provide read-mostly capabilities. Additionally, it was heavy to query, as the API provides no way of `bulk-read` all the values. Even in a worker, that would require getting all the keys first and then query one by one. If we take into consideration limitations of having up to `6` connections active and `50ms` for a worker execution (CPU time, not wall time), this might be not enough. From the budget point of view, I doubt that my blog will be able to write more than 1 million per month, so we're on the safe side here. Definitely, the key value store in workers is not one size fits all solution and should be not treated as one.

The second idea was using Azure Functions. The first important thing was to separate the execution and processing from accepting the request. SLA just for Azure Storage Queues is much higher than a combined SLA for Functions + Storage, so using SAS-secured queue was a good thing to go. With the function, it performs a single execution for each log, appending it to the blob.

The last one was based on a direct call to the [append-block](https://docs.microsoft.com/en-us/rest/api/storageservices/append-block) and resulted in dramatic drop in number of operations and calls.

### Summary

I hope this post provides you with some options, with some trail of my decision making in this situation. It also shows, that being given specific conditions, one tool is not always the right one, and the decision could be context specific and limit-specific (lack of API).

This example could be tried with other APIs like something for data ingress or totally different platform than Cloudflare Workers but so far it provides good enough results for playing with the gathered data a bit more.
