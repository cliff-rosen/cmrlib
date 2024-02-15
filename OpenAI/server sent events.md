# Server Sent Events

process openai using eventstream
update flask code per best practices for sse
process flask using eventstream
process flask using react

## Overview
client sends request to server:
```
GET /events HTTP/1.1
Host: example.com
Accept: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Authorization: Bearer MY_TOKEN
```

server replies with ok
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
```

server then replies with any number of messages separated by \n\n:
```
data: {"sender": "Server", "message": "Welcome to the SSE connection!"}

data: {"value": 1}

```

server closes connection when done without requirement for closing message
## OpenAI Stream format
```
data: {"id":"chatcmpl-8sYMaINPl8SKF8YXrJcTuE5zhIaGW","object":"chat.completion.chunk","created":1708012496,"model":"gpt-3.5-turbo-0613","system_fingerprint":null,"choices":[{"index":0,"delta":{"content":" revealed"},"logprobs":null,"finish_reason":null}]}

data: {"id":"chatcmpl-8sYMaINPl8SKF8YXrJcTuE5zhIaGW","object":"chat.completion.chunk","created":1708012496,"model":"gpt-3.5-turbo-0613","system_fingerprint":null,"choices":[{"index":0,"delta":{"content":" the"},"logprobs":null,"finish_reason":null}]}

data: {"id":"chatcmpl-8sYMaINPl8SKF8YXrJcTuE5zhIaGW","object":"chat.completion.chunk","created":1708012496,"model":"gpt-3.5-turbo-0613","system_fingerprint":null,"choices":[{"index":0,"delta":{"content":" existence"},"logprobs":null,"finish_reason":null}]}

data: {"id":"chatcmpl-8sYMaINPl8SKF8YXrJcTuE5zhIaGW","object":"chat.completion.chunk","created":1708012496,"model":"gpt-3.5-turbo-0613","system_finger

data: {"id":"chatcmpl-8sYMaINPl8SKF8YXrJcTuE5zhIaGW","object":"chat.completion.chunk","created":1708012496,"model":"gpt-3.5-turbo-0613","system_fingerprint":null,"choices":[{"index":0,"delta":{},"logprobs":null,"finish_reason":"length"}]}

data: [DONE]
```
A given data message can be parsed as follows:
```
// finish reason => data.choices[0].finish_reason
// delta content => data.choices[0].delta.content

data = '{"id":"chatcmpl-8sYMaINPl8SKF8YXrJcTuE5zhIaGW","object":"chat.completion.chunk"...}'
obj = JSON.parse(data)
response = obj.choices[0]
if(!response.finish_reason)
    delta_content = response.delta.content
```

## Flask
API endpoint retrieves generator from OpenAI client and generates stream of events capturing completion.
```
def generate(messages, temperature, stream=False):
    response = ""

    try:
        completion = client.chat.completions.create(
            model=COMPLETION_MODEL,
            messages=messages,
            max_tokens=MAX_TOKENS,
            temperature=temperature,
            stream=stream,
        )
        for chunk in completion:
            message = chunk.choices[0].delta.content
            print(message, end="", flush=True)
            yield message

    except Exception as e:
        print("query_model error: ", str(e))
        logger.warning("get_answer.query_model error:" + str(e))
        response = "We're sorry, the server was too busy to handle this response.  Please try again."

    finally:
        print("finally")

def get_hello():
    return model.generate(messages, 0.0, True)

class Hello(Resource):
    def get(self):
        return Response(hello.get_hello(), mimetype="text/event-stream")
```

## JavaScript
Browser requests stream from server and then listens for and processes events
### With EventStream
The EventStream library simpifies SSE processing, but I don't think it works with OpenAI because OpenAI requires posting data to open the connection
```
const  API_URL = 'http://127.0.0.1/hello'
const eventSource = new EventSource(API_URL); // Replace with your server endpoint

eventSource.onmessage = (event) => {
  // Process the received event
  const data = JSON.parse(event.data); // Assuming JSON data
  console.log(`Received event: ${data.event}`);
  console.log(`Data: ${data.data}`);

  // Update your UI or application based on the data
  // ...
};

eventSource.onerror = (error) => {
  console.error("Error:", error);
  // Handle connection errors or other issues
};

// Optionally, add event listeners for specific event types:
eventSource.addEventListener("data", (event) => {
  // Handle "new-message" events specifically
});
```
### Without EventStream
```
const  API_URL = 'http://127.0.0.1/hello'
const response = await fetch(API_URL});

if (!response.ok) {
  const message = `An error has occurred: ${response.status}`;
  throw new Error(message);
}

const reader = response.body.getReader();
try {
while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  const chunk = new TextDecoder().decode(value);
  text = getText(chunk)
  console.log("Received chunk: ", text);
  write(text)
}
```

## React
