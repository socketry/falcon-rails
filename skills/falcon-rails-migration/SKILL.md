---
name: falcon-rails-migration
description: Migrate an existing Rails application from Puma or another Rack server to Falcon while preserving development and production behavior. Use for migration planning, implementation, or review; not for new Rails applications or feature-specific streaming work.
---

# Falcon Rails Migration

Treat the migration as a server and concurrency-model change, not just a Gemfile edit. Preserve the existing deployment contract until Falcon has been verified.

## Migration Approach

1. Update the application's dependencies in this order:

	```bash
	bundle add falcon
	bundle remove puma
	bundle add falcon-rails
	bundle add agent-context
	bundle add agent-skills
	```

	Install the context documentation and skills provided by those gems:

	```bash
	bundle exec bake agent:context:install
	bundle exec bake agent:skills:install
	```

	Add `/.agents` to `.gitignore`; it contains generated skills which can be reinstalled from the bundled gems.

2. Run the test suite, then inventory the previous server contract: commands, bind address, port, TLS termination, worker count, preload behavior, timeouts, health checks, graceful shutdown, and deployment manifests.
3. Boot Falcon locally and exercise ordinary requests before changing production configuration.
4. Identify and test features sensitive to the server runtime, including streaming responses, WebSockets or Action Cable, request-local state, background jobs, and long-running requests.
5. Audit code for assumptions that break under fiber concurrency: thread-local request state, shared mutable objects, unbounded task creation, blocking native or CPU-heavy work, and resources held across waits.
6. Translate the production entrypoint to `falcon host` and `falcon.rb`, preserving the deployment contract rather than copying a generic configuration. Do not use `falcon serve` for production.
7. Remove obsolete Puma commands and configuration only after the Falcon path is covered by tests and deployment checks.

## Additional Context

From the context installed in step one, read `.context/falcon-rails/getting-started.md`, `.context/falcon/rails-integration.md`, and `.context/falcon/deployment.md`. Read the Async best-practices and thread-safety context when reviewing application compatibility.

## Verification

Run the full application suite, then test the application through Falcon. Verify readiness and liveness checks, proxy behavior, database pool use, graceful shutdown, and representative concurrent load. If the application uses streaming or WebSockets, use the corresponding Falcon Rails skill for protocol-specific checks.
