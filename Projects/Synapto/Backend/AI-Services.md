# Backend AI Services

**Complete guide to Synapto's AI services architecture and implementation**

---

## Overview

The backend AI system consists of **three main components** that work together to automate email handling:

1. **EmailClassifier** - Classifies incoming emails and extracts structured data
2. **ResponseGenerator** - Generates tenant-specific responses using templates
3. **ToolOrchestrator** - Autonomous agent that can call tools to gather information

All AI services use **Groq API** with llama-3.3-70b-versatile as the primary model.

---

## AIService Facade

**Location**: `backend/app/services/ai_service.py`

The AIService is the main entry point for all AI operations:

```python
class AIService:
    """Facade for AI operations."""
    
    def __init__(self, tenant: Tenant):
        self.tenant = tenant
        self.groq_client = GroqClient()
        
        # Initialize components
        self.classifier = EmailClassifier(tenant, self.groq_client)
        self.generator = ResponseGenerator(tenant, self.groq_client)
        self.orchestrator = ToolOrchestrator(tenant, self.groq_client)
    
    async def classify_email(self, email_content: str) -> ClassificationResult:
        """Classify email and extract details."""
        return await self.classifier.classify(email_content)
    
    async def generate_response(
        self, 
        classification: ClassificationResult,
        tone_sliders: dict
    ) -> str:
        """Generate response with tenant-specific tone."""
        return await self.generator.generate(classification, tone_sliders)
    
    async def orchestrate(self, query: str, conversation_id: str) -> str:
        """Run autonomous agent with tool calling."""
        return await self.orchestrator.execute(query, conversation_id)
```

**Key Features**:
- Tenant-aware (all components receive tenant object)
- Groq client with circuit breaker for resilience
- Async/await for non-blocking I/O

---

## EmailClassifier

**Location**: `backend/app/services/ai_components/email_classifier.py`

Classifies incoming emails into categories and extracts structured information.

### Classification Categories

```python
class EmailCategory(Enum):
    BOOKING_REQUEST = "booking_request"
    BOOKING_MODIFICATION = "booking_modification" 
    GENERAL_INQUIRY = "general_inquiry"
    COMPLAINT = "complaint"
    CANCELLATION = "cancellation"
    PAYMENT = "payment"
    SPAM = "spam"
```

### Implementation

```python
class EmailClassifier:
    """Classifies emails and extracts structured data."""
    
    async def classify(self, email_content: str) -> ClassificationResult:
        """
        Classify email and extract details.
        
        Returns:
            ClassificationResult with:
              - category: EmailCategory
              - confidence: float (0-1)
              - extracted_data: dict with customer info
              - sentiment: str (positive/neutral/negative)
        """
        # Build tenant-specific prompt
        prompt = self._build_classification_prompt(email_content)
        
        # Call Groq API with JSON mode
        response = await self.groq_client.chat_completion(
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
            model="llama-3.3-70b-versatile"
        )
        
        # Parse structured output
        result = json.loads(response.choices[0].message.content)
        
        return ClassificationResult(
            category=EmailCategory(result["category"]),
            confidence=result["confidence"],
            extracted_data=result["extracted_data"],
            sentiment=result["sentiment"]
        )
```

### Extracted Data Example

```json
{
  "category": "booking_request",
  "confidence": 0.95,
  "extracted_data": {
    "customer_name": "Anna Hansen",
    "customer_email": "anna@example.com",
    "customer_phone": "+45 12 34 56 78",
    "service_type": "private",
    "preferred_date": "2026-01-15",
    "address": "Nørrebrogade 123, 2200 København N",
    "square_meters": 85,
    "special_requests": "Need windows cleaned"
  },
  "sentiment": "positive"
}
```

### Prompt Engineering

The classifier uses a **structured prompt** with:
- Tenant-specific service types (private vs commercial)
- Tenant-specific pricing context
- Clear JSON schema for output
- Few-shot examples

See [[Backend/Prompt-Engineering]] for complete prompt templates.

---

## ResponseGenerator

**Location**: `backend/app/services/ai_components/response_generator.py`

Generates appropriate responses using tenant-specific templates and tone.

### Tone Sliders

Rendetalje and FB Rengøring use different tones:

```python
# Rendetalje (Premium)
RENDETALJE_TONE = {
    "formality": 0.8,      # High formality
    "enthusiasm": 0.6,     # Moderate enthusiasm
    "detail_level": 0.7,   # Detailed responses
    "language_style": "luxury"
}

# FB Rengøring (Commercial)
FB_TONE = {
    "formality": 0.6,      # Medium formality
    "enthusiasm": 0.4,     # Lower enthusiasm
    "detail_level": 0.5,   # Concise responses
    "language_style": "efficient"
}
```

### Implementation

```python
class ResponseGenerator:
    """Generates tenant-specific email responses."""
    
    async def generate(
        self, 
        classification: ClassificationResult,
        tone_sliders: dict = None
    ) -> str:
        """
        Generate response using template + LLM.
        
        Process:
        1. Select template based on category
        2. Populate template with extracted data
        3. Apply tone adjustments via LLM
        4. Add tenant-specific signature
        """
        # Get base template
        template = self._get_template(classification.category)
        
        # Populate with data
        populated = template.format(**classification.extracted_data)
        
        # Apply tone via LLM
        tone_sliders = tone_sliders or self.tenant.tone_settings
        prompt = self._build_tone_prompt(populated, tone_sliders)
        
        response = await self.groq_client.chat_completion(
            messages=[{"role": "user", "content": prompt}],
            model="llama-3.3-70b-versatile",
            temperature=0.7  # Creative for natural language
        )
        
        # Add signature
        final_response = response.choices[0].message.content
        final_response += f"\n\n{self.tenant.tone_settings['signature']}"
        
        return final_response
```

### Response Templates

Templates are stored in `backend/prompts/response_templates/`:

**booking_request.txt**:
```
Kære {customer_name},

Tak for din henvendelse om rengøring.

Vi har modtaget din forespørgsel om {service_type} rengøring 
på adressen {address} ({square_meters} m²).

Vores pris er {price} DKK baseret på {hours} timer á {hourly_rate} DKK.

Kan vi bekræfte booking til {preferred_date} kl. {preferred_time}?

Med venlig hilsen
[SIGNATURE_PLACEHOLDER]
```

---

## ToolOrchestrator

**Location**: `backend/app/services/ai_components/tool_orchestrator.py`

Autonomous agent that can call tools to gather information before responding.

### Available Tools

```python
AVAILABLE_TOOLS = [
    {
        "name": "get_customer_info",
        "description": "Retrieve customer information by email",
        "parameters": {
            "email": {"type": "string", "description": "Customer email"}
        }
    },
    {
        "name": "check_availability",
        "description": "Check booking availability for date range",
        "parameters": {
            "start_date": {"type": "string", "format": "YYYY-MM-DD"},
            "end_date": {"type": "string", "format": "YYYY-MM-DD"}
        }
    },
    {
        "name": "calculate_price",
        "description": "Calculate price for service",
        "parameters": {
            "service_type": {"type": "string"},
            "square_meters": {"type": "number"},
            "hours": {"type": "number"}
        }
    }
]
```

### Implementation Pattern

```python
class ToolOrchestrator:
    """Autonomous agent with tool calling."""
    
    MAX_ITERATIONS = 3  # Prevent infinite loops
    
    async def execute(self, query: str, conversation_id: str) -> str:
        """
        Execute agent loop with tool calling.
        
        Flow:
        1. LLM decides if tool call needed
        2. If yes: Execute tool, add result to context
        3. Repeat until LLM provides final answer
        4. Max 3 iterations to prevent loops
        """
        conversation_history = await self._load_history(conversation_id)
        conversation_history.append({"role": "user", "content": query})
        
        for iteration in range(self.MAX_ITERATIONS):
            # Ask LLM with tool definitions
            response = await self.groq_client.chat_completion(
                messages=conversation_history,
                tools=AVAILABLE_TOOLS,
                model="llama-3.3-70b-versatile"
            )
            
            # Check if tool call requested
            if response.choices[0].message.tool_calls:
                # Execute tool
                tool_call = response.choices[0].message.tool_calls[0]
                tool_result = await self._execute_tool(
                    tool_call.function.name,
                    json.loads(tool_call.function.arguments)
                )
                
                # Add to conversation
                conversation_history.append({
                    "role": "assistant",
                    "content": None,
                    "tool_calls": [tool_call]
                })
                conversation_history.append({
                    "role": "tool",
                    "content": json.dumps(tool_result),
                    "tool_call_id": tool_call.id
                })
            else:
                # Final answer provided
                final_answer = response.choices[0].message.content
                await self._save_history(conversation_id, conversation_history)
                return final_answer
        
        # Max iterations reached
        return "I apologize, but I need more information to help you."
```

### Example Flow

**User Query**: "Can I book cleaning for January 15th?"

**Iteration 1**:
- LLM calls `check_availability(start_date="2026-01-15", end_date="2026-01-15")`
- Tool returns: `{"available": true, "time_slots": ["09:00", "13:00"]}`

**Iteration 2**:
- LLM has availability data
- Provides final answer: "Yes! We have availability on January 15th at 09:00 or 13:00. Which time works best for you?"

---

## Groq Client with Circuit Breaker

**Location**: `backend/app/services/groq/client.py`

Resilient wrapper around Groq API with circuit breaker pattern.

### Circuit Breaker Configuration

```python
from pybreaker import CircuitBreaker

GROQ_BREAKER = CircuitBreaker(
    fail_max=5,              # Open after 5 failures
    timeout_duration=60,     # Stay open for 60 seconds
    name="groq_api"
)

class GroqClient:
    """Groq API client with circuit breaker."""
    
    @GROQ_BREAKER
    async def chat_completion(self, **kwargs) -> ChatCompletion:
        """
        Call Groq API with circuit breaker protection.
        
        If circuit is OPEN (too many failures):
          - Raises CircuitBreakerError immediately
          - Prevents cascading failures
          - Returns after timeout_duration
        """
        try:
            response = await self.client.chat.completions.create(**kwargs)
            return response
        except Exception as e:
            logger.error(f"Groq API error: {e}")
            raise
```

### Fallback Strategy

When circuit breaker opens:

```python
try:
    response = await groq_client.chat_completion(...)
except CircuitBreakerError:
    # Circuit is open - use fallback
    return await self._get_fallback_response(category)

def _get_fallback_response(self, category: EmailCategory) -> str:
    """Return pre-written response when AI unavailable."""
    FALLBACKS = {
        EmailCategory.BOOKING_REQUEST: (
            "Tak for din henvendelse. Vi vender tilbage hurtigst muligt."
        ),
        # ... other categories
    }
    return FALLBACKS.get(category, "Tak for din besked.")
```

---

## Context Window Management

**Location**: `backend/app/services/memory_service.py`

Manages conversation memory with sliding window + summarization.

### Strategy

```python
class MemoryService:
    """Manages conversation memory with context window limits."""
    
    MAX_MESSAGES = 20  # Keep recent 20 messages
    
    async def get_conversation_context(
        self, 
        conversation_id: str
    ) -> List[dict]:
        """
        Get conversation context with smart truncation.
        
        If > 20 messages:
        1. Summarize old messages (1-10)
        2. Keep recent messages (11-20) verbatim
        3. Reduces tokens by 45-80%
        """
        messages = await self._load_messages(conversation_id)
        
        if len(messages) <= self.MAX_MESSAGES:
            return messages
        
        # Split into old + recent
        old_messages = messages[:-self.MAX_MESSAGES]
        recent_messages = messages[-self.MAX_MESSAGES:]
        
        # Summarize old messages
        summary = await self._summarize_messages(old_messages)
        
        # Combine summary + recent
        return [
            {"role": "system", "content": f"Previous context: {summary}"},
            *recent_messages
        ]
    
    async def _summarize_messages(self, messages: List[dict]) -> str:
        """Use Groq to summarize old conversation."""
        prompt = f"Summarize this conversation:\n{json.dumps(messages)}"
        
        response = await self.groq_client.chat_completion(
            messages=[{"role": "user", "content": prompt}],
            model="llama-3.3-70b-versatile",
            max_tokens=200  # Short summary
        )
        
        return response.choices[0].message.content
```

### Token Savings

- **Before**: 10,000 tokens (50 messages)
- **After**: 2,500 tokens (summary + 20 recent)
- **Reduction**: 75%

---

## Testing AI Services

### Test Structure

```python
# tests/test_ai_service.py
import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_email_classification(tenant_rendetalje):
    """Test email classifier with mock Groq response."""
    
    # Mock Groq response
    mock_response = {
        "choices": [{
            "message": {
                "content": json.dumps({
                    "category": "booking_request",
                    "confidence": 0.95,
                    "extracted_data": {"customer_name": "Test"}
                })
            }
        }]
    }
    
    with patch.object(GroqClient, 'chat_completion', new=AsyncMock(return_value=mock_response)):
        classifier = EmailClassifier(tenant_rendetalje, GroqClient())
        
        result = await classifier.classify("I want to book cleaning")
        
        assert result.category == EmailCategory.BOOKING_REQUEST
        assert result.confidence == 0.95
```

---

## Related Documentation
- [[Multi-Tenant-Guide]] - Tenant-specific AI configuration
- [[Backend/Groq-Integration]] - Groq API details
- [[Backend/Prompt-Engineering]] - Prompt templates
- [[Testing/AI-Testing]] - AI testing strategies

#ai #backend #groq #llm #automation
