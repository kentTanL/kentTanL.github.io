---
layout: post
title: MongoDB Using $FIRST with Aggregation
tags: [MongoDB]
categories: [MongoDB]
image:
  background: triangular.png
---

Lets assume that MongoDB is being used as a data store for an e-commerce application(similar to Amazon.com) on which users can shop for and buy the items they desire.

All the users purchases are stored in a collection named 'purchase_history'. Each document in the collection will be structured as below

{% highlight yaml %}

{
	"_id": ObjectId("581006910d96be013c3ee72d"),
	"itemId": "1112-3333-4442-1111",
	"userId": "353236",
	"category": "sports",
	"purchaseDate": ISODate("2016-10-26T01:27:45.195Z"),
	"purchasePrice": 32.3,
	"currency": "DOLLARS"
}

{% endhighlight %}



The site also has a section on the page which displays the recent 5 categories associated with the products the users shopped.

For example if the user purchased 10 items of different categories in the following order :


sports
kitchen
sports
furniture
sports
sports
clothing
jewellery
sports
shoes


The "Recent 5 categories" section with show the following data

sports
kitchen
furniture
clothing 
jewellery


<figure class="center">
	<img src="/images/mongo/recent-categories.png" height="800px"></img>
</figure>


Logically thinking, we can achieve the above result if we do the following steps :

1. Sort the data in the 'purchase_history' by 'purchaseDate' descending order so that the most recent purchase is at the top.
2. Filter distinct values for the 'categories' in the sorted result of step #1.
3. Extract the top 5 values.

The above can be achieved through Mongo query using Aggregation framework combined with Group and $FIRST.

{% highlight yaml %}
 
 db.purchase_history.aggregate([{
   $match:{userId:"353236"}},
   {$sort : { "purchaseDate" : -1}} , {$group:{_id:"$category" , 'purchaseDate': {$first: '$purchaseDate'}}},{$sort : { purchaseDate : -1} }]); 
                     
 
 
 Overview :
 
 1. Find all records associated with the userId.
 2. Sort by purchaseDate descending order.
 3. Group by category and project 'purchaseDate' as the first record in each group. The first record in this case will be the record with the latest "purchaseDate' for a each category.
 4. Resort again by the projected purchaseDate becase the 'group' operation does not maintain any previous ordering.
 
 
 
{% endhighlight %}

