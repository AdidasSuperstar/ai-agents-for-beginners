# Semantic Kernel Notebook Explanation

## What This Code Does

This cell runs a two-turn conversation with your `TravelAgent`.

The two user messages are:

```python
user_inputs = [
    "Plan me a day trip.",
    "I don't like that destination. Plan me another vacation.",
]
```

So the agent first plans a trip, then the user rejects the destination and asks for another one.

The important part is this:

```python
thread: ChatHistoryAgentThread | None = None
```

`thread` stores the conversation history. After the first message, Semantic Kernel returns an updated thread:

```python
thread = response.thread
```

That means the second user message is not treated as a brand-new conversation. The agent knows the user already rejected the first destination.

Inside this loop:

```python
async for response in agent.invoke_stream(
    messages=user_input,
    thread=thread,
):
```

the agent is called in streaming mode. Instead of waiting for one final response, Semantic Kernel sends back pieces as they happen.

Those pieces can be different types:

```python
FunctionCallContent
```

Means the model is trying to call a tool/function, such as:

```python
get_random_destination()
```

from your `DestinationsPlugin`.

```python
FunctionResultContent
```

Means Semantic Kernel has executed the function and is returning the result back to the model. For example:

```text
Berlin, Germany
```

```python
StreamingTextContent
```

Means normal assistant text, streamed piece by piece.

So this section:

```python
if isinstance(item, FunctionCallContent):
```

captures the tool call.

This section:

```python
elif isinstance(item, FunctionResultContent):
```

captures the result of the tool call.

And this section:

```python
elif isinstance(item, StreamingTextContent) and item.text:
    full_response.append(item.text)
```

collects the final natural-language answer from the assistant.

At the end, the code builds some HTML so the notebook output looks nicer. It shows:

- the user message
- expandable function call details
- the assistant response

So in plain English: this cell sends two messages to the agent, lets the agent call your destination plugin, keeps conversation memory between turns, and displays the streamed response nicely in the notebook.

## Semantic Kernel Vs Agent Framework

Semantic Kernel and Microsoft Agent Framework are related ideas, but they sit at different levels.

Semantic Kernel is more like an application SDK for orchestration. You define plugins, connect a model, create an agent, and your Python code controls the flow. In this notebook, your local Python class:

```python
DestinationsPlugin
```

becomes a tool the model can call.

The Semantic Kernel pieces are:

```python
OpenAIChatCompletion
ChatCompletionAgent
kernel_function
```

`OpenAIChatCompletion` connects to the model.

`ChatCompletionAgent` wraps that model and gives it agent behavior.

`kernel_function` exposes your Python method as a callable tool.

Microsoft Agent Framework, especially when used with Foundry, is more platform-oriented. You often create or connect to agents managed through Azure AI Foundry. It is better suited for production patterns like hosted agents, Azure identity, tracing, evaluation, deployment, and enterprise governance.

A simple way to think about it:

```text
Semantic Kernel:
You build and run the agent logic in your app/notebook.

Agent Framework + Foundry:
You often work with agents connected to Azure AI Foundry resources and production services.
```

## Why GitHub Models Here?

This notebook uses GitHub Models because it is simpler for a beginner course sample.

With GitHub Models, you usually only need:

```python
GITHUB_TOKEN
GITHUB_ENDPOINT
GITHUB_MODEL_ID
```

and the endpoint:

```python
https://models.github.ai/inference
```

That endpoint behaves like an OpenAI-compatible chat completions API. Because of that, Semantic Kernel can use:

```python
OpenAIChatCompletion
```

even though the model is being served by GitHub Models.

So this code:

```python
client = AsyncOpenAI(
    api_key=os.environ.get("GITHUB_TOKEN"),
    base_url=os.environ.get("GITHUB_ENDPOINT", "https://models.github.ai/inference"),
)

chat_completion_service = OpenAIChatCompletion(
    ai_model_id=os.environ.get("GITHUB_MODEL_ID", "gpt-4o-mini"),
    async_client=client,
)
```

means:

"Create an OpenAI-compatible client, point it at GitHub Models, then give that client to Semantic Kernel."

## Can We Use Foundry Models Instead?

Yes. Semantic Kernel can be used with Azure OpenAI / Foundry-hosted models too. GitHub Models is not required by Semantic Kernel.

The reason this sample uses GitHub Models is mostly convenience:

- no Azure resource needs to be provisioned
- no Foundry project setup is required
- easier for learners to run locally
- good for demonstrating the Semantic Kernel concepts: agent, tool calling, streaming, and conversation thread

If you use Foundry or Azure OpenAI, the agent logic can stay mostly the same. The main thing you would change is the model connection setup: endpoint, credential, model/deployment name, and possibly the Semantic Kernel connector class.

So the core difference is not:

```text
Semantic Kernel = GitHub Models
Agent Framework = Foundry
```

It is more like:

```text
This notebook uses Semantic Kernel with GitHub Models because it is lightweight for learning.

A production version might use Semantic Kernel or Agent Framework with Azure AI Foundry, depending on how much platform management you want.
```

One small note: your `DestinationsPlugin` is the actual tool. The model does not manually pick from the list itself. It decides when to call the tool, Semantic Kernel runs the Python function, then the model uses the function result to write the final travel answer.