---
name: falcon-rails-websockets
description: Implement or review WebSocket endpoints in a Rails application running on Falcon. Use for bidirectional persistent connections, including raw WebSockets or existing Action Cable integrations; not for one-way SSE or finite HTTP streaming.
---

# Falcon Rails WebSockets

Determine whether the application already uses Action Cable or raw WebSockets. Extend the established abstraction unless the requested change requires a migration.

## Controller Pattern

For a raw WebSocket endpoint, use the Rails adapter to assign the upgraded response:

```ruby
require "async/websocket/adapters/rails"

class ChatController < ApplicationController
	skip_before_action :verify_authenticity_token, only: :connect
	
	def connect
		return head(:unauthorized) unless current_user
		
		self.response = Async::WebSocket::Adapters::Rails.open(request) do |connection|
			Sync do
				while message = connection.read
					payload = JSON.parse(message.buffer)
					connection.send_text(JSON.generate(handle_message(payload)))
					connection.flush
				end
			rescue Protocol::WebSocket::ClosedError
				# The client disconnected.
			end
		end
	end
end
```

Keep authentication and authorization before accepting the connection. Adapt message handling to the application's protocol, including validation and error responses.

## Connection Requirements

- Define the handshake contract: route, authentication, authorization, origin policy, and any subprotocol. Scope CSRF exemptions narrowly.
- Define a versionable message contract with explicit handling for malformed, unknown, and oversized messages.
- Give one connection handler clear ownership of the socket and its child tasks. Coordinate concurrent producers and serialize writes.
- Bound outbound buffering and define what happens when a client is slow. Do not hold Active Record connections or transactions while waiting for messages.
- On close, error, cancellation, or server shutdown, stop child tasks and release subscriptions.

## Additional Context

Install the Falcon Rails context:

```bash
bundle exec bake agent:context:install --gem falcon-rails
```

Read `.context/falcon-rails/websockets.md` for additional client code, routing, adapter details, and a complete example.

## Verification

Test with a real WebSocket client through Falcon. Cover successful upgrade, authentication and origin rejection, valid and invalid messages, slow consumers, abrupt disconnects, multiple concurrent clients, and graceful server shutdown. Confirm that reconnect behavior does not leak tasks or duplicate application effects.
