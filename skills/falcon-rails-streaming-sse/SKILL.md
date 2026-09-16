---
name: falcon-rails-streaming-sse
description: Implement or review HTTP streaming and Server-Sent Events in a Rails application running on Falcon. Use for progressive responses and server-to-client event streams; not for bidirectional WebSocket communication.
---

# Falcon Rails Streaming and SSE

Choose ordinary HTTP streaming for finite progressive output or custom framing. Choose SSE for a long-lived, server-to-client event channel with browser-managed reconnection.

## Controller Patterns

For a finite streaming response, assign a `Rack::Response` with a callable body:

```ruby
class ExportController < ApplicationController
	def show
		body = proc do |stream|
			each_record do |record|
				stream.write("#{JSON.generate(record)}\n")
			end
		end
		
		self.response = Rack::Response[200, {"content-type" => "application/x-ndjson"}, body]
	end
end
```

For SSE, use the same response shape with event-stream headers and SSE framing:

```ruby
class EventsController < ApplicationController
	def index
		body = proc do |stream|
			event_source.each do |event|
				stream.write("data: #{JSON.generate(event)}\n\n")
			end
		end
		
		self.response = Rack::Response[200, {
			"content-type" => "text/event-stream",
			"cache-control" => "no-cache",
		}, body]
	end
end
```

Adapt `each_record` and `event_source` to the application's producer. The callable body owns that producer's lifetime.

## Lifecycle Requirements

- Keep authentication and authorization in the normal Rails request path before starting the response body.
- On completion, failure, or client disconnect, stop producers and release subscriptions promptly.
- Preserve backpressure. Do not place an unbounded queue between producers and a slow client, and do not accumulate the complete response in memory.
- Avoid holding an Active Record transaction or checked-out connection while waiting for future events.
- For SSE, decide whether reconnecting clients need event IDs, replay, retry timing, or heartbeats.

## Additional Context

Install the Falcon Rails context:

```bash
bundle exec bake agent:context:install --gem falcon-rails
```

Read `.context/falcon-rails/http-streaming.md` and `.context/falcon-rails/server-sent-events.md` for additional examples, client code, routing, and framing details.

## Verification

Test through Falcon with a client that consumes incrementally. Verify headers and framing, first-byte latency, normal completion, client cancellation, producer failure, and cleanup. For SSE, also test reconnection and any replay or duplicate-event behavior. Check the production proxy for response buffering and idle timeouts.
