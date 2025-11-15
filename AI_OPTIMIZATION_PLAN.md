# AI Assistant Token Optimization & Enhancement Plan

## Overview
This document outlines a comprehensive, long-term plan to optimize token usage, improve quality, and implement professional-grade enhancements for the Nerco AI Assistant.

**Branch:** `feature/ai-token-optimization`  
**Goal:** Reduce token usage by 60-80% while maintaining or improving quality, and add professional features like RAG, token tracking, and smart context management.

---

## Current State Analysis

### Token Consumption Breakdown
- **System Prompt:** 15,000-50,000 tokens (node types + workspace)
- **Chat History:** 5,000-30,000 tokens (shared across models)
- **Function Definitions:** ~216 tokens
- **Total per Request:** 20,000-100,000 tokens
- **No Token Tracking:** Users can't see usage
- **No Cost Visibility:** No cost estimates

### Quality Requirements
- ✅ Must preserve flow quality (no degradation)
- ✅ Must handle flow updates correctly (full node state needed)
- ✅ Must maintain conversation context
- ✅ Must support multi-model usage efficiently

---

## Optimization Strategy: Phased Approach

### Phase 0: Foundation & Measurement (Week 1-2)
**Goal:** Establish baseline metrics and tracking infrastructure

#### Tasks:
1. **Token Tracking System**
   - Extract `usage.prompt_tokens`, `usage.completion_tokens`, `usage.total_tokens` from API responses
   - Store per-request token usage
   - Calculate session totals
   - Track per-model/provider usage

2. **Metrics Dashboard**
   - Display token usage in UI (current request, session total)
   - Show cost estimates (if pricing data available)
   - Add token usage indicator in chat interface

3. **Profiling & Baseline**
   - Log token usage per request type (question, create flow, update flow)
   - Track average tokens per operation
   - Identify token consumption patterns
   - Establish quality baseline metrics

4. **A/B Testing Framework**
   - Set up framework to compare optimizations
   - Track quality metrics (flow success rate, user satisfaction)
   - Monitor for quality degradation

**Deliverables:**
- Token tracking system implemented
- UI showing token usage
- Baseline metrics documented
- A/B testing framework ready

**Success Criteria:**
- Users can see token usage
- Baseline metrics established
- No quality degradation

---

### Phase 1: Safe Optimizations (Week 3-4)
**Goal:** Implement optimizations with zero quality risk

#### 1.1 Per-Model Chat History
**Impact:** 50-70% reduction in cross-model token waste

- Separate `chatHistory` storage per `provider+model+apiKeyId` combination
- Switch history when user changes model
- Clear history option per model
- Preserve history across sessions (localStorage)

**Implementation:**
```javascript
// Structure: chatHistory[provider][model][apiKeyId] = [...]
// Benefits: No cross-model confusion, cleaner context
```

#### 1.2 System Prompt Caching
**Impact:** 30-50% reduction in system prompt tokens

- Cache node type definitions (refresh on palette changes)
- Cache workspace summaries (refresh on flow changes)
- Cache system prompt templates
- Invalidate cache on relevant events

**Implementation:**
```javascript
// Cache key: hash of nodeTypes + workspace state
// Refresh triggers: palette changes, flow modifications
```

#### 1.3 Conditional Context Inclusion
**Impact:** 40-60% reduction for question-only requests

- **Questions:** Minimal context (node types summary only)
- **Flow Creation:** Node types + workspace summary
- **Flow Updates:** Full workspace (required for quality)

**Implementation:**
```javascript
// Detect intent from user message
// Include context based on operation type
```

**Deliverables:**
- Per-model history working
- System prompt caching implemented
- Conditional context based on operation type
- Quality maintained (A/B tested)

**Success Criteria:**
- 40-60% token reduction for questions
- No quality degradation
- Users can switch models without confusion

---

### Phase 2: Smart Context Management (Week 5-7)
**Goal:** Intelligent context optimization with quality preservation

#### 2.1 Workspace Context Optimization
**Impact:** 30-50% reduction in workspace tokens

- **For Questions:** Send summary (node count, types, names)
- **For Flow Creation:** Send minimal node info (id, type, name, position)
- **For Flow Updates:** Send full node state (required)

**Implementation:**
```javascript
// Compress workspace based on operation type
// Preserve full state only when needed
```

#### 2.2 Message Summarization
**Impact:** 50-70% reduction in history tokens for long conversations

- Keep last 5-10 messages verbatim
- Summarize older messages (>10 messages old)
- Preserve key information (flows created, errors, decisions)
- Store summaries in history

**Implementation:**
```javascript
// Summarize old messages when history > threshold
// Preserve: flows, errors, important decisions
// Compress: general conversation
```

#### 2.3 Intelligent Truncation
**Impact:** Better context window utilization

- Prioritize important messages (flows, errors)
- Smart truncation (not just oldest first)
- Progressive summarization as history grows

**Deliverables:**
- Workspace compression working
- Message summarization implemented
- Smart truncation logic
- Quality maintained

**Success Criteria:**
- 50-70% token reduction in long conversations
- Important context preserved
- No quality degradation

---

### Phase 3: RAG Implementation (Week 8-12)
**Goal:** Implement Retrieval Augmented Generation for node types and patterns

#### 3.1 Knowledge Base Construction
**Components:**
- Node type documentation (from registry)
- Node property schemas
- Common flow patterns
- Best practices
- Example flows
- Troubleshooting guides

**Storage:**
- Vector database (e.g., Chroma, Pinecone, or in-memory with embeddings)
- Embeddings for semantic search
- Metadata for filtering

#### 3.2 Retrieval System
**Features:**
- Semantic search for relevant node types
- Retrieve only relevant documentation
- Include relevant examples and patterns
- Filter by operation type (create vs update)

**Implementation:**
```javascript
// On request:
// 1. Analyze user query (intent, keywords)
// 2. Retrieve relevant node types (semantic search)
// 3. Retrieve relevant examples/patterns
// 4. Include only retrieved info in context
```

#### 3.3 Integration
- Replace full node type list with retrieved subset
- Include relevant examples in context
- Add pattern suggestions
- Maintain quality while reducing tokens

**Deliverables:**
- Knowledge base built
- Retrieval system working
- RAG integrated into AI service
- Significant token reduction

**Success Criteria:**
- 60-80% reduction in node type tokens
- Quality maintained or improved
- Relevant examples included

---

### Phase 4: Advanced Optimizations (Week 13-16)
**Goal:** Fine-tuning and advanced techniques

#### 4.1 Semantic Compression
- Use embeddings to identify redundant information
- Remove duplicate context
- Keep unique information only

#### 4.2 Progressive Context Loading
- Start with minimal context
- If AI needs more, include in follow-up
- Reduce initial token usage

#### 4.3 Fine-Tuning (Long-term)
- Fine-tune smaller model on Node-RED tasks
- Reduces need for large context
- Better domain understanding
- Lower costs

**Deliverables:**
- Semantic compression working
- Progressive loading implemented
- Fine-tuning research completed

---

## Implementation Details

### File Structure
```
packages/node_modules/@node-red/editor-client/src/js/ui/
├── chatbot.js (main UI)
├── openai-service.js (AI service)
├── ai-models.js (model registry)
├── ai-token-tracker.js (NEW - token tracking)
├── ai-context-manager.js (NEW - context optimization)
├── ai-rag-service.js (NEW - RAG implementation)
└── ai-history-manager.js (NEW - per-model history)
```

### Key Components

#### 1. Token Tracker (`ai-token-tracker.js`)
```javascript
// Responsibilities:
// - Extract token usage from API responses
// - Store per-request and session totals
// - Calculate costs
// - Display in UI
```

#### 2. Context Manager (`ai-context-manager.js`)
```javascript
// Responsibilities:
// - Conditional context inclusion
// - Workspace compression
// - System prompt caching
// - Context optimization based on operation type
```

#### 3. History Manager (`ai-history-manager.js`)
```javascript
// Responsibilities:
// - Per-model history storage
// - History switching
// - Message summarization
// - Smart truncation
```

#### 4. RAG Service (`ai-rag-service.js`)
```javascript
// Responsibilities:
// - Knowledge base management
// - Semantic search
// - Retrieval of relevant context
// - Example/pattern inclusion
```

---

## Quality Assurance

### Testing Strategy
1. **Unit Tests:** Each optimization component
2. **Integration Tests:** End-to-end flow creation/updates
3. **A/B Tests:** Compare optimized vs. baseline
4. **Quality Metrics:**
   - Flow success rate
   - User satisfaction
   - Error rate
   - Token usage reduction

### Quality Gates
- ✅ No quality degradation (A/B tested)
- ✅ Flow creation success rate maintained
- ✅ Flow update accuracy maintained
- ✅ User satisfaction maintained or improved

---

## Success Metrics

### Token Usage
- **Target:** 60-80% reduction in average tokens per request
- **Baseline:** 20,000-100,000 tokens/request
- **Target:** 5,000-20,000 tokens/request

### Quality
- **Flow Success Rate:** Maintain ≥95%
- **User Satisfaction:** Maintain or improve
- **Error Rate:** Maintain or reduce

### User Experience
- **Token Visibility:** Users see usage
- **Cost Transparency:** Users see cost estimates
- **Model Switching:** Seamless with separate history

---

## Timeline

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| Phase 0 | Week 1-2 | Token tracking, baseline metrics |
| Phase 1 | Week 3-4 | Per-model history, caching, conditional context |
| Phase 2 | Week 5-7 | Workspace optimization, summarization |
| Phase 3 | Week 8-12 | RAG implementation |
| Phase 4 | Week 13-16 | Advanced optimizations |

**Total Duration:** ~16 weeks (4 months)

---

## Risk Mitigation

### Quality Risks
- **Risk:** Optimizations reduce quality
- **Mitigation:** A/B test everything, measure quality metrics, rollback capability

### Complexity Risks
- **Risk:** Over-engineering
- **Mitigation:** Start simple, iterate, measure impact

### Performance Risks
- **Risk:** RAG adds latency
- **Mitigation:** Cache embeddings, optimize retrieval, measure performance

---

## Future Enhancements

### Long-term (6+ months)
1. **Fine-tuning:** Custom model for Node-RED
2. **Multi-modal:** Support for flow diagrams
3. **Collaborative:** Shared knowledge base
4. **Analytics:** Usage analytics and insights

---

## Notes

- All optimizations must be A/B tested
- Quality is non-negotiable
- Incremental approach (one phase at a time)
- Measure everything
- User feedback is critical

---

## Getting Started

1. Review and approve this plan
2. Set up development environment
3. Start with Phase 0 (Foundation & Measurement)
4. Iterate based on results

---

**Last Updated:** [Current Date]  
**Status:** Planning Phase  
**Next Steps:** Begin Phase 0 implementation

