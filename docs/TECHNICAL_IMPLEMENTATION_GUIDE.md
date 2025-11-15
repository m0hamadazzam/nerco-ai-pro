# Technical Implementation Guide: AI Token Optimization

## Architecture Overview

### Current Architecture
```
chatbot.js → openai-service.js → API Provider
     ↓              ↓
chatHistory    systemPrompt
```

### Optimized Architecture
```
chatbot.js → ai-history-manager.js → ai-context-manager.js → ai-token-tracker.js
     ↓              ↓                        ↓                      ↓
perModelHistory  contextOptimization    tokenTracking         openai-service.js
                                                                    ↓
                                                              ai-rag-service.js
                                                                    ↓
                                                              API Provider
```

---

## Phase 0: Foundation & Measurement

### 1. Token Tracker Implementation

#### File: `ai-token-tracker.js`
```javascript
RED.aiTokenTracker = (function() {
    var sessionStats = {
        totalTokens: 0,
        totalCost: 0,
        requestCount: 0,
        perModel: {},
        perOperation: {}
    };

    function extractTokenUsage(apiResponse, provider) {
        // Extract from different provider formats
        if (provider === 'openai') {
            return {
                promptTokens: apiResponse.usage?.prompt_tokens || 0,
                completionTokens: apiResponse.usage?.completion_tokens || 0,
                totalTokens: apiResponse.usage?.total_tokens || 0
            };
        }
        // Add other providers...
    }

    function trackUsage(usage, model, operation, provider) {
        // Track per request
        // Track per session
        // Track per model
        // Track per operation type
    }

    function calculateCost(tokens, model, provider) {
        // Use pricing data from ai-models.js
        // Calculate cost based on model pricing
    }

    return {
        track: trackUsage,
        getSessionStats: function() { return sessionStats; },
        reset: function() { sessionStats = {...}; }
    };
})();
```

#### Integration Points:
- `openai-service.js`: Extract usage after API call
- `chatbot.js`: Display in UI

#### UI Components:
- Token counter in chat header
- Session stats in settings
- Per-request tooltip

---

### 2. Metrics Dashboard

#### UI Location: Chat interface header
```html
<div class="red-ui-chatbot-token-stats">
    <span class="token-counter">2,450 tokens</span>
    <span class="session-total">15,230 tokens</span>
    <span class="cost-estimate">~$0.05</span>
</div>
```

#### Storage:
- Session stats in memory
- Persistent stats in localStorage
- Optional: Backend analytics

---

## Phase 1: Safe Optimizations

### 1.1 Per-Model History Manager

#### File: `ai-history-manager.js`
```javascript
RED.aiHistoryManager = (function() {
    var histories = {}; // { "provider:model:apiKeyId": [...] }
    var currentKey = null;

    function getHistoryKey(provider, model, apiKeyId) {
        return `${provider}:${model}:${apiKeyId}`;
    }

    function getHistory(provider, model, apiKeyId) {
        var key = getHistoryKey(provider, model, apiKeyId);
        if (!histories[key]) {
            histories[key] = [];
            // Load from localStorage if exists
            loadFromStorage(key);
        }
        return histories[key];
    }

    function switchHistory(provider, model, apiKeyId) {
        var newKey = getHistoryKey(provider, model, apiKeyId);
        if (currentKey !== newKey) {
            // Save current history
            if (currentKey) {
                saveToStorage(currentKey);
            }
            // Load new history
            currentKey = newKey;
            return getHistory(provider, model, apiKeyId);
        }
        return histories[currentKey] || [];
    }

    function addMessage(role, content) {
        if (currentKey) {
            histories[currentKey].push({ role, content });
            saveToStorage(currentKey);
        }
    }

    function clearHistory(provider, model, apiKeyId) {
        var key = getHistoryKey(provider, model, apiKeyId);
        delete histories[key];
        localStorage.removeItem(`ai_history_${key}`);
    }

    function loadFromStorage(key) {
        var stored = localStorage.getItem(`ai_history_${key}`);
        if (stored) {
            try {
                histories[key] = JSON.parse(stored);
            } catch (e) {
                histories[key] = [];
            }
        }
    }

    function saveToStorage(key) {
        if (histories[key]) {
            localStorage.setItem(`ai_history_${key}`, JSON.stringify(histories[key]));
        }
    }

    return {
        getHistory: getHistory,
        switchHistory: switchHistory,
        addMessage: addMessage,
        clearHistory: clearHistory
    };
})();
```

#### Integration:
- `chatbot.js`: Use instead of `chatHistory` array
- Switch history when model changes
- Clear history button in UI

---

### 1.2 System Prompt Caching

#### File: `ai-context-manager.js` (partial)
```javascript
RED.aiContextManager = (function() {
    var cache = {
        nodeTypes: null,
        nodeTypesHash: null,
        workspaceSummary: null,
        workspaceHash: null,
        systemPrompt: null
    };

    function getNodeTypesHash(nodeTypes) {
        // Create hash of node types for cache invalidation
        return JSON.stringify(Object.keys(nodeTypes).sort());
    }

    function getWorkspaceHash(workspace) {
        // Create hash of workspace for cache invalidation
        return JSON.stringify(workspace.map(n => n.id).sort());
    }

    function getCachedNodeTypes(currentNodeTypes) {
        var currentHash = getNodeTypesHash(currentNodeTypes);
        if (cache.nodeTypes && cache.nodeTypesHash === currentHash) {
            return cache.nodeTypes;
        }
        // Cache miss - update cache
        cache.nodeTypes = currentNodeTypes;
        cache.nodeTypesHash = currentHash;
        return cache.nodeTypes;
    }

    function invalidateCache() {
        cache = {
            nodeTypes: null,
            nodeTypesHash: null,
            workspaceSummary: null,
            workspaceHash: null,
            systemPrompt: null
        };
    }

    // Listen for palette changes
    RED.events.on('palette:changed', invalidateCache);
    RED.events.on('workspace:changed', function() {
        cache.workspaceSummary = null;
        cache.workspaceHash = null;
    });

    return {
        getCachedNodeTypes: getCachedNodeTypes,
        invalidateCache: invalidateCache
    };
})();
```

---

### 1.3 Conditional Context Inclusion

#### Operation Detection:
```javascript
function detectOperationType(userMessage, conversationHistory) {
    // Simple keyword-based detection (can be improved with ML)
    var message = userMessage.toLowerCase();
    
    if (message.includes('create') || message.includes('make') || 
        message.includes('build') || message.includes('generate')) {
        return 'create_flow';
    }
    
    if (message.includes('update') || message.includes('modify') || 
        message.includes('change') || message.includes('edit')) {
        return 'update_flow';
    }
    
    // Check if there are nodes in workspace
    if (nodesInWorkspace && nodesInWorkspace.length > 0) {
        // If workspace has nodes, might be update
        if (message.includes('add') || message.includes('remove') || 
            message.includes('connect')) {
            return 'update_flow';
        }
    }
    
    return 'question';
}
```

#### Context Builder:
```javascript
function buildContext(operationType, nodeTypes, workspace) {
    switch(operationType) {
        case 'question':
            return {
                nodeTypes: summarizeNodeTypes(nodeTypes), // Just names and descriptions
                workspace: null // Not needed for questions
            };
        
        case 'create_flow':
            return {
                nodeTypes: nodeTypes, // Full definitions
                workspace: summarizeWorkspace(workspace) // Summary to avoid conflicts
            };
        
        case 'update_flow':
            return {
                nodeTypes: nodeTypes, // Full definitions
                workspace: workspace // Full state required
            };
    }
}
```

---

## Phase 2: Smart Context Management

### 2.1 Workspace Compression

#### Summarization Functions:
```javascript
function summarizeWorkspace(workspace) {
    if (!workspace || workspace.length === 0) {
        return null;
    }
    
    return {
        nodeCount: workspace.length,
        nodeTypes: [...new Set(workspace.map(n => n.type))],
        nodes: workspace.map(n => ({
            id: n.id,
            type: n.type,
            name: n.name || n.id,
            x: n.x,
            y: n.y
            // Minimal info for conflict detection
        }))
    };
}

function summarizeNodeTypes(nodeTypes) {
    // Return only essential info for questions
    return Object.keys(nodeTypes).map(type => ({
        type: type,
        name: nodeTypes[type].name || type,
        description: nodeTypes[type].help || '',
        // Exclude full schema, examples, etc.
    }));
}
```

---

### 2.2 Message Summarization

#### Summarization Strategy:
```javascript
function shouldSummarize(history, maxMessages = 10) {
    return history.length > maxMessages;
}

function summarizeOldMessages(history, keepRecent = 10) {
    if (history.length <= keepRecent) {
        return history;
    }
    
    var recent = history.slice(-keepRecent);
    var old = history.slice(0, -keepRecent);
    
    // Summarize old messages
    var summary = createSummary(old);
    
    return [
        { role: 'system', content: `Previous conversation summary: ${summary}` },
        ...recent
    ];
}

function createSummary(messages) {
    // Extract key information:
    // - Flows created
    // - Errors encountered
    // - Important decisions
    // - User preferences
    
    var summary = [];
    
    messages.forEach(msg => {
        if (msg.role === 'assistant' && msg.isFlow) {
            summary.push(`Created flow: ${msg.description}`);
        }
        if (msg.role === 'assistant' && msg.isUpdate) {
            summary.push(`Updated flow: ${msg.description}`);
        }
        // Add more extraction logic...
    });
    
    return summary.join('; ');
}
```

---

## Phase 3: RAG Implementation

### 3.1 Knowledge Base Structure

#### Data Sources:
1. **Node Type Registry** (from `RED.nodes.registry`)
   - Node definitions
   - Property schemas
   - Help text
   - Examples

2. **Common Patterns** (curated)
   - Common flow patterns
   - Best practices
   - Example flows

3. **Documentation** (external)
   - Node-RED docs
   - Community examples

#### Storage Format:
```javascript
{
    id: "node-type-inject",
    type: "node_type",
    content: "Inject node documentation...",
    metadata: {
        nodeType: "inject",
        category: "common",
        tags: ["trigger", "input"]
    },
    embedding: [0.123, 0.456, ...] // Vector embedding
}
```

---

### 3.2 Embedding Generation

#### Options:
1. **Client-side:** Use a lightweight embedding model (e.g., Transformers.js)
2. **Server-side:** Generate embeddings on backend
3. **Pre-computed:** Generate embeddings at build time

#### Implementation:
```javascript
// Using Transformers.js (client-side)
import { pipeline } from '@xenova/transformers';

var embedder = null;

async function getEmbedder() {
    if (!embedder) {
        embedder = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');
    }
    return embedder;
}

async function generateEmbedding(text) {
    var model = await getEmbedder();
    var result = await model(text);
    return Array.from(result.data);
}
```

---

### 3.3 Retrieval System

#### Semantic Search:
```javascript
RED.aiRAGService = (function() {
    var knowledgeBase = [];
    var embeddings = [];

    function initialize() {
        // Load knowledge base
        // Generate or load embeddings
    }

    function search(query, topK = 5) {
        // 1. Generate query embedding
        var queryEmbedding = generateEmbedding(query);
        
        // 2. Calculate similarity (cosine similarity)
        var similarities = embeddings.map((emb, idx) => ({
            index: idx,
            similarity: cosineSimilarity(queryEmbedding, emb)
        }));
        
        // 3. Sort by similarity
        similarities.sort((a, b) => b.similarity - a.similarity);
        
        // 4. Return top K results
        return similarities.slice(0, topK).map(s => knowledgeBase[s.index]);
    }

    function retrieveRelevantContext(userMessage, operationType) {
        // Search for relevant node types
        var nodeTypeResults = search(userMessage + ' node types', 10);
        
        // Search for relevant examples
        var exampleResults = search(userMessage + ' examples', 5);
        
        // Filter by operation type if needed
        if (operationType === 'create_flow') {
            // Include more examples
        }
        
        return {
            nodeTypes: nodeTypeResults.filter(r => r.type === 'node_type'),
            examples: exampleResults.filter(r => r.type === 'example'),
            patterns: exampleResults.filter(r => r.type === 'pattern')
        };
    }

    return {
        initialize: initialize,
        retrieveRelevantContext: retrieveRelevantContext
    };
})();
```

---

### 3.4 Integration with AI Service

#### Modified System Prompt:
```javascript
function getSystemPrompt(retrievedContext, nodesInWorkspace) {
    return {
        role: "system",
        content: `You are a Node-RED assistant...

RELEVANT NODE TYPES (based on your query):
${formatRetrievedNodeTypes(retrievedContext.nodeTypes)}

RELEVANT EXAMPLES:
${formatRetrievedExamples(retrievedContext.examples)}

WORKSPACE CONTEXT:
${formatWorkspace(nodesInWorkspace)}

...`
    };
}
```

---

## Testing Strategy

### Unit Tests
```javascript
// Test token tracking
test('tracks token usage correctly', () => {
    // ...
});

// Test history management
test('switches history when model changes', () => {
    // ...
});

// Test context optimization
test('includes minimal context for questions', () => {
    // ...
});
```

### Integration Tests
```javascript
// Test end-to-end flow creation
test('creates flow with optimized context', async () => {
    // ...
});

// Test flow update with full context
test('updates flow correctly with full workspace', async () => {
    // ...
});
```

### A/B Tests
```javascript
// Compare optimized vs baseline
test('optimized version maintains quality', async () => {
    var baseline = await createFlowBaseline();
    var optimized = await createFlowOptimized();
    
    assert.equal(baseline.success, optimized.success);
    assert(optimized.tokens < baseline.tokens * 0.5); // 50% reduction
});
```

---

## Migration Plan

### Step 1: Add New Components (Non-breaking)
- Add new files alongside existing code
- No changes to existing functionality

### Step 2: Feature Flags
- Add feature flags for each optimization
- Allow gradual rollout

### Step 3: Gradual Migration
- Migrate one component at a time
- Test thoroughly before next step

### Step 4: Cleanup
- Remove old code
- Optimize and refactor

---

## Performance Considerations

### Caching Strategy
- Cache embeddings (don't regenerate)
- Cache retrieved context (short TTL)
- Cache system prompts (invalidate on changes)

### Memory Management
- Limit history size per model
- Clean up old cache entries
- Use efficient data structures

### Latency
- RAG retrieval should be <100ms
- Use async operations
- Pre-compute when possible

---

## Monitoring & Analytics

### Metrics to Track
- Token usage per request
- Token reduction percentage
- Quality metrics (flow success rate)
- User satisfaction
- Performance (latency)

### Logging
- Log all token usage
- Log context optimization decisions
- Log RAG retrieval results
- Log quality metrics

---

## Rollback Plan

### Feature Flags
- Each optimization behind a flag
- Can disable individually
- Quick rollback capability

### Version Control
- Tag releases
- Keep old code for rollback
- Document breaking changes

---

## Documentation

### User Documentation
- Token usage explanation
- Cost estimation guide
- History management guide

### Developer Documentation
- Architecture overview
- Component documentation
- API documentation
- Testing guide

---

**Last Updated:** [Current Date]  
**Status:** Technical Specification  
**Next Steps:** Begin Phase 0 implementation

