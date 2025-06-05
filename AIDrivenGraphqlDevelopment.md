To provide a consolidated, complete end-to-end solution based on our discussion, I’ll compile the key components, code, and explanations into a single, cohesive response. This will include the optimized GraphQL MCP server and the enhanced LangGraph GraphQL agent with FastAPI, incorporating all discussed features (singleton instances, token refresh, retries, metrics, etc.). I’ll structure it clearly for reference and provide instructions to export it as a PDF for download.

However, since I cannot directly generate or serve a PDF file, I’ll format the content in a way that you can easily copy into a document editor (e.g., Microsoft Word, Google Docs) or a Markdown-to-PDF converter (e.g., Pandoc, Typora) to create a PDF. I’ll also include guidance on how to export the content as a PDF at the end.

---

# Consolidated End-to-End Solution: GraphQL MCP Server and LangGraph GraphQL Agent

This document provides a complete, production-grade solution for a GraphQL-based natural language query system, comprising two components:
1. **GraphQL MCP Server**: Handles GraphQL schema introspection, query generation, and execution using MCP (Multi-Cloud Platform).
2. **LangGraph GraphQL Agent**: A FastAPI-based agent that processes user queries, integrates with the MCP server, and exposes API endpoints.

The solution incorporates optimizations like singleton instances, Azure OpenAI token refresh, connection pooling, retries, schema caching, metrics, and detailed comments for developer clarity.

## Solution Overview
- **Repositories**:
  - `graphql-mcp-server`: MCP server with GraphQL tools and resources.
  - `langgraph-graphql-agent`: FastAPI-based LangGraph agent for query processing.
- **Features**:
  - **Singleton Instances**: Single instances for `GraphQLMCPServer` and `AzureChatOpenAI` to reduce overhead.
  - **Token Refresh**: Periodic refresh of Azure OpenAI tokens (placeholder; replace with `msal`).
  - **Production-Grade**: Retries, connection pooling, schema caching, metrics, and logging.
  - **FastAPI Endpoints**: Generic endpoints to process queries for any GraphQL endpoint.
- **Assumptions**:
  - Retail GraphQL schema with `Product`, `Order`, `Customer`, and `OrderFilter`.
  - Azure OpenAI for LLM and embeddings.
  - MCP server runs at `http://localhost:8000/mcp`.
- **Date/Time**: June 4, 2025, 09:22 AM IST.

## Repository 1: `graphql-mcp-server`

### Structure
```
graphql-mcp-server/
├── graphql_mcp_server.py
├── utils.py
├── config.py
├── .env
├── requirements.txt
└── README.md
```

### `requirements.txt`
```text
mcp
gql
aiohttp
python-dotenv
loguru
openai
tenacity
langchain_openai
```

### `.env`
```env
AZURE_OPENAI_API_KEY=your-azure-openai-api-key
AZURE_OPENAI_ENDPOINT=your-azure-openai-endpoint
AZURE_OPENAI_API_VERSION=2023-05-15
AZURE_OPENAI_DEPLOYMENT_NAME=your-deployment-name
GRAPHQL_ENDPOINT=https://retail-graphql.example.com/graphql
TOKEN_REFRESH_INTERVAL=1800
SCHEMA_CACHE_TTL=3600
```

### `config.py`
```python
from dotenv import load_dotenv
import os

load_dotenv()

class Config:
    """Centralized configuration for the GraphQL MCP server."""
    AZURE_OPENAI_API_KEY = os.getenv("AZURE_OPENAI_API_KEY")
    AZURE_OPENAI_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
    AZURE_OPENAI_API_VERSION = os.getenv("AZURE_OPENAI_API_VERSION")
    AZURE_OPENAI_DEPLOYMENT_NAME = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME")
    GRAPHQL_ENDPOINT = os.getenv("GRAPHQL_ENDPOINT")
    TOKEN_REFRESH_INTERVAL = int(os.getenv("TOKEN_REFRESH_INTERVAL", 1800))  # Token refresh interval in seconds
    SCHEMA_CACHE_TTL = int(os.getenv("SCHEMA_CACHE_TTL", 3600))  # Schema cache TTL in seconds
    MAX_RETRIES = 3  # Maximum retry attempts for transient failures
    RETRY_BACKOFF = 2  # Base backoff time for retries in seconds
    MAX_METADATA_EXAMPLES = 2  # Maximum metadata examples to store
    LLM_MODEL = "AzureChatOpenAI"  # LLM model name
```

### `utils.py`
```python
import re
import json
from loguru import logger
from openai import AsyncOpenAI
from dotenv import load_dotenv
import os
import numpy as np
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from datetime import datetime, timedelta
import threading
from config import Config

load_dotenv()

# Configure logging with rotation and debug level
logger.remove()
logger.add("graphql_agent.log", rotation="10 MB", level="DEBUG", format="{time} {level} {message}")

# Initialize OpenAI client for embeddings
openai_client = AsyncOpenAI(api_key=os.getenv("AZURE_OPENAI_API_KEY"), base_url=os.getenv("AZURE_OPENAI_ENDPOINT"))

class Metrics:
    """Tracks server performance metrics such as query counts and latency."""
    def __init__(self):
        self.lock = threading.Lock()  # Ensure thread-safe updates
        self.queries_executed = 0  # Total queries executed
        self.query_errors = 0  # Total query errors
        self.total_latency = 0.0  # Cumulative query latency in seconds

    def record_query(self, latency: float):
        """Records a successful query execution with its latency.
        
        Args:
            latency (float): Query execution time in seconds.
        """
        with self.lock:
            self.queries_executed += 1
            self.total_latency += latency

    def record_error(self):
        """Records a query error."""
        with self.lock:
            self.query_errors += 1

    def get_stats(self):
        """Returns current metrics as a dictionary.
        
        Returns:
            dict: Metrics including query count, error count, and average latency.
        """
        with self.lock:
            avg_latency = self.total_latency / self.queries_executed if self.queries_executed > 0 else 0
            return {
                "queries_executed": self.queries_executed,
                "query_errors": self.query_errors,
                "avg_query_latency_ms": avg_latency * 1000
            }

metrics = Metrics()

class SchemaCache:
    """Caches the GraphQL schema to avoid redundant introspection."""
    def __init__(self, ttl_seconds):
        self.cache = None  # Cached schema
        self.expiry = None  # Cache expiry time
        self.ttl = ttl_seconds  # Time-to-live in seconds
        self.lock = threading.Lock()  # Ensure thread-safe access

    def get(self):
        """Retrieves the cached schema if valid.
        
        Returns:
            dict or None: Cached schema or None if expired.
        """
        with self.lock:
            if self.cache and self.expiry > datetime.utcnow():
                return self.cache
            return None

    def set(self, schema):
        """Stores a schema in the cache with a TTL.
        
        Args:
            schema (dict): GraphQL schema to cache.
        """
        with self.lock:
            self.cache = schema
            self.expiry = datetime.utcnow() + timedelta(seconds=self.ttl)

schema_cache = SchemaCache(ttl_seconds=Config.SCHEMA_CACHE_TTL)

def validate_input(user_query: str):
    """Validates the user query for length and invalid characters.
    
    Args:
        user_query (str): User input query.
        
    Raises:
        ValueError: If query length is invalid or contains forbidden characters.
    """
    if not user_query or len(user_query) > 2000:
        logger.error("Invalid query length")
        raise ValueError("Query length must be between 1 and 2000 characters")
    if re.search(r'[<>{};]', user_query):
        logger.error("Query contains invalid characters")
        raise ValueError("Query contains invalid characters")
    logger.debug(f"Validated input: {user_query}")

def format_argument_value(arg_value, arg_type):
    """Formats a GraphQL argument value based on its type.
    
    Args:
        arg_value: Argument value to format.
        arg_type (str): GraphQL type (e.g., String, Int, [String]).
        
    Returns:
        str: Formatted argument value.
        
    Raises:
        Exception: If formatting fails.
    """
    try:
        if arg_type.endswith('!') or arg_type in ['String', 'ID']:
            return f'"{arg_value}"' if isinstance(arg_value, str) else str(arg_value)
        elif arg_type in ['Int', 'Float']:
            return str(arg_value)
        elif arg_type.startswith('[') and arg_type.endswith(']'):
            return f'[{", ".join(format_argument_value(v, arg_type[1:-1]) for v in arg_value)}]'
        elif arg_type == 'Boolean':
            return str(arg_value).lower()
        else:
            return f'{{ {", ".join(f"{k}: {format_argument_value(v, 'String')}" for k, v in arg_value.items())} }}'
        logger.debug(f"Formatted argument: {arg_value} as {arg_type}")
    except Exception as e:
        logger.error(f"Failed to format argument {arg_value}: {e}")
        raise

@retry(
    stop=stop_after_attempt(Config.MAX_RETRIES),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(Exception)
)
async def get_embedding(text: str) -> list:
    """Generates an embedding for the given text using OpenAI.
    
    Args:
        text (str): Text to embed.
        
    Returns:
        list: Embedding vector.
        
    Raises:
        Exception: If embedding generation fails after retries.
    """
    try:
        response = await openai_client.embeddings.create(input=text, model="text-embedding-ada-002")
        embedding = response.data[0].embedding
        logger.debug(f"Generated embedding for text: {text[:50]}...")
        return embedding
    except Exception as e:
        logger.error(f"Failed to generate embedding: {e}")
        raise

def cosine_similarity(vec1: list, vec2: list) -> float:
    """Computes cosine similarity between two vectors.
    
    Args:
        vec1 (list): First vector.
        vec2 (list): Second vector.
        
    Returns:
        float: Cosine similarity score (0.0 if computation fails).
    """
    try:
        vec1 = np.array(vec1)
        vec2 = np.array(vec2)
        return np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2))
    except Exception as e:
        logger.error(f"Failed to compute cosine similarity: {e}")
        return 0.0

def format_metadata_for_prompt(initial_metadata: list, query_metadata: list):
    """Formats metadata for inclusion in LLM prompts.
    
    Args:
        initial_metadata (list): Static metadata with entity/attribute descriptions.
        query_metadata (list): Dynamic user-confirmed query metadata.
        
    Returns:
        str: JSON-formatted metadata string.
    """
    try:
        examples = []
        for meta in initial_metadata[:2]:
            examples.append({
                "graphql_query": meta["graphql_query"],
                "entity_descriptions": meta["entity_descriptions"],
                "attribute_descriptions": meta["attribute_descriptions"]
            })
        for meta in query_metadata[:2]:
            examples.append({
                "user_query": meta["user_query"],
                "entities": meta["entities"],
                "arguments": meta["arguments"],
                "graphql_query": meta["graphql_query"]
            })
        return json.dumps(examples, indent=2)
    except Exception as e:
        logger.error(f"Failed to format metadata: {e}")
        return ""

def get_default_fields(schema, entity):
    """Selects default scalar fields for an entity if none specified.
    
    Args:
        schema (dict): GraphQL schema.
        entity (str): Entity name (e.g., Product).
        
    Returns:
        list: List of default field names (up to 3).
    """
    try:
        for t in schema['__schema']['types']:
            if t['name'] == entity and t['kind'] == 'OBJECT':
                fields = [f['name'] for f in t.get('fields', []) if f['type'].get('kind') != 'OBJECT']
                return fields[:3]
        return []
    except Exception as e:
        logger.error(f"Failed to get default fields for {entity}: {e}")
        return []

def infer_default_arguments(user_query, schema, entity):
    """Infers default arguments for ambiguous queries (e.g., 'recent').
    
    Args:
        user_query (str): User query.
        schema (dict): GraphQL schema.
        entity (str): Entity name (e.g., Order).
        
    Returns:
        dict: Default arguments (e.g., createdAt filter).
    """
    try:
        defaults = {}
        if "recent" in user_query.lower():
            for t in schema['__schema']['types']:
                if t['kind'] == 'INPUT_OBJECT' and t['name'].startswith(entity):
                    for f in t.get('inputFields', []):
                        if f['name'] == 'createdAt' and 'DateTime' in f['type'].get('name', ''):
                            defaults['createdAt'] = {'gte': '2025-05-01'}
        return defaults
    except Exception as e:
        logger.error(f"Failed to infer default arguments for {entity}: {e}")
        return {}
```

### `graphql_mcp_server.py`
```python
import asyncio
import json
from mcp.server.fastmcp import FastMCP
from gql import Client, gql
from gql.transport.aiohttp import AIOHTTPTransport
from langchain_openai import AzureChatOpenAI
from config import Config
from utils import validate_input, logger, format_argument_value, format_metadata_for_prompt, get_embedding, cosine_similarity, get_default_fields, infer_default_arguments, metrics, schema_cache
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import time
import threading

class GraphQLMCPServer:
    """Singleton server for handling GraphQL queries via MCP."""
    _instance = None
    _lock = threading.Lock()

    @classmethod
    def get_instance(cls):
        """Returns the singleton instance of GraphQLMCPServer.
        
        Returns:
            GraphQLMCPServer: Singleton instance.
        """
        with cls._lock:
            if cls._instance is None:
                cls._instance = cls(
                    endpoint=Config.GRAPHQL_ENDPOINT,
                    azure_api_key=Config.AZURE_OPENAI_API_KEY,
                    azure_endpoint=Config.AZURE_OPENAI_ENDPOINT,
                    azure_api_version=Config.AZURE_OPENAI_API_VERSION,
                    deployment_name=Config.AZURE_OPENAI_DEPLOYMENT_NAME
                )
                # Start token refresh in background
                asyncio.create_task(cls._instance._refresh_token_loop())
            return cls._instance

    def __init__(self, endpoint, azure_api_key, azure_endpoint, azure_api_version, deployment_name):
        """Initializes the server with GraphQL and LLM configurations.
        
        Args:
            endpoint (str): GraphQL API endpoint.
            azure_api_key (str): Azure OpenAI API key.
            azure_endpoint (str): Azure OpenAI endpoint.
            azure_api_version (str): Azure OpenAI API version.
            deployment_name (str): Azure OpenAI deployment name.
            
        Raises:
            RuntimeError: If instantiated directly instead of via get_instance().
        """
        if GraphQLMCPServer._instance is not None:
            raise RuntimeError("Use get_instance() to access GraphQLMCPServer")
        self.endpoint = endpoint
        self.transport = AIOHTTPTransport(url=endpoint, timeout=30)  # Enable connection pooling
        self.client = Client(transport=self.transport, fetch_schema_from_transport=True)
        self.llm = AzureChatOpenAI(
            openai_api_key=azure_api_key,
            azure_endpoint=azure_endpoint,
            openai_api_version=azure_api_version,
            deployment_name=deployment_name,
            temperature=0
        )
        self.token = azure_api_key
        self.token_expiry = time.time() + Config.TOKEN_REFRESH_INTERVAL
        logger.info(f"Initialized GraphQLMCPServer for endpoint: {endpoint}")

    async def _refresh_token(self):
        """Refreshes the Azure OpenAI token (placeholder).
        
        Updates the LLM instance with the new token.
        
        Raises:
            Exception: If token refresh fails.
        """
        try:
            # Placeholder: Replace with Azure token refresh (e.g., msal)
            new_token = Config.AZURE_OPENAI_API_KEY
            self.token = new_token
            self.token_expiry = time.time() + Config.TOKEN_REFRESH_INTERVAL
            self.llm = AzureChatOpenAI(
                openai_api_key=new_token,
                azure_endpoint=Config.AZURE_OPENAI_ENDPOINT,
                openai_api_version=Config.AZURE_OPENAI_API_VERSION,
                deployment_name=Config.AZURE_OPENAI_DEPLOYMENT_NAME,
                temperature=0
            )
            logger.info("Refreshed Azure OpenAI token")
        except Exception as e:
            logger.error(f"Failed to refresh token: {e}")
            raise

    async def _refresh_token_loop(self):
        """Runs a background loop to refresh the token periodically."""
        while True:
            await asyncio.sleep(Config.TOKEN_REFRESH_INTERVAL)
            await self._refresh_token()

    @retry(
        stop=stop_after_attempt(Config.MAX_RETRIES),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type(Exception)
    )
    async def introspect_schema(self):
        """Introspects the GraphQL schema from the endpoint.
        
        Returns:
            dict: Introspected schema.
            
        Raises:
            Exception: If introspection fails after retries.
        """
        try:
            cached_schema = schema_cache.get()
            if cached_schema:
                logger.debug("Using cached schema")
                return cached_schema

            query = gql("""
                query IntrospectionQuery {
                    __schema {
                        queryType { name }
                        types {
                            name
                            kind
                            fields {
                                name
                                args { name type { name kind ofType { name kind } } }
                                type { name kind ofType { name kind fields { name type { name kind ofType { name kind } } } }
                            }
                            inputFields { name type { name kind ofType { name kind } } }
                        }
                    }
                }
            """)
            logger.debug("Executing schema introspection query")
            start_time = time.time()
            result = await self.client.execute_async(query)
            latency = time.time() - start_time
            metrics.record_query(latency)
            schema_cache.set(result)
            logger.info("Schema introspected")
            return result
        except Exception as e:
            metrics.record_error()
            logger.error(f"Failed to introspect schema: {e}")
            raise

    async def extract_query_entities(self, user_query: str, schema: dict, initial_metadata: list, query_metadata: list) -> dict:
        """Extracts entities and fields from a user query.
        
        Args:
            user_query (str): User input query.
            schema (dict): GraphQL schema.
            initial_metadata (list): Static metadata.
            query_metadata (list): Dynamic metadata.
            
        Returns:
            dict: Entities and their fields.
            
        Raises:
            Exception: If entity extraction fails.
        """
        try:
            validate_input(user_query)
            user_embedding = await get_embedding(user_query)
            entity_scores = []
            for meta in initial_metadata + query_metadata:
                for entity, desc in meta.get("entity_descriptions", {}).items():
                    desc_embedding = await get_embedding(desc)
                    score = cosine_similarity(user_embedding, desc_embedding)
                    entity_scores.append((entity, score))
            entity_scores.sort(key=lambda x: x[1], reverse=True)
            top_entities = [e[0] for e in entity_scores[:3]]

            prompt = await mcp.get_prompt("extract_entities_prompt")(
                schema=self._format_schema(schema, include_nested=True),
                metadata=format_metadata_for_prompt(initial_metadata, query_metadata),
                user_query=user_query
            )
            logger.debug(f"Extracting entities for query: {user_query}")
            start_time = time.time()
            response = await self.llm.ainvoke(prompt)
            latency = time.time() - start_time
            metrics.record_query(latency)
            result = json.loads(response.content)

            for entity in result["entities"]:
                if not result["fields"].get(entity):
                    result["fields"][entity] = get_default_fields(schema, entity)
            result["entities"] = [e for e in result["entities"] if e in top_entities]
            logger.info(f"Extracted entities: {result}")
            return result
        except Exception as e:
            metrics.record_error()
            logger.error(f"Failed to extract entities: {e}")
            raise

    async def extract_arguments(self, user_query: str, schema: dict, entities: list, initial_metadata: list, query_metadata: list) -> dict:
        """Extracts arguments for entities from a user query.
        
        Args:
            user_query (str): User input query.
            schema (dict): GraphQL schema.
            entities (list): List of entity names.
            initial_metadata (list): Static metadata.
            query_metadata (list): Dynamic metadata.
            
        Returns:
            dict: Arguments for each entity.
            
        Raises:
            Exception: If argument extraction fails.
        """
        try:
            user_embedding = await get_embedding(user_query)
            attr_scores = []
            for meta in initial_metadata + query_metadata:
                for attr, desc in meta.get("attribute_descriptions", {}).items():
                    desc_embedding = await get_embedding(desc)
                    score = cosine_similarity(user_embedding, desc_embedding)
                    attr_scores.append((attr, score))
            attr_scores.sort(key=lambda x: x[1], reverse=True)

            prompt = await mcp.get_prompt("extract_arguments_prompt")(
                schema=self._format_schema(schema, include_args=True),
                metadata=format_metadata_for_prompt(initial_metadata, query_metadata),
                entities=entities,
                user_query=user_query
            )
            logger.debug(f"Extracting arguments for entities: {entities}")
            start_time = time.time()
            response = await self.llm.ainvoke(prompt)
            latency = time.time() - start_time
            metrics.record_query(latency)
            result = json.loads(response.content)

            for entity in entities:
                if not result.get(entity):
                    result[entity] = {"filter": infer_default_arguments(user_query, schema, entity)}
            logger.info(f"Extracted arguments: {result}")
            return result
        except Exception as e:
            metrics.record_error()
            logger.error(f"Failed to extract arguments: {e}")
            raise

    async def build_graphql_query(self, entities: dict, arguments: dict, schema: dict, initial_metadata: list, query_metadata: list, error_feedback: str = "") -> str:
        """Builds a GraphQL query from entities and arguments.
        
        Args:
            entities (dict): Entities and their fields.
            arguments (dict): Arguments for entities.
            schema (dict): GraphQL schema.
            initial_metadata (list): Static metadata.
            query_metadata (list): Dynamic metadata.
            error_feedback (str): Validation error feedback.
            
        Returns:
            str: GraphQL query string.
            
        Raises:
            Exception: If query building fails.
        """
        try:
            prompt = await mcp.get_prompt("build_query_prompt")(
                schema=self._format_schema(schema, include_nested=True),
                metadata=format_metadata_for_prompt(initial_metadata, query_metadata),
                entities=entities,
                arguments=arguments,
                error_feedback=error_feedback
            )
            logger.debug(f"Building GraphQL query with entities: {entities}")
            start_time = time.time()
            response = await self.llm.ainvoke(prompt)
            latency = time.time() - start_time
            metrics.record_query(latency)
            query = response.content.strip()
            logger.info(f"Built GraphQL query: {query}")
            return query
        except Exception as e:
            metrics.record_error()
            logger.error(f"Failed to build GraphQL query: {e}")
            raise

    @retry(
        stop=stop_after_attempt(Config.MAX_RETRIES),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type(Exception)
    )
    async def validate_graphql_query(self, query: str) -> tuple[bool, str]:
        """Validates a GraphQL query against the endpoint.
        
        Args:
            query (str): GraphQL query string.
            
        Returns:
            tuple: (is_valid, error_message).
            
        Raises:
            Exception: If validation fails after retries.
        """
        try:
            gql_query = gql(query)
            start_time = time.time()
            await self.client.execute_async(gql_query)
            latency = time.time() - start_time
            metrics.record_query(latency)
            logger.info(f"Validated GraphQL query: {query}")
            return True, ""
        except Exception as e:
            metrics.record_error()
            logger.error(f"Invalid GraphQL query: {query}, Error: {e}")
            return False, str(e)

    @retry(
        stop=stop_after_attempt(Config.MAX_RETRIES),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type(Exception)
    )
    async def execute_graphql_query(self, query: str) -> dict:
        """Executes a validated GraphQL query.
        
        Args:
            query (str): GraphQL query string.
            
        Returns:
            dict: Query response.
            
        Raises:
            ValueError: If execution fails after retries.
        """
        try:
            gql_query = gql(query)
            start_time = time.time()
            response = await self.client.execute_async(gql_query)
            latency = time.time() - start_time
            metrics.record_query(latency)
            logger.info(f"Query executed: {query}")
            logger.debug(f"Query response: {response}")
            return response
        except Exception as e:
            metrics.record_error()
            logger.error(f"Error executing GraphQL query: {query}, Error: {str(e)}")
            raise ValueError(f"Query execution failed: {str(e)}")

    async def update_initial_metadata(self, schema: dict):
        """Updates initial metadata with new entities and attributes.
        
        Args:
            schema (dict): GraphQL schema.
            
        Raises:
            Exception: If metadata update fails.
        """
        try:
            metadata = await mcp.get_resource("initial_metadata")
            existing_entities = {k for meta in metadata["examples"] for k in meta["entity_descriptions"].keys()}
            new_entities = set()
            new_attributes = set()

            for t in schema['__schema']['types']:
                if t['kind'] == 'OBJECT' and t['name'] not in ['Query', '__Schema', '__Type']:
                    new_entities.add(t['name'])
                    for f in t.get('fields', []):
                        new_attributes.add(f"{t['name']}.{f['name']}")
                elif t['kind'] == 'INPUT_OBJECT':
                    for f in t.get('inputFields', []):
                        new_attributes.add(f"{t['name']}.{f['name']}")

            for entity in new_entities - existing_entities:
                metadata["examples"].append({
                    "graphql_query": "",
                    "entity_descriptions": {entity: f"Represents a {entity.lower()} in the retail system."},
                    "attribute_descriptions": {}
                })
            for meta in metadata["examples']:
                for attr in new_attributes:
                    if attr.split('.')[0] in meta["entity_descriptions"] and attr not in meta["attribute_descriptions"]:
                        meta["attribute_descriptions"][attr] = f"Represents the {attr.split('.')[1]} of {attr.split('.')[0]}."
            await mcp.update_resource("initial_metadata", metadata)
            logger.info("Updated initial_metadata with new entities/attributes")
        except Exception as e:
            metrics.record_error()
            logger.error(f"Failed to update initial_metadata: {e}")
            raise

    def _format_schema(self, schema, include_nested=False, include_args=False):
        """Formats the GraphQL schema for LLM prompts.
        
        Args:
            schema (dict): GraphQL schema.
            include_nested (bool): Whether to include nested fields.
            include_args (bool): Whether to include input arguments.
            
        Returns:
            str: Formatted schema string.
            
        Raises:
            Exception: If formatting fails.
        """
        try:
            query_type = schema['__schema']['query']['name']
            types = schema['__schema']['types']
            formatted = f"Query Type: {query_type}\n"
            for t in types:
                if t['kind'] == 'OBJECT' and t['name'] == query_type:
                    formatted += "Fields:\n"
                    for field in t.get('fields', []):
                        args = [f"{arg['name']}: {self._format_type(arg['type'])}" for arg in field.get('args', [])]
                        field_type = self._format_type(field['type'])
                        formatted += f" {field['name']}({', '.join(args)}): {field_type}\n"
                        if include_nested and field['type'].get('kind') == 'OBJECT':
                            nested_fields = self._get_nested_fields(field['type']['name'])
                            formatted += f" Nested fields for {field['type']['name']}:\n"
                            formatted += "\n".join(f"      {nf}" for nf in nested_fields) + "\n"
                elif t['kind'] == 'INPUT_OBJECT' and include_args:
                    formatted += f"Input Type: {t['name']}\n"
                    for input_field in t.get('inputFields', []):
                        formatted += f" {input_field['name']}: {self._format_type(input_field['type'])}\n"
            logger.debug("Formatted schema for LLM prompt")
            return formatted
        except Exception as e:
            logger.error(f"Failed to format schema: {e}")
            raise

    def _format_type(self, type_info):
        """Formats a GraphQL type for display.
        
        Args:
            type_info (dict): Type information.
            
        Returns:
            str: Formatted type string.
            
        Raises:
            Exception: If type formatting fails.
        """
        try:
            if type_info.get('kind') == 'NON_NULL':
                return f"{self._format_type(type_info['ofType'])}!"
            if type_info.get('kind') == 'LIST':
                return f"[{self._format_type(type_info['ofType'])}]"
            return type_info.get('name', '')
        except Exception as e:
            logger.error(f"Failed to format type: {e}")
            raise

    def _get_nested_fields(self, schema, type_name):
        """Retrieves scalar fields for a nested object type.
        
        Args:
            schema (dict): GraphQL schema.
            type_name (str): Object type name.
            
        Returns:
            list: Scalar field names.
            
        Raises:
            Exception: If field retrieval fails.
        """
        try:
            for t in schema['__schema']['types']:
                if t['name'] == type_name and t['kind'] == 'OBJECT':
                    return [f['name'] for f in t.get('fields', []) if f['type'].get('kind') != 'OBJECT']
            return []
        except Exception as e:
            logger.error(f"Failed to get nested fields for {type_name}: {e}")
            raise

# Initial Metadata
INITIAL_METADATA = {
    "examples": [
        {
            "graphql_query": "query {\n  allProducts(filter: {maxPrice: 100}) {\n    id\n    name\n    price\n  }\n}",
            "entity_descriptions": {
                "Product": "Represents a product with details like name, price, and category."
            },
            "attribute_descriptions": {
                "Product.id": "Unique identifier for a product.",
                "Product.name": "Name of the product.",
                "Product.price": "Price of the product.",
                "ProductFilter.maxPrice": "Maximum price for filtering products."
            }
        },
        {
            "graphql_query": "query {\n  allOrders(filter: {customerCity: \"New York\", minProductPrice: 50}) {\n    id\n    customer {\n      name\n      city\n    }\n    products {\n      name\n      price\n    }\n}",
            "entity_descriptions": {
                "Order": "Represents a customer purchase with details like customer and products.",
                "Customer": "Represents a customer with details like name and city.",
                "Product": "Represents a product with details like name, price, and category."
            },
            "attribute_descriptions": {
                "Order.id": "Unique identifier for an order.",
                "Order.customer": "The customer who placed the order.",
                "Order.products": "List of products in the order.",
                "OrderFilter.customerCity": "The city of the customer placing the order.",
                "OrderFilter.minProductPrice": "Minimum price of products in the order.",
                "Customer.name": "Name of the customer.",
                "Customer.city": "City of the customer.",
                "Product.name": "Name of the product.",
                "Product.price": "Price of the product."
            }
        },
        {
            "graphql_query": "query {\n  allOrders(filter: {createdAt: {gte: \"2025-05-01\"}}) {\n    id\n    total\n  }\n}",
            "entity_descriptions": {
                "Order": "Represents a customer purchase with details like customer and products."
            },
            "attribute_descriptions": {
                "Order.id": "Unique identifier for an order.",
                "Order.total": "Total amount of the order.",
                "OrderFilter.createdAt": "Date when the order was created, used for filtering recent orders."
            }
        }
    ]
}

# Initialize MCP server
mcp = FastMCP("GraphQLAgent")

# MCP Prompts
@mcp.prompt()
def extract_entities_prompt(schema: str, metadata: str, user_query: str) -> str:
    """Prompt for extracting entities and fields from a user query."""
    return f"""
    Given a GraphQL schema, a user query, and metadata with entity/attribute descriptions, identify the relevant entities and fields.
    Schema (simplified):
    {schema}
    
    Metadata (with descriptions and examples):
    {metadata}
    
    User Query: {user_query}
    
    Instructions:
    - If the query is ambiguous (e.g., "customer details"), select common fields like id, name.
    - For terms like "recent", infer time-based filters (e.g., last 30 days).
    - Return a JSON object with:
      - entities: List of entity names (e.g., ["Order", "Customer", "Product"])
      - fields: Dictionary mapping each entity to a list of relevant field names
    """

@mcp.prompt()
def extract_arguments_prompt(schema: str, metadata: str, entities: list, user_query: str) -> str:
    """Prompt for extracting arguments for entities."""
    return f"""
    Given a GraphQL schema, a user query, entities, and metadata with descriptions, extract required arguments.
    Schema (simplified):
    {schema}
    
    Metadata (with descriptions and examples):
    {metadata}
    
    Entities: {entities}
    User Query: {user_query}
    
    Instructions:
    - If arguments are missing or ambiguous (e.g., "recent orders"), infer defaults (e.g., createdAt filter for last 30 days).
    - Use metadata examples to guide argument selection.
    - Output a JSON object mapping each entity to its required arguments and values.
      Example: {{"Order": {{"filter": {{"customerCity": "New York", "minPrice": 50}}}}}}
    """

@mcp.prompt()
def build_query_prompt(schema: str, metadata: str, entities: dict, arguments: dict, error_feedback: str = "") -> str:
    """Prompt for building a GraphQL query."""
    return f"""
    Given entities, arguments, a GraphQL schema, metadata, and optional error feedback, build a valid GraphQL query.
    Schema (simplified):
    {schema}
    
    Metadata (with descriptions and examples):
    {metadata}
    
    Entities: {json.dumps(entities)}
    Arguments: {json.dumps(arguments)}
    
    Error Feedback (if any): {error_feedback}
    
    Instructions:
    - Ensure the query is valid per the schema.
    - If error feedback is provided, fix issues (e.g., incorrect field names, missing required arguments).
    - Output a valid GraphQL query string with proper nesting and argument formatting.
    """

# MCP Resources
@mcp.resource()
async def graphql_schema():
    """Stores the introspected GraphQL schema."""
    server = GraphQLMCPServer.get_instance()
    return await server.introspect_schema()

@mcp.resource()
async def config():
    """Stores configuration settings for the server."""
    return {
        "graphql_endpoint": Config.GRAPHQL_ENDPOINT,
        "llm_model": Config.LLM_MODEL,
        "max_metadata_examples": Config.MAX_METADATA_EXAMPLES,
        "token_refresh_interval": Config.TOKEN_REFRESH_INTERVAL,
        "schema_cache_ttl": Config.SCHEMA_CACHE_TTL,
        "max_retries": Config.MAX_RETRIES
    }

@mcp.resource()
async def initial_metadata():
    """Stores static metadata with entity/attribute descriptions."""
    return INITIAL_METADATA

@mcp.resource()
async def query_metadata():
    """Stores dynamic user-confirmed queries metadata."""
    return {"examples": []}

# MCP Tools
@mcp.tool()
async def introspect_schema() -> dict:
    """Introspects the GraphQL schema and updates resources."""
    server = GraphQLMCPServer.get_instance()
    schema = await server.introspect_schema()
    await mcp.update_resource("graphql_schema", schema)
    await server.update_initial_metadata(schema)
    return schema

@mcp.tool()
async def extract_query_entities(user_query: str) -> dict:
    """Extracts entities and fields from a user query."""
    server = GraphQLMCPServer.get_instance()
    schema = await mcp.get_resource("graphql_schema")
    initial_metadata = (await mcp.get_resource("initial_metadata"))["examples"]
    query_metadata = (await mcp.get_resource("query_metadata"))["examples"]
    return await server.extract_query_entities(user_query, schema, initial_metadata, query_metadata)

@mcp.tool()
async def extract_arguments(user_query: str, entities: dict) -> dict:
    """Extracts arguments for entities from a user query."""
    server = GraphQLMCPServer.get_instance()
    schema = await mcp.get_resource("graphql_schema")
    initial_metadata = (await mcp.get_resource("initial_metadata"))["examples"]
    query_metadata = (await mcp.get_resource("query_metadata"))["examples"]
    return await server.extract_arguments(user_query, schema, entities['entities'], initial_metadata, query_metadata)

@mcp.tool()
async def build_graphql_query(entities: dict, arguments: dict, error_feedback: str = "") -> str:
    """Builds a GraphQL query from entities and arguments."""
    server = GraphQLMCPServer.get_instance()
    schema = await mcp.get_resource("graphql_schema")
    initial_metadata = (await mcp.get_resource("initial_metadata"))["examples"]
    query_metadata = (await mcp.get_resource("query_metadata"))["examples"]
    return await server.build_graphql_query(entities, arguments, schema, initial_metadata, query_metadata, error_feedback)

@mcp.tool()
async def validate_graphql_query(query: str) -> tuple[bool, str]:
    """Validates a GraphQL query."""
    server = GraphQLMCPServer.get_instance()
    return await server.validate_graphql_query(query)

@mcp.tool()
async def execute_graphql_query(query: str) -> dict:
    """Executes a validated GraphQL query."""
    server = GraphQLMCPServer.get_instance()
    return await server.execute_graphql_query(query)

@mcp.tool()
async def confirm_response(user_query: str, entities: dict, arguments: dict, graphql_query: str, response: dict):
    """Confirms a response and updates query metadata."""
    try:
        metadata = await mcp.get_resource("query_metadata")
        metadata["examples"].append({
            "user_query": user_query,
            "entities": entities,
            "arguments": arguments,
            "graphql_query": graphql_query,
            "response": response
        })
        config = await mcp.get_resource("config")
        max_examples = config.get("max_metadata_examples", 2)
        metadata["examples"] = metadata["examples"][-max_examples:]
        await mcp.update_resource("query_metadata", metadata)
        logger.info(f"Updated query_metadata with new example for query: {user_query}")
    except Exception as e:
        metrics.record_error()
        logger.error(f"Failed to confirm response: {response}")
        raise

@mcp.tool()
async def update_initial_metadata():
    """Updates initial metadata with new entities/attributes."""
    server = GraphQLMCPServer.get_instance()
    schema = await mcp.get_resource("graphql_schema")
    await server.update_initial_metadata(schema)

@mcp.tool()
async def get_metrics():
    """Returns server performance metrics."""
    return metrics.get_stats()

if __name__ == "__main__":
    logger.info("Starting GraphQL MCP server")
    mcp.run(transport="streamable-http")
```

### `README.md`
```markdown
# GraphQL MCP Server

A production-grade MCP server for translating natural language queries into GraphQL queries.

## Setup
1. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Configure `.env` with Azure OpenAI and GraphQL endpoint details.
3. Run the server:
   ```bash
   python graphql_mcp_server.py
   ```

## Features
- CLI singleton `GraphQLMCPServer` for efficiency.
- CLI-based token refresh for Azure CLI OpenAI.
- CLI-based connection pooling, retries, schema caching, and metrics.
- CLI-based detailed comments for developer clarity.
```

---

## Repository 2: `langgraph-graphql-agent`

### Structure
```
langgraph-graphql-agent/
├── app/
│   ├── __init__.py
│   ├── agent.py
│   ├── api.py
│   └── models.py
├── utils.py
├── config.py
├── .env
├── requirements.txt
├── main.py
└── README.md
```

### `requirements.txt`
```text
langgraph
langchain
langchain_openai
langchain_mcp_adapters
python-dotenv
loguru
openai
tenacity
fastapi
uvicorn
pydantic
```

### `.env`
```env
AZURE_OPENAI_API_KEY=your-azure-openai-api-key
AZURE_OPENAI_ENDPOINT=https://your-azure-openai-endpoint
AZURE_OPENAI_API_VERSION=2023-05-15
AZURE_OPENAI_DEPLOYMENT_NAME=your-deployment-name
GRAPHQL_ENDPOINT=https://retail-graphql.com/graphql
TOKEN_REFRESH_INTERVAL=1800
MCP_SERVER_URL=http://localhost:8000/mcp
API_PORT=8001
```

### `config.py`
```python
from dotenv import load_dotenv
import os

load_dotenv()

class Config:
    """Centralized configuration for the LangGraph agent."""
    AZURE_OPENAI_API_KEY = os.getenv("AZURE_OPENAI_API_KEY")
    AZURE_OPENAI_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
    AZURE_OPENAI_API_VERSION = os.getenv("AZURE_OPENAI_VERSION")
    AZURE_OPENAI_DEPLOYMENT_NAME = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME")
    GRAPHQL_ENDPOINT = os.getenv("GRAPHQL_ENDPOINT")
    TOKEN_REFRESH_INTERVAL = int(os.getenv("TOKEN_REFRESH_INTERVAL", 1800))  # Token refresh interval in seconds
    MAX_RETRIES = 3  # Maximum retry attempts for transient failures
    RETRY_BACKOFF = 30  # Base backoff time for retries in seconds
    MCP_SERVER_URL = os.getenv("MCP_SERVER_URL", "http://localhost:8000/mcp")  # MCP server URL
    API_PORT = int(os.getenv("API_PORT", 8001))  # FastAPI port
    LLM_MODEL = "AzureChatOpenAI"  # LLM model name
```

### `utils.py`
```python
import re
from loguru import logger
from openai import AsyncOpenAI
from dotenv import load_dotenv
import os
import numpy as np
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from datetime import datetime, timedelta
import threading
from config import Config

load_dotenv()

# Configure logging with rotation and debug level
logger.remove()
logger.add("graphql-agent.log", rotation="10 MB", level="DEBUG", format="{time} {level} {message}")

# Initialize OpenAI client for embeddings
openai_client = AsyncOpenAI(api_key=os.getenv("AZURE_OPENAI_API_KEY"), base_url=os.getenv("AZURE_OPENAI_ENDPOINT")))

class Metrics:
    """Tracks agent performance metrics such as query counts and latency."""
    def __init__(self):
        self.lock = threading.Lock()  # Ensure thread-safe updates
        self.queries_executed = 0  # Total queries executed
        self.query_errors = 0  # Total query errors
        self.total_latency = 0.0  # Cumulative query latency in seconds

    def record_query(self, latency: float):
        """Records a successful query execution with its latency.
        
        Args:
            latency (float): Query execution time in seconds.
        """
        """
        with self.lock:
            self.queries += 1
            self.total_latency += latency

    def record_error(self):
        """Records a query error."""
        with self.lock:
            self.query_errors += 1

    def get_metrics(self):
        """Returns the current metrics as a dictionary.
        
        Returns:
            dict: Metrics including query count, error count, and average latency.
        """
        with self.lock:
            avg_latency = self.total_latency / self.queries_executed if self.queries_executed > 0 else 0
            return {
                "queries_executed": self.queries_executed,
                "query_errors": self.query_errors,
                "avg_query_latency_ms": avg_latency * 1000
            }

metrics = Metrics()

def validate_input(user_query: str):
    """Validates the user query for length and invalid characters.
    
    Args:
        user_query (str): User input query.
        
    Raises:
        ValueError: If query length is invalid or contains forbidden characters.
    """
    if not user_query or len(user_query) > 2000:
        logger.error("Invalid query length")
        raise ValueError("Query length must be between 1 and 2000 characters")
    if re.search(r'[<>{}]', user_query):
        logger.error("Query contains invalid characters")
        raise ValueError("Query contains invalid characters")
    logger.debug(f"Validated input: {user_query}")

@retry(
    stop=stop_after_attempt(Config.MAX_RETRIES),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(Exception)
)
async def get_embedding(text: str) -> list:
    """Generates an async embedding for a given text using OpenAI.
    
    Args:
        text (str): Text to embed.
        
    Returns:
        list: Embedding vector.
    """
    try:
        response = await openai_client.embeddings.create(input=text, model="text-embedding-ada-002")
        embedding = response.data[0].embedding
        logger.debug(f"Generated embedding for text: {text[:50]}...")
        return embedding
    except Exception as e:
        logger.error(f"Failed to generate embedding: {e}")
        raise

def cosine_similarity(vec1: list, vec2: list) -> float:
    """Computes cosine similarity between two vectors.
    
    Args:
        vec1 (list): First vector.
        vec2 (list): Second vector.
        
    Returns:
        float: Cosine similarity score (0.0 if computation fails).
    """
    try:
        vec1 = np.array(vec1)
        vec2 = np.array(vec2)
        return np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2))
    except Exception as e:
        logger.error(f"Failed to compute cosine similarity: {e}")
        return 0.0
```

### `app/models.py`
```python
from pydantic import BaseModel, HttpUrl, Field
from typing import Optional

class GraphQLQueryRequest(BaseModel):
    """Request model for GraphQL query endpoint."""
    user_query: str = Field(..., description="Natural language query from the user")
    graphql_endpoint: HttpUrl = Field(..., description="GraphQL API endpoint URL")

class GraphQLQueryResponse(BaseModel):
    """Response model for GraphQL query endpoint."""
    graphql_query: str = Field(..., description="Generated GraphQL query")
    result: dict = Field(..., description="Query execution result")
    entities: dict = Field(..., description="Extracted entities and fields")
    arguments: dict = Field(..., description="Extracted arguments")
    clarified_query: Optional[str] = Field(None, description="Clarified user query if ambiguous")

class MetricsResponse(BaseModel):
    """Response model for metrics endpoint."""
    queries_executed: int = Field(..., description="Total queries executed")
    query_errors: int = Field(..., description="Total query errors")
    avg_query_latency_ms: float = Field(..., description="Average query latency in milliseconds")

class HealthResponse(BaseModel):
    """Response model for health endpoint."""
    status: str = Field(..., description="Agent health status")
```

### `app/agent.py`
```python
import asyncio
import time
from langchain_openai import AzureChatOpenAI
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent
from config import Config
from utils import validate_input, logger, metrics
from threading import Lock

class LangGraphAgent:
    """Singleton class for managing the LangGraph GraphQL agent."""
    _instance = None
    _lock = Lock()

    @classmethod
    def get_instance(cls):
        """Returns the singleton instance of LangGraphAgent.
        
        Returns:
            LangGraphAgent: Singleton instance.
        """
        with cls._lock:
            if cls._instance is None:
                cls._instance = cls()
                # Start token refresh in background task
                asyncio.create_task(cls._instance._refresh_token_loop())
            return cls._instance

    def __init__(self):
        """Initializes the agent with LLM configuration.
        
        Raises:
            RuntimeError: If instantiated directly.
        """
        if LangGraphAgent._instance is not None:
            raise RuntimeError("Use get_instance() to access LangGraphAgent")
        self.llm = AzureChatOpenAI(
            openai_api_key=Config.AZURE_OPENAI_API_KEY,
            azure_endpoint=Config.AZURE_OPENAI_ENDPOINT,
            openai_api_version=Config.AZURE_OPENAI_API_VERSION,
            deployment_name=Config.AZURE_OPENAI_API_DEPLOYMENT_NAME,
            temperature=0
        )
        self.token = Config.AZURE_OPENAI_API_KEY
        self.token_expiry = time.time() + Config.TOKEN_REFRESH_INTERVAL
        logger.info("Initialized LangGraphAgent with AzureChatOpenAI")

    async def _refresh_token(self):
        """Refreshes the Azure OpenAI token (placeholder).
        
        Updates the LLM instance with a new token.
        
        Raises:
            Exception: If token refresh fails.
        """
        try:
            # Placeholder: Replace with actual Azure token refresh (e.g., msal)
            new_token = Config.AZURE_OPENAI_API_KEY
            self.token = new_token
            self.token_expiry = time.time() + Config.TOKEN_REFRESH_INTERVAL
            self.llm = AzureChatOpenAI(
                openai_api_key=new_token,
                azure_endpoint=Config.AZURE_OPENAI_ENDPOINT,
                openai_api_version=Config.AZURE_OPENAI_API_VERSION,
                deployment_name=Config.AZURE_OPENAI_TOKEN_DEPLOYMENT_NAME,
                temperature=0
            )
            logger.info("Refreshed Azure OpenAI token")
        except Exception as e:
            logger.error(f"Failed to refresh token: {error}")
            raise

    async def _refresh_token(selfloop):
        """Runs a background loop to refresh the token periodically."""
        while True:
            await asyncio.sleep(Config.TOKEN_REFRESH_INTERVAL)
            await self._refresh_token()

    async def clarify_query(self, user_query: str) -> str:
        """Clarifies ambiguous user queries using the LLM.
        
        Args:
            user_query (str): User input query.
            
        Returns:
            str: Clarified query or original if no clarification needed.
            
        Raises:
            Exception: If clarification fails (returns original query).
        """
        try:
            ambiguous_terms = ["details", "recent", "all"]
            if any(term in user_query.lower() for term in ambiguous_terms):
                prompt = f"""
                The query "{user_query}" is ambiguous. Suggest specific details to clarify (e.g., fields, filters).
                Examples:
                - For "customer details": Specify fields like id, name, city.
                - For "recent orders": Specify a time range (e.g., last 30 days).
                Return a clarified query or the original if no clarification needed.
                """
                start_time = time.time()
                response = await self.llm.asyncinvoke(prompt)
                latency = time.time() - start_time
                metrics.record_query(latency)
                clarified_query = response.content.strip()
                logger.info(f"Clarified query: {clarified_query}")
                return clarified_query
            return user_query
        except Exception as e:
            metrics.record_error()
            logger.error(f"Failed to clarify query: {e}")
            return user_query

    async def process_query(self, user_query: str, graphql_endpoint: str) -> dict:
        """Processes a user query and executes it against the GraphQL endpoint.
        
        Args:
            user_query (str): User input query.
            graphql_endpoint (str): GraphQL API endpoint.
            
        Returns:
            dict: Query results including GraphQL query, result, entities, arguments, and clarified query.
            
        Raises:
            ValueError: If input validation fails.
            Exception: If query processing fails.
        """
        try:
            validate_input(user_query)
            start_time = time.time()

            # Connect to MCP server
            async with MultiServerMCPClient() as client:
                await client.connect_to_server(
                    "graphql",
                    url=Config.MCP_SERVER_URL,
                    transport="http-streamable"
                )
                tools = await client.get_tools()
                logger.info(f"Connected to MCP server with tools: {[tool.name for tool in tools]}")

                # Create agent
                agent = create_react_agent(self.llm, tools)

                # Clarify query
                clarified_query = await self.clarify_query(user_query)
                logger.info(f"Processing clarified query: {clarified_query}")

                # Extract entities and arguments
                entities = await tools[tools.index(next(t for t in tools if t.name == "extract_query_entities"))](clarified_query)
                arguments = await tools[tools.index(next(t for t in tools if t.name == "extract_arguments"))](clarified_query, entities)

                # Build and validate query
                max_attempts = Config.MAX_RETRIES
                error_feedback = ""
                graphql_query = None
                for attempt in range(max_attempts):
                    graphql_query = await tools[tools.index(next(t for t in tools if t.name == "build_graphql_query"))](entities, arguments, error_feedback)
                    is_valid, error = await tools[tools.index(next(t for t in tools if t.name == "validate_graphql_query"))](graphql_query)
                    if is_valid:
                        break
                    error_feedback = error
                    logger.warning(f"Query validation failed (attempt {attempt + 1}): {error}")

                if is_valid:
                    # Execute query
                    result = await tools[tools.index(next(t for t in tools if t.name == "execute_graphql_query"))](graphql_query)
                    logger.info(f"Query result: {result}")

                    # Confirm response
                    await tools[tools.index(next(t for t in tools if t.name == "confirm_response"))](
                        clarified_query, entities, arguments, graphql_query, result
                    )

                    latency = time.time() - start_time
                    metrics.record_query(latency)
                    return {
                        "graphql_query": graphql_query,
                        "result": result,
                        "entities": entities,
                        "arguments": arguments,
                        "clarified_query": clarified_query
                    }
                else:
                    metrics.record_error()
                    logger.error("Failed to generate valid query after max attempts")
                    raise ValueError("Failed to generate valid query")

        except Exception as e:
            metrics.record_error()
            logger.error(f"Error processing query: {e}")
            raise
```

### `app/api.py`
```python
from fastapi import FastAPI, HTTPException
from .agent import LangGraphAgent
from .models import GraphQLQueryRequest, GraphQLQueryResponse, MetricsResponse, HealthResponse
from utils import logger, metrics
from config import Config

app = FastAPI(title="LangGraph GraphQL Agent", version="1.0.0")

@app.post("/graphql/query", response_model=GraphQLQueryResponse)
async def process_graphql_query(request: GraphQLQueryRequest):
    """Processes a GraphQL query from a user question.
    
    Args:
        request (GraphQLQueryRequest): Request containing user query and GraphQL endpoint.
        
    Returns:
        GraphQLQueryResponse: Response with query results and metadata.
        
    Raises:
        HTTPException: If query processing fails.
    """
    try:
        agent = LangGraphAgent.get_instance()
        result = await agent.process_query(request.user_query, str(request.graphql_endpoint))
        logger.info(f"Processed query: {request.user_query}")
        return GraphQLQueryResponse(**result)
    except Exception as e:
        logger.error(f"Failed to process query: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/metrics", response_model=MetricsResponse)
async def get_metrics():
    """Retrieves agent performance metrics.
    
    Returns:
        MetricsResponse: Current metrics.
    """
    stats = metrics.get_stats()
    logger.info(f"Retrieved metrics: {stats}")
    return MetricsResponse(**stats)

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Checks the agent's health.
    
    Returns:
        HealthResponse: Health status.
    """
    try:
        agent = LangGraphAgent.get_instance()
        # Simple health check by invoking LLM
        await agent.llm.ainvoke("Health check")
        logger.debug("Health check passed")
        return HealthResponse(status="healthy")
    except Exception as e:
        logger.error(f"Health check failed: {e}")
        raise HTTPException(status_code=500, detail="Agent is unhealthy")
```

### `main.py`
```python
import uvicorn
from config import Config

if __name__ == "__main__":
    uvicorn.run("app.api:app", host="0.0.0.0", port=Config.API_PORT, reload=True)
```

### `README.md`
```markdown
# LangGraph GraphQL Agent

A FastAPI-based LangGraph agent for processing natural language GraphQL queries.

## Setup
1. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Configure `.env` with Azure OpenAI and MCP server details.
3. Start the MCP server (`graphql-mcp-server`).
4. Run the agent:
   ```bash
   python main.py
   ```

## Endpoints
- `POST /graphql/query`: Process a user query for a GraphQL endpoint.
- `GET /metrics`: Retrieve performance metrics.
- `GET /health`: Check agent health.

## Features
- Singleton `AzureChatOpenAI` with token refresh.
- FastAPI for generic query processing.
- Retries, metrics, and logging.
- Detailed comments for clarity.
```

---

## Example Usage

### Inputs
- **GraphQL Endpoint**: `https://retail-graphql.example.com/graphql`
- **User Query**: “Find recent orders with customer details”
- **Date/Time**: June 4, 2025, 09:22 AM IST

### Steps
1. **Start MCP Server**:
   ```bash
   cd graphql-mcp-server
   python graphql_mcp_server.py
   ```

2. **Start Agent**:
   ```bash
   cd langgraph-graphql-agent
   python main.py
   ```

3. **Query via API**:
   ```bash
   curl -X POST http://localhost:8001/graphql/query \
   -H "Content-Type: application/json" \
   -d '{
       "user_query": "Find recent orders with customer details",
       "graphql_endpoint": "https://retail-graphql.example.com/graphql"
   }'
   ```
   **Response**:
   ```json
   {
     "graphql_query": "query {\n  allOrders(filter: {createdAt: {gte: \"2025-05-01\"}}) {\n    id\n    customer {\n      name\n      city\n    }\n  }\n}",
     "result": {
       "allOrders": [
         {
           "id": "1",
           "customer": {
             "name": "John Doe",
             "city": "New York"
           }
         }
       ]
     },
     "entities": {
       "entities": ["Order", "Customer"],
       "fields": {
         "Order": ["id", "customer"],
         "Customer": ["name", "city"]
       }
     },
     "arguments": {
       "Order": {
         "filter": {
           "createdAt": {"gte": "2025-05-01"}
         }
       }
     },
     "clarified_query": "Find orders from the last 30 days with customer name and city"
   }
   ```

4. **Check Metrics**:
   ```bash
   curl http://localhost:8001/metrics
   ```
   **Response**:
   ```json
   {
     "queries_executed": 1,
     "query_errors": 0,
     "avg_query_latency_ms": 1200.5
   }
   ```

5. **Check Health**:
   ```bash
   curl http://localhost:8001/health
   ```
   **Response**:
   ```json
   {
     "status": "healthy"
   }
   ```

---

## Production Features
- **Efficiency**: Singleton instances for `GraphQLMCPServer` and `AzureChatOpenAI`.
- **Reliability**: Token refresh (placeholder; use `msal` for production).
- **Scalability**: Connection pooling, retries, and async processing.
- **Observability**: Metrics and structured logging.
- **Modularity**: Separate repositories and FastAPI endpoints.
- **Clarity**: Detailed comments for developers.

**Token Refresh Implementation** (for production):
```python
from msal import ConfidentialClientApplication

async def _refresh_token(self):
    app = ConfidentialClientApplication(
        client_id="your-client-id",
        client_credential="your-client-secret",
        authority="https://login.microsoftonline.com/your-tenant-id"
    )
    result = await asyncio.get_event_loop().run_in_executor(
        None, app.acquire_token_for_client, ["https://cognitiveservices.azure.com/.default"]
    )
    self.token = result["access_token"]
    self.llm = AzureChatOpenAI(openai_api_key=self.token, ...)
```

---

## Exporting as PDF

To create a PDF from this content:

1. **Copy to Document Editor**:
   - Copy the entire response into a text editor like Microsoft Word or Google Docs.
   - Format headings, code blocks, and text for readability.
   - Export as PDF (`File > Download > PDF` in Google Docs or `File > Save As > PDF` in Word).

2. **Use Markdown-to-PDF Converter**:
   - Save the response as a `.md` file (e.g., `solution.md`).
   - Install Pandoc: `sudo apt-get install pandoc` (Linux) or equivalent.
   - Convert to PDF:
     ```bash
     pandoc solution.md -o solution.pdf --pdf-engine=xelatex
     ```
   - Alternatively, use Typora or an online converter like `markdowntopdf.com`.

3. **Browser Print**:
   - Copy the response into a Markdown viewer (e.g., VS Code with Markdown preview).
   - Print to PDF using your browser (`Ctrl+P > Save as PDF`).

---

This consolidated solution includes all code, configurations, and instructions. Let me know if you need specific sections expanded, additional features, or assistance with PDF conversion!