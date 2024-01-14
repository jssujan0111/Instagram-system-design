
  ##  System Design Case Study 2

                       Topic: Instagram
## Introduction: 
Instagram is a free social app for sharing photos and videos with captions. Posts can be organized using hashtags and geotags, making them searchable. Posts are visible to followers and the public if tagged. Users can set profiles to private for restricted access to followers.
## Requirements
# Functional Requirements
•	**Post photos and videos:** The users can post photos and videos on Instagram.

**•	Follow and unfollow users:** The users can follow and unfollow other users on Instagram.

**•	Like or dislike posts:** The users can like or dislike posts of the accounts they follow.

**•	Search photos and videos:** The users can search photos and videos based on captions and location.

**•	Generate news feed:** The users can view the news feed consisting of the photos and videos (in chronological order) from all the users they follow. Users can also view suggested and promoted photos in their news feed.

# Non functional Requirements
**•	Scalability:** The system should be scalable to handle millions of users in terms of computational resources and storage.

**•	Latency:** The latency to generate a news feed should be low.

**•	Availability:** The system should be highly available.

**•	Durability:** Any uploaded content (photos and videos) should never get lost.

**•	Consistency:** We can compromise a little on consistency. It is acceptable if the content (photos or videos) takes time to show in followers’ feeds located in a distant region.

**•	Reliability:** The system must be able to tolerate hardware and software failures.

# Extended Requirement
1.	Metrics and Analytics
3.	Accessibility
4.	Integrations with third party service
# Estimation and Constraints
### Traffic:
As a social media application, It’s important to consider that read requests will be significantly more frequent than write requests, with a ratio approximately **100:1.**

As per as our requirements we have **300 million monthly active users**. So we have **10 million daily active users.** 

Let’s assume that Each user posts 5 post per day including images or videos. So this gives up **50 million post per day**

**10million * 5 post  = 50 million / per day**

Now, we assume that 10 percent videos and 90 percent images & text. 

**50 million * 10 percent = 5 million / per day. (videos)

50 million * 90 percent = 45 million / per day. (images and texts)**

What would be request per second (RPS) for our system?

**50 million / (24 hrs * 3600s ) ~= 5k per second.**

### Storage
If we assume that Each post is about **150 kb** on average (including images and videos) , we will require **7.5TB** of database storage everyday.

**50 million * 150 kb ~= 7.5 TB / per day.**

And for 10 years , we will require 27 PB of database storage.

**7500GB * 10 * 365 =  27 PB**

### Bandwidth
As our system is handling **7.5 TB** of ingress database everyday , 
we will require a  minimum bandwidth of around **75mb** per second.
**7.5TB / ( 24 hours * 3600s ) =~ 75MB**

## High Level Estimation
|**Type** |	**Estimate**|  
|-------------|--|
|    Daily Active Users (DAU)| 10 million | 
|  Request Per Second| 5k/s |
| Storage Per Day| =~ 7.5TB |
| Storage After 10 Years| 	=~ 27 PB|
| Bandwidth	| =~ 75 MB/s |
##  Data Model
 
![enter image description here](https://i.ibb.co/mcvF2kY/instagram.png)
## Database
While our data model seems quite relational, we don't necessarily need to store everything in a single database, as this can limit our scalability and quickly become a bottleneck. 
We will split the data between different services each having ownership over a particular table. 
Then we can use a relational database such as PostgreSQL or a distributed NoSQL database such as Apache Cassandra for our use case.

N.B:  We can store photos in a distributed file storage like HDFS or S3.

# API Design
## API Endpoints::


| Name   | Path | Method  	|Param	  | Return 
|--|--|--|--|--|
| New post | POST  | /posts | userID, images, videos | New post |	
|Follow unfollow |	POST |	/follow |	followed, followerID| Boolean |
|Get newsfeed/timeline |	GET |	/feed |	userID |	List of posts

# High Level Design
##  Which Architecture should we choose?
    1.	Monolithic
    2.	Micro service
**To scale easily we can choose micro service architecture.**
## Services
     1.	User services
     2. News feed services
     3.	Media services
     4.	Notifications services
     5.	Analytics Services
## Inter service Communication
Since our architecture is microservices-based, services will be communicating with each other as well. Generally, REST or HTTP performs well but we can further improve the performance using gRPC which is more lightweight and efficient. 
Service discovery is another thing we will have to take into account. We can also use a service mesh that enables managed, observable, and secure communication between individual services.

## Generate timeline/newsfeed
 ### Feed Generation:
Let's assume we want to generate the feed for user A, we will perform the following steps: 

    1. Retrieve the IDs of all the users and entities (hashtags, topics, etc.) user A follows.
    2. Fetch the relevant tweets for each of the retrieved IDs.
    3. Use a ranking algorithm to rank the tweets based on parameters such as relevance, time, engagement, etc.
    4. Return the ranked tweets data to the client in a paginated manner.

   Note: Feed generation is an intensive process and can take quite a lot of time, especially for users following a lot of people. To improve the performance, the feed can be pre-generated and stored in the cache, then we can have a mechanism to periodically update the feed and apply our ranking algorithm to the new tweets.

### Publishing Feeds
Publishing is the step where the feed data is pushed according to each specific user. This can be a quite heavy operation, as a user may have millions of friends or followers. To deal with this, we have three different approaches:

    1. Pull Model (or Fan-out on load) 
    2. Push Model (or Fan-out on write) 
    3. Hybrid Model
### PULL Model
When a user creates a tweet, and a follower reloads their newsfeed, the feed is created and stored in memory. The most recent feed is only loaded when the user requests it. This approach reduces the number of write operations on our database. The downside of this approach is that the users will not be able to view recent feeds unless they "pull" the data from the server, which will increase the number of read operations on the server.
### PUSH Model
In this model, once a user creates a tweet, it is "pushed" to all the follower's feeds immediately. This prevents the system from having to go through a user's entire followers list to check for updates. However, the downside of this approach is that it would increase the number of write operations on the database.
### Hybrid Model
A third approach is a hybrid model between the pull and push model. It combines the beneficial features of the above two models and tries to provide a balanced approach between the two. The hybrid model allows only users with a lesser number of followers to use the push model. For users with a higher number of followers such as celebrities, the pull model is used.
Where do we store the timeline?
We store a user’s timeline against a userID in a key-value store. Upon request, we fetch the data from the key-value store and show it to the user. The key is userID, while the value is timeline content (links to photos and videos). Because the storage size of the value is often limited to a few MegaBytes, we can store the timeline data in a blob and put the link to the blob in the value of the key as we approach the size limit.

## Ranking Algorithm
**Rank = Affinity x Weight x Decay** 

Affinity: is the "closeness" of the user to the creator of the edge. If a user frequently likes, comments, or messages the edge creator, then the value of affinity will be higher, resulting in a higher rank for the post. 

**Weight:**  is the value assigned according to each edge. A comment can have a higher weightage than likes, and thus a post with more comments is more likely to get a higher rank.

 **Decay:** is the measure of the creation of the edge. The older the edge, the lesser will be the value of decay and eventually the rank. Nowadays, algorithms are much more complex and ranking is done using machine learning models which can take thousands of factors into consideration.
## Search
Sometimes traditional DBMS are not performant enough, we need something which allows us to store, search, and analyze huge volumes of data quickly and in near real-time and give results within milliseconds. Elasticsearch can help us with this use case. Elasticsearch is a distributed, free and open search and analytics engine for all types of data, including textual, numerical, geospatial, structured, and unstructured. It is built on top of Apache Lucene.
## Identify trending topics
Trending functionality will be based on top of the search functionality. We can cache the most frequently searched queries, hashtags, and topics in the last N seconds and update them every M seconds using some sort of batch job mechanism. Our ranking algorithm can also be applied to the trending topics to give them more weight and personalize them for the user.
## Notifications
Push notifications are an integral part of any social media platform. We can use a message queue or a message broker such as Apache Kafka with the notification service to dispatch requests to Firebase Cloud Messaging (FCM) or Apple Push Notification Service (APNS) which will handle the delivery of the push notifications to user devices.
# DETAILED DESIGN
### Data Partitioning
To scale out our databases we will need to partition our data. Horizontal partitioning (aka Sharding) can be a good first step. We can use partitions schemes such as: 

     1.  Hash-Based Partitioning 
     2.	List-Based Partitioning 
     3.	Range Based Partitioning 
     4.	Composite Partitioning 
The above approaches can still cause uneven data and load distribution, we can solve this using Consistent hashing.
## Mutual Friends/followers
For mutual friends, we can build a social graph for every user. Each node in the graph will represent a user and a directional edge will represent followers and followees. After that, we can traverse the followers of a user to find and suggest a mutual friend. This would require a graph database such as Neo4j and ArangoDB. This is a pretty simple algorithm, to improve our suggestion accuracy, we will need to incorporate a recommendation model which uses machine learning as part of our algorithm.
## Metrics and Analytics
Recording analytics and metrics is one of our extended requirements. As we will be using Apache Kafka to publish all sorts of events, we can process these events and run analytics on the data using Apache Spark which is an open-source unified analytics engine for large-scale data processing.
## Caching
**Which cache eviction policy to use?** 

We can use solutions like Redis or Memcached and cache 20% of the daily traffic but what kind of cache eviction policy would best fit our needs? 
**Least Recently Used (LRU)** can be a good policy for our system. In this policy, we discard the least recently used key first. 
How to handle cache miss?
 Whenever there is a cache miss, our servers can hit the database directly and update the cache with the new entries.
Media Access and Storage
As we know, most of our storage space will be used for storing media files such as images, videos, or other files. Our media service will be handling both access and storage of the user media files.
 But where can we store files at scale?
 Well, object storage is what we're looking for. Object stores break data files up into pieces called objects. It then stores those objects in a single repository, which can be spread out across multiple networked systems. We can also use distributed file storage such as HDFS or GlusterFS.
# Content Delivery Network
### Content Delivery Network (CDN)
 Content Delivery Network (CDN) increases content availability and redundancy while reducing bandwidth costs. Generally, static files such as images, and videos are served from CDN. We can use services like Amazon CloudFront or Cloudflare CDN for this use case.

# Detailed Design implementation
![enter image description here](https://i.ibb.co/dkwX6XM/detailed-design-instragram.png)

## Identify Bottlenecks
 ● "What if one of our services crashes?"
 
 ● "How will we distribute our traffic between our components?" 
 
● "How can we reduce the load on our database?" 

● "How to improve the availability of our cache?" 

● "How can we make our notification system more robust?" 

● "How can we reduce media storage costs"?

## Resolve Bottlenecks
Resolve Bottlenecks To make our system more resilient we can do the following:

● Running multiple instances of our Servers and Key Generation Service. 

● Introducing load balancers between clients, servers, databases, and cache servers. 

● Using multiple read replicas for our database as it's a read-heavy system. 

● Exactly once delivery and message ordering is challenging in a distributed system, we can use a dedicated message broker such as Apache Kafka or NATS to make our notification system more robust. ● We can add media processing and compression capabilities to the media service to compress large files which will save a lot of storage space and reduce cost.

                          Thank You
