# AI Chatbot Design and RAG Implementation

## Overview

This document details the comprehensive design of the AI chatbot system, including the RAG (Retrieval-Augmented Generation) mechanism, secure data retrieval using RBAC, embedding pipeline, and user-role-scoped query responses.

## Architecture Components

### 1. AI Gateway Service

The AI Gateway serves as the central orchestrator for all AI interactions, managing authentication, routing, and response formatting.

```python
class AIGateway:
    def __init__(self):
        self.auth_service = AuthService()
        self.rbac_service = RBACService()
        self.rag_engine = RAGEngine()
        self.context_manager = ContextManager()
        self.action_executor = ActionExecutor()
        self.llm_providers = LLMProviderManager()
    
    async def process_chat_message(self, message: ChatMessage, user_context: UserContext) -> ChatResponse:
        # 1. Authenticate and authorize user
        user = await self.auth_service.validate_token(user_context.token)
        permissions = await self.rbac_service.get_user_permissions(user)
        
        # 2. Parse message intent and extract entities
        intent = await self.parse_intent(message.content)
        entities = await self.extract_entities(message.content)
        
        # 3. Retrieve relevant context using RAG
        context_docs = await self.rag_engine.retrieve_context(
            query=message.content,
            user_permissions=permissions,
            tenant_id=user.tenant_id
        )
        
        # 4. Generate response
        response = await self.generate_response(
            message=message.content,
            context=context_docs,
            user=user,
            intent=intent,
            entities=entities
        )
        
        # 5. Execute actions if requested
        if intent.requires_action:
            action_result = await self.action_executor.execute(
                action=intent.action,
                parameters=entities,
                user=user
            )
            response.actions = action_result
        
        # 6. Log interaction for audit
        await self.log_interaction(user, message, response)
        
        return response
```

### 2. RAG Engine Implementation

The RAG engine combines retrieval and generation to provide contextually relevant responses based on the organization's data.

#### 2.1 Document Embedding Pipeline

```python
class EmbeddingPipeline:
    def __init__(self):
        self.embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
        self.vector_db = QdrantClient()
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200
        )
    
    async def process_document(self, document: Document, tenant_id: str):
        """Process and index a document for RAG retrieval"""
        
        # 1. Extract text content
        text_content = await self.extract_text(document)
        
        # 2. Split into chunks
        chunks = self.text_splitter.split_text(text_content)
        
        # 3. Generate embeddings
        embeddings = self.embedding_model.encode(chunks)
        
        # 4. Create metadata for each chunk
        chunk_documents = []
        for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
            chunk_doc = {
                'id': f"{document.id}_{i}",
                'content': chunk,
                'embedding': embedding.tolist(),
                'metadata': {
                    'document_id': document.id,
                    'document_type': document.type,
                    'tenant_id': tenant_id,
                    'source_module': document.module,
                    'permissions': document.required_permissions,
                    'created_at': document.created_at.isoformat(),
                    'chunk_index': i
                }
            }
            chunk_documents.append(chunk_doc)
        
        # 5. Store in vector database
        await self.vector_db.upsert(
            collection_name=f"tenant_{tenant_id}",
            points=chunk_documents
        )
        
        return len(chunk_documents)
```

#### 2.2 Real-time Data Synchronization

```python
class DataSynchronizer:
    def __init__(self):
        self.embedding_pipeline = EmbeddingPipeline()
        self.kafka_consumer = KafkaConsumer()
        self.vector_db = QdrantClient()
    
    async def sync_business_data(self, event: BusinessEvent):
        """Sync business data changes to vector database"""
        
        if event.event_type in ['employee.created', 'employee.updated']:
            await self.sync_employee_data(event)
        elif event.event_type in ['invoice.created', 'invoice.updated']:
            await self.sync_invoice_data(event)
        elif event.event_type in ['contact.created', 'contact.updated']:
            await self.sync_contact_data(event)
        # ... handle other entity types
    
    async def sync_employee_data(self, event: BusinessEvent):
        """Convert employee data to searchable document"""
        employee_data = event.data
        
        # Create searchable document
        document_content = f"""
        Employee: {employee_data['first_name']} {employee_data['last_name']}
        Employee ID: {employee_data['employee_id']}
        Department: {employee_data['department']['name']}
        Position: {employee_data['position']}
        Email: {employee_data['email']}
        Hire Date: {employee_data['hire_date']}
        Status: {employee_data['status']}
        """
        
        document = Document(
            id=f"employee_{employee_data['id']}",
            content=document_content,
            type='employee',
            module='hrm',
            required_permissions=['hrm.employee.read'],
            tenant_id=event.tenant_id
        )
        
        await self.embedding_pipeline.process_document(document, event.tenant_id)
```

#### 2.3 Permission-Aware Retrieval

```python
class PermissionAwareRetriever:
    def __init__(self):
        self.vector_db = QdrantClient()
        self.embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
    
    async def retrieve_context(self, query: str, user_permissions: List[str], tenant_id: str, limit: int = 10) -> List[Document]:
        """Retrieve relevant documents based on user permissions"""
        
        # 1. Generate query embedding
        query_embedding = self.embedding_model.encode([query])[0]
        
        # 2. Build permission filter
        permission_filter = self.build_permission_filter(user_permissions)
        
        # 3. Search vector database
        search_results = await self.vector_db.search(
            collection_name=f"tenant_{tenant_id}",
            query_vector=query_embedding.tolist(),
            query_filter=permission_filter,
            limit=limit,
            score_threshold=0.7
        )
        
        # 4. Convert results to documents
        documents = []
        for result in search_results:
            doc = Document(
                id=result.payload['document_id'],
                content=result.payload['content'],
                type=result.payload['document_type'],
                module=result.payload['source_module'],
                score=result.score,
                metadata=result.payload
            )
            documents.append(doc)
        
        return documents
    
    def build_permission_filter(self, user_permissions: List[str]) -> Dict:
        """Build Qdrant filter based on user permissions"""
        
        # Create OR conditions for each permission the user has
        permission_conditions = []
        for permission in user_permissions:
            permission_conditions.append({
                "key": "permissions",
                "match": {"any": [permission]}
            })
        
        return {
            "should": permission_conditions
        }
```

### 3. Context Management System

The context manager maintains conversation state and user context across interactions.

```python
class ContextManager:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.session_timeout = 3600  # 1 hour
    
    async def get_conversation_context(self, user_id: str, conversation_id: str) -> ConversationContext:
        """Retrieve conversation context from cache"""
        
        cache_key = f"conversation:{user_id}:{conversation_id}"
        cached_context = await self.redis_client.get(cache_key)
        
        if cached_context:
            return ConversationContext.from_json(cached_context)
        
        # Create new context if not found
        context = ConversationContext(
            user_id=user_id,
            conversation_id=conversation_id,
            messages=[],
            current_module=None,
            active_entities={},
            user_preferences=await self.get_user_preferences(user_id)
        )
        
        await self.save_conversation_context(context)
        return context
    
    async def update_conversation_context(self, context: ConversationContext, message: ChatMessage, response: ChatResponse):
        """Update conversation context with new message and response"""
        
        # Add message to history
        context.messages.append({
            'role': 'user',
            'content': message.content,
            'timestamp': message.timestamp
        })
        
        context.messages.append({
            'role': 'assistant',
            'content': response.content,
            'timestamp': response.timestamp,
            'actions': response.actions
        })
        
        # Update active entities
        if response.entities:
            context.active_entities.update(response.entities)
        
        # Update current module if changed
        if response.module:
            context.current_module = response.module
        
        # Trim old messages to prevent context overflow
        if len(context.messages) > 20:
            context.messages = context.messages[-20:]
        
        await self.save_conversation_context(context)
    
    async def save_conversation_context(self, context: ConversationContext):
        """Save conversation context to cache"""
        
        cache_key = f"conversation:{context.user_id}:{context.conversation_id}"
        await self.redis_client.setex(
            cache_key,
            self.session_timeout,
            context.to_json()
        )
```

### 4. Intent Recognition and Entity Extraction

```python
class IntentRecognizer:
    def __init__(self):
        self.nlp = spacy.load("en_core_web_sm")
        self.intent_patterns = self.load_intent_patterns()
    
    async def parse_intent(self, message: str) -> Intent:
        """Parse user intent from message"""
        
        doc = self.nlp(message.lower())
        
        # Extract action verbs
        action_verbs = [token.lemma_ for token in doc if token.pos_ == "VERB"]
        
        # Match against known patterns
        for pattern in self.intent_patterns:
            if self.matches_pattern(doc, pattern):
                return Intent(
                    type=pattern['type'],
                    action=pattern['action'],
                    confidence=pattern['confidence'],
                    requires_action=pattern.get('requires_action', False),
                    parameters=self.extract_parameters(doc, pattern)
                )
        
        # Default to query intent
        return Intent(
            type='query',
            action='search',
            confidence=0.5,
            requires_action=False
        )
    
    def load_intent_patterns(self) -> List[Dict]:
        """Load intent recognition patterns"""
        return [
            {
                'type': 'create',
                'action': 'create_record',
                'patterns': ['create', 'add', 'new', 'make'],
                'entities': ['employee', 'invoice', 'contact', 'project'],
                'requires_action': True,
                'confidence': 0.9
            },
            {
                'type': 'search',
                'action': 'search_records',
                'patterns': ['show', 'find', 'search', 'get', 'list'],
                'entities': ['employee', 'invoice', 'contact', 'project'],
                'requires_action': False,
                'confidence': 0.8
            },
            {
                'type': 'update',
                'action': 'update_record',
                'patterns': ['update', 'change', 'modify', 'edit'],
                'entities': ['employee', 'invoice', 'contact', 'project'],
                'requires_action': True,
                'confidence': 0.9
            },
            {
                'type': 'report',
                'action': 'generate_report',
                'patterns': ['report', 'generate', 'export', 'download'],
                'entities': ['sales', 'financial', 'employee', 'inventory'],
                'requires_action': True,
                'confidence': 0.85
            },
            {
                'type': 'navigate',
                'action': 'navigate_to',
                'patterns': ['go to', 'open', 'navigate', 'show me'],
                'entities': ['dashboard', 'page', 'module'],
                'requires_action': True,
                'confidence': 0.8
            }
        ]
```

### 5. Action Execution Framework

```python
class ActionExecutor:
    def __init__(self):
        self.business_services = BusinessServiceRegistry()
        self.rbac_service = RBACService()
        self.audit_logger = AuditLogger()
    
    async def execute(self, action: str, parameters: Dict, user: User) -> ActionResult:
        """Execute AI-requested action with proper authorization"""
        
        # 1. Validate permissions
        if not await self.rbac_service.can_execute_action(user, action, parameters):
            return ActionResult(
                success=False,
                error="Insufficient permissions to execute this action",
                error_code="PERMISSION_DENIED"
            )
        
        # 2. Route to appropriate handler
        try:
            if action == 'create_record':
                result = await self.handle_create_record(parameters, user)
            elif action == 'search_records':
                result = await self.handle_search_records(parameters, user)
            elif action == 'update_record':
                result = await self.handle_update_record(parameters, user)
            elif action == 'generate_report':
                result = await self.handle_generate_report(parameters, user)
            elif action == 'navigate_to':
                result = await self.handle_navigation(parameters, user)
            else:
                result = ActionResult(
                    success=False,
                    error=f"Unknown action: {action}",
                    error_code="UNKNOWN_ACTION"
                )
        
        except Exception as e:
            result = ActionResult(
                success=False,
                error=str(e),
                error_code="EXECUTION_ERROR"
            )
        
        # 3. Log action for audit
        await self.audit_logger.log_ai_action(
            user=user,
            action=action,
            parameters=parameters,
            result=result
        )
        
        return result
    
    async def handle_create_record(self, parameters: Dict, user: User) -> ActionResult:
        """Handle record creation requests"""
        
        entity_type = parameters.get('entity_type')
        entity_data = parameters.get('data', {})
        
        if entity_type == 'employee':
            service = self.business_services.get_service('hrm')
            employee = await service.create_employee(entity_data, user)
            return ActionResult(
                success=True,
                data={'employee': employee.to_dict()},
                message=f"Employee {employee.full_name} created successfully"
            )
        
        elif entity_type == 'invoice':
            service = self.business_services.get_service('finance')
            invoice = await service.create_invoice(entity_data, user)
            return ActionResult(
                success=True,
                data={'invoice': invoice.to_dict()},
                message=f"Invoice {invoice.invoice_number} created successfully"
            )
        
        # ... handle other entity types
        
        return ActionResult(
            success=False,
            error=f"Unknown entity type: {entity_type}",
            error_code="UNKNOWN_ENTITY_TYPE"
        )
```

### 6. Response Generation with LLM Integration

```python
class ResponseGenerator:
    def __init__(self):
        self.llm_providers = {
            'openai': OpenAIProvider(),
            'anthropic': AnthropicProvider(),
            'local': LocalLLMProvider()
        }
        self.default_provider = 'openai'
    
    async def generate_response(self, query: str, context_docs: List[Document], user: User, intent: Intent) -> str:
        """Generate contextual response using LLM"""
        
        # 1. Build context from retrieved documents
        context_text = self.build_context_text(context_docs)
        
        # 2. Create system prompt based on user role and intent
        system_prompt = self.create_system_prompt(user, intent)
        
        # 3. Build user prompt with context
        user_prompt = self.create_user_prompt(query, context_text, user)
        
        # 4. Get LLM provider for user's tenant
        provider_name = await self.get_tenant_llm_provider(user.tenant_id)
        provider = self.llm_providers.get(provider_name, self.llm_providers[self.default_provider])
        
        # 5. Generate response
        response = await provider.generate_response(
            system_prompt=system_prompt,
            user_prompt=user_prompt,
            max_tokens=1000,
            temperature=0.7
        )
        
        # 6. Post-process response for security
        filtered_response = await self.filter_sensitive_information(response, user)
        
        return filtered_response
    
    def create_system_prompt(self, user: User, intent: Intent) -> str:
        """Create role-specific system prompt"""
        
        base_prompt = f"""
        You are an intelligent ERP assistant for {user.organization.name}.
        You are helping {user.full_name} who has the role of {user.primary_role}.
        
        Current context:
        - User's department: {user.department.name if user.department else 'N/A'}
        - User's permissions: {', '.join(user.permissions)}
        - Current module: {intent.module if hasattr(intent, 'module') else 'General'}
        
        Guidelines:
        1. Only provide information the user is authorized to access
        2. Be helpful and professional
        3. If asked to perform actions, confirm before proceeding
        4. Provide specific, actionable responses
        5. Reference relevant data from the context when available
        6. If you cannot access certain information due to permissions, explain this clearly
        """
        
        # Add role-specific instructions
        if 'hrm' in user.permissions:
            base_prompt += "\n- You can help with employee management, leave requests, and HR processes"
        if 'finance' in user.permissions:
            base_prompt += "\n- You can help with invoices, payments, and financial reports"
        if 'crm' in user.permissions:
            base_prompt += "\n- You can help with contacts, leads, and sales processes"
        
        return base_prompt
    
    def create_user_prompt(self, query: str, context: str, user: User) -> str:
        """Create user prompt with context"""
        
        return f"""
        Context from {user.organization.name}'s data:
        {context}
        
        User question: {query}
        
        Please provide a helpful response based on the available context and the user's permissions.
        """
    
    async def filter_sensitive_information(self, response: str, user: User) -> str:
        """Filter sensitive information from response based on user permissions"""
        
        # Remove salary information if user doesn't have payroll permissions
        if 'hrm.payroll.read' not in user.permissions:
            response = re.sub(r'\$[\d,]+(?:\.\d{2})?', '[SALARY REDACTED]', response)
        
        # Remove sensitive financial data if not authorized
        if 'finance.sensitive.read' not in user.permissions:
            response = re.sub(r'profit.*?\$[\d,]+', '[FINANCIAL DATA REDACTED]', response, flags=re.IGNORECASE)
        
        # Remove personal information if not HR
        if 'hrm.personal.read' not in user.permissions:
            response = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '[SSN REDACTED]', response)  # SSN
            response = re.sub(r'\b\d{3}-\d{3}-\d{4}\b', '[PHONE REDACTED]', response)  # Phone
        
        return response
```

### 7. Security and Audit Framework

```python
class AISecurityManager:
    def __init__(self):
        self.audit_logger = AuditLogger()
        self.rate_limiter = RateLimiter()
        self.content_filter = ContentFilter()
    
    async def validate_ai_request(self, request: ChatRequest, user: User) -> ValidationResult:
        """Validate AI request for security and compliance"""
        
        # 1. Rate limiting
        if not await self.rate_limiter.check_limit(user.id, 'ai_chat', limit=100, window=3600):
            return ValidationResult(
                valid=False,
                error="Rate limit exceeded",
                error_code="RATE_LIMIT_EXCEEDED"
            )
        
        # 2. Content filtering
        if await self.content_filter.contains_malicious_content(request.message):
            await self.audit_logger.log_security_event(
                user=user,
                event_type="MALICIOUS_CONTENT_DETECTED",
                details={"message": request.message}
            )
            return ValidationResult(
                valid=False,
                error="Message contains prohibited content",
                error_code="CONTENT_VIOLATION"
            )
        
        # 3. Prompt injection detection
        if await self.detect_prompt_injection(request.message):
            await self.audit_logger.log_security_event(
                user=user,
                event_type="PROMPT_INJECTION_ATTEMPT",
                details={"message": request.message}
            )
            return ValidationResult(
                valid=False,
                error="Potential prompt injection detected",
                error_code="PROMPT_INJECTION"
            )
        
        return ValidationResult(valid=True)
    
    async def detect_prompt_injection(self, message: str) -> bool:
        """Detect potential prompt injection attempts"""
        
        injection_patterns = [
            r'ignore\s+previous\s+instructions',
            r'system\s*:\s*you\s+are',
            r'forget\s+everything',
            r'new\s+instructions',
            r'override\s+your\s+programming',
            r'act\s+as\s+if\s+you\s+are',
            r'pretend\s+to\s+be'
        ]
        
        message_lower = message.lower()
        for pattern in injection_patterns:
            if re.search(pattern, message_lower):
                return True
        
        return False
```

### 8. Performance Optimization

```python
class PerformanceOptimizer:
    def __init__(self):
        self.cache = Redis()
        self.metrics = MetricsCollector()
    
    async def optimize_retrieval(self, query: str, user_context: UserContext) -> List[Document]:
        """Optimize document retrieval with caching and pre-filtering"""
        
        # 1. Check cache for similar queries
        cache_key = self.generate_cache_key(query, user_context.permissions)
        cached_results = await self.cache.get(cache_key)
        
        if cached_results:
            self.metrics.increment('rag.cache_hit')
            return json.loads(cached_results)
        
        # 2. Perform retrieval with optimizations
        start_time = time.time()
        
        # Pre-filter by permissions to reduce search space
        permission_filter = self.build_efficient_filter(user_context.permissions)
        
        # Use approximate search for better performance
        results = await self.vector_db.search(
            query_vector=self.get_query_embedding(query),
            filter=permission_filter,
            limit=20,  # Get more results for better ranking
            ef=128,    # Increase search accuracy
            score_threshold=0.6
        )
        
        # 3. Re-rank results for relevance
        ranked_results = await self.rerank_results(results, query, user_context)
        
        # 4. Cache results
        await self.cache.setex(
            cache_key,
            300,  # 5 minutes
            json.dumps([doc.to_dict() for doc in ranked_results[:10]])
        )
        
        retrieval_time = time.time() - start_time
        self.metrics.histogram('rag.retrieval_time', retrieval_time)
        
        return ranked_results[:10]
```

This comprehensive AI chatbot design provides a robust, secure, and scalable foundation for intelligent ERP interactions while maintaining strict data access controls and audit trails.