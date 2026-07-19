

API Types:
	Rest 
	[[GraphQL Foundations|Graphql]]
	GRPC


Pagination:
	Page based : / events?page=1&limit=25
	Cursor based:  /events ? cursor={lastEventId}&limit=25
		This is useful when writes are too many and  order is changed while user is browsing


Versioning startegies :
	Header versioning 
	URL versioning


Security:
	 Authentication and authorization
	 Rate limiting and throttling
		 - **Per-user limits**: 1000 requests per hour per authenticated user
		- **Per-IP limits**: 100 requests per hour for unauthenticated requests
		- **Endpoint-specific limits**: 10 booking attempts per minute to prevent ticket scalping
 