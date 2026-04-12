# Event-Driven Integration Patterns

**Reference document for:** [SKILL.md](SKILL.md) - event-driven-architecture skill

Use this skill when integrating external APIs, SDKs, or message brokers into your event-driven system. This skill covers SDK method selection, consumer group semantics, defensive response parsing, and schema constraint alignment.

## When to Use

- Choosing between SDK methods (generate vs chat, subscribe vs fetch)
- Setting up consumer groups with proper identity semantics
- Parsing responses from external APIs
- Aligning configuration/prompts with database schema
- Handling non-deterministic API responses

---

## 1. SDK API Selection: Match Method to Use Case

### Antipattern: Choosing Method by Convenience

**Problem:** Selecting a method just because it's simpler or more familiar, without understanding what it does.

Example (any language):
```
❌ WRONG: Use simple/completion endpoint
   - Response has no role structure (just raw text)
   - Can't separate system instructions from user data
   - No multi-turn conversation support
```

### Pattern: Choose Method Based on Requirements

**Use messaging/conversation API (chat, messages, etc.) when you need:**
- System prompts or role-based instructions (system, user, assistant roles)
- Multi-turn conversations (context preservation)
- Structured messaging (request/response with role metadata)
- Ability to separate instructions from input data

**Use simple completion API (generate, completion, etc.) when:**
- Simple text-in, text-out is enough
- You have a single prompt string
- No need for role-based structure or context

### Correct Implementation Pattern

Structured message format (language-independent):
```
Request to messaging API:
  messages: [
    {"role": "system", "content": instructions_string},
    {"role": "user", "content": input_data_string}
  ]
  model: model_name
  stream: false

Response: structured object with message content accessible
```

**Language-specific examples:**

Python (asyncio):
```python
messages = [
    {"role": "system", "content": system_instructions},
    {"role": "user", "content": user_data}
]
response = await client.chat(model=model_name, messages=messages)
result = response.message.content
```

JavaScript/Node.js (async):
```javascript
const messages = [
    {role: "system", content: systemInstructions},
    {role: "user", content: userData}
];
const response = await client.chat({model: modelName, messages});
const result = response.message.content;
```

Go:
```go
messages := []Message{
    {Role: "system", Content: systemInstructions},
    {Role: "user", Content: userData},
}
response := client.Chat(ctx, &ChatRequest{Model: modelName, Messages: messages})
result := response.Message.Content
```

**Key differences:**
- Completion API: response = raw text string
- Messaging API: response = structured object with role metadata and content

**Action:** Before implementing, document:
- What method are you using? (chat/messages/completion/generate/other)
- Why this method? (justify: "need system prompts", "simple completion", etc.)
- How do you access the response? (understand your language's SDK structure)

---

## 2. Consumer Group Semantics: Identity vs Group

### Antipattern: Same Name for Both

```
❌ WRONG: durable_name and deliver_group identical
   subscriber_config = {
       subject: "work",
       durable_name: "batch-workers",   # Same value
       deliver_group: "batch-workers"   # = same value
   }

   Problem: Can't distinguish which instance processed message N
   Debugging harder; no per-instance tracking; no resumption by instance
```

### Pattern: Separate Identity and Group

```
✅ RIGHT: Unique durable_name per instance, shared deliver_group
   
   worker-1: {
       subject: "work",
       durable_name: "worker-1",        # Unique per instance (hostname, UUID, pod name)
       deliver_group: "batch-workers"   # Shared across all workers in group
   }
   
   worker-2: {
       subject: "work",
       durable_name: "worker-2",        # Different identity, same group
       deliver_group: "batch-workers"   # Same group
   }
   
   Benefits:
   - Each instance has unique identity (worker-1, worker-2, worker-3)
   - All share same deliver_group for load balancing
   - Each resumes from own position on restart
   - Observability: track which instance processed each message
   - Dead-letter: route to specific instance if needed
```

**How to generate durable_name:**

Language-independent pattern:
```
Option A: Use hostname
  durable_name = get_system_hostname()  # Returns: worker-pod-123

Option B: Use environment variable (Kubernetes)
  durable_name = get_env("HOSTNAME") || get_env("POD_NAME") || "worker-unknown"

Option C: Use UUID (for truly stateless workers)
  durable_name = "worker-" + generate_uuid().substring(0, 8)

Deliver group always shared:
  deliver_group = "batch-workers"  (constant for all instances)
```

Language-specific examples:

Python:
```python
import socket, os, uuid

# Option A: Hostname
durable_name = socket.gethostname()

# Option B: K8s env var
durable_name = os.getenv("HOSTNAME", "worker-unknown")

# Option C: UUID
durable_name = f"worker-{uuid.uuid4().hex[:8]}"

deliver_group = "batch-workers"
```

JavaScript:
```javascript
const os = require('os');
const {v4: uuidv4} = require('uuid');

// Option A: Hostname
const durableName = os.hostname();

// Option B: K8s env var
const durableName = process.env.HOSTNAME || "worker-unknown";

// Option C: UUID
const durableName = `worker-${uuidv4().substring(0, 8)}`;

const deliverGroup = "batch-workers";
```

Go:
```go
import ("os"; "net"; uuid "github.com/google/uuid")

// Option A: Hostname
durableName, _ := os.Hostname()

// Option B: K8s env var
durableName := os.Getenv("HOSTNAME")
if durableName == "" { durableName = "worker-unknown" }

// Option C: UUID
durableName := "worker-" + uuid.New().String()[:8]

deliverGroup := "batch-workers"
```

**Documentation required:**
```markdown
## Consumer Group Setup

**durable_name:** Per-instance unique identifier
  - Value: Pod hostname (Kubernetes) or UUID (if stateless)
  - Purpose: Track which instance processed each message
  - Persisted: Yes, resumes from last ack on restart

**deliver_group:** Shared across all instances
  - Value: "batch-workers" (constant for all instances)
  - Purpose: Group membership for round-robin distribution
  - Behavior: NATS round-robins messages among subscribers with same deliver_group

**Why separate:**
  - Identity (durable_name) ≠ Group (deliver_group)
  - Same identity across restarts (resumption)
  - Same group name across all instances (load balancing)
```

---

## 3. Defensive Response Parsing

### Antipattern: Assume Well-Formed Responses

```
❌ WRONG: Assume API always returns valid JSON with required fields
   response_data = parse_json(response_text)
   summary = response_data["summary"]      # Crashes if "summary" missing
   priority = response_data["priority"]    # Crashes if "priority" missing
```

### Pattern: Defensive Parsing with Fallbacks

Language-independent pattern:
```
✅ RIGHT: 
   1. Try to parse response
   2. Use safe dictionary access (get() or equivalent)
   3. Provide sensible default for each field
   4. Catch parse errors (non-JSON, malformed)
   5. Return fallback values on error
```

**Why this matters:**
- External APIs are non-deterministic (network issues, timeouts, format changes)
- Generative APIs sometimes return plain text instead of structured format
- Error responses may be HTML, plain text, or different format
- API versions/migrations can change response structure
- Timeouts and retries can change response content

**Defensive parsing checklist:**
```
1. Wrap parsing in try-catch/try-except
2. Use safe field access (get, optional fields, null coalescing)
3. Provide sensible defaults for all required fields
4. Handle parse errors explicitly
5. Return fallback values when parsing fails
6. Log what happened (for debugging)
```

**Language-specific examples:**

Python:
```python
import json

try:
    result_json = json.loads(response.text)
    summary = result_json.get("summary", response.text)  # Safe field access
    priority = result_json.get("priority", "medium")     # Default value
except (json.JSONDecodeError, ValueError, TypeError):
    summary = response.text  # Fallback: use whole response
    priority = "medium"      # Fallback: sensible default
```

JavaScript:
```javascript
let summary, priority;
try {
    const resultJson = JSON.parse(responseText);
    summary = resultJson.summary || responseText;      // Safe field access
    priority = resultJson.priority || "medium";        // Default value
} catch (error) {
    summary = responseText;  // Fallback: use whole response
    priority = "medium";     // Fallback: sensible default
}
```

Go:
```go
var result map[string]interface{}
err := json.Unmarshal([]byte(responseText), &result)

var summary, priority string
if err == nil {
    // Parse succeeded; use safe field access
    if s, ok := result["summary"].(string); ok {
        summary = s
    } else {
        summary = responseText
    }
    if p, ok := result["priority"].(string); ok {
        priority = p
    } else {
        priority = "medium"
    }
} else {
    // Parse failed; use fallbacks
    summary = responseText
    priority = "medium"
}
```

**Test cases for defensive parsing:**

```
Test 1: Valid response with all fields
  Input: {"summary": "ok", "priority": "high"}
  Expected: summary="ok", priority="high"
  
Test 2: Missing optional fields
  Input: {"summary": "ok"}
  Expected: summary="ok", priority="medium" (default)
  
Test 3: Non-JSON response
  Input: "Error: timeout"
  Expected: summary="Error: timeout", priority="medium" (defaults)
  
Test 4: Empty response
  Input: ""
  Expected: summary="", priority="medium" (default)
  
Test 5: Malformed JSON
  Input: {summary: ok}  (missing quotes)
  Expected: Parse fails, returns fallbacks
```

---

## 4. Configuration & Schema Alignment

### Antipattern: Configuration Examples Don't Match Schema

```
❌ WRONG: Configuration tells external API to return "med" but database expects "medium"

Database Schema:
  ALTER TABLE records ADD CONSTRAINT priority_check 
    CHECK (priority IN ('low', 'medium', 'high', 'critical'))

Configuration Sent to API:
  "Return priority as one of: high, med, low"

Result: API returns "med" → Database rejects with constraint violation
```

### Pattern: Align Configuration with Constraints

Language-independent approach:
```
✅ RIGHT:
   1. List all constrained fields in response (enums, formats)
   2. Check database schema for exact allowed values
   3. Update configuration/prompt to show exact values, not abbreviations
   4. Provide example response with exact values
   5. Test that external API follows exact format
```

**Alignment checklist:**

```
Before writing configuration/prompt:
  [ ] List all enum/constrained fields in expected response
  [ ] Read database schema for exact values (not abbreviations)
  [ ] Document in configuration with exact values
  [ ] Provide example response with exact values
  [ ] Create test case that verifies config values match schema

Common value mismatches to watch for:
  ❌ "med" → database expects "medium"
  ❌ "high", "low" → database expects "high_priority", "low_priority"
  ❌ "true" (string) → database expects boolean true
  ❌ "2024-01" (partial ISO 8601) → database expects full timestamp
  ❌ "USD" → database expects integer currency code (840)
```

**Correct pattern (all languages):**

```
Configuration document:
  "Return a response with these fields:
   - summary (string): brief description
   - priority (string): exactly one of: 'low', 'medium', 'high', 'critical'
   - status (boolean): true if action needed, false otherwise
   - timestamp (ISO 8601 timestamp): when evaluation was done
   
   Example:
   {
     'summary': 'Medium risk detected',
     'priority': 'medium',
     'status': true,
     'timestamp': '2024-01-15T10:30:00Z'
   }"
```

**Test for constraint violations (language-independent):**

```
Test: Configuration values match schema

1. Extract valid values from configuration
   valid_priorities = ["low", "medium", "high", "critical"]
   valid_statuses = [true, false]

2. Read database schema for same fields
   db_priorities = ["low", "medium", "high", "critical"]
   db_statuses = [true, false]

3. Verify they match exactly
   assert valid_priorities == db_priorities
   assert valid_statuses == db_statuses

4. When external API returns data, validate against schema
   api_response = parse_response(...)
   assert api_response.priority IN db_priorities
   assert typeof(api_response.status) == boolean
   assert is_valid_iso8601_timestamp(api_response.timestamp)
```

Python example:
```python
def validate_config_matches_schema():
    # Config says these are valid
    config_priorities = ["low", "medium", "high", "critical"]
    
    # Database enforces these
    db_priorities = ["low", "medium", "high", "critical"]
    
    assert set(config_priorities) == set(db_priorities), \
        f"Config {config_priorities} != DB {db_priorities}"
    
    # When API returns data, test it matches
    api_response = '{"priority": "med"}'
    parsed = json.loads(api_response)
    assert parsed["priority"] in db_priorities, \
        f"API returned '{parsed['priority']}' not in {db_priorities}"
```

---

## 5. External API Call Semantics

### Publisher Behavior (Stateless)

**Pattern: Publisher is unaware of consumers**

```
✅ Publisher publishes to generic topic/channel/queue
   - Does not know who will consume the message
   - Does not implement routing logic
   - Does not implement load balancing
   - Lets broker handle message distribution

Publisher responsibility:
  1. Accept work (data, record ID, metadata)
  2. Serialize to wire format (JSON, protobuf, etc.)
  3. Publish to topic/stream
  4. Return success/failure to caller
```

Language-specific examples:

Python (asyncio):
```python
async def publish_work(record_id: str, data: dict):
    """Publish work; let broker handle distribution"""
    await broker.publish(
        topic="work",  # Generic topic; not per-entity
        message={
            "record_id": record_id,
            "data": data,
            "timestamp": datetime.now().isoformat()
        }
    )
```

JavaScript (async):
```javascript
async function publishWork(recordId, data) {
    await broker.publish({
        topic: "work",  // Generic topic
        message: {
            recordId: recordId,
            data: data,
            timestamp: new Date().toISOString()
        }
    });
}
```

Go:
```go
func (p *Publisher) PublishWork(ctx context.Context, recordID string, data interface{}) error {
    return p.broker.Publish(ctx, "work", &WorkMessage{
        RecordID:  recordID,
        Data:      data,
        Timestamp: time.Now().UTC(),
    })
}
```

**Publisher rules (all languages):**
- No consumer group logic in publisher
- No topic sharding by entity (use generic topic)
- No routing decisions
- Just serialize and publish; broker handles rest

### Consumer Behavior (Queue Group Aware)

**Pattern: Consumer specifies how it participates in load balancing**

```
✅ Consumer subscribes with:
   1. Topic/channel to listen to
   2. Unique consumer identity (for resumption on restart)
   3. Consumer group (for load balancing across instances)
```

Language-specific examples:

Python (asyncio):
```python
async def subscribe_to_work():
    """Subscribe with queue group for round-robin distribution"""
    subscription = await broker.subscribe(
        topic="work",
        consumer_id=get_instance_id(),      # Unique per instance
        consumer_group="batch-workers"      # Shared across instances
    )
    
    async for message in subscription:
        result = await process_message(message)
        await message.acknowledge()  # Only after successful processing
```

JavaScript (async):
```javascript
async function subscribeToWork() {
    const subscription = await broker.subscribe({
        topic: "work",
        consumerId: getInstanceId(),        // Unique per instance
        consumerGroup: "batch-workers"      // Shared across instances
    });
    
    for await (const message of subscription) {
        await processMessage(message);
        await message.acknowledge();        // After successful processing
    }
}
```

Go:
```go
func (c *Consumer) SubscribeToWork(ctx context.Context) error {
    sub, err := c.broker.Subscribe(ctx, &SubscribeRequest{
        Topic:         "work",
        ConsumerID:    c.getInstanceID(),   // Unique per instance
        ConsumerGroup: "batch-workers",     // Shared across instances
    })
    if err != nil {
        return err
    }
    
    for {
        msg := <-sub.Messages()
        if err := c.processMessage(ctx, msg); err != nil {
            // Handle error; may retry
            continue
        }
        sub.Acknowledge(msg.ID)  // After successful processing
    }
}
```

**Consumer rules (all languages):**
- Specify consumer_group/deliver_group for load balancing (shared across all instances)
- Specify consumer_id/durable_name unique per instance (for resumption)
- Acknowledge ONLY after successful processing (at-least-once semantics)
- Don't acknowledge on parse/processing errors (let broker retry)

---

## 6. String Configuration Format

### Antipattern: Multi-Segment Configuration Becomes Wrong Type

**Problem (language-specific):**

Python:
```python
# ❌ WRONG: Parentheses with multiple strings creates tuple
_instructions = (
    "Instruction 1. ",
    "Instruction 2. ",
    "Instruction 3. "
)
type(_instructions)  # <class 'tuple'> — NOT a string!
api_call(prompt=_instructions)  # Type error: expected str, got tuple
```

JavaScript:
```javascript
// ❌ WRONG: Array instead of concatenated string
const _instructions = [
    "Instruction 1. ",
    "Instruction 2. ",
    "Instruction 3. "
];
typeof _instructions  // "object" (array) — NOT a string!
api.call({prompt: _instructions})  // Type error
```

Go:
```go
// ❌ WRONG: Slice instead of concatenated string
instructions := []string{
    "Instruction 1. ",
    "Instruction 2. ",
    "Instruction 3. ",
}
// When passed to API expecting string, type error
```

### Pattern: Explicit String Concatenation

**Language-independent principle:**
```
Configuration is a single string.
Build it by explicitly concatenating segments.
Verify the result is a string before passing to API.
```

**Language-specific patterns:**

Python (implicit concatenation in parentheses):
```python
# ✅ RIGHT: Adjacent string literals concatenate automatically
_instructions = (
    "You are a data analyzer. "
    "Review the input carefully. "
    "Return JSON with summary and priority. "
    "Priority must be one of: 'low', 'medium', 'high', 'critical'. "
    "\n"
    "Example: {\"summary\": \"ok\", \"priority\": \"medium\"}"
)
type(_instructions)  # <class 'str'> ✅
api_call(prompt=_instructions)  # Works ✅
```

JavaScript (explicit concatenation):
```javascript
// ✅ RIGHT: Use + operator or template strings
const _instructions = 
    "You are a data analyzer. " +
    "Review the input carefully. " +
    "Return JSON with summary and priority. " +
    "Priority must be one of: 'low', 'medium', 'high', 'critical'. " +
    "\n" +
    'Example: {"summary": "ok", "priority": "medium"}';

// Or template string:
const _instructions = `
You are a data analyzer.
Review the input carefully.
Return JSON with summary and priority.
Priority must be one of: 'low', 'medium', 'high', 'critical'.

Example: {"summary": "ok", "priority": "medium"}
`;

typeof _instructions  // "string" ✅
api.call({prompt: _instructions})  // Works ✅
```

Go (explicit concatenation):
```go
// ✅ RIGHT: Use + operator or strings package
var instructions string
instructions = "You are a data analyzer. " +
    "Review the input carefully. " +
    "Return JSON with summary and priority. " +
    "Priority must be one of: 'low', 'medium', 'high', 'critical'. " +
    "\nExample: {\"summary\": \"ok\", \"priority\": \"medium\"}"

// Or using strings.Builder:
var sb strings.Builder
sb.WriteString("You are a data analyzer.\n")
sb.WriteString("Review the input carefully.\n")
// ...
instructions := sb.String()

// Verify type before passing
api.Call(ctx, &Request{Prompt: instructions})  // Works ✅
```

**Verification checklist:**
```
Before passing configuration to external API:
  [ ] Result is a single string, not array/tuple/list
  [ ] String is non-empty
  [ ] No accidental concatenation errors
  [ ] Type check passes in your language
  [ ] Documentation explains format
```

**Documentation for clarity (all languages):**

```
Config format:
  This configuration is a single multi-line string.
  It is passed directly to the external API.
  Do NOT wrap in array, tuple, or list.
  Do NOT split into segments unless explicitly concatenating to one string.
  
  Example usage:
    api_call(prompt=config_prompt)        # ✅ Pass as string
    api_call(prompt=[config_prompt])      # ❌ Don't wrap in array
    api_call(prompts=[prompt1, prompt2])  # ❌ Different parameter entirely
```

---

## Checklist: Integration Patterns

Before integrating any external API or SDK:

- [ ] **SDK method choice documented** (why chat vs generate, why subscribe vs fetch)
- [ ] **Consumer identity semantics** (durable_name unique per instance, deliver_group shared)
- [ ] **Response parsing is defensive** (try-except, .get() with defaults, fallbacks)
- [ ] **Configuration values match schema** (exact enum values, no abbreviations)
- [ ] **Publisher is stateless** (no routing logic, generic topic)
- [ ] **Consumer specifies queue group** (deliver_group for load balancing)
- [ ] **String formats verified** (config is string, not tuple)
- [ ] **Response format tested** (valid JSON, missing fields, non-JSON responses)
- [ ] **Ack behavior correct** (acknowledge only after processing for at-least-once)

---

## Testing Strategy

For each integration, write tests covering:

```python
def test_integration_happy_path():
    """SDK method works with valid input"""
    # Verify response structure matches expectations
    
def test_integration_missing_fields():
    """Missing optional fields handled gracefully"""
    # Verify fallback values
    
def test_integration_invalid_format():
    """Non-JSON or unexpected format handled"""
    # Verify fallback behavior
    
def test_integration_constraint_alignment():
    """Response values pass database constraints"""
    # Verify enum values match database CHECK constraints
    
def test_consumer_identity():
    """Consumer group semantics correct"""
    # Verify durable_name is unique per instance
    # Verify deliver_group is shared across instances
```

---

## Summary: Integration Decision Tree

```
Choosing an API method?
  → Understand use case (role-based? multi-turn? simple completion?)
  → Choose method that supports your use case
  → Document why

Setting up consumer group?
  → Use unique durable_name per instance (identity)
  → Use shared deliver_group for group membership
  → Test per-instance resumption

Parsing external response?
  → Wrap in try-except
  → Use .get() not [] for dict access
  → Provide sensible defaults
  → Handle non-JSON responses

Configuring external API?
  → List all constrained fields
  → Match database schema exactly (not abbreviations)
  → Provide example response with exact values
  → Test for constraint violations

Publishing work?
  → Stateless: just send to topic
  → No routing logic
  → Let broker handle distribution

Consuming work?
  → Specify deliver_group for load balancing
  → Specify durable_name for per-instance identity
  → Acknowledge after processing (at-least-once)
```

