---
title: Building a small internal gem for FCM usage
tags:
  - firebase
  - ruby
created: 2026-03-03
updated:
status: draft
---
I had an idea for building a small internal gem for my rails backend, and make it feel very close to the Firebase Admin SDK.

A few key realities first:

- **FCM HTTP v1** is the right backend API to use. ([Firebase](https://firebase.google.com/docs/cloud-messaging/send/v1-api?utm_source=chatgpt.com "Send a Message using FCM HTTP v1 API - Firebase"))
- The old Ruby `fcm` gem is **server-key / legacy style**, not what you want for HTTP v1. ([rubydoc.info](https://www.rubydoc.info/gems/fcm/1.0.1?utm_source=chatgpt.com "File: README – Documentation for fcm (1.0.1)"))
- **Multicast and “sendAll” are Admin SDK conveniences**. With raw HTTP v1 you typically "fan out" by sending multiple requests (HTTP/2 makes this efficient). Google also deprecated the old batch endpoint approach. ([Stack Overflow](https://stackoverflow.com/questions/76563436/firebase-http-v1-api-and-no-batch-send-anymore?utm_source=chatgpt.com "Firebase HTTP v1 API and no batch send anymore?"))
- For OAuth2/service accounts in Ruby, use Google’s official `googleauth` library. ([GitHub](https://github.com/googleapis/google-auth-library-ruby?utm_source=chatgpt.com "googleapis/google-auth-library-ruby"))

Below I have a gem design that covers my 3 requirements and matches my desired API surface.
## Gem goals

### 1) Access token (service account → short-lived OAuth2)

Use `googleauth` to mint and auto-refresh access tokens for the **Firebase Messaging scope** (commonly `https://www.googleapis.com/auth/firebase.messaging`). The send requests must be authorized with OAuth2 for HTTP v1. ([Firebase](https://firebase.google.com/docs/cloud-messaging/send/v1-api?utm_source=chatgpt.com "Send a Message using FCM HTTP v1 API - Firebase"))

### 2) Admin-SDK-like API

- `send(message)` → single token/topic/condition
- `send_multicast(message, tokens:)` → same payload to many device tokens (fanout)
- `send_all(messages)` → list of different messages (fanout)

This mirrors Admin SDK concepts (even though the transport differs). ([Firebase](https://firebase.google.com/docs/cloud-messaging/send/admin-sdk?utm_source=chatgpt.com "Send a Message using Firebase Admin SDK"))

##  Structure

```
my_fcm/
  lib/
    my_fcm.rb
    my_fcm/config.rb
    my_fcm/client.rb
    my_fcm/auth.rb
    my_fcm/errors.rb
    my_fcm/result.rb
```
### Let's go with minimum dependencies:

- `googleauth` (OAuth2 service account) ([GitHub](https://github.com/googleapis/google-auth-library-ruby?utm_source=chatgpt.com "googleapis/google-auth-library-ruby"))
- `faraday` (HTTP)

## Core implementation

### `lib/my_fcm.rb`

```ruby
# frozen_string_literal: true
require "my_fcm/config"
require "my_fcm/client"

module MyFcm
  class << self
    def configure
      yield(config)
    end

    def config
      @config ||= Config.new
    end

    def client
      @client ||= Client.new(config:)
    end
  end
end
```

### `lib/my_fcm/config.rb`

```ruby
# frozen_string_literal: true
module MyFcm
  class Config
    attr_accessor :project_id, :service_account_json, :service_account_io, :timeout, :open_timeout

    def initialize
      @timeout = 10
      @open_timeout = 5
    end
  end
end
```

### `lib/my_fcm/auth.rb`

```ruby
# frozen_string_literal: true
require "googleauth"

module MyFcm
  class Auth
    SCOPE = "https://www.googleapis.com/auth/firebase.messaging"

    def initialize(config:)
      @config = config
      @fetcher = build_fetcher
    end

    def access_token
      # googleauth caches & refreshes internally; returns a hash like { "access_token" => "...", "expires_in" => ... }
      token_hash = @fetcher.fetch_access_token!
      token_hash.fetch("access_token")
    end

    private

    def build_fetcher
      if @config.service_account_io
        Google::Auth::ServiceAccountCredentials.make_creds(
          json_key_io: @config.service_account_io,
          scope: [SCOPE],
        )
      elsif @config.service_account_json
        Google::Auth::ServiceAccountCredentials.make_creds(
          json_key_io: StringIO.new(@config.service_account_json),
          scope: [SCOPE],
        )
      else
        raise ArgumentError, "Provide Config#service_account_json or Config#service_account_io"
      end
    end
  end
end
```

### `lib/my_fcm/client.rb`

```ruby
# frozen_string_literal: true
require "faraday"
require "json"
require "my_fcm/auth"
require "my_fcm/result"

module MyFcm
  class Client
    FCM_HOST = "https://fcm.googleapis.com"

    def initialize(config:)
      @config = config
      @auth = Auth.new(config:)
      @http = Faraday.new(url: FCM_HOST) do |f|
        f.request :json
        f.response :json, content_type: /\bjson$/
      end
    end

    # --- Admin-like API ---

    # message_hash must be shaped like { message: { token/topic/condition..., android/apns..., data..., notification... } }
    def send(message_hash)
      post_send(message_hash)
    end

    # Same message to many device tokens (fanout)
    def send_multicast(message_hash, tokens:, max_concurrency: 20)
      # HTTP v1 doesn't accept an array of tokens in one call; you fan out (HTTP/2 multiplexing helps). :contentReference[oaicite:7]{index=7}
      base = deep_dup(message_hash)
      results = parallel_map(tokens, max_concurrency:) do |token|
        msg = deep_dup(base)
        msg[:message] ||= {}
        msg[:message][:token] = token
        post_send(msg)
      end
      Result.multicast(tokens:, results:)
    end

    # List of different messages (fanout)
    def send_all(messages, max_concurrency: 20)
      results = parallel_map(messages, max_concurrency:) { |m| post_send(m) }
      Result.batch(results:)
    end

    private

    def post_send(payload)
      project_id = @config.project_id or raise ArgumentError, "Config#project_id required"

      res = @http.post("/v1/projects/#{project_id}/messages:send") do |req|
        req.headers["Authorization"] = "Bearer #{@auth.access_token}"
        req.options.timeout = @config.timeout
        req.options.open_timeout = @config.open_timeout
        req.body = payload
      end

      Result.single(http_status: res.status, body: res.body)
    rescue Faraday::Error => e
      Result.single(http_status: nil, body: { "error" => { "message" => e.message } })
    end

    def parallel_map(items, max_concurrency:)
      # Keep this gem dependency-light: simple thread pool
      q = Queue.new
      items.each_with_index { |item, idx| q << [idx, item] }

      out = Array.new(items.size)
      workers = [max_concurrency, items.size].min.times.map do
        Thread.new do
          while (pair = q.pop(true) rescue nil)
            idx, item = pair
            out[idx] = yield(item)
          end
        end
      end
      workers.each(&:join)
      out
    end

    def deep_dup(obj)
      Marshal.load(Marshal.dump(obj))
    end
  end
end
```

### `lib/my_fcm/result.rb`

```ruby
# frozen_string_literal: true
module MyFcm
  class Result
    attr_reader :http_status, :body

    def initialize(http_status:, body:)
      @http_status = http_status
      @body = body
    end

    def ok?
      http_status && http_status >= 200 && http_status < 300 && !error?
    end

    def error?
      body.is_a?(Hash) && body.key?("error")
    end

    def error_code
      body.dig("error", "status")
    end

    def error_message
      body.dig("error", "message")
    end

    # factories
    def self.single(http_status:, body:) = new(http_status:, body:)

    def self.multicast(tokens:, results:)
      ok = []
      bad = []
      results.each_with_index do |r, idx|
        (r.ok? ? ok : bad) << { token: tokens[idx], result: r }
      end
      { success: ok, failure: bad, results: results }
    end

    def self.batch(results:)
      { success: results.count(&:ok?), failure: results.count { |r| !r.ok? }, results: results }
    end
  end
end
```

## Usage in Rails

```ruby
# config/initializers/my_fcm.rb
MyFcm.configure do |c|
  c.project_id = ENV.fetch("FIREBASE_PROJECT_ID")
  c.service_account_json = ENV.fetch("FIREBASE_SERVICE_ACCOUNT_JSON") # store JSON as env var
end
```

Send to one device:

```ruby
payload = {
  message: {
    token: device_token,
    android: { priority: "HIGH" },
    data: { type: "incoming_order", order_id: "12345" }
  }
}

MyFcm.client.send(payload)
```

Send to many devices:

```ruby
MyFcm.client.send_multicast(
  { message: { android: { priority: "HIGH" }, data: { type: "incoming_order", order_id: "12345" } } },
  tokens: device_tokens,
  max_concurrency: 50
)
```

Send a list of different messages:

```ruby
messages = device_tokens.map do |t|
  { message: { token: t, data: { type: "ping", ts: Time.now.to_i.to_s } } }
end

MyFcm.client.send_all(messages, max_concurrency: 50)
```

## Notes:

1. **No true multicast endpoint in HTTP v1**  
    Admin SDK exposes multicast/sendAll ergonomics, but underneath it’s effectively optimized fanout. You’re implementing the same thing by fanning out requests (and optionally using HTTP/2 / concurrency). ([Stack Overflow](https://stackoverflow.com/questions/76563436/firebase-http-v1-api-and-no-batch-send-anymore?utm_source=chatgpt.com "Firebase HTTP v1 API and no batch send anymore?"))
2. **Handle invalid tokens**  
    When FCM returns errors like “NOT_FOUND / UNREGISTERED / NOT_REGISTERED” (wording varies), remove that token from your DB.
3. **Prefer topics for large fanout**  
    For very large audiences, topic messaging is what FCM is designed for. ([Firebase](https://firebase.google.com/docs/cloud-messaging/topic-messaging?utm_source=chatgpt.com "Topic Messaging - Firebase"))

## More similar to Admin SDK?

## What's next?

- [ ] `async` / `concurrent-ruby` for concurrency
- [ ] `oj` for JSON
- [ ] Automatic retries with backoff on transient errors
- [ ] Rate limiting
- [ ] Token caching via memory/Redis (optional)
- [ ] Structured error types
- [ ] More close to Admin SDK:

```ruby
MyFcm.send_to_token(token, data:, notification:, android:)
MyFcm.send_to_tokens(tokens, data:, notification:, android:) # multicast
MyFcm.send_messages([ ... ]) # send_all
```

