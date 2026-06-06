# Snippetbox
#### Video Demo:  https://www.youtube.com/watch?v=l0eCucHL1Og
#### Description:
Snippetbox is a full-stack web application built in Go that allows users to share short text snippets, similar to Pastebin. Users can sign up for an account, log in, and create snippets with a title, content, and an expiry period of 1, 7, or 365 days. Each snippet gets its own URL and is automatically hidden once it expires. Authenticated users can also view and update their account details, including changing their password.

The application is served over HTTPS using a TLS certificate and uses MySQL for persistent storage. It features session management (via the `scs` library), CSRF protection (via `nosurf`), and bcrypt password hashing for security. HTML templates are compiled at startup into a cache and embedded directly into the binary using Go's `embed` package, meaning no external files are needed at runtime.

The codebase follows a clean layered architecture: the `cmd/web` package handles all HTTP routing, middleware, and request handling; the `internal/models` package encapsulates all database logic behind interfaces, making it easy to swap in mock implementations for testing; and the `internal/validator` package provides reusable form validation helpers. Middleware is composed using the `alice` library into named chains — a standard chain for all requests, a dynamic chain for session and CSRF handling, and a protected chain that requires authentication.

This project was built as a learning exercise following Alex Edwards' book *Let's Go*, progressively introducing Go web development patterns including middleware chaining, template caching, interface-based dependency injection, and integration testing.
