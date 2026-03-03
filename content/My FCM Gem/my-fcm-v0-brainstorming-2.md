---
title: Building a small internal gem for FCM usage - Part 2
tags:
  - firebase
  - ruby
created: 2026-03-03
updated:
status: draft
---
Below is an upgraded baseline that adds:

- **Oj** for JSON encode/decode (faster + less GC)
- **concurrent-ruby** thread pool for fanout (`send_multicast`, `send_all`)
- **token caching** (so you don’t mint a new OAuth token per request)
- optional **retry with exponential backoff** for transient failures

## 1) Gemspec

```ruby
# my_fcm.gemspec
spec.add_dependency "faraday", ">= 2.0"
spec.add_dependency "googleauth", ">= 1.9"
spec.add_dependency "concurrent-ruby", ">= 1.2"
spec.add_dependency "oj", ">= 3.16"
```

## 2) Config

```ruby
# lib/my_fcm/config.rb
module MyFcm
  class Config
    attr_accessor :project_id, :service_account_json, :service_account_io
    attr_accessor :timeout, :open_timeout

    # concurrency
    attr_accessor :max_threads, :pool_queue, :shutdown_timeout

    # retry
    attr_accessor :retries, :retry_base_delay, :retry_max_delay

    def initialize
      @timeout = 10
      @open_timeout = 5

      @max_threads = 40
      @pool_queue = 1000
      @shutdown_timeout = 5

      @retries = 2
      @retry_base_delay = 0.2
      @retry_max_delay = 2.0
    end
  end
end
```

## 3) OAuth token provider with caching

Googleauth will refresh, but **you still don’t want to call `fetch_access_token!` every send**. Cache it until it’s near expiry.

```ruby
# lib/my_fcm/auth.rb
require "googleauth"
require "concurrent"

module MyFcm
  class Auth
    SCOPE = "https://www.googleapis.com/auth/firebase.messaging"
    SKEW_SECONDS = 60 # refresh 1 min early

    def initialize(config:)
      @config = config
      @creds = build_creds
      @mutex = Mutex.new
      @cached_token = nil
      @expires_at = Time.at(0)
    end

    def access_token
      now = Time.now
      return @cached_token if @cached_token && now < (@expires_at - SKEW_SECONDS)

      @mutex.synchronize do
        now = Time.now
        return @cached_token if @cached_token && now < (@expires_at - SKEW_SECONDS)

        token_hash = @creds.fetch_access_token!
        @cached_token = token_hash.fetch("access_token")
        expires_in = token_hash["expires_in"] || 3600
        @expires_at = now + expires_in.to_i
        @cached_token
      end
    end

    private

    def build_creds
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

## 4) Client using Oj + concurrent pool

Key points:

- encode request body with `Oj.dump`
- parse response with `Oj.load`
- `send_multicast` and `send_all` use a thread pool

```ruby
# lib/my_fcm/client.rb
require "faraday"
require "oj"
require "concurrent"
require "my_fcm/auth"
require "my_fcm/result"

module MyFcm
  class Client
    FCM_HOST = "https://fcm.googleapis.com"

    TRANSIENT_HTTP = [408, 425, 429, 500, 502, 503, 504].freeze

    def initialize(config:)
      @config = config
      @auth = Auth.new(config:)
      @pool = Concurrent::ThreadPoolExecutor.new(
        min_threads: 1,
        max_threads: @config.max_threads,
        max_queue: @config.pool_queue,
        fallback_policy: :caller_runs,
      )

      @http = Faraday.new(url: FCM_HOST) do |f|
        f.adapter Faraday.default_adapter
      end
    end

    def shutdown!
      @pool.shutdown
      @pool.wait_for_termination(@config.shutdown_timeout)
    end

    # Single message: payload = { message: { token/topic/condition..., android/apns..., data..., notification... } }
    def send(payload)
      post_send(payload)
    end

    # Same message to many tokens (fanout)
    def send_multicast(payload, tokens:)
      base = deep_dup(payload)

      futures = tokens.map do |token|
        Concurrent::Promises.future_on(@pool, token) do |t|
          msg = deep_dup(base)
          msg[:message] ||= {}
          msg[:message][:token] = t
          post_send(msg)
        end
      end

      results = Concurrent::Promises.zip(*futures).value!
      Result.multicast(tokens:, results:)
    end

    # Many different messages (fanout)
    def send_all(messages)
      futures = messages.map do |msg|
        Concurrent::Promises.future_on(@pool) { post_send(msg) }
      end
      results = Concurrent::Promises.zip(*futures).value!
      Result.batch(results:)
    end

    private

    def post_send(payload)
      project_id = @config.project_id or raise ArgumentError, "Config#project_id required"
      path = "/v1/projects/#{project_id}/messages:send"

      with_retries do
        res = @http.post(path) do |req|
          req.headers["Authorization"] = "Bearer #{@auth.access_token}"
          req.headers["Content-Type"] = "application/json"
          req.headers["Accept"] = "application/json"
          req.options.timeout = @config.timeout
          req.options.open_timeout = @config.open_timeout
          req.body = Oj.dump(payload, mode: :compat)
        end

        body = parse_json(res.body)
        Result.single(http_status: res.status, body:)
      end
    rescue Faraday::Error => e
      Result.single(http_status: nil, body: { "error" => { "message" => e.message } })
    end

    def with_retries
      attempt = 0
      begin
        result = yield
        return result unless transient?(result)

        raise "transient_http_#{result.http_status}"
      rescue => e
        attempt += 1
        raise e if attempt > @config.retries

        sleep retry_delay(attempt)
        retry
      end
    end

    def transient?(result)
      return false unless result.respond_to?(:http_status)
      TRANSIENT_HTTP.include?(result.http_status)
    end

    def retry_delay(attempt)
      base = @config.retry_base_delay.to_f
      max  = @config.retry_max_delay.to_f
      # exponential backoff + jitter
      delay = [base * (2 ** (attempt - 1)), max].min
      delay * (0.5 + rand) # jitter: 0.5x - 1.5x
    end

    def parse_json(body)
      return {} if body.nil? || body == ""
      body.is_a?(Hash) ? body : Oj.load(body)
    rescue Oj::ParseError
      { "raw" => body.to_s }
    end

    def deep_dup(obj)
      Marshal.load(Marshal.dump(obj))
    end
  end
end
```

## 5) Result helpers (same as before, but keep)

```ruby
# lib/my_fcm/result.rb
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

## 6) Nice “Admin-ish” convenience API (optional but recommended)

```ruby
# lib/my_fcm/messaging.rb
module MyFcm
  module Messaging
    module_function

    def send_to_token(token, data:, notification: nil, android: nil, apns: nil)
      payload = {
        message: {
          token: token,
          data: stringify(data),
        }
      }
      payload[:message][:notification] = notification if notification
      payload[:message][:android] = android if android
      payload[:message][:apns] = apns if apns
      MyFcm.client.send(payload)
    end

    def send_to_tokens(tokens, data:, notification: nil, android: nil, apns: nil)
      payload = {
        message: {
          data: stringify(data),
        }
      }
      payload[:message][:notification] = notification if notification
      payload[:message][:android] = android if android
      payload[:message][:apns] = apns if apns
      MyFcm.client.send_multicast(payload, tokens: tokens)
    end

    def send_messages(messages)
      MyFcm.client.send_all(messages)
    end

    def stringify(hash)
      # FCM data values must be strings
      hash.transform_values { |v| v.nil? ? "" : v.to_s }
    end
  end
end
```

## Example usage in Rails

```ruby
MyFcm.configure do |c|
  c.project_id = ENV.fetch("FIREBASE_PROJECT_ID")
  c.service_account_json = ENV.fetch("FIREBASE_SERVICE_ACCOUNT_JSON")
  c.max_threads = 60
  c.retries = 2
end

MyFcm::Messaging.send_to_token(
  device_token,
  data: { type: "incoming_order", order_id: 12345 },
  android: { priority: "HIGH" }
)

MyFcm::Messaging.send_to_tokens(
  tokens,
  data: { type: "incoming_order", order_id: 12345 },
  android: { priority: "HIGH" }
)
```

## Two important production notes

1. **FCM `data` values must be strings** : The helper above `stringify` enforces that (prevents subtle delivery/SDK issues).
2. **We still need "fanout" for multicast** : HTTP v1 doesn’t accept an array of tokens in one request, so concurrency is the right approach.

Next:

- A `MyFcm::Errors` map to classify “invalid token / unregistered” responses and return a list of tokens to delete
- Rails `ActiveSupport::Notifications` hooks for observability
- Optional Redis token cache (only needed if we run many processes and want fewer token mints)