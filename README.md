# Ruby GigaChat

[![Gem Version](https://img.shields.io/gem/v/gigachat.svg)](https://rubygems.org/gems/gigachat)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/neonix20b/gigachat/blob/main/LICENSE.txt)

## Installation

### Bundler

Add this line to your application's Gemfile:

```ruby
gem "gigachat"
```

And then execute:

```bash
$ bundle install
```

### Gem install

Or install with:

```bash
$ gem install gigachat
```

and require with:

```ruby
require "gigachat"
```

## Usage

Get your API key from [developers.sber.ru](https://developers.sber.ru/).

### Quickstart

For a quick test you can pass your token directly to a new client:

```ruby
client = GigaChat::Client.new(
  api_type: "GIGACHAT_API_CORP", # or GIGACHAT_API_PERS, GIGACHAT_API_B2B
  client_base64: "Yjgy...VhYw==" # your authorization data
)
```

### With Config

For a more robust setup, you can configure the gem with your API keys, for example in an `gigachat.rb` initializer file. Never hardcode secrets into your codebase - instead use something like [dotenv](https://github.com/motdotla/dotenv) to pass the keys safely into your environments.

```ruby
GigaChat.configure do |config|
  config.api_type = "GIGACHAT_API_CORP" # or GIGACHAT_API_PERS, GIGACHAT_API_B2B
  config.client_base64 = ENV.fetch("GIGACHAT_CLIENT_KEY")
end
```

Then you can create a client like this:

```ruby
client = GigaChat::Client.new
```

You can still override the config defaults when making new clients; any options not included will fall back to any global config set with GigaChat.configure. e.g. in this example the api_type, etc. will fallback to any set globally using GigaChat.configure, with only the client_base64 overridden:

```ruby
client = GigaChat::Client.new(client_base64: "secret_token_goes_here")
```

#### Custom timeout or base URI

The default timeout for any request using this library is 120 seconds. You can change that by passing a number of seconds to the `request_timeout` when initializing the client. You can also change the base URI used for all requests, eg. to use observability tools like [Helicone](https://docs.helicone.ai/quickstart/integrate-in-one-line-of-code), and add arbitrary other headers:

```ruby
client = GigaChat::Client.new(
    client_base64: "secret_token_goes_here",
    uri_base: "https://oai.hconeai.com/",
    uri_auth: "https://localhost:5362/",
    request_timeout: 240,
    extra_headers: {
      "X-Proxy-TTL" => "43200",
      "X-Proxy-Refresh": "true",
    }
)
```

or when configuring the gem:

```ruby
GigaChat.configure do |config|
    config.client_base64 = ENV.fetch("GIGACHAT_CLIENT_KEY")
    config.log_errors = true # Optional
    config.uri_base = "https://oai.hconeai.com/" # Optional
    config.request_timeout = 240 # Optional
    config.extra_headers = {
      "X-Proxy-TTL" => "43200",
      "X-Proxy-Refresh": "true"
    } # Optional
end
```

#### Extra Headers per Client

You can dynamically pass headers per client object, which will be merged with any headers set globally with GigaChat.configure:

```ruby
client = GigaChat::Client.new(client_base64: "secret_token_goes_here")
client.add_headers("X-Proxy-TTL" => "43200")
```

#### Logging

##### Errors

By default, `gigachat` does not log any `Faraday::Error`s encountered while executing a network request to avoid leaking data (e.g. 400s, 500s, SSL errors and more - see [here](https://www.rubydoc.info/github/lostisland/faraday/Faraday/Error) for a complete list of subclasses of `Faraday::Error` and what can cause them).

If you would like to enable this functionality, you can set `log_errors` to `true` when configuring the client:

```ruby
  client = GigaChat::Client.new(log_errors: true)
```

##### Faraday middleware

You can pass [Faraday middleware](https://lostisland.github.io/faraday/#/middleware/index) to the client in a block, eg. to enable verbose logging with Ruby's [Logger](https://ruby-doc.org/3.2.2/stdlibs/logger/Logger.html):

```ruby
  client = GigaChat::Client.new do |f|
    f.response :logger, Logger.new($stdout), bodies: true
  end
```

### Counting Tokens

GigaChat parses prompt text into [tokens](https://help.openai.com/en/articles/4936856-what-are-tokens-and-how-to-count-them), which are words or portions of words. Counting tokens can help you estimate your [costs](https://developers.sber.ru/docs/ru/gigachat/api/tariffs). It can also help you ensure your prompt text size is within the max-token limits of your model's context window, and choose an appropriate [`max_tokens`](https://developers.sber.ru/docs/ru/gigachat/api/reference/rest/post-chat) completion parameter so your response will fit as well.

To estimate the token-count of your text:

```ruby
GigaChat.rough_token_count("Your text")
```

If you need a more accurate count, try [tiktoken_ruby](https://github.com/IAPark/tiktoken_ruby).

### Models

There are different models that can be used to generate text. For a full list and to retrieve information about a single model:

```ruby
client.models.list
client.models.retrieve(id: "GigaChat-Pro")
```

### Chat

GigaChat is a model that can be used to generate text in a conversational style. You can use it to [generate a response](https://developers.sber.ru/docs/ru/gigachat/api/reference/rest/post-chat) to a sequence of messages:

```ruby
response = client.chat(
    parameters: {
        model: "GigaChat-Pro", # Required.
        messages: [{ role: "user", content: "Hello!"}], # Required.
        temperature: 0.7,
    })
puts response.dig("choices", 0, "message", "content")
# => "Hello! How may I assist you today?"
```

#### Streaming Chat

You can stream from the API in realtime, which can be much faster and used to create a more engaging user experience. Pass a [Proc](https://ruby-doc.org/core-2.6/Proc.html) (or any object with a `#call` method) to the `stream` parameter to receive the stream of completion chunks as they are generated. Each time one or more chunks is received, the proc will be called once with each chunk, parsed as a Hash. If GigaChat returns an error, `gigachat` will raise a Faraday error.

```ruby
client.chat(
    parameters: {
        model: "GigaChat-Pro", # Required.
        messages: [{ role: "user", content: "Describe a character called Anna!"}], # Required.
        temperature: 0.7,
        update_interval: 1,
        stream: proc do |chunk, _bytesize|
            print chunk.dig("choices", 0, "delta", "content")
        end
    })
# => "Anna is a young woman in her mid-twenties, with wavy chestnut hair that falls to her shoulders..."
```

### Functions

You can describe and pass in functions and the model will intelligently choose to output a JSON object containing arguments to call them - eg., to use your method `get_current_weather` to get the weather in a given location.

```ruby

def get_current_weather(location:, unit: "fahrenheit")
  # Here you could use a weather api to fetch the weather.
  "The weather in #{location} is nice 🌞 #{unit}"
end

messages = [
  {
    "role": "user",
    "content": "What is the weather like in San Francisco?",
  },
]

response =
  client.chat(
    parameters: {
      model: "GigaChat-Pro",
      messages: messages,  # Defined above because we'll use it again
      functions: [
        {
          type: "function",
          function: {
            name: "get_current_weather",
            description: "Get the current weather in a given location",
            parameters: {  # Format: https://json-schema.org/understanding-json-schema
              type: :object,
              properties: {
                location: {
                  type: :string,
                  description: "The city and state, e.g. San Francisco, CA",
                },
                unit: {
                  type: "string",
                  enum: %w[celsius fahrenheit],
                },
              },
              required: ["location"],
            },
          },
        }
      ],
      function_call: "none"  # Optional, defaults to "auto"
                               # Can also put "none" or specific functions, see docs
    },
  )
```

You can learn more at [this link](https://developers.sber.ru/docs/ru/gigachat/api/function-calling).

### Embeddings

You can use the embeddings endpoint to get a vector of numbers representing an input. You can then compare these vectors for different inputs to efficiently check how similar the inputs are.

```ruby
response = client.embeddings(
    parameters: {
        model: "GigaChat",
        input: "The food was delicious and the waiter..."
    }
)

puts response.dig("data", 0, "embedding")
# => Vector representation of your embedding
```

### Image Generation

GigaChat API will return an image if the `#chat` method receives an appropriate message, for example, "Draw a pink cat". Images are generated in binary form in JPG format using the built-in text2image function.

In response, GigaChat returns the identifier of the created image, which can be downloaded using the `POST /files/:file_id/content` request. 

To create an image, the request must include the "function_call": "auto" parameter, which allows the model to determine the need to call the text2image function. When creating images using the built-in function, the model returns a response with the result `"finish_reason": "stop"`.

You can learn more at [this link](https://developers.sber.ru/docs/ru/gigachat/api/images-generation).

### Errors

HTTP errors can be caught like this:

```
  begin
    GigaChat::Client.new.models.retrieve(id: "GigaChat")
  rescue Faraday::Error => e
    raise "Got a Faraday error: #{e}"
  end
```

## Contributing

Bug reports and pull requests are welcome on GitHub at <https://github.com/neonix20b/gigachat>. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [code of conduct](https://github.com/neonix20b/gigachat/blob/main/CODE_OF_CONDUCT.md).

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in the Ruby OpenAI project's codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/neonix20b/gigachat/blob/main/CODE_OF_CONDUCT.md).
