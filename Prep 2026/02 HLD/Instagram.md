# Design a Photo Sharing App Like Instagram - Practice Notes

![[Excalidraw/Instagram.excalidraw]]

### Design a Photo Sharing App Like Instagram Guided Practice - November 4, 2026

<!-- EXCALIDRAW_PRACTICE practiceId="cmw0yflqn0ydj01ady8l69u8y" -->

#### Key Takeaways

**Requirements**

* For media-heavy apps like Instagram, prioritize availability over consistency (AP in CAP theorem). Users expect content to always load, even if a like count or feed is slightly stale. Eventual consistency is an acceptable trade-off here.

* Always quantify non-functional requirements with specific numbers. For example, feed load time under 500ms is a concrete target that drives decisions like caching strategy and CDN usage. Vague goals like 'fast' are harder to design around.

* Explicitly listing out-of-scope features during requirements gathering is a strong interview habit. It shows you can focus the design and prevents scope creep, which interviewers appreciate as a signal of real-world engineering judgment.

**Core Entities**

* Avoid creating separate tables for similar entity types like ImagePost and VideoPost. Instead, use a single Post entity with a 'media_type' field. This keeps the schema flexible and avoids duplicating shared fields like captions, likes, and timestamps.

* A Follow relationship in a social graph is best modeled as a join table with two user ID columns, like follower_id and followee_id. This lets you efficiently query both who a user follows and who follows them.

**API**

* When a REST endpoint targets a specific resource, put the identifier in the URL path rather than the request body. For example, use POST /follows/{userId} instead of POST /follows with a followedUserId in the body. This makes the target resource explicit and follows REST conventions for resource-oriented design.

* Never upload large files like photos or videos directly in a POST request body. Instead, use a presigned URL flow: the client asks your server for a temporary upload URL, uploads the file directly to object storage (like S3) using that URL, then sends only the resulting media reference ID to your server. This avoids overloading your API servers with large binary payloads.

* The current user's identity should always come from an Authorization header containing a token, not from a field in the request body or URL. This is both a security best practice and a standard convention because it keeps authentication separate from the request payload and prevents callers from spoofing other user IDs.

**High Level Design**

* When using presigned URLs for media uploads, the flow has three steps: the client requests a presigned URL from your app server, the client uploads the file directly to blob storage using that URL, and blob storage returns a media ID that your app server then stores on the post record. This keeps large file transfers off your application servers entirely.

* A follow relationship is a directed edge, meaning user A following user B does not mean user B follows user A. Store it as a follower table with two columns, followerId and followedId, plus a createdAt timestamp. This simple structure is enough to answer all follow queries and supports analytics and debugging later.

* For a fan out on read feed, you need two key database indexes to keep reads fast. Add an index on followerId in the follow table so you can quickly look up who a user follows. Add a composite index on (userId, createdAt) in the post table so you can efficiently fetch posts from multiple users sorted by time.

* When paginating a sorted feed, the order of operations matters. First fetch posts from all followed users, then merge and sort by createdAt descending, and only then apply your cursor and limit to slice out the correct page. Applying pagination before sorting will return incorrect results.

**Deep Dives**

* When a user crosses the celebrity threshold and switches from fan-out-on-write to fan-out-on-read, stale precomputed feed entries still exist in followers' caches. You need a plan for this transition, such as marking those entries as invalid or letting them expire via TTL, so followers do not see duplicate or missing posts during the switchover.

* For large media uploads, the correct flow is: your server creates an upload session and generates pre-signed URLs for each multipart chunk, the client uploads chunks directly to object storage in parallel, and then object storage fires a completion event to your backend. This keeps application servers out of the data path entirely and avoids treating a partially uploaded file as ready.

* CDN delivery alone is not enough for media-heavy feeds. You should preprocess uploaded media into multiple resolutions and bitrates after upload, then serve the best-fit variant based on the client's screen size and network speed. This reduces bandwidth and speeds up rendering without changing your storage architecture.

​

#### My Answers

**Requirements**

**Q:** What are the non-functional requirements for this system?

> Availability >> consistency for photos and videos Feed response time << 500ms Photo and videos should render with minimal latency Scalable upto 500M DAU with 100M posts Out of scope: Authorization and authentication GDPR compliance and other countries rule compliance Analytics about users and posts

**Core Entities**

**Q:** What are the core entities of the system?

> Post: will have user and media. Image and video post not separate.\
> Media: can be images/videos.\
> Follow: relationship between two users\
> User: client of our system, will have user details

**API**

**Q:** What are the main API endpoints that this system will need?

> GET /feed?limit={pageSize}&from={cursor} -------------> posts[]\
> POST /posts ----------------------------------------> postId { "media": mediaUploadId, // uploaded directly using presigned-uploads "caption": "My cool photo!", }\
> POST /follows ----------------------------------------> {success:true/false} { "followedUserId": "123" }

**High Level Design**

**Q:** How will users create posts with photos or videos?

> Clients when trying to post will request present URLs. Start uploading media there, get a media ID. Then call our post API. which creates the post. With caption, media ID is which creates the post. With caption, media ID is , and other info, if needed. Then via our API gateway, the request is to our post service, which will insert a post record in our database, database. Since the data is quite structured, let's use Postgres for our Then via our API gateway, the request is to our post service, which will insert a post record in our database, database. Since the data is quite structured, let's use Postgres for our database. And it will get a new post row. Which will have post ID, user ID. Media IDs related to that post. Caption created at, and other fields. Which will have post ID, user ID. Media IDs related to that post. Caption created at, and other fields.

**Q:** How will users follow other users?

> For following a use, another user. We'll make a API call. To our follower service via our API gateway, which is responsible for routing authentication. Etcetera. To our follower service via our API gateway, which is responsible for routing authentication. Etcetera. The follower service will. The follower API will. Take followed user ID. And then put. follower API will. Take followed user ID. And then put. Either. We can. For, we can extract follower ID. Either from our token or. From a header, whichever we prefer. And then add a record in our follower table. With followed ID and follower ID. Either from our token or. From a header, whichever we prefer. And then add a record in our follower table. With followed ID and follower ID. Both are user IDs, which identify who was, who followed whom. This table can have extra data if you want, but for now. We are trying to keep it simple. We are trying to keep it simple.

**Q:** How will users see a chronological feed of posts from users they follow?

> How it will primarily function is. Client will make any pay call to the post service with. Client will make any pay call to the post service with. A get call with. Get my feed with cursor and limit. The post service. Queries database for. All the user the. Get my feed with cursor and limit. The post service. Queries database for. All the user the. Feed requesting user follows. And then requests then queries the post table for all the posts from those users. And merges them, sorts them, and applies the cursor and limit on them and returns to client. For a user following many users, this is really quite slow. But I think in deep dives, the question is gonna come up. That how we make food generation. Fast. That how we make food generation. Fast.

**Deep Dives**

**Q:** How would you scale the feed generation to support users who follow thousands of accounts while maintaining low latency?

> We can try to. Precompute. Precompute. Our feed for all the users. Let's say when a user... creates a post. creates a post. We trigger off async operations. Long running operations, which upgrade feed, which add that post to feed of all the user. Which follow the feed , which add that post to feed of all the user. Which follow the user which created the post. Though this will work for most of users. Because Because , post creation to post viewing ratio is very high. to post viewing ratio is very high. But for celebrities, this will create a problem. For example, let's say Taylor Swift, she is followed by twenty million people. So, writing her posts to feed off twenty million people will take very long time. So, we can maybe use a combination, a hybrid approach. use a combination, a hybrid approach. Where for all... for, let's say, non-celebrity users. We... for, let's say, non-celebrity users. We... directly fan out. When the post is being written. But for celebrity users. We fan out when the feed is being read. When the post is being written. But for celebrity users. We fan out when the feed is being read. So how will our feed query work when a. User makes a feed query, we will take its pre computed feed. User makes a feed query, we will take its pre computed feed. And add to that the celebrity, the post from, recent post from the celebrities which this user follows.

**Q:** How would you choose and evolve the threshold between regular and celebrity accounts so that traffic spikes or fast follower growth do not cause unstable feed behavior?

> If we simply rely on follower count. For identifying a user as celebrity or not. That can create problems because. Let's say a user sees a sudden growth. we simply rely on follower count. For identifying a user as celebrity or not. That can create problems because. Let's say a user sees a sudden growth. Of its. Posts. Or its followers. Then. Our feed generation can run into hiccups. So to ensure. We are. Stabily generating feed by the mechanisms which we have described in the previous steps. Then. Our feed generation can run into hiccups. So to ensure. We are. Stabily generating feed by the mechanisms which we have described in the previous steps. We can. Say like. Say like. We have to discard bursts. In follower growth. Let's say. In follower growth. Let's say. If we have a threshold for. Ten thousand users as. For categorizing a, an user as celebrity. If you stay above ten thousand for. If you stay above ten thousand for. Let's say, twenty four hours, then only you are categorized as. Celebrity. Celebrity. Till in that time. We can handle the fee. The way we were previously handling.

**Q:** How would you handle the upload of large media files efficiently, particularly videos that could be up to 4GB in size?

> For the efficient handling of large media uploads, we'll implement a client-side multi-part upload system using S3's BSL and URLs. First, when a user initiates a post, our post service creates a post record. With still suspending. And generates BSL URL. For S3 multi-part upload. These URLs allow the client to upload chunks directly to S3 without going through our servers. The client then splits the video into chunks. With still suspending. And generates BSL URL. For S3 multi-part upload. These URLs allow the client to upload chunks directly to S3 without going through our servers. The client then splits the video into chunks. One to four MB each let's say. And uploads them in parallel directly to S3 using the BSL URLs. This approach has several advantages. It's resilient to network interruptions. Since failed chunks can be retrieved independently. It reduces load on our application servers since they are not handling the binary data. And it leverages S3. This approach has several advantages. It's resilient to network interruptions. Since failed chunks can be retrieved independently. It reduces load on our application servers since they are not handling the binary data. And it leverages S3. Highly scalable infrastructure for storage. To track completion we'll configure S3 event notifications to trigger a lambda function when all chunks are successfully uploaded. This function will update the post status to complete. And add the final S3 object key to our post metadata. The server driven approach is more reliable than trusting clients to record upload completion. To track completion we'll configure S3 event notifications to trigger a lambda function when all chunks are successfully uploaded. This function will update the post status to complete. And add the final S3 object key to our post metadata. The server driven approach is more reliable than trusting clients to record upload completion.

**Q:** How would you make the upload flow idempotent so a client can safely retry part uploads and the final completion call without creating duplicate or corrupted media state?

> To handle this item potency. For uploads. Phase one, we can, during initiation, we need a competency. We can have client send an a competency key header. For every upload request. Or For every upload request. Or , and server checks for that already existing key and returns the existing upload ID instead of creating a duplicate session. While uploading parts , we can maintain a state table if a chunk is recent due to network drop. The server checks the ledger. If it already marked success with a matching checksum, it returns okay immediately without rewriting. The server checks the ledger. If it already marked success with a matching checksum, it returns okay immediately without rewriting. During completion, we use idempotency key and the database transaction to move state from uploading. to committing. to completed. to completed. Storage provider completions is three. Is inherently unimportant with the same. Part list. Part list. Also, in case we can run a background worker to abort and clean up the orphaned sessions stuck in initiated or uploading for more than twenty four hours. initiated or uploading for more than twenty four hours.

**Q:** How would you ensure fast media delivery to users globally, with photos and videos rendering quickly regardless of a user's location?

> We use CDN to deliver our images and videos to the users. Our media processors. Can. Process a video or image. For multiple resolutions. Can. Process a video or image. For multiple resolutions. And store them and serve them. Via CDN. Via CDN. Depending on users. Network speed or let's say screen size. Whatever the users need. We can get that resolution from our CDN. Network speed or let's say screen size. Whatever the users need. We can get that resolution from our CDN. This will help us reduce the consumption of users net. Bandwidth. Bandwidth.

​