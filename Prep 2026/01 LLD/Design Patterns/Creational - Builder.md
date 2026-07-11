[[index|← Design Patterns]]

A builder is a helper that lets you create a complex object step by step without worrying about the order or messy construction details. It's used when an object has many optional parts or configuration choices.

This shows up when designing things like HTTP requests, database queries, or configuration objects. Instead of a constructor with ten parameters where half are null, you build the object incrementally.


```python 
from typing import Optional

# NOTE: Builder is less common in Python. Python has better alternatives like
# dataclasses with default values, keyword arguments, or simple dictionaries.
# This pattern adds unnecessary complexity for most Python use cases.

class HttpRequest:
    def __init__(self):
        self.url: Optional[str] = None
        self.method: Optional[str] = None
        self.headers: dict[str, str] = {}
        self.body: Optional[str] = None

    class Builder:
        def __init__(self):
            self._request = HttpRequest()

        def url(self, url: str) -> 'HttpRequest.Builder':
            self._request.url = url
            return self

        def method(self, method: str) -> 'HttpRequest.Builder':
            self._request.method = method
            return self

        def header(self, key: str, value: str) -> 'HttpRequest.Builder':
            self._request.headers[key] = value
            return self

        def body(self, body: str) -> 'HttpRequest.Builder':
            self._request.body = body
            return self

        def build(self) -> 'HttpRequest':
            # Validate required fields
            if self._request.url is None:
                raise ValueError("URL is required")
            return self._request

# Usage
request = (HttpRequest.Builder()
    .url("https://api.example.com")
    .method("POST")
    .header("Content-Type", "application/json")
    .body('{"key": "value"}')
    .build())


```