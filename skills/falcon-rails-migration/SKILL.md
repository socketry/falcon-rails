---
name: falcon-rails-migration
description: Migrate an existing Rails application from Puma or another Rack server to Falcon while preserving development and production behavior. Use for migration planning, implementation, or review; not for new Rails applications or feature-specific streaming work.
---

# Falcon Rails Migration

Treat the migration as a server and concurrency-model change, not just a Gemfile edit. Preserve the existing deployment contract until Falcon has been verified.

## Migration Approach

1. Establish a working baseline with the current server and tests.
2. Inventory the current server contract: commands, bind address, port, TLS termination, worker count, preload behavior, timeouts, health checks, graceful shutdown, and deployment manifests.
3. Identify features sensitive to the server runtime, including streaming responses, WebSockets or Action Cable, request-local state, background jobs, and long-running requests.
4. Add `falcon-rails` and boot Falcon locally while retaining the existing server as a fallback.
5. Exercise ordinary requests and every server-sensitive feature before changing production configuration.
6. Audit code for assumptions that break under fiber concurrency: thread-local request state, shared mutable objects, unbounded task creation, blocking native or CPU-heavy work, and resources held across waits.
7. Translate the production entrypoint to `falcon host` and `falcon.rb`, preserving the deployment contract rather than copying a generic configuration. Do not use `falcon serve` for production.
8. Remove the previous server and its configuration only after the Falcon path is covered by tests and deployment checks.

## Additional Context

Install the relevant context:

```bash
bundle exec bake agent:context:install --gem falcon-rails
bundle exec bake agent:context:install --gem falcon
bundle exec bake agent:context:install --gem async
```

Read `.context/falcon-rails/getting-started.md`, `.context/falcon/rails-integration.md`, and `.context/falcon/deployment.md`. Read the Async best-practices and thread-safety context when reviewing application compatibility.

## Verification

Run the full application suite, then test the application through Falcon. Verify readiness and liveness checks, proxy behavior, database pool use, graceful shutdown, and representative concurrent load. If the application uses streaming or WebSockets, use the corresponding Falcon Rails skill for protocol-specific checks.
