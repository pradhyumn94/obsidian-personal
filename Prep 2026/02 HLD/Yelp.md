---
status: Done
category: Search
date: 2026-10-25
---

# Design a Local Business Review Site Like Yelp - Practice Notes

![[Excalidraw/Yelp.excalidraw.md]]

See also: [[Proximity Search]] · [[PostgreSQL]] · [[Prep 2026/03 Engineering Notes/DB Indexing|DB Indexing]] · [[Elasticsearch]] · [[Handling Contention]] · [[CAP Theorem]].

### Design a Local Business Review Site Like Yelp Guided Practice - October 25, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmvnsbecl1fu401ad1vluwceu" -->

#### Key Takeaways

**Requirements**

* When stating latency requirements, always pair them with a concrete number like under 500ms for search or under 100ms for autocomplete. Vague goals like 'low latency' are harder to defend in an interview than specific, measurable targets.

* For systems where data freshness is not critical, like reviews on Yelp, choose availability over consistency using CAP theorem reasoning. This means the system stays up even if some nodes are out of sync, and data eventually propagates. This trade-off is called eventual consistency.

* Anchor your scale estimates to a DAU number and work outward. For example, 100M DAU might mean roughly 1000 requests per second at peak if each user makes a few requests per day. Having this math ready shows you understand the operational demands of the system.

**Core Entities**

* For a Yelp-like system, the three core entities are User, Business, and Review. User creates Reviews, Reviews belong to a Business, and a Business can have many Reviews. Knowing these relationships upfront shows you understand the domain before jumping into infrastructure.

**API**

* When designing a search endpoint that takes a geographic location, prefer explicit separate query parameters like lat and lng over a single ambiguous location field. This removes ambiguity about the expected format and makes the API contract clearer for consumers without requiring extra documentation.

**High Level Design**

* When describing a request flow verbally, always walk through every hop in order including the API gateway. Say 'client sends request to API gateway, gateway routes to business service, business service queries the database' rather than skipping the gateway. Interviewers check that your verbal explanation matches your diagram.

* When explaining how a database returns results, say it queries indexed fields such as name, category, and location rather than saying it scans for matches. A scan implies a full table scan which is slow, while querying indexed fields is the correct and efficient behavior you are actually designing for.

* When a write operation happens, describe it as persisting a new row in the relevant table rather than saying the service updates the details. This makes it clear you are inserting a new record rather than modifying an existing one, which is an important distinction for correctness and for showing you understand basic database write semantics.

* Always include a userId field on review records in your data model. This ties each review to its author, which is required for displaying who wrote the review, enforcing one-review-per-user rules, and supporting future features like flagging or editing reviews.

**Deep Dives**

* When using PostgreSQL with PostGIS and GIN indexes together, you write a single SQL query with both predicates (e.g., ST_DWithin AND a full-text match) and let the query planner combine the indexes internally. Never split this into two separate queries and intersect the results in application code, because that pulls large result sets over the network before filtering, which is slow and wasteful.

* When combining a geospatial filter and a text filter in one query, apply the most selective filter first to shrink the candidate set. Radius filtering with PostGIS is usually more selective than text search, so structure your query so the location predicate runs first and reduces the rows the text filter has to scan.

* For named geographic regions like cities or neighborhoods, model them as polygons not points. A single lat/long with a radius is inaccurate for irregularly shaped areas. Store polygons in a locations table and precompute which region each business belongs to at write time, so search reads become a simple filter instead of an expensive spatial join.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements?

> Low latency for search (<500ms) Highly available and eventual consistent Support 100M DAU and 10M business Out Of Scope Protect user data and GDPR Protection against spam and abuse

**Core Entities**

**Q:** What are the core entities of the system?

> User - people who will view and review business Business - register on platform. Name location (lat,long), description details Review - submitted for a business by a user

**API**

**Q:** What are the external REST APIs that your system will expose in order to satisfy the functional requirements?

> // search GET /businesses?query&location&category&page -> Business[]\
> // get business details GET /businesses/:businessId -> Business\
> // paginated reviews by recency say 10 on every page GET /businesses/:businessId/reviews?page= -> Review[]\
> // Post a review POST /businesses/:businessId/reviews { rating: number, text?: string }

**High Level Design**

**Q:** How will users be able to search for businesses?

> Client will call business service. With term location category. Location can be. User's current location. Or user can specify a location. And then when the service reaches out to database. Which in this instance is a Postgres. Server. And it scans whatever matches. Whatever services match the query, and then return those businesses to. Our client. For handling location related queries, let's say , within two kilometers of radius. We can use PostGIS Index.

**Q:** How will users be able to view a businesses details and reviews?

> Client will reach out. To business service. Via API gateway. You calling the get APIs. With the business ID parameter. Business service will then. Fetch the business. And its reviews. And return there. Another way to see this can be one network call returns business details. And then a page initiate. View of reviews can return. The reviews for the.

**Q:** How will users be able to leave reviews on businesses?

> Plans will call or post. U P I. And why I beg it will reach up to review service. With each review will have. User ID, business ID, text rating. And optional media. Sources, videos or images. Review service then updates these to our details.

**Deep Dives**

**Q:** How would you efficiently calculate and update the average rating for businesses to ensure it's readily available in search results and still accurate up to the minute?

> We'll keep appreciating a number of reviews on business. Self. In business table itself. Whenever user submits a review, we can use pre-calculated average rating and number of reviews to calculate the new average rating. And then update it to the business table. While also saving the review. You tell me top challenge can be that what if two users try to update at the same time? We can, since we do not need a very explicit lock. As the write volume is not that much, very few people compared to how many people are reading the business data will leave their views. We can take an approach of optimistic and currency control. Proceed. Which leads the reputation and number of reviews and before calculates the new one and then before going to commit the new calculations. Checks again if number of reviews are same. If they have changed. In between, it runs the process again. Fails and runs the, retries the process. Otherwise. Process.

**Q:** How would you modify your system to ensure that a user can only leave one review per business?

> An application level check will be a bad idea. As we will have to. Read the user's review for that business. Keep a memory and then check whether it exists or not. Take a database call on a pre-review, which can overwhelm our database if not applied properly. Another one can be , right solution can be, we can apply a constraint on our database, where the constraint is based on user ID and business ID. With the unique constraint in place, database will reject trying to update the same row, and we just need to handle that error gracefully and pass it down to the client.

**Q:** How would you structure the write flow so that creating the review and updating the business average rating stay consistent when the unique constraint rejects a duplicate submission?

> Creating the review and updating the business. Will be. Transaction. Since we are using. Same database. But different tables for this. This can be easier.

**Q:** How can you improve search to handle complex queries more efficiently?

> For more complex queries. On text, maybe we can use an extension. Postgres extension. Gen, a genindex for a full text search. And. Postgis index. For a location based queries. Will. Help our system. And since the data is not that much. Indexing size is not that big a problem.

**Q:** How would you structure the query so the database can efficiently combine radius filtering with term and category matching without doing unnecessary work?

> . Our system has different indexes for different purposes. Post TIS index for location related filtering. And gene index. For full text search. We can apply both the predicates to our database query and have the query planner. Do. Most of the work for us. And get a array of businesses, which is filtered by both.

**Q:** How would you modify your system to allow searching by predefined location names such as cities (e.g., 'San Francisco') or neighborhoods (e.g., 'Mission District')? Assume you have a finite list of supported location names.

> We need to know two things. To solve this. Whether a particular business which is identified via. Latitude and longitude. It falls in a particular. Area polygon or not. To know. That we can. Create a new table in our database that makes location, that maps location names to polygons. These polygons can be sourced from an. Publicly available data sets. GeoAPI. And then to implement this, create a locations table with columns for name. For example, let's say. Los Angeles, Washington, Francisco, the type, city, neighborhood, and polygon to store the geographical data. This geographical data can be in the form of GeoJSON. Populate this table with data from our chosen sources. Index the name column for efficiency. This allows us to quickly translate a location name into its. corresponding polygon for use in geographic queries. Now how to filter businesses falling into a particular polygon, or a particular shape? Postgress via PostGIS extension and Elasticsearch have an handy for working with polygons, which they are. In which they call. Geo shapes or geopoints. In case of Elasticsearch we can simply add a. New geo shape. Field to our business documents. And use the geo shape query to. Find businesses that exist within our polygons. Doing this bounding box search on every request isn't efficient enough. So instead of filtering on the bounding box for each request, we can pre-compute the area. for each business on creation, and store them as a list of location and requests in our database.

​