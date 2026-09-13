# Design a File Storage Service Like Dropbox - Practice Notes

### Design a File Storage Service Like Dropbox Guided Practice - September 13, 2026

! [[Dropbox.excalidraw]] 
#### Key Takeaways

**Requirements**

* Resumable uploads are a key non-functional requirement for any system handling large file uploads. If a network interruption occurs mid-upload, the system should track how much was already uploaded and resume from that point rather than starting over. This matters especially when files can be tens of gigabytes.

* For a file sync system like Dropbox, low latency needs to be tied specifically to cross-device sync speed, not just general response time. A concrete and interview-ready way to phrase this is: file changes should propagate to all other devices within roughly one minute of being saved.

* Functional requirements define what the system must do, like supporting files up to 50 GB. Non-functional requirements define how the system should behave, like being available 99.99% of the time or syncing changes within one minute. Keeping these two categories separate helps you structure your requirements clearly in an interview.

**Core Entities**

* Store file metadata (name, size, owner, timestamps, file path/URL) separately from the raw binary file content. Metadata lives in a relational database for fast lookups and querying, while the actual file bytes go into blob/object storage like S3. This split avoids loading large binary data just to answer simple questions like 'what files does this user have?'

**API**

* REST endpoints should be named after resources, not actions. Use POST /files to upload and GET /files/{fileId} to download, not POST /upload or GET /download. This makes your API predictable and idiomatic, which interviewers notice.

* Every mutating endpoint like POST /files should return something useful in the response, such as a fileId or the created file metadata. Without this, the client has no way to reference the resource in future requests like downloading or sharing it.

* A sync or changes endpoint that can return large result sets should always support pagination. Return a limit parameter on the request and a cursor or next_page_token in the response so clients can fetch results in manageable chunks instead of getting one massive payload.

* User identity should be passed in an Authorization header, not in the URL or request body. URLs can be logged by proxies and servers, which would expose sensitive credentials. Headers are the standard and safer place for auth tokens.

**High Level Design**

* A push notification alone is not enough for sync. It is only a hint that something changed. The client must follow up with a 'get changes' request to a change service, passing a cursor like a version number or timestamp, to get back the actual list of files that changed since its last sync. Without this query, the client cannot reliably know what to download, delete, or rename.

* The source of truth for what changed should be your file metadata database, not storage events. Storage can signal that bytes changed, but file names, deletes, moves, and renames are tracked in metadata records. The change service reads from that metadata using a version or timestamp cursor to return a reliable diff to the client.

* The full remote sync apply flow has three steps. First the client receives a push or polls for a signal. Then it calls the change service to get a list of changed items since its last cursor. Then it downloads updated file contents from blob storage when needed and writes those changes into the local folder so the device mirrors cloud state. Make sure you can walk through all three steps explicitly.

* When designing a local sync agent, the local SQLite database should store per-file metadata like path, last modified time, size, and sync status. This lets the agent quickly compare local state against remote state and decide what needs to be uploaded or downloaded, even after a restart.

**Deep Dives**

* When syncing large files, use per-chunk hashing (fingerprinting) to detect changes. Both client and server compute a hash for each chunk, compare the fingerprints, and transfer only the chunks whose hashes differ. Without this step, chunking alone does not tell you which chunks actually changed.

* Fixed-size chunking breaks down when bytes are inserted or deleted near the start of a file because it shifts all subsequent chunk boundaries, making unchanged data appear new. Content-defined (variable-size) chunking solves this by anchoring boundaries to the file content itself, so inserts and deletes only affect nearby chunks.

* In a multipart upload flow, S3 is the authoritative source of truth for which parts are durable. On reconnect, the server should call S3's List Parts API to get the confirmed ETag list, sync its own chunk metadata to match, and then tell the client which parts still need uploading. Any part that was in-flight at disconnect will not appear in the list and must be retried.

* A resumable upload session should have an explicit session ID that the client references on reconnect. This makes it clear which upload is being resumed when multiple large uploads are in flight at once, and keeps the state machine easy to reason about.

​