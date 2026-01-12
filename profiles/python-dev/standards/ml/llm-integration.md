## LLM Integration Best Practices

### LLM Provider Setup
- **Multiple Providers**: Support multiple LLM providers (OpenAI, Anthropic, Google, Ollama)
- **Unified Interface**: Create consistent interface across providers
- **Environment Config**: Store API keys in environment variables
- **Fallback Strategy**: Implement fallback to alternative providers

```python
import os
from enum import Enum
from typing import Optional
from openai import AsyncOpenAI
from anthropic import AsyncAnthropic
import google.generativeai as genai

class LLMProvider(str, Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    GOOGLE = "google"
    OLLAMA = "ollama"

class LLMClient:
    """Unified LLM client supporting multiple providers."""

    def __init__(self):
        self.openai_client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))
        self.anthropic_client = AsyncAnthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
        genai.configure(api_key=os.getenv("GOOGLE_API_KEY"))

    async def generate(
        self,
        prompt: str,
        provider: LLMProvider = LLMProvider.OPENAI,
        model: Optional[str] = None,
        max_tokens: int = 1000,
        temperature: float = 0.7
    ) -> str:
        """Generate completion from specified provider."""
        if provider == LLMProvider.OPENAI:
            return await self._openai_generate(prompt, model or "gpt-4", max_tokens, temperature)
        elif provider == LLMProvider.ANTHROPIC:
            return await self._anthropic_generate(prompt, model or "claude-3-opus-20240229", max_tokens, temperature)
        elif provider == LLMProvider.GOOGLE:
            return await self._google_generate(prompt, model or "gemini-pro", max_tokens, temperature)
        else:
            raise ValueError(f"Unsupported provider: {provider}")
```

### OpenAI Integration
- **Async Client**: Use AsyncOpenAI for async operations
- **Error Handling**: Handle rate limits and API errors
- **Streaming**: Support streaming for long responses
- **Function Calling**: Use function calling for structured outputs

```python
from openai import AsyncOpenAI, RateLimitError, APIError
import asyncio

client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))

async def openai_completion(
    prompt: str,
    model: str = "gpt-4",
    max_tokens: int = 1000,
    temperature: float = 0.7,
    retry_count: int = 3
) -> str:
    """Generate completion with retry logic."""
    for attempt in range(retry_count):
        try:
            response = await client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
                max_tokens=max_tokens,
                temperature=temperature
            )
            return response.choices[0].message.content

        except RateLimitError:
            if attempt < retry_count - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                await asyncio.sleep(wait_time)
            else:
                raise

        except APIError as e:
            print(f"API Error: {e}")
            raise

# Streaming response
async def openai_stream(prompt: str, model: str = "gpt-4"):
    """Stream completion token by token."""
    stream = await client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    async for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

# Usage
async for token in openai_stream("Explain machine learning"):
    print(token, end='', flush=True)
```

### Anthropic Claude Integration
- **Messages API**: Use Anthropic's messages API
- **System Prompts**: Utilize system prompts effectively
- **Streaming**: Stream responses for better UX

```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

async def claude_completion(
    prompt: str,
    model: str = "claude-3-opus-20240229",
    max_tokens: int = 1000,
    temperature: float = 0.7,
    system_prompt: Optional[str] = None
) -> str:
    """Generate completion with Claude."""
    message = await client.messages.create(
        model=model,
        max_tokens=max_tokens,
        temperature=temperature,
        system=system_prompt or "You are a helpful AI assistant.",
        messages=[
            {"role": "user", "content": prompt}
        ]
    )
    return message.content[0].text

# Streaming with Claude
async def claude_stream(prompt: str, model: str = "claude-3-opus-20240229"):
    """Stream completion from Claude."""
    async with client.messages.stream(
        model=model,
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    ) as stream:
        async for text in stream.text_stream:
            yield text
```

### LangChain Integration
- **Chains**: Use LangChain chains for complex workflows
- **Prompts**: Use prompt templates for consistency
- **Memory**: Implement conversation memory
- **Agents**: Use agents for tool-using workflows

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.chains import LLMChain
from langchain.memory import ConversationBufferMemory

# Initialize LLM
llm = ChatOpenAI(
    model_name="gpt-4",
    temperature=0.7,
    openai_api_key=os.getenv("OPENAI_API_KEY")
)

# Create prompt template
prompt_template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful AI assistant specialized in {domain}."),
    ("human", "{input}")
])

# Create chain
chain = LLMChain(
    llm=llm,
    prompt=prompt_template
)

# Run chain
result = await chain.arun(domain="data science", input="Explain feature engineering")

# Chain with memory
memory = ConversationBufferMemory()

chain_with_memory = LLMChain(
    llm=llm,
    prompt=prompt_template,
    memory=memory
)

# Conversation
response1 = await chain_with_memory.arun(domain="python", input="What are decorators?")
response2 = await chain_with_memory.arun(domain="python", input="Give me an example")
```

### Embeddings
- **Batch Processing**: Process embeddings in batches
- **Caching**: Cache embeddings to avoid recomputation
- **Multiple Models**: Support different embedding models

```python
from openai import AsyncOpenAI
from sentence_transformers import SentenceTransformer
import numpy as np

# OpenAI embeddings
openai_client = AsyncOpenAI()

async def get_openai_embeddings(
    texts: list[str],
    model: str = "text-embedding-ada-002"
) -> list[list[float]]:
    """Get embeddings from OpenAI."""
    response = await openai_client.embeddings.create(
        model=model,
        input=texts
    )
    return [item.embedding for item in response.data]

# Local embeddings with Sentence Transformers
embedding_model = SentenceTransformer('all-MiniLM-L6-v2')

def get_local_embeddings(
    texts: list[str],
    batch_size: int = 32
) -> np.ndarray:
    """Get embeddings using local model."""
    return embedding_model.encode(
        texts,
        batch_size=batch_size,
        show_progress_bar=True,
        convert_to_numpy=True
    )

# Embedding cache
from functools import lru_cache
import hashlib

@lru_cache(maxsize=1000)
def cached_embedding(text: str) -> list[float]:
    """Cache embeddings for reuse."""
    return embedding_model.encode([text])[0].tolist()
```

### Token Counting
- **tiktoken**: Use tiktoken for accurate token counting
- **Budget Management**: Track and limit token usage
- **Cost Estimation**: Estimate API costs before calling

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4") -> int:
    """Count tokens in text for specific model."""
    encoding = tiktoken.encoding_for_model(model)
    return len(encoding.encode(text))

def estimate_cost(
    prompt: str,
    completion_length: int,
    model: str = "gpt-4"
) -> float:
    """Estimate API cost for completion."""
    prompt_tokens = count_tokens(prompt, model)
    total_tokens = prompt_tokens + completion_length

    # Pricing (example for gpt-4)
    if model == "gpt-4":
        prompt_cost = prompt_tokens * 0.03 / 1000
        completion_cost = completion_length * 0.06 / 1000
        return prompt_cost + completion_cost
    else:
        # Add other model pricing
        return 0.0

# Check before calling API
prompt = "Long prompt text..."
token_count = count_tokens(prompt)

if token_count > 4000:
    print(f"Warning: Prompt has {token_count} tokens, may exceed context window")
```

### Prompt Engineering
- **Templates**: Use reusable prompt templates
- **Few-Shot Examples**: Include examples in prompts
- **Clear Instructions**: Write clear, specific instructions
- **System Prompts**: Use system prompts to set behavior

```python
from string import Template

# Prompt template
ANALYSIS_PROMPT = Template("""
You are an expert data analyst. Analyze the following data and provide insights.

Data:
$data

Focus areas:
$focus_areas

Provide:
1. Key patterns and trends
2. Notable anomalies
3. Actionable recommendations

Response format: JSON
""")

# Few-shot prompt
FEW_SHOT_CLASSIFICATION = """
Classify the sentiment of the following texts.

Examples:
Text: "I love this product!"
Sentiment: positive

Text: "This is terrible and doesn't work."
Sentiment: negative

Text: "It's okay, nothing special."
Sentiment: neutral

Now classify:
Text: "{text}"
Sentiment:
"""

# Using templates
prompt = ANALYSIS_PROMPT.substitute(
    data=data_summary,
    focus_areas="- Sales trends\n- Customer behavior\n- Revenue patterns"
)

response = await llm_client.generate(prompt)
```

### Structured Output with Pydantic
- **JSON Mode**: Use JSON mode for structured output
- **Pydantic Models**: Define response schemas
- **Validation**: Validate LLM outputs

```python
from pydantic import BaseModel, Field
from typing import List
import json

class SentimentAnalysis(BaseModel):
    """Schema for sentiment analysis output."""
    text: str
    sentiment: str = Field(..., pattern="^(positive|negative|neutral)$")
    confidence: float = Field(..., ge=0.0, le=1.0)
    keywords: List[str]

async def analyze_sentiment_structured(text: str) -> SentimentAnalysis:
    """Get structured sentiment analysis."""
    prompt = f"""
    Analyze the sentiment of the following text and return a JSON response.

    Text: {text}

    Return JSON with this exact structure:
    {{
        "text": "original text",
        "sentiment": "positive/negative/neutral",
        "confidence": 0.0-1.0,
        "keywords": ["keyword1", "keyword2"]
    }}
    """

    response = await openai_client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"}
    )

    result_json = json.loads(response.choices[0].message.content)
    return SentimentAnalysis(**result_json)
```

### Conversation Management
- **Message History**: Maintain conversation history
- **Context Window**: Manage context window limits
- **Memory Strategies**: Implement memory strategies (buffer, summary, entity)

```python
from collections import deque
from typing import List, Dict

class ConversationManager:
    """Manage conversation history with token limits."""

    def __init__(self, max_tokens: int = 4000):
        self.messages: deque = deque()
        self.max_tokens = max_tokens

    def add_message(self, role: str, content: str):
        """Add message to history."""
        self.messages.append({"role": role, "content": content})
        self._truncate_if_needed()

    def _count_tokens(self) -> int:
        """Count total tokens in conversation."""
        total = 0
        for msg in self.messages:
            total += count_tokens(msg["content"])
        return total

    def _truncate_if_needed(self):
        """Remove old messages if exceeding token limit."""
        while self._count_tokens() > self.max_tokens and len(self.messages) > 1:
            self.messages.popleft()

    def get_messages(self) -> List[Dict]:
        """Get all messages for API call."""
        return list(self.messages)

# Usage
conversation = ConversationManager(max_tokens=4000)

conversation.add_message("system", "You are a helpful assistant.")
conversation.add_message("user", "What is machine learning?")

response = await openai_client.chat.completions.create(
    model="gpt-4",
    messages=conversation.get_messages()
)

conversation.add_message("assistant", response.choices[0].message.content)
```

### RAG (Retrieval Augmented Generation)
- **Vector Search**: Use vector databases for retrieval
- **Context Injection**: Inject retrieved context into prompts
- **Reranking**: Rerank results for relevance

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.docstore.document import Document

# Initialize vector store
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(
    collection_name="documents",
    embedding_function=embeddings,
    persist_directory="./chroma_db"
)

async def rag_query(
    query: str,
    top_k: int = 3
) -> str:
    """Answer query using RAG."""
    # Retrieve relevant documents
    docs = vectorstore.similarity_search(query, k=top_k)

    # Build context from retrieved docs
    context = "\n\n".join([doc.page_content for doc in docs])

    # Create prompt with context
    prompt = f"""
    Answer the question based on the following context. If the context doesn't
    contain relevant information, say so.

    Context:
    {context}

    Question: {query}

    Answer:
    """

    # Generate response
    response = await openai_client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )

    return response.choices[0].message.content
```

### Error Handling and Retries
- **Exponential Backoff**: Implement exponential backoff for rate limits
- **Timeout Handling**: Handle timeouts gracefully
- **Fallback Providers**: Fall back to alternative providers

```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def llm_call_with_retry(prompt: str) -> str:
    """LLM call with automatic retry."""
    try:
        response = await openai_client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            timeout=30.0
        )
        return response.choices[0].message.content

    except RateLimitError:
        print("Rate limit hit, retrying...")
        raise  # Will trigger retry

    except APIError as e:
        print(f"API error: {e}")
        # Try fallback provider
        return await fallback_provider_call(prompt)
```

### Batch Processing
- **Concurrent Requests**: Process multiple requests concurrently
- **Rate Limiting**: Respect API rate limits
- **Progress Tracking**: Track progress for long batches

```python
import asyncio
from asyncio import Semaphore

async def process_batch(
    prompts: List[str],
    max_concurrent: int = 5
) -> List[str]:
    """Process multiple prompts concurrently with rate limiting."""
    semaphore = Semaphore(max_concurrent)

    async def process_one(prompt: str) -> str:
        async with semaphore:
            return await llm_call_with_retry(prompt)

    # Process all prompts concurrently
    tasks = [process_one(prompt) for prompt in prompts]
    results = await asyncio.gather(*tasks)

    return results

# Usage
prompts = ["prompt 1", "prompt 2", "prompt 3", ...]
results = await process_batch(prompts, max_concurrent=5)
```

### Monitoring and Logging
- **Track Metrics**: Track token usage, latency, costs
- **Structured Logging**: Use structured logging for LLM calls
- **Performance Metrics**: Monitor performance metrics

```python
import logging
import time
from dataclasses import dataclass
from datetime import datetime

@dataclass
class LLMCallMetrics:
    timestamp: datetime
    provider: str
    model: str
    prompt_tokens: int
    completion_tokens: int
    latency_ms: float
    cost: float

class LLMMonitor:
    """Monitor LLM API calls."""

    def __init__(self):
        self.metrics: List[LLMCallMetrics] = []
        self.logger = logging.getLogger(__name__)

    async def tracked_call(
        self,
        prompt: str,
        provider: str = "openai",
        model: str = "gpt-4"
    ) -> str:
        """Make LLM call with tracking."""
        start_time = time.time()

        response = await openai_client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}]
        )

        latency = (time.time() - start_time) * 1000

        metrics = LLMCallMetrics(
            timestamp=datetime.now(),
            provider=provider,
            model=model,
            prompt_tokens=response.usage.prompt_tokens,
            completion_tokens=response.usage.completion_tokens,
            latency_ms=latency,
            cost=self._calculate_cost(response.usage, model)
        )

        self.metrics.append(metrics)

        self.logger.info(
            "LLM call completed",
            extra={
                "provider": provider,
                "model": model,
                "tokens": response.usage.total_tokens,
                "latency_ms": latency,
                "cost": metrics.cost
            }
        )

        return response.choices[0].message.content

    def _calculate_cost(self, usage, model: str) -> float:
        """Calculate API call cost."""
        # Add pricing logic
        return 0.0

    def get_summary(self) -> Dict:
        """Get summary of all calls."""
        if not self.metrics:
            return {}

        total_calls = len(self.metrics)
        total_tokens = sum(m.prompt_tokens + m.completion_tokens for m in self.metrics)
        total_cost = sum(m.cost for m in self.metrics)
        avg_latency = sum(m.latency_ms for m in self.metrics) / total_calls

        return {
            "total_calls": total_calls,
            "total_tokens": total_tokens,
            "total_cost": total_cost,
            "avg_latency_ms": avg_latency
        }
```
