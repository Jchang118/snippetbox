# Snippetbox
#### Video Demo:  https://www.youtube.com/watch?v=l0eCucHL1Og
#### Description:

## Overview

Snippetbox is a full-stack web application built in Go that allows users to create and share short text snippets, similar to Pastebin or GitHub Gist. Users can sign up for an account, log in, and publish snippets with a title, content, and a chosen expiry period of 1, 7, or 365 days. Each snippet is assigned its own unique URL and is automatically hidden from view once it expires. Authenticated users can also view their account profile and update their password at any time.

## Project Structure

The project follows a clean, layered architecture that separates concerns across three main areas:

- **`cmd/web/`** — the application entry point and all HTTP-related logic. This includes `main.go` (server setup and dependency injection), `routes.go` (URL routing and middleware chain assembly), `handlers.go` (request handlers and form structs), `middleware.go` (reusable middleware functions), `helpers.go` (rendering and error utilities), `templates.go` (template cache construction), and `context.go` (custom context keys for passing authentication state through the request lifecycle).

- **`internal/models/`** — the data layer. `snippets.go` and `users.go` each define a model struct wrapping a `*sql.DB` connection pool, along with a corresponding interface (`SnippetModelInterface`, `UserModelInterface`). Using interfaces means the handlers never depend directly on the concrete database implementation, which makes it straightforward to substitute mock models during testing. The `mock/` subdirectory contains those in-memory fakes, and `testdata/` holds SQL scripts to set up and tear down a real test database.

- **`internal/validator/`** — a small but reusable package providing a `Validator` struct that collects both field-level errors (e.g. "title cannot be blank") and non-field errors (e.g. "email or password is incorrect"), along with standalone helper functions like `NotBlank`, `MinChars`, `MaxChars`, `MaxBytes`, `PermittedValue`, and `Matches`. Each form struct in `handlers.go` embeds this `Validator` directly, so validation logic stays close to the form definition.

- **`ui/`** — all front-end assets. HTML templates are organized into a base layout (`base.tmpl`), reusable partials (`nav.tmpl`), and individual page templates under `pages/`. Static assets (CSS, JavaScript, images) live under `static/`. The `efs.go` file uses Go's `//go:embed` directive to bundle the entire `html/` and `static/` directories directly into the compiled binary, so no separate file deployment is needed.

- **`tls/`** — a self-signed TLS certificate and private key used to serve the application over HTTPS during development.

## Key Design Decisions

**Dependency injection via a central `application` struct.** Rather than using global variables, all shared dependencies — the logger, database models, template cache, form decoder, and session manager — are stored as fields on a single `application` struct defined in `main.go`. Every handler and helper is a method on this struct, giving them access to shared state without tight coupling.

**Interface-based models for testability.** Both `SnippetModel` and `UserModel` satisfy interface types, meaning tests can inject lightweight mock implementations that operate in memory rather than hitting a real database. This pattern keeps integration tests fast and deterministic while still allowing end-to-end tests against a real MySQL instance when needed.

**Template caching.** At startup, `newTemplateCache()` parses every page template together with the base layout and partials, storing the resulting `*template.Template` values in a `map[string]*template.Template`. Subsequent requests look up the pre-compiled template by name rather than parsing files on every render, which is both faster and catches template errors at boot time rather than at runtime.

**Middleware chaining with `alice`.** Middleware is composed into three named chains using the `alice` library: a `standard` chain applied to all requests (panic recovery, request logging, common security headers), a `dynamic` chain for pages that need sessions, CSRF protection, and authentication state, and a `protected` chain that additionally enforces that the user is logged in — redirecting anonymous visitors to the login page and saving the originally requested path so they can be returned there after signing in.

**Security.** The application enforces HTTPS via TLS, sets strict HTTP security headers (Content Security Policy, X-Frame-Options, X-Content-Type-Options, Referrer-Policy), generates CSRF tokens for every form using `nosurf`, stores sessions server-side in MySQL with a 12-hour lifetime and Secure cookie flag, and hashes passwords with bcrypt at cost factor 12.

**Form decoding and validation.** HTML form submissions are decoded into typed Go structs using the `go-playground/form` library, driven by `form:"fieldname"` struct tags. Validation rules are applied by calling `CheckField` on the embedded `Validator`, which accumulates errors without short-circuiting. If any errors exist, the form is re-rendered with the user's input and inline error messages, giving a smooth user experience without losing entered data.

## What I Learned

This project was built as a learning exercise following Alex Edwards' book *Let's Go*. Through it I gained hands-on experience with Go's standard `net/http` package, structured logging with `log/slog`, HTML templating, MySQL integration via `database/sql`, password hashing, session management, CSRF mitigation, TLS configuration, the `embed` package, writing both unit tests and integration tests in Go, and organising a non-trivial Go project into coherent packages. It provided a solid foundation for understanding how production-quality Go web applications are structured and why each design decision — from interface-based models to template caching — matters in practice.
