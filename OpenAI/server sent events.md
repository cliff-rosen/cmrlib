# Server Sent Events

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

## Flask
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


## JavaScript
### With EventStream
```
const eventSource = new EventSource("127.0.0.1/hello"); // Replace with your server endpoint

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
eventSource.addEventListener("new-message", (event) => {
  // Handle "new-message" events specifically
});
```
### Without EventStream
```
endpoint = 'API_URL'
const data = {
  model: 'gpt-3.5-turbo',
  messages: messages,
  temperature: 0.7,
  max_tokens: 150,
  top_p: 1.0,
  frequency_penalty: 0.0,
  presence_penalty: 0.0,
  stream: stream
};
    
const response = await fetch(endpoint, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${apiKey}`,
  },
  body: JSON.stringify(data),
});

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
