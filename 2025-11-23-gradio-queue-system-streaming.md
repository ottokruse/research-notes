---
title: Gradio Queue System and Streaming Connection Stability
date: 2025-11-23
tags: [gradio, streaming, websockets, sse, python, connection-management]
---

# Gradio Queue System and Streaming Connection Stability

## Executive Summary

Research into Gradio's queue system reveals that connection drops during streaming (especially with inactive browser tabs) are caused by a combination of:

1. **Browser tab throttling** - Browsers aggressively throttle inactive tabs, causing SSE data loss
2. **Frontend rendering bottleneck** - UI can't keep up with rapid backend updates
3. **No backpressure mechanism** - Backend continues generating updates regardless of frontend capacity
4. **Limited connection recovery** - SSE doesn't support automatic reconnection with event replay

**Key Solution**: Throttle backend yields to 10-20 updates/second and batch updates before yielding.

---

## 1. Gradio Queue System Overview

### What is Gradio's Queue System?

Every Gradio app includes a **built-in queuing system** that manages concurrent user requests using **Server-Side Events (SSE)**.

**Key characteristics**:
- Uses SSE instead of traditional HTTP POST requests
- Automatically handles concurrent users (scales to thousands)
- Provides real-time progress updates and ETAs
- Prevents timeouts (SSE connections don't timeout like POST requests ~1 minute)
- Controls resource usage (prevents OOM errors with heavy ML operations)

### Default Behavior

- **Each event listener gets its own queue** (e.g., `button.click()`)
- **"Single-function-single-worker" model**: Default concurrency limit is 1
- Workers process requests for their assigned function type only
- This prevents multiple workers from loading the same ML model simultaneously

### Configuration Options

#### Global Configuration (`Blocks.queue()`)

```python
demo.queue(
    default_concurrency_limit=10,  # Default for all event listeners
    max_size=100                   # Max events in queue at any moment
)
```

- `default_concurrency_limit`: Default is 1, set higher for lightweight operations
- `max_size`: Queue capacity (None = unlimited). Users see "queue full" when exceeded

#### Per-Event Configuration

```python
button.click(
    fn, 
    inputs, 
    outputs, 
    concurrency_limit=5,        # Max concurrent executions for this event
    concurrency_id="gpu_queue", # Share queue across multiple events
    queue=True                   # Use queue (default) or bypass
)
```

**When to use `queue=False`**:
- Extremely fast functions (simple calculations, UI updates)
- No heavy resource usage
- Immediate response critical for UX

**When to use `queue=True`** (default):
- Time-consuming operations (ML inference, data processing)
- Streaming operations
- Need progress bars or status updates
- Resource control required

---

## 2. Connection Management and Inactive Browser Tabs

### Critical Issue: Browser Tab Throttling

When a browser tab becomes **inactive** (backgrounded), browsers throttle JavaScript execution and network activity for performance optimization. This directly impacts Gradio's SSE connections.

**Issue #9243 - Data Loss When Tab Becomes Inactive**:
- **Problem**: Gradio streaming pauses when tab is inactive
- **Impact**: Data transmitted while inactive is **lost permanently**
- **Example**: Switching tabs for 12 seconds = 12 seconds of streamed data lost
- **Root Cause**: Browser throttling, not a Gradio bug

**Browser Behavior**:
- Chrome limits to **4-6 SSE connections per domain**
- Background tabs get throttled timers (minimum 1 second intervals)
- Network requests are deprioritized
- SSE message processing is delayed

### Heartbeat Mechanism

Gradio implements a **heartbeat system** to maintain SSE connections:

```python
# From queueing.py
progress_update_sleep_when_free = 0.01  # Check every 10ms
```

**How it works**:
- Periodic heartbeat messages keep connections alive
- Client repeatedly connects to check application status
- Endpoint: `/gradio_api/heartbeat/{session_id}`

**Limitations**:
- **Issue #6239**: SSE does NOT detect browser closing as disconnection
- Sessions remain in queue even after tabs/browsers close
- No native browser API to reliably detect inactive tabs for SSE

**Known Problems**:
- **Issue #6319**: `KeyError: 'heartbeat'` after ~15-16 seconds of processing
- **Issue #9999**: Heartbeat can timeout after 120 seconds on cloud platforms (AWS App Runner)

### Timeout Settings

**No Direct Timeout Configuration**:
- Gradio doesn't expose SSE timeout configuration to users
- Timeouts controlled by:
  - Reverse proxies (nginx, AWS App Runner: 120s default)
  - Browser connection limits
  - Network infrastructure

**SSE Advantages**:
- Don't timeout like POST requests (typically 1 minute)
- Allow real-time ETA updates during processing
- Support multiple server-to-client updates

### Known Issues with Inactive Tabs

**Issue #11248 - SSE Reconnection**:
- Gradio doesn't implement `Last-Event-ID` for connection recovery
- Network timeouts cause unrecoverable connection loss
- EventSource auto-reconnects but can't resume from last event

**Issue #11458 - SSE Deprecation Discussion**:
- MCP (Model Context Protocol) deprecated SSE in favor of Streamable HTTP
- Reasons: Long-lived connections strain infrastructure, poor HTTP/2 support, difficult recovery

---

## 3. High-Frequency Updates and Streaming Behavior

### How Gradio Handles Rapid Updates

**Streaming Mechanism**:
- Uses generator functions with `yield`
- Each yield sends a separate SSE message
- **No inherent backend throttling** of yields
- Uses thread pool (default max 40 threads from FastAPI)

**Built-in Frontend Throttling (Gradio 4.x+)**:
- **PR #7084 (January 2024)**: Auto-throttles to **20 updates/second (50ms intervals)**
- Prevents browser event stack overflow
- Improved performance by 10x in some cases
- SSE events have higher priority than WebSocket, causing lag if too fast

### Frontend Rendering Bottleneck

**Issue #5914 - UI Can't Keep Up**:
- Even when backend yields rapidly, UI displays at coarser granularity
- **Root cause**: Number of DOM elements significantly impacts rendering
- With 300+ unused textboxes, streaming becomes significantly slower
- Chrome uses 100% CPU core during rendering
- Backend finishes while UI still catching up

**Issue #7952 - Buffering Behavior**:
- Users yield at token granularity but UI batches/buffers multiple yields
- Backend processes each yield individually
- Frontend decides when to actually render

### No Explicit Backpressure Mechanism

From `queueing.py` source analysis:
- **No built-in throttling** of yields in queue system
- **No backpressure signals** from frontend to backend
- Backend continues processing regardless of frontend rendering speed
- Updates queue up in memory if frontend can't keep up

**Progress updates ARE throttled**:
```python
async def start_progress_updates(self):
    """
    Progress updates can be very frequent, so we check at regular intervals
    and send only the most recent update. Consecutive updates between sends
    will overwrite each other.
    """
```

**Regular yields are NOT throttled**: Sent as fast as generated

### Memory Considerations

```python
# From queueing.py
pending_messages_per_session: LRUCache[str, ThreadQueue] = LRUCache(2000)
```

- Maintains 2000 sessions worth of pending messages
- Each session has event queue, message queue, state objects
- With frequent updates, memory can grow significantly

### gr.State Behavior with Streaming

**How it works**:
- Session-scoped: Each user session has own state
- Persists across yields throughout generator execution
- Deep copied on completion before sending

**Best practice**:
```python
def efficient_streaming_with_state(input, state):
    # Don't accumulate unbounded data
    if len(state.history) > 100:
        state.history = state.history[-50:]  # Keep only recent items
    
    for update in generate_updates(input):
        state.current = update
        yield state.current, state
```

---

## 4. Solutions and Workarounds

### Preventing Connection Drops

#### A. Backend Throttling (Manual)

**Time-Based Throttling**:
```python
import time

class Throttler:
    def __init__(self, min_interval=0.05):  # 50ms = 20 updates/sec
        self.min_interval = min_interval
        self.last_time = 0
    
    def should_yield(self):
        current = time.time()
        if current - self.last_time >= self.min_interval:
            self.last_time = current
            return True
        return False

def streaming_fn(input_text):
    throttler = Throttler(0.05)
    response = ""
    
    for token in generate_tokens(input_text):
        response += token
        if throttler.should_yield():
            yield response
    
    yield response  # Always yield final result
```

#### B. Batch Updates for LLM Agent Traces

```python
def stream_agent_traces(query):
    trace_buffer = []
    last_yield_time = time.time()
    min_yield_interval = 0.05  # 50ms = 20 updates/sec max
    
    for step in agent.run(query):
        trace_buffer.append(step)
        current_time = time.time()
        
        # Yield only every 50ms to avoid overwhelming UI
        if current_time - last_yield_time >= min_yield_interval:
            yield format_traces(trace_buffer)
            last_yield_time = current_time
            trace_buffer = []
    
    # Final yield for remaining traces
    if trace_buffer:
        yield format_traces(trace_buffer)
```

#### C. Token-Level Batching

```python
def chatbot_response(prompt):
    partial_response = ""
    token_buffer = []
    
    for i, token in enumerate(llm_stream(prompt)):
        token_buffer.append(token)
        
        # Yield every 3-5 tokens instead of every token
        if i % 3 == 0 or is_final_token:
            partial_response += ''.join(token_buffer)
            token_buffer = []
            yield partial_response
```

#### D. Smart Batching (Word Boundaries)

```python
def streaming_fn(prompt):
    partial = ""
    
    for chunk in llm_stream(prompt):
        partial += chunk
        
        # Yield at natural word boundaries
        if partial.endswith((' ', '\n', '.', ',', '!', '?')):
            yield partial
    
    yield partial  # Final
```

### Configuration for Stability

#### Nginx Configuration for SSE

```nginx
location / {
    proxy_pass http://gradio_app;
    
    # Essential for SSE
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    
    # Disable buffering
    proxy_buffering off;
    proxy_cache off;
    
    # Timeouts for long-running connections
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    
    # Required headers
    proxy_set_header X-Accel-Buffering no;
    proxy_set_header Cache-Control no-cache;
}
```

#### Queue Configuration Example

```python
with gr.Blocks() as demo:
    # Component setup...
    
    btn.click(
        fn=my_function,
        inputs=inputs,
        outputs=outputs,
        concurrency_limit=5,  # Max 5 concurrent executions
        concurrency_id="gpu_queue"  # Shared queue across functions
    )

# Global queue settings
demo.queue(
    default_concurrency_limit=10,  # Default for all functions
    max_size=100  # Max queue length
).launch()
```

### Connection Recovery Patterns

#### Frontend Monitoring (JavaScript)

```python
import gradio as gr

connection_monitor_js = """
function() {
    let reconnectAttempts = 0;
    const maxReconnects = 5;
    
    window.addEventListener('error', function(e) {
        if (e.message.includes('connection') || e.message.includes('network')) {
            console.log('Connection error detected');
            if (reconnectAttempts < maxReconnects) {
                reconnectAttempts++;
                setTimeout(() => {
                    window.location.reload();
                }, 2000 * reconnectAttempts);  // Exponential backoff
            }
        }
    });
}
"""

with gr.Blocks(js=connection_monitor_js) as demo:
    # Your interface
    pass
```

#### Backend Heartbeat Pattern

```python
import time

def long_running_task(input_data):
    last_heartbeat = time.time()
    
    for i, result in enumerate(process_data(input_data)):
        # Send heartbeat every 30 seconds
        if time.time() - last_heartbeat > 30:
            yield {"status": "heartbeat", "data": None}
            last_heartbeat = time.time()
        
        yield {"status": "data", "data": result}
```

#### Client-Side Retry Logic

```python
from gradio_client import Client
import time

def predict_with_retry(client, *args, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.predict(*args)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
                
                # Recreate client connection
                client = Client(client.src)
            else:
                raise

client = Client("http://your-app-url")
result = predict_with_retry(client, input_data)
```

### Alternative Patterns

#### Background Processing with UI Polling

```python
import threading
import time

class StreamBuffer:
    def __init__(self):
        self.buffer = ""
        self.lock = threading.Lock()
    
    def append(self, text):
        with self.lock:
            self.buffer += text
    
    def get(self):
        with self.lock:
            return self.buffer

def background_generator(prompt, buffer):
    for token in llm_stream(prompt):
        buffer.append(token)

def ui_updater(prompt):
    buffer = StreamBuffer()
    
    # Start background thread
    thread = threading.Thread(target=background_generator, args=(prompt, buffer))
    thread.start()
    
    # Update UI at controlled rate
    while thread.is_alive():
        yield buffer.get()
        time.sleep(0.1)  # Update every 100ms
    
    yield buffer.get()  # Final update
```

### Handling Inactive Tabs

#### Page Visibility API

```python
visibility_monitor_js = """
function() {
    document.addEventListener('visibilitychange', () => {
        if (document.hidden) {
            console.warn('Tab inactive - streaming may be interrupted');
            // Could display warning to user
        } else {
            console.log('Tab active - resuming normal operation');
        }
    });
}
"""

with gr.Blocks(js=visibility_monitor_js) as demo:
    gr.Markdown("""
    ⚠️ **Important**: Keep this tab active during streaming operations.
    Data may be lost if the tab is inactive.
    """)
    # Rest of interface
```

---

## 5. Recommendations for LLM Agent Trace Streaming

Based on the research, here are specific recommendations for streaming LLM agent traces:

### Immediate Actions

1. **Implement Backend Throttling**:
   - Target 10-20 updates/second (50-100ms intervals)
   - Batch multiple trace events before yielding

2. **Reduce UI Complexity**:
   - Minimize number of components in layout
   - Hide unused components dynamically
   - Use conditional rendering

3. **Warn Users About Tab Activity**:
   - Display prominent message about keeping tab active
   - Use Page Visibility API to detect when tab becomes inactive

### Code Example for Agent Traces

```python
import time
import gradio as gr

def stream_agent_traces(query, state):
    """Stream agent traces with intelligent batching."""
    
    if state is None:
        state = {"traces": [], "last_update": time.time()}
    
    trace_batch = []
    last_yield = time.time()
    min_interval = 0.05  # 50ms between yields
    batch_size = 5  # Or yield every 5 traces
    
    for trace in agent.execute(query):
        trace_batch.append(trace)
        current_time = time.time()
        
        # Yield based on time OR batch size
        should_yield = (
            current_time - last_yield >= min_interval or
            len(trace_batch) >= batch_size
        )
        
        if should_yield:
            state["traces"].extend(trace_batch)
            state["last_update"] = current_time
            
            # Format for display (only last N traces if needed)
            display_traces = state["traces"][-100:]  # Keep UI manageable
            
            yield format_traces_output(display_traces), state
            
            last_yield = current_time
            trace_batch = []
    
    # Final yield for any remaining traces
    if trace_batch:
        state["traces"].extend(trace_batch)
        yield format_traces_output(state["traces"][-100:]), state

# Gradio interface
with gr.Blocks() as demo:
    gr.Markdown("""
    # LLM Agent Trace Viewer
    
    ⚠️ **Keep this tab active during execution** - inactive tabs may lose streaming data.
    """)
    
    state = gr.State()
    query_input = gr.Textbox(label="Query")
    traces_output = gr.JSON(label="Agent Traces")
    run_btn = gr.Button("Run Agent")
    
    run_btn.click(
        stream_agent_traces,
        inputs=[query_input, state],
        outputs=[traces_output, state],
        concurrency_limit=3  # Allow 3 concurrent agent runs
    )

demo.queue(
    default_concurrency_limit=3,
    max_size=50
).launch()
```

### Architecture Recommendations

1. **Consider WebSocket Alternative**: For mission-critical streaming where data loss is unacceptable
2. **Server-Side Buffering**: Store traces server-side, allow client to request missing data
3. **Checkpoint System**: Save agent state periodically, allow resume from checkpoint
4. **Polling Fallback**: Implement polling mode for unstable connections

### Testing Strategy

1. Test with browser DevTools network throttling
2. Test with tab switching during active streaming
3. Monitor browser console for errors
4. Test behind nginx/reverse proxy
5. Test on mobile devices (more aggressive throttling)

---

## Key Takeaways

✅ **Gradio 4.x auto-throttles frontend** to 20 updates/sec  
⚠️ **Inactive tabs cause permanent data loss** - fundamental browser behavior  
🔧 **Implement backend throttling** - 50-100ms between yields recommended  
📦 **Batch updates** before yielding - don't yield every token/event  
🚨 **Warn users** to keep tabs active during streaming  
🔄 **No automatic connection recovery** - implement retry logic at app level  
🎯 **UI complexity matters** - minimize components for better streaming performance  
🚀 **Upgrade to Gradio 4.x+** if on older version for automatic frontend throttling

---

## Sources

- [Gradio Official Documentation - Queuing](https://www.gradio.app/guides/queuing)
- [Gradio GitHub Issue #9243 - Data Loss with Inactive Tabs](https://github.com/gradio-app/gradio/issues/9243)
- [Gradio GitHub Issue #6239 - SSE Connection Detection](https://github.com/gradio-app/gradio/issues/6239)
- [Gradio GitHub Issue #11248 - SSE Reconnection Support](https://github.com/gradio-app/gradio/issues/11248)
- [Gradio GitHub Issue #5914 - Frontend Rendering Bottleneck](https://github.com/gradio-app/gradio/issues/5914)
- [Gradio GitHub Issue #7952 - Buffering Behavior](https://github.com/gradio-app/gradio/issues/7952)
- [Gradio GitHub PR #7084 - Frontend Throttling](https://github.com/gradio-app/gradio/pull/7084)
- [Gradio GitHub Issue #6319 - Heartbeat Errors](https://github.com/gradio-app/gradio/issues/6319)
- [Gradio GitHub Issue #9999 - Cloud Platform Timeouts](https://github.com/gradio-app/gradio/issues/9999)
- [Gradio GitHub Issue #11458 - SSE Deprecation Discussion](https://github.com/gradio-app/gradio/issues/11458)
- [Nginx SSE Configuration](https://serverfault.com/questions/801628)
- [MDN - Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)

---

*Research conducted: November 23, 2025*
