# Cloud Data

Data lives in the cloud, it has weight, and PuffinDB is the **Cloud Data Engine** working against the cloud's gravitational pull.

## Data Weight
When thinking about data, one should not think in terms of small data or big data. This volumetric classification isn't really helpful anymore. Instead, one should think in terms of **weight** `W`. Data has **mass** (the larger the dataset, the larger its mass `m`), and lives within a cloud that exerts a gravitational pull (weight) on it, with a fixed [gravitational constant](https://en.wikipedia.org/wiki/Gravitational_constant) `g`. The larger data gets, the stronger a gravitational pull is exerted on it by the cloud it resides in. According to Newton's second law of motion: `W = m·g`

Following this analogy, a cloud's gravitational constant is a factor of its internal bandwidth (how fast can you move data from an object store to a compute engine), its external bandwidth (how fast can you download data from the cloud to your client computer), and its egress cost (how much does it cost to do the latter). Of course, like any other analogy, ours is imperfect, but it should be helpful to illustrate certain important points that follow.

## Data at Rest
At rest, data lives in a Lake ([Iceberg](https://iceberg.apache.org/), [Delta Lake](https://delta.io/), [Hudi](https://hudi.apache.org/)) backed by an Object Store ([Amazon S3](https://aws.amazon.com/s3/), [Azure Blob Storage](https://azure.microsoft.com/en-us/products/storage/blobs), [Google Cloud Storage](https://cloud.google.com/storage)). According to our analogy, it is laying still on Earth's surface, wasting very little energy to do so (translation: object stores are dirt cheap).

## Data in Motion
Because the object store's advent established a clear separation between storage and compute, data cannot be queried or updated directly from the object store (with the exception of trivial queries done with [`SelectObjectContent`](https://docs.aws.amazon.com/AmazonS3/latest/API/API_SelectObjectContent.html)). Instead, it needs to be moved from the object store to a compute node (serverless function, serverless container, container, or client), then queried or updated from there.

This data movement is expensive, for two mains reasons: first, storage and compute capacities increase faster than network bandwidth; second, bits cannot move faster than the speed of light (they move quite a bit slower actually, and never in a straight line). This means that **bandwidth** and **latency** are a cloud's main limiting factors. They are facts of life that we must learn to live with.

With that in mind, one could think of moving data from the object store to a compute node within the same [availability zone](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html) as taking a flight from Paris to Tokyo. It is no doubt expensive, but nowhere near as much as putting a satellite on orbit around the Earth, or going to Mars. Following our analogy, moving data from the object store to a [Content Delivery Network](https://en.wikipedia.org/wiki/Content_delivery_network) (CDN) for edge computing is akin to putting a satellite on low-Earth orbit, while downloading a full dataset from the object store to your laptop computer for local computing is a bit like establishing a human colony on Mars. It certainly can be done, but is it really worth the expense?

Can we justify this analogy in any quantitative manner? Absolutely!

If we assume our Paris-Tokyo flight to cost $1,000 and our payload (passenger and luggage) to be 100kg, we get a cost of $10/kg. As a point of comparison, SpaceX's [Falcon Heavy](https://www.spacex.com/vehicles/falcon-heavy/) offers low-Earth orbit deliveries for as little as $1,400/kg, which is 140 times more expensive. And current missions to Mars cost about $200,000/kg, or 142 times more that low-Earth orbit deliveries. In other words, each jump is about two orders of magnitude more expensive than the previous one. Granted, SpaceX promises Starship missions to Mars for as little as $50/kg, but that remains to be seen, and low-cost carriers routinely take passengers across continents for less than $50. Furthermore, sending you and your suitcase to Mars is one thing, but sending along all the equipment and supplies necessary for your survival once you get there is another thing altogether. You get the point...

From the hotel room where I am writing this article, I enjoy (suffer from) a bandwidth of just over 60 Mbps. As a point of comparison, a [`p4d.24xlarge`](https://aws.amazon.com/ec2/instance-types/p4/) instance gives me 400 Gbps of bandwith, which is 6,667 times greater. And the 3,000 Lambda functions that I can provision within a second or less give me an aggregated bandwidth from S3 of 2.4 Tbps, or 40,000 times greater. Therefore, if going to Mars is 20,000 times more expensive than flying from Paris to Tokyo, downloading a dataset to your laptop is indeed akin to going to Mars...

## Data Caching
As a result, since we cannot bring compute to data (until object stores get [upgraded](docs/Future-Proofing.md) with a full SQL engine like [DuckDB](https://duckdb.org/)), we need to bring data to compute, but we need to do so at the lowest possible cost. This is called **caching**, and it is the most important feature of any distributed database engine. Now, when we write about **cost**, we mean two things: first, money; second, latency, because time is money. Said another way, I want to query (or update) my dataset as quickly as possible, for as little money as possible.

As explained earlier, data at rest lives in the object store, therefore will have to be moved from there to some compute engine. We could use a traditional container for that (*e.g.* [EC2](https://aws.amazon.com/ec2/) instance), but its bandwidth is limited (400 Gbps at most). Instead, we are probably better off using a fleet of serverless functions (*e.g.* [AWS Lambda](https://aws.amazon.com/lambda/)): 3,000 of them can be provisioned within less than a second, and offer an aggregated bandwidth from S3 of 2.4 Tbps (6 times more). This is called **scale out** (we will come back to that later in the article).

But when a query is made on a dataset (or a particular subset of a dataset) by a data analyst, it is quite likely that another query will be made on the same dataset soon after. Therefore, we will want to cache onto our compute engine(s) the data that we have just moved from the object store. This is very easy when using a traditional container, but it can also be done using serverless functions and containers.

From there, we will soon realize that scaling out queries across multiple compute nodes works well for certain queries or certain query parts, but is much more difficult for others. Therefore, we will also want to involve one bigger compute engine (we like to call it a **monostore**) that can be used to cache large amounts of data and process really complex queries against them. For example, Amazon Web Services now offer their [`u-24tb1.112xlarge`](https://aws.amazon.com/ec2/instance-types/high-memory/) instance on demand, with 448 vCPUs and 24 TB of RAM.

Unfortunately, it only comes with 100 Gbps of network bandwidth (4 to 8 times more would be more appropriate), which means that filling its RAM with data would take at least 1,920 seconds (32 minutes). Even if you load data compressed at a 5× factor and cache it uncompressed in memory, you will need to wait 384 seconds (over 6 minutes) for the full dataset to be downloaded. And when you do so, you will want to scale out data download from the object store to the monostore by using a fleet of serverless functions in between, especially if data can be filtered at the source (*i.e.* **filter pushdown**).

Then, you will realize that caching data on the monostore could be done at four different levels:

1. Compressed on local solid state drives ([NVMe](https://en.wikipedia.org/wiki/NVM_Express) ideally)
2. Compressed in CPU memory
3. Uncompressed in CPU memory
4. Uncompressed in GPU memory (if you are lucky enough to use a GPU-accelerated monostore like the [`p4de.24xlarge`](https://aws.amazon.com/ec2/instance-types/p4/))

After that, you might consider caching closer to their users small to medium datasets (*Cf.* [Dataset Sizes](https://github.com/stoic-doc/Community/discussions/905)) that are used very frequently, using a [Content Delivery Network](https://en.wikipedia.org/wiki/Content_delivery_network) (CDN) and some edge serverless functions for querying. And only when you have exhausted all these options should you consider caching datasets all the way to the client. Welcome to Mars!

This incremental caching of data closer and closer to the client and the distribution of queries across layers of caching is called **scale up**.

## Scale Out and Scale Up
In order to take advantage of these complementary **scale out** and **scale up** strategies, the [distributed query engine](docs/Query%20Engine.md) will need a very powerful [distributed query planner](docs/Query%20Planner.md).