# Plugin System Architecture

## Overview

This document describes the comprehensive plugin system that allows AI services, Google integrations, and custom extensions to be seamlessly integrated into the ERP system with proper isolation and security.

## Plugin Architecture

### 1. Core Plugin Framework

The plugin system is built around a modular architecture that provides secure isolation, dependency management, and lifecycle control.

```python
# Core plugin interface
from abc import ABC, abstractmethod
from typing import Dict, Any, List, Optional
from dataclasses import dataclass
from enum import Enum

class PluginType(Enum):
    AI_PROVIDER = "ai_provider"
    INTEGRATION = "integration"
    BUSINESS_LOGIC = "business_logic"
    UI_EXTENSION = "ui_extension"
    WORKFLOW = "workflow"
    NOTIFICATION = "notification"

@dataclass
class PluginMetadata:
    name: str
    version: str
    description: str
    author: str
    plugin_type: PluginType
    dependencies: List[str]
    permissions: List[str]
    config_schema: Dict[str, Any]
    api_version: str = "1.0"

class PluginInterface(ABC):
    """Base interface for all plugins"""
    
    @property
    @abstractmethod
    def metadata(self) -> PluginMetadata:
        """Return plugin metadata"""
        pass
    
    @abstractmethod
    async def initialize(self, config: Dict[str, Any], context: 'PluginContext') -> bool:
        """Initialize the plugin with configuration"""
        pass
    
    @abstractmethod
    async def activate(self) -> bool:
        """Activate the plugin"""
        pass
    
    @abstractmethod
    async def deactivate(self) -> bool:
        """Deactivate the plugin"""
        pass
    
    @abstractmethod
    async def health_check(self) -> Dict[str, Any]:
        """Return plugin health status"""
        pass
    
    @abstractmethod
    async def cleanup(self) -> bool:
        """Clean up plugin resources"""
        pass

class PluginContext:
    """Context provided to plugins for system interaction"""
    
    def __init__(self, tenant_id: str, user_context: Optional[Dict] = None):
        self.tenant_id = tenant_id
        self.user_context = user_context or {}
        self._services = {}
        self._config = {}
    
    def get_service(self, service_name: str):
        """Get system service instance"""
        return self._services.get(service_name)
    
    def get_config(self, key: str, default=None):
        """Get configuration value"""
        return self._config.get(key, default)
    
    def log(self, level: str, message: str, **kwargs):
        """Log message through system logger"""
        # Implementation for logging
        pass
    
    def emit_event(self, event_type: str, data: Dict[str, Any]):
        """Emit system event"""
        # Implementation for event emission
        pass
```

### 2. Plugin Manager

The plugin manager handles plugin lifecycle, dependency resolution, and security enforcement.

```python
import importlib
import inspect
import asyncio
from pathlib import Path
from typing import Dict, List, Optional, Type
import yaml
import json

class PluginManager:
    """Central plugin management system"""
    
    def __init__(self):
        self.plugins: Dict[str, PluginInterface] = {}
        self.plugin_configs: Dict[str, Dict] = {}
        self.plugin_states: Dict[str, str] = {}  # loaded, active, inactive, error
        self.dependency_graph: Dict[str, List[str]] = {}
        self.security_manager = PluginSecurityManager()
        self.sandbox_manager = PluginSandboxManager()
    
    async def discover_plugins(self, plugin_directories: List[str]) -> List[str]:
        """Discover available plugins in specified directories"""
        
        discovered_plugins = []
        
        for directory in plugin_directories:
            plugin_path = Path(directory)
            if not plugin_path.exists():
                continue
            
            for plugin_dir in plugin_path.iterdir():
                if plugin_dir.is_dir():
                    manifest_path = plugin_dir / "plugin.yaml"
                    if manifest_path.exists():
                        try:
                            plugin_info = await self._load_plugin_manifest(manifest_path)
                            discovered_plugins.append(plugin_info['name'])
                        except Exception as e:
                            self.log_error(f"Failed to load plugin manifest {manifest_path}: {e}")
        
        return discovered_plugins
    
    async def load_plugin(self, plugin_name: str, plugin_path: str) -> bool:
        """Load a plugin from the specified path"""
        
        try:
            # 1. Load plugin manifest
            manifest_path = Path(plugin_path) / "plugin.yaml"
            manifest = await self._load_plugin_manifest(manifest_path)
            
            # 2. Security validation
            if not await self.security_manager.validate_plugin(plugin_path, manifest):
                raise SecurityError(f"Plugin {plugin_name} failed security validation")
            
            # 3. Load plugin module
            plugin_module = await self._load_plugin_module(plugin_path, manifest)
            
            # 4. Instantiate plugin class
            plugin_class = getattr(plugin_module, manifest['main_class'])
            plugin_instance = plugin_class()
            
            # 5. Validate plugin interface
            if not isinstance(plugin_instance, PluginInterface):
                raise ValueError(f"Plugin {plugin_name} does not implement PluginInterface")
            
            # 6. Store plugin
            self.plugins[plugin_name] = plugin_instance
            self.plugin_configs[plugin_name] = manifest
            self.plugin_states[plugin_name] = "loaded"
            
            # 7. Build dependency graph
            self._update_dependency_graph(plugin_name, manifest.get('dependencies', []))
            
            return True
            
        except Exception as e:
            self.log_error(f"Failed to load plugin {plugin_name}: {e}")
            self.plugin_states[plugin_name] = "error"
            return False
    
    async def initialize_plugin(self, plugin_name: str, config: Dict[str, Any], tenant_id: str) -> bool:
        """Initialize a loaded plugin with configuration"""
        
        if plugin_name not in self.plugins:
            raise ValueError(f"Plugin {plugin_name} not loaded")
        
        plugin = self.plugins[plugin_name]
        
        try:
            # 1. Validate configuration
            if not await self._validate_plugin_config(plugin_name, config):
                raise ValueError(f"Invalid configuration for plugin {plugin_name}")
            
            # 2. Create plugin context
            context = PluginContext(tenant_id)
            await self._setup_plugin_context(context, plugin_name)
            
            # 3. Initialize plugin in sandbox
            success = await self.sandbox_manager.execute_in_sandbox(
                plugin_name,
                plugin.initialize,
                config,
                context
            )
            
            if success:
                self.plugin_states[plugin_name] = "initialized"
            
            return success
            
        except Exception as e:
            self.log_error(f"Failed to initialize plugin {plugin_name}: {e}")
            self.plugin_states[plugin_name] = "error"
            return False
    
    async def activate_plugin(self, plugin_name: str) -> bool:
        """Activate an initialized plugin"""
        
        if plugin_name not in self.plugins:
            raise ValueError(f"Plugin {plugin_name} not loaded")
        
        if self.plugin_states[plugin_name] != "initialized":
            raise ValueError(f"Plugin {plugin_name} not initialized")
        
        plugin = self.plugins[plugin_name]
        
        try:
            # 1. Check dependencies
            if not await self._check_dependencies(plugin_name):
                raise ValueError(f"Dependencies not met for plugin {plugin_name}")
            
            # 2. Activate plugin
            success = await self.sandbox_manager.execute_in_sandbox(
                plugin_name,
                plugin.activate
            )
            
            if success:
                self.plugin_states[plugin_name] = "active"
                await self._register_plugin_hooks(plugin_name)
            
            return success
            
        except Exception as e:
            self.log_error(f"Failed to activate plugin {plugin_name}: {e}")
            self.plugin_states[plugin_name] = "error"
            return False
    
    async def deactivate_plugin(self, plugin_name: str) -> bool:
        """Deactivate an active plugin"""
        
        if plugin_name not in self.plugins:
            return True  # Already not loaded
        
        plugin = self.plugins[plugin_name]
        
        try:
            # 1. Unregister hooks
            await self._unregister_plugin_hooks(plugin_name)
            
            # 2. Deactivate plugin
            success = await self.sandbox_manager.execute_in_sandbox(
                plugin_name,
                plugin.deactivate
            )
            
            if success:
                self.plugin_states[plugin_name] = "inactive"
            
            return success
            
        except Exception as e:
            self.log_error(f"Failed to deactivate plugin {plugin_name}: {e}")
            return False
    
    async def unload_plugin(self, plugin_name: str) -> bool:
        """Completely unload a plugin"""
        
        if plugin_name not in self.plugins:
            return True
        
        try:
            # 1. Deactivate if active
            if self.plugin_states[plugin_name] == "active":
                await self.deactivate_plugin(plugin_name)
            
            # 2. Cleanup plugin resources
            plugin = self.plugins[plugin_name]
            await self.sandbox_manager.execute_in_sandbox(
                plugin_name,
                plugin.cleanup
            )
            
            # 3. Remove from manager
            del self.plugins[plugin_name]
            del self.plugin_configs[plugin_name]
            del self.plugin_states[plugin_name]
            
            # 4. Update dependency graph
            self._remove_from_dependency_graph(plugin_name)
            
            # 5. Cleanup sandbox
            await self.sandbox_manager.cleanup_sandbox(plugin_name)
            
            return True
            
        except Exception as e:
            self.log_error(f"Failed to unload plugin {plugin_name}: {e}")
            return False
```

### 3. Plugin Security Manager

```python
import hashlib
import os
import ast
import subprocess
from pathlib import Path

class PluginSecurityManager:
    """Manages plugin security validation and sandboxing"""
    
    def __init__(self):
        self.allowed_imports = {
            'standard': [
                'json', 'datetime', 'uuid', 'typing', 'dataclasses',
                'asyncio', 'logging', 'pathlib', 'urllib', 'base64'
            ],
            'third_party': [
                'requests', 'aiohttp', 'pydantic', 'sqlalchemy',
                'redis', 'celery', 'kafka', 'elasticsearch'
            ]
        }
        self.forbidden_operations = [
            'exec', 'eval', 'compile', '__import__',
            'open', 'file', 'input', 'raw_input'
        ]
    
    async def validate_plugin(self, plugin_path: str, manifest: Dict) -> bool:
        """Comprehensive plugin security validation"""
        
        try:
            # 1. Validate manifest
            if not await self._validate_manifest(manifest):
                return False
            
            # 2. Scan plugin code
            if not await self._scan_plugin_code(plugin_path):
                return False
            
            # 3. Check file permissions
            if not await self._check_file_permissions(plugin_path):
                return False
            
            # 4. Validate dependencies
            if not await self._validate_dependencies(manifest.get('dependencies', [])):
                return False
            
            # 5. Generate and verify checksum
            if not await self._verify_plugin_integrity(plugin_path, manifest):
                return False
            
            return True
            
        except Exception as e:
            self.log_error(f"Security validation failed for {plugin_path}: {e}")
            return False
    
    async def _scan_plugin_code(self, plugin_path: str) -> bool:
        """Scan plugin code for security vulnerabilities"""
        
        plugin_dir = Path(plugin_path)
        
        for py_file in plugin_dir.rglob("*.py"):
            try:
                with open(py_file, 'r', encoding='utf-8') as f:
                    code = f.read()
                
                # Parse AST for analysis
                tree = ast.parse(code)
                
                # Check for forbidden operations
                for node in ast.walk(tree):
                    if isinstance(node, ast.Call):
                        if hasattr(node.func, 'id') and node.func.id in self.forbidden_operations:
                            self.log_security_violation(
                                f"Forbidden operation '{node.func.id}' found in {py_file}"
                            )
                            return False
                    
                    # Check imports
                    elif isinstance(node, (ast.Import, ast.ImportFrom)):
                        if not self._validate_import(node):
                            return False
                
            except Exception as e:
                self.log_error(f"Failed to scan {py_file}: {e}")
                return False
        
        return True
    
    def _validate_import(self, node) -> bool:
        """Validate import statements"""
        
        if isinstance(node, ast.Import):
            for alias in node.names:
                if not self._is_allowed_import(alias.name):
                    self.log_security_violation(f"Forbidden import: {alias.name}")
                    return False
        
        elif isinstance(node, ast.ImportFrom):
            module = node.module
            if module and not self._is_allowed_import(module):
                self.log_security_violation(f"Forbidden import from: {module}")
                return False
        
        return True
    
    def _is_allowed_import(self, module_name: str) -> bool:
        """Check if import is allowed"""
        
        # Check standard library
        if module_name in self.allowed_imports['standard']:
            return True
        
        # Check third-party libraries
        if module_name in self.allowed_imports['third_party']:
            return True
        
        # Check if it's a system module (starts with system prefix)
        if module_name.startswith('erp_system.'):
            return True
        
        return False
```

### 4. Plugin Sandbox Manager

```python
import docker
import tempfile
import shutil
from typing import Any, Callable

class PluginSandboxManager:
    """Manages plugin execution in isolated sandboxes"""
    
    def __init__(self):
        self.docker_client = docker.from_env()
        self.sandboxes: Dict[str, Dict] = {}
        self.resource_limits = {
            'memory': '256m',
            'cpu_quota': 50000,  # 50% of one CPU
            'network': 'none',   # No network access by default
            'read_only': True
        }
    
    async def create_sandbox(self, plugin_name: str, config: Dict) -> str:
        """Create isolated sandbox for plugin execution"""
        
        try:
            # 1. Create temporary directory for plugin
            temp_dir = tempfile.mkdtemp(prefix=f"plugin_{plugin_name}_")
            
            # 2. Copy plugin files to sandbox
            plugin_path = config.get('plugin_path')
            if plugin_path:
                shutil.copytree(plugin_path, f"{temp_dir}/plugin")
            
            # 3. Create Docker container
            container = self.docker_client.containers.create(
                image='python:3.11-slim',
                command=['python', '-c', 'import time; time.sleep(3600)'],  # Keep alive
                mem_limit=self.resource_limits['memory'],
                cpu_quota=self.resource_limits['cpu_quota'],
                network_mode=self.resource_limits['network'],
                read_only=self.resource_limits['read_only'],
                volumes={
                    temp_dir: {'bind': '/app', 'mode': 'ro'}
                },
                working_dir='/app',
                environment={
                    'PLUGIN_NAME': plugin_name,
                    'PYTHONPATH': '/app'
                }
            )
            
            # 4. Start container
            container.start()
            
            # 5. Store sandbox info
            self.sandboxes[plugin_name] = {
                'container': container,
                'temp_dir': temp_dir,
                'created_at': datetime.utcnow()
            }
            
            return container.id
            
        except Exception as e:
            self.log_error(f"Failed to create sandbox for {plugin_name}: {e}")
            raise
    
    async def execute_in_sandbox(self, plugin_name: str, func: Callable, *args, **kwargs) -> Any:
        """Execute function in plugin sandbox with timeout and monitoring"""
        
        if plugin_name not in self.sandboxes:
            await self.create_sandbox(plugin_name, {})
        
        sandbox = self.sandboxes[plugin_name]
        container = sandbox['container']
        
        try:
            # 1. Prepare execution code
            execution_code = self._prepare_execution_code(func, args, kwargs)
            
            # 2. Execute with timeout
            exec_result = container.exec_run(
                cmd=['python', '-c', execution_code],
                timeout=30,  # 30 second timeout
                demux=True
            )
            
            # 3. Parse result
            if exec_result.exit_code == 0:
                return self._parse_execution_result(exec_result.output)
            else:
                raise RuntimeError(f"Plugin execution failed: {exec_result.output}")
                
        except Exception as e:
            self.log_error(f"Sandbox execution failed for {plugin_name}: {e}")
            raise
    
    async def cleanup_sandbox(self, plugin_name: str):
        """Clean up plugin sandbox"""
        
        if plugin_name not in self.sandboxes:
            return
        
        sandbox = self.sandboxes[plugin_name]
        
        try:
            # 1. Stop and remove container
            container = sandbox['container']
            container.stop(timeout=5)
            container.remove()
            
            # 2. Clean up temporary directory
            temp_dir = sandbox['temp_dir']
            shutil.rmtree(temp_dir, ignore_errors=True)
            
            # 3. Remove from tracking
            del self.sandboxes[plugin_name]
            
        except Exception as e:
            self.log_error(f"Failed to cleanup sandbox for {plugin_name}: {e}")
```

## AI Plugin Implementation

### 1. AI Provider Plugin Interface

```python
from abc import abstractmethod
from typing import List, Dict, Any, AsyncGenerator

class AIProviderPlugin(PluginInterface):
    """Base class for AI provider plugins"""
    
    @abstractmethod
    async def generate_response(self, prompt: str, context: Dict[str, Any]) -> str:
        """Generate AI response"""
        pass
    
    @abstractmethod
    async def generate_streaming_response(self, prompt: str, context: Dict[str, Any]) -> AsyncGenerator[str, None]:
        """Generate streaming AI response"""
        pass
    
    @abstractmethod
    async def generate_embeddings(self, texts: List[str]) -> List[List[float]]:
        """Generate text embeddings"""
        pass
    
    @abstractmethod
    async def validate_model_config(self, config: Dict[str, Any]) -> bool:
        """Validate model configuration"""
        pass

# Example OpenAI Plugin
class OpenAIPlugin(AIProviderPlugin):
    """OpenAI provider plugin"""
    
    @property
    def metadata(self) -> PluginMetadata:
        return PluginMetadata(
            name="openai_provider",
            version="1.0.0",
            description="OpenAI GPT integration plugin",
            author="ERP System Team",
            plugin_type=PluginType.AI_PROVIDER,
            dependencies=[],
            permissions=["ai.generate", "ai.embeddings"],
            config_schema={
                "api_key": {"type": "string", "required": True},
                "model": {"type": "string", "default": "gpt-4"},
                "temperature": {"type": "number", "default": 0.7},
                "max_tokens": {"type": "integer", "default": 1000}
            }
        )
    
    async def initialize(self, config: Dict[str, Any], context: PluginContext) -> bool:
        """Initialize OpenAI client"""
        try:
            import openai
            
            self.client = openai.AsyncOpenAI(
                api_key=config['api_key']
            )
            self.model = config.get('model', 'gpt-4')
            self.temperature = config.get('temperature', 0.7)
            self.max_tokens = config.get('max_tokens', 1000)
            
            # Test connection
            await self.client.models.list()
            
            return True
            
        except Exception as e:
            context.log('error', f"Failed to initialize OpenAI plugin: {e}")
            return False
    
    async def generate_response(self, prompt: str, context: Dict[str, Any]) -> str:
        """Generate response using OpenAI"""
        
        try:
            response = await self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": context.get('system_prompt', '')},
                    {"role": "user", "content": prompt}
                ],
                temperature=self.temperature,
                max_tokens=self.max_tokens
            )
            
            return response.choices[0].message.content
            
        except Exception as e:
            raise RuntimeError(f"OpenAI generation failed: {e}")
    
    async def generate_streaming_response(self, prompt: str, context: Dict[str, Any]) -> AsyncGenerator[str, None]:
        """Generate streaming response"""
        
        try:
            stream = await self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": context.get('system_prompt', '')},
                    {"role": "user", "content": prompt}
                ],
                temperature=self.temperature,
                max_tokens=self.max_tokens,
                stream=True
            )
            
            async for chunk in stream:
                if chunk.choices[0].delta.content:
                    yield chunk.choices[0].delta.content
                    
        except Exception as e:
            raise RuntimeError(f"OpenAI streaming failed: {e}")
```

### 2. Local LLM Plugin

```python
class LocalLLMPlugin(AIProviderPlugin):
    """Local LLM provider plugin using Ollama"""
    
    @property
    def metadata(self) -> PluginMetadata:
        return PluginMetadata(
            name="local_llm_provider",
            version="1.0.0",
            description="Local LLM integration using Ollama",
            author="ERP System Team",
            plugin_type=PluginType.AI_PROVIDER,
            dependencies=["ollama"],
            permissions=["ai.generate", "ai.embeddings"],
            config_schema={
                "model": {"type": "string", "default": "llama2"},
                "host": {"type": "string", "default": "http://localhost:11434"},
                "temperature": {"type": "number", "default": 0.7},
                "context_length": {"type": "integer", "default": 4096}
            }
        )
    
    async def initialize(self, config: Dict[str, Any], context: PluginContext) -> bool:
        """Initialize Ollama client"""
        try:
            import ollama
            
            self.client = ollama.AsyncClient(host=config.get('host', 'http://localhost:11434'))
            self.model = config.get('model', 'llama2')
            self.temperature = config.get('temperature', 0.7)
            self.context_length = config.get('context_length', 4096)
            
            # Test connection
            await self.client.list()
            
            return True
            
        except Exception as e:
            context.log('error', f"Failed to initialize Local LLM plugin: {e}")
            return False
    
    async def generate_response(self, prompt: str, context: Dict[str, Any]) -> str:
        """Generate response using local LLM"""
        
        try:
            response = await self.client.generate(
                model=self.model,
                prompt=prompt,
                options={
                    'temperature': self.temperature,
                    'num_ctx': self.context_length
                }
            )
            
            return response['response']
            
        except Exception as e:
            raise RuntimeError(f"Local LLM generation failed: {e}")
```

## Google Services Integration Plugin

### 1. Google Services Plugin

```python
class GoogleServicesPlugin(PluginInterface):
    """Google services integration plugin"""
    
    @property
    def metadata(self) -> PluginMetadata:
        return PluginMetadata(
            name="google_services",
            version="1.0.0",
            description="Google Workspace integration (Gmail, Drive, Calendar)",
            author="ERP System Team",
            plugin_type=PluginType.INTEGRATION,
            dependencies=["google-auth", "google-api-python-client"],
            permissions=["integration.google", "email.send", "files.upload"],
            config_schema={
                "credentials_file": {"type": "string", "required": True},
                "scopes": {"type": "array", "default": [
                    "https://www.googleapis.com/auth/gmail.send",
                    "https://www.googleapis.com/auth/drive.file"
                ]}
            }
        )
    
    async def initialize(self, config: Dict[str, Any], context: PluginContext) -> bool:
        """Initialize Google services"""
        try:
            from google.oauth2.credentials import Credentials
            from googleapiclient.discovery import build
            
            # Load credentials
            credentials = Credentials.from_authorized_user_file(
                config['credentials_file'],
                config['scopes']
            )
            
            # Initialize services
            self.gmail_service = build('gmail', 'v1', credentials=credentials)
            self.drive_service = build('drive', 'v3', credentials=credentials)
            
            return True
            
        except Exception as e:
            context.log('error', f"Failed to initialize Google services: {e}")
            return False
    
    async def send_email(self, to: str, subject: str, body: str, attachments: List[str] = None) -> bool:
        """Send email via Gmail"""
        try:
            import base64
            from email.mime.text import MIMEText
            from email.mime.multipart import MIMEMultipart
            from email.mime.base import MIMEBase
            from email import encoders
            
            # Create message
            message = MIMEMultipart()
            message['to'] = to
            message['subject'] = subject
            
            # Add body
            message.attach(MIMEText(body, 'html'))
            
            # Add attachments
            if attachments:
                for file_path in attachments:
                    with open(file_path, 'rb') as f:
                        part = MIMEBase('application', 'octet-stream')
                        part.set_payload(f.read())
                        encoders.encode_base64(part)
                        part.add_header(
                            'Content-Disposition',
                            f'attachment; filename= {os.path.basename(file_path)}'
                        )
                        message.attach(part)
            
            # Send message
            raw_message = base64.urlsafe_b64encode(message.as_bytes()).decode()
            
            result = self.gmail_service.users().messages().send(
                userId='me',
                body={'raw': raw_message}
            ).execute()
            
            return bool(result.get('id'))
            
        except Exception as e:
            raise RuntimeError(f"Gmail send failed: {e}")
    
    async def upload_to_drive(self, file_path: str, folder_id: str = None) -> str:
        """Upload file to Google Drive"""
        try:
            from googleapiclient.http import MediaFileUpload
            
            file_metadata = {
                'name': os.path.basename(file_path)
            }
            
            if folder_id:
                file_metadata['parents'] = [folder_id]
            
            media = MediaFileUpload(file_path, resumable=True)
            
            file = self.drive_service.files().create(
                body=file_metadata,
                media_body=media,
                fields='id'
            ).execute()
            
            return file.get('id')
            
        except Exception as e:
            raise RuntimeError(f"Drive upload failed: {e}")
```

## Plugin Configuration and Management

### 1. Plugin Configuration Schema

```yaml
# plugin.yaml - Plugin manifest file
name: "openai_provider"
version: "1.0.0"
description: "OpenAI GPT integration plugin"
author: "ERP System Team"
license: "MIT"

plugin_type: "ai_provider"
api_version: "1.0"

main_class: "OpenAIPlugin"
main_file: "openai_plugin.py"

dependencies:
  - "openai>=1.0.0"
  - "aiohttp>=3.8.0"

permissions:
  - "ai.generate"
  - "ai.embeddings"
  - "network.external"

config_schema:
  type: "object"
  properties:
    api_key:
      type: "string"
      description: "OpenAI API key"
      required: true
      sensitive: true
    model:
      type: "string"
      description: "Model to use"
      default: "gpt-4"
      enum: ["gpt-3.5-turbo", "gpt-4", "gpt-4-turbo"]
    temperature:
      type: "number"
      description: "Sampling temperature"
      default: 0.7
      minimum: 0.0
      maximum: 2.0
    max_tokens:
      type: "integer"
      description: "Maximum tokens to generate"
      default: 1000
      minimum: 1
      maximum: 4000

hooks:
  - name: "ai_request"
    description: "Handle AI generation requests"
    method: "generate_response"
  - name: "ai_embedding"
    description: "Generate text embeddings"
    method: "generate_embeddings"

health_check:
  endpoint: "/health"
  interval: 60
  timeout: 10

resources:
  memory_limit: "512MB"
  cpu_limit: "0.5"
  network_access: true
  file_system_access: false

security:
  sandbox: true
  network_isolation: false
  file_permissions: "read-only"
```

### 2. Plugin Registry and Discovery

```python
class PluginRegistry:
    """Central registry for plugin discovery and management"""
    
    def __init__(self):
        self.registered_plugins: Dict[str, Dict] = {}
        self.plugin_hooks: Dict[str, List[str]] = {}
        self.plugin_manager = PluginManager()
    
    async def register_plugin_directory(self, directory: str):
        """Register a directory for plugin discovery"""
        
        plugins = await self.plugin_manager.discover_plugins([directory])
        
        for plugin_name in plugins:
            plugin_path = os.path.join(directory, plugin_name)
            await self.register_plugin(plugin_name, plugin_path)
    
    async def register_plugin(self, plugin_name: str, plugin_path: str):
        """Register a single plugin"""
        
        try:
            # Load plugin manifest
            manifest_path = os.path.join(plugin_path, "plugin.yaml")
            with open(manifest_path, 'r') as f:
                manifest = yaml.safe_load(f)
            
            # Register plugin
            self.registered_plugins[plugin_name] = {
                'path': plugin_path,
                'manifest': manifest,
                'status': 'registered'
            }
            
            # Register hooks
            for hook in manifest.get('hooks', []):
                hook_name = hook['name']
                if hook_name not in self.plugin_hooks:
                    self.plugin_hooks[hook_name] = []
                self.plugin_hooks[hook_name].append(plugin_name)
            
        except Exception as e:
            self.log_error(f"Failed to register plugin {plugin_name}: {e}")
    
    async def get_plugins_for_hook(self, hook_name: str) -> List[str]:
        """Get plugins that implement a specific hook"""
        return self.plugin_hooks.get(hook_name, [])
    
    async def execute_hook(self, hook_name: str, *args, **kwargs) -> List[Any]:
        """Execute all plugins for a specific hook"""
        
        results = []
        plugins = await self.get_plugins_for_hook(hook_name)
        
        for plugin_name in plugins:
            try:
                if plugin_name in self.plugin_manager.plugins:
                    plugin = self.plugin_manager.plugins[plugin_name]
                    
                    # Find hook method
                    manifest = self.registered_plugins[plugin_name]['manifest']
                    hook_config = next(
                        (h for h in manifest.get('hooks', []) if h['name'] == hook_name),
                        None
                    )
                    
                    if hook_config:
                        method_name = hook_config['method']
                        method = getattr(plugin, method_name, None)
                        
                        if method:
                            result = await method(*args, **kwargs)
                            results.append({
                                'plugin': plugin_name,
                                'result': result,
                                'success': True
                            })
                
            except Exception as e:
                results.append({
                    'plugin': plugin_name,
                    'error': str(e),
                    'success': False
                })
        
        return results
```

This comprehensive plugin system provides secure, isolated, and manageable integration of AI services, Google integrations, and custom extensions into the ERP system.