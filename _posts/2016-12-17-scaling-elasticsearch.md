---
layout: post-no-feature
title: "Scaling Elasticsearch to Handle Lots of Data"
description: "To scale for very large datasets, use Elasticsearch with filtered aliases"
category: articles
tags: [Elasticsearch, Scaling, Ruby, Full text search, user data flow]
---

Elasticsearch is a great data store for implementing full-text search and analytics. If your use-case warrants it, I would highly recommend using it. That said, there are certain applications where you need to be careful how you go about indexing your data.

### User-based Data Flow

Suppose you are a huge medical insurance company with millions of insurees, and for each insuree you have reams of patient data. Now suppose you'd like to be able to efficiently search that patient data and you decide to use Elasticsearch. How would you go about doing this?

A logical approach would be to store each patient data under its own patient index. My patient data would be stored under my index, and your patient data would be stored under your index. This is known as the [user-based data flow](https://www.elastic.co/guide/en/elasticsearch/guide/current/user-based.html).

Having an index per patient works if the number of patients is not too large. But our insurance company has millions of patients which means it also has millions of Elasticsearch indices. In this scenario, Elasticsearch would slow to a crawl.

There is a solution: We can trick the user into thinking we have an index per patient by using index aliases. Aliases are basically names that map to a specific index. In our case, we have millions of aliases mapping to a handful of primary indices. From a user perspective, each of these aliases looks like an individual index.

### Some Examples with Code

In the following code examples, I'm going to be using Ruby and Elasticsearch version 2.4.2. Ruby has an excellent [Elasticsearch gem](https://github.com/elastic/elasticsearch-ruby) that makes it easy to demonstrate what's going on. The documentation is also very good. I'm running Elasticsearch locally on port 9200.

Let's require the Elasticsearch gem and Ruby's JSON library, and make sure our Elasticsearch instance is up and running.

{% gist a7a7083623f4bda23450538bc08b69b1 setup.rb %}

Running ```client.info``` gives us the following result:

{% gist a7a7083623f4bda23450538bc08b69b1 setup_results.rb %}

Now we need to create the primary index that all the aliases will reference. I'm calling this index ```patients```. I couldn't find a convenient command in the Elasticsearch gem to perform a basic PUT HTTP request so for these types of requests I'm resorting to using Faraday.

{% gist a7a7083623f4bda23450538bc08b69b1 primary_index.rb %}

Now let's create aliases for two patients named ```PatientA``` and ```PatientB```.

{% gist a7a7083623f4bda23450538bc08b69b1 create_alias.rb %}

As you can see, the alias mapping is done through the ```patient_name``` field. For those still following, you may have noticed that there is an extra ```routing``` field in the request body. This field is important and I'll explain why next.

Typically, patient data would be stored in several different shards. When a search is performed within a particular patient's data, Elasticsearch would need to search several different shards and then combine those results. This is ineffecient. A better way would be to store all the data for a specific patient in one shard. Elasticsearch allows us to do this through routes. By providing a specific route, Elasticsearch will know to store a particular patient's data in the shard specified by the route. That's what the ```routing``` field is for.

We are now ready to store some documents for each patient. In a production setting we may be storing huge amounts of data. We can improve indexing performance by disabling refresh and using the bulk API. Even though this example is using a tiny data set, I'll use both for demonstration purposes.

{% gist a7a7083623f4bda23450538bc08b69b1 index_docs.rb %}

Now we can perform the search!

{% gist a7a7083623f4bda23450538bc08b69b1 search.rb %}

Notice that when we performed the search on the primary index ```patients``` we can see that all 5 shards were involved in the search. When we searched on the alias ```PatientA``` only 1 shard was used. It looks like the aliasing is working. It also looks like the routing is working since only 1 shard was used when searching with the alias.

### Conclusion

If you expect to have a very large amount of data stored in your Elasticsearch instance, you will need to use index aliases with routing, otherwise your cluster's performance will degrade significantly.
