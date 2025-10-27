"""
Production-Grade PathRAG System for FinTech Metadata Retrieval
Optimizes Text-to-SQL accuracy by extracting graph-based relational context
"""

import os
import logging
from typing import List, Dict, Tuple, Optional, Set
from dataclasses import dataclass, field
from enum import Enum
import json
from collections import defaultdict
import heapq

import numpy as np
import networkx as nx
from rdflib import Graph, Namespace, RDF, RDFS, OWL
from rdflib.term import URIRef, Literal
from sentence_transformers import SentenceTransformer
from rank_bm25 import BM25Okapi
import torch

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


class NodeType(Enum):
    """Types of nodes in the knowledge graph"""
    TABLE = "table"
    COLUMN = "column"
    RELATIONSHIP = "relationship"
    METRIC = "metric"
    CONSTRAINT = "constraint"
    BUSINESS_TERM = "business_term"


@dataclass
class GraphNode:
    """Represents a node in the knowledge graph"""
    id: str
    name: str
    node_type: NodeType
    description: str = ""
    properties: Dict = field(default_factory=dict)
    text_chunks: List[str] = field(default_factory=list)
    embedding: Optional[np.ndarray] = None
    

@dataclass
class GraphEdge:
    """Represents an edge in the knowledge graph"""
    source: str
    target: str
    relation_type: str
    weight: float = 1.0
    properties: Dict = field(default_factory=dict)
    text_chunks: List[str] = field(default_factory=list)


@dataclass
class PathResult:
    """Result from path finding"""
    path: List[str]
    score: float
    textual_representation: str
    metadata: Dict


class RDFKnowledgeGraphLoader:
    """Loads and processes RDF/Turtle files into a knowledge graph"""
    
    def __init__(self, namespace_mappings: Dict[str, str] = None):
        self.graph = Graph()
        self.namespace_mappings = namespace_mappings or {}
        self.nx_graph = nx.DiGraph()
        
    def load_turtle_files(self, file_paths: List[str]) -> None:
        """Load multiple Turtle files"""
        for file_path in file_paths:
            try:
                logger.info(f"Loading RDF file: {file_path}")
                self.graph.parse(file_path, format="turtle")
                logger.info(f"Successfully loaded {file_path}")
            except Exception as e:
                logger.error(f"Error loading {file_path}: {e}")
                raise
    
    def extract_schema_metadata(self) -> Tuple[List[GraphNode], List[GraphEdge]]:
        """Extract database schema metadata from RDF graph"""
        nodes = []
        edges = []
        
        # Define common ontology namespaces
        SCHEMA = Namespace("http://schema.org/")
        DB = Namespace("http://fintech.com/schema/")
        
        # Extract tables
        for subj in self.graph.subjects(RDF.type, DB.Table):
            node = self._extract_table_node(subj, DB, SCHEMA)
            if node:
                nodes.append(node)
        
        # Extract columns
        for subj in self.graph.subjects(RDF.type, DB.Column):
            node = self._extract_column_node(subj, DB, SCHEMA)
            if node:
                nodes.append(node)
        
        # Extract relationships
        for subj in self.graph.subjects(RDF.type, DB.ForeignKey):
            edge = self._extract_relationship_edge(subj, DB)
            if edge:
                edges.append(edge)
        
        # Extract metrics and business terms
        for subj in self.graph.subjects(RDF.type, DB.Metric):
            node = self._extract_metric_node(subj, DB, SCHEMA)
            if node:
                nodes.append(node)
        
        logger.info(f"Extracted {len(nodes)} nodes and {len(edges)} edges from RDF")
        return nodes, edges
    
    def _extract_table_node(self, subj, DB, SCHEMA) -> Optional[GraphNode]:
        """Extract table metadata"""
        name = self._get_label(subj)
        if not name:
            return None
        
        description = str(self.graph.value(subj, SCHEMA.description) or "")
        schema_name = str(self.graph.value(subj, DB.schemaName) or "")
        
        properties = {
            "schema": schema_name,
            "uri": str(subj)
        }
        
        text_chunk = f"Table {name}"
        if schema_name:
            text_chunk += f" in schema {schema_name}"
        if description:
            text_chunk += f": {description}"
        
        return GraphNode(
            id=self._create_id(subj),
            name=name,
            node_type=NodeType.TABLE,
            description=description,
            properties=properties,
            text_chunks=[text_chunk]
        )
    
    def _extract_column_node(self, subj, DB, SCHEMA) -> Optional[GraphNode]:
        """Extract column metadata"""
        name = self._get_label(subj)
        if not name:
            return None
        
        description = str(self.graph.value(subj, SCHEMA.description) or "")
        data_type = str(self.graph.value(subj, DB.dataType) or "")
        table = self.graph.value(subj, DB.belongsToTable)
        table_name = self._get_label(table) if table else ""
        
        properties = {
            "data_type": data_type,
            "table": table_name,
            "uri": str(subj),
            "nullable": str(self.graph.value(subj, DB.nullable) or "true"),
            "primary_key": str(self.graph.value(subj, DB.primaryKey) or "false")
        }
        
        text_chunk = f"Column {name}"
        if table_name:
            text_chunk += f" in table {table_name}"
        if data_type:
            text_chunk += f" of type {data_type}"
        if description:
            text_chunk += f": {description}"
        
        return GraphNode(
            id=self._create_id(subj),
            name=name,
            node_type=NodeType.COLUMN,
            description=description,
            properties=properties,
            text_chunks=[text_chunk]
        )
    
    def _extract_relationship_edge(self, subj, DB) -> Optional[GraphEdge]:
        """Extract foreign key relationships"""
        from_col = self.graph.value(subj, DB.fromColumn)
        to_col = self.graph.value(subj, DB.toColumn)
        
        if not from_col or not to_col:
            return None
        
        from_id = self._create_id(from_col)
        to_id = self._create_id(to_col)
        
        from_name = self._get_label(from_col)
        to_name = self._get_label(to_col)
        
        text_chunk = f"{from_name} references {to_name}"
        
        return GraphEdge(
            source=from_id,
            target=to_id,
            relation_type="foreign_key",
            weight=2.0,  # Higher weight for FK relationships
            text_chunks=[text_chunk]
        )
    
    def _extract_metric_node(self, subj, DB, SCHEMA) -> Optional[GraphNode]:
        """Extract business metric metadata"""
        name = self._get_label(subj)
        if not name:
            return None
        
        description = str(self.graph.value(subj, SCHEMA.description) or "")
        formula = str(self.graph.value(subj, DB.formula) or "")
        
        properties = {
            "formula": formula,
            "uri": str(subj)
        }
        
        text_chunk = f"Metric {name}"
        if description:
            text_chunk += f": {description}"
        if formula:
            text_chunk += f". Calculated as: {formula}"
        
        return GraphNode(
            id=self._create_id(subj),
            name=name,
            node_type=NodeType.METRIC,
            description=description,
            properties=properties,
            text_chunks=[text_chunk]
        )
    
    def _get_label(self, uri) -> str:
        """Get human-readable label from URI"""
        if not uri:
            return ""
        label = self.graph.value(uri, RDFS.label)
        if label:
            return str(label)
        # Fallback to URI fragment
        return str(uri).split("/")[-1].split("#")[-1]
    
    def _create_id(self, uri) -> str:
        """Create consistent ID from URI"""
        return str(uri).split("/")[-1].split("#")[-1].lower()
    
    def build_networkx_graph(self, nodes: List[GraphNode], 
                            edges: List[GraphEdge]) -> nx.DiGraph:
        """Build NetworkX graph from nodes and edges"""
        G = nx.DiGraph()
        
        # Add nodes
        for node in nodes:
            G.add_node(
                node.id,
                name=node.name,
                node_type=node.node_type.value,
                description=node.description,
                properties=node.properties,
                text_chunks=node.text_chunks
            )
        
        # Add edges
        for edge in edges:
            if edge.source in G.nodes and edge.target in G.nodes:
                G.add_edge(
                    edge.source,
                    edge.target,
                    relation_type=edge.relation_type,
                    weight=edge.weight,
                    text_chunks=edge.text_chunks
                )
        
        # Add implicit table-column edges
        for node_id, data in G.nodes(data=True):
            if data.get('node_type') == NodeType.COLUMN.value:
                table_name = data['properties'].get('table', '').lower()
                if table_name and table_name in G.nodes:
                    G.add_edge(
                        table_name,
                        node_id,
                        relation_type="has_column",
                        weight=1.5,
                        text_chunks=[f"{data['properties'].get('table')} has column {data['name']}"]
                    )
        
        logger.info(f"Built NetworkX graph: {G.number_of_nodes()} nodes, {G.number_of_edges()} edges")
        return G


class OptimalPathRAG:
    """
    Production-grade PathRAG implementation with direct query embedding
    Optimized for FinTech metadata retrieval
    """
    
    def __init__(self, 
                 embedding_model: str = "all-MiniLM-L6-v2",
                 max_path_length: int = 4,
                 top_k_nodes: int = 8,
                 top_k_paths: int = 5):
        
        self.embedding_model = SentenceTransformer(embedding_model)
        self.max_path_length = max_path_length
        self.top_k_nodes = top_k_nodes
        self.top_k_paths = top_k_paths
        
        self.graph: Optional[nx.DiGraph] = None
        self.node_embeddings: Dict[str, np.ndarray] = {}
        self.bm25: Optional[BM25Okapi] = None
        self.node_texts: Dict[str, str] = {}
        self.tokenized_corpus: List[List[str]] = []
        self.node_ids_ordered: List[str] = []
        
        logger.info(f"Initialized OptimalPathRAG with model: {embedding_model}")
    
    def build_graph(self, rdf_files: List[str]) -> None:
        """Load RDF files and build knowledge graph"""
        loader = RDFKnowledgeGraphLoader()
        loader.load_turtle_files(rdf_files)
        
        nodes, edges = loader.extract_schema_metadata()
        self.graph = loader.build_networkx_graph(nodes, edges)
        
        self._build_embeddings()
        self._build_bm25_index()
    
    def _build_embeddings(self) -> None:
        """Create embeddings for all nodes"""
        logger.info("Building node embeddings...")
        
        for node_id, data in self.graph.nodes(data=True):
            # Combine all textual information
            text_parts = [
                data.get('name', ''),
                data.get('description', ''),
                ' '.join(data.get('text_chunks', []))
            ]
            combined_text = ' '.join(filter(None, text_parts))
            self.node_texts[node_id] = combined_text
            
            # Create embedding
            embedding = self.embedding_model.encode(combined_text)
            self.node_embeddings[node_id] = embedding
            self.graph.nodes[node_id]['embedding'] = embedding
        
        logger.info(f"Created embeddings for {len(self.node_embeddings)} nodes")
    
    def _build_bm25_index(self) -> None:
        """Build BM25 index for lexical matching"""
        logger.info("Building BM25 index...")
        
        self.node_ids_ordered = list(self.graph.nodes())
        corpus = [self.node_texts[node_id] for node_id in self.node_ids_ordered]
        
        # Tokenize corpus
        self.tokenized_corpus = [text.lower().split() for text in corpus]
        self.bm25 = BM25Okapi(self.tokenized_corpus)
        
        logger.info("BM25 index built successfully")
    
    def retrieve_initial_nodes_hybrid(self, query: str) -> List[str]:
        """
        Hybrid retrieval: BM25 + Vector similarity
        This is the optimal approach per our discussion
        """
        # 1. BM25 retrieval (lexical matching)
        tokenized_query = query.lower().split()
        bm25_scores = self.bm25.get_scores(tokenized_query)
        
        # Get top BM25 matches
        bm25_top_indices = np.argsort(bm25_scores)[-self.top_k_nodes:][::-1]
        bm25_candidates = {
            self.node_ids_ordered[idx]: bm25_scores[idx] 
            for idx in bm25_top_indices
        }
        
        # 2. Vector similarity (semantic matching)
        query_embedding = self.embedding_model.encode(query)
        vector_scores = {}
        
        for node_id, node_embedding in self.node_embeddings.items():
            similarity = np.dot(query_embedding, node_embedding) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(node_embedding)
            )
            vector_scores[node_id] = similarity
        
        # Get top vector matches
        vector_candidates = dict(sorted(
            vector_scores.items(), 
            key=lambda x: x[1], 
            reverse=True
        )[:self.top_k_nodes])
        
        # 3. Hybrid fusion (normalize and combine)
        all_candidates = set(bm25_candidates.keys()) | set(vector_candidates.keys())
        
        # Normalize scores
        max_bm25 = max(bm25_candidates.values()) if bm25_candidates else 1.0
        max_vector = max(vector_candidates.values()) if vector_candidates else 1.0
        
        hybrid_scores = {}
        for node_id in all_candidates:
            bm25_norm = bm25_candidates.get(node_id, 0) / max_bm25
            vector_norm = vector_candidates.get(node_id, 0) / max_vector
            
            # Weighted combination (70% vector, 30% BM25)
            hybrid_scores[node_id] = 0.7 * vector_norm + 0.3 * bm25_norm
        
        # Return top-k
        top_nodes = sorted(
            hybrid_scores.items(), 
            key=lambda x: x[1], 
            reverse=True
        )[:self.top_k_nodes]
        
        result_nodes = [node_id for node_id, _ in top_nodes]
        
        logger.info(f"Retrieved {len(result_nodes)} initial nodes for query: {query[:50]}...")
        return result_nodes
    
    def find_all_paths_optimized(self, start_nodes: List[str]) -> List[List[str]]:
        """
        Find all paths using BFS with pruning
        Optimized to avoid exponential explosion
        """
        all_paths = []
        visited_paths = set()
        
        for start_node in start_nodes:
            if start_node not in self.graph:
                continue
            
            # BFS queue: (current_node, path, depth)
            queue = [(start_node, [start_node], 0)]
            
            while queue:
                current, path, depth = queue.pop(0)
                
                # Add current path
                path_tuple = tuple(path)
                if len(path) >= 2 and path_tuple not in visited_paths:
                    all_paths.append(path)
                    visited_paths.add(path_tuple)
                
                # Stop if max depth reached
                if depth >= self.max_path_length - 1:
                    continue
                
                # Explore neighbors (both directions for undirected exploration)
                neighbors = set(self.graph.successors(current)) | set(self.graph.predecessors(current))
                
                for neighbor in neighbors:
                    if neighbor not in path:  # Avoid cycles
                        new_path = path + [neighbor]
                        queue.append((neighbor, new_path, depth + 1))
        
        logger.info(f"Found {len(all_paths)} candidate paths")
        return all_paths
    
    def calculate_path_score(self, path: List[str]) -> float:
        """
        Calculate comprehensive path relevance score
        Considers: edge weights, path length, node types
        """
        if len(path) < 2:
            return 0.0
        
        score = 0.0
        
        # Edge weight score
        for i in range(len(path) - 1):
            if self.graph.has_edge(path[i], path[i+1]):
                edge_data = self.graph[path[i]][path[i+1]]
                score += edge_data.get('weight', 1.0)
            elif self.graph.has_edge(path[i+1], path[i]):
                edge_data = self.graph[path[i+1]][path[i]]
                score += edge_data.get('weight', 1.0)
        
        # Length penalty (prefer shorter, more direct paths)
        length_penalty = 1.0 / (1.0 + 0.3 * (len(path) - 2))
        score *= length_penalty
        
        # Node type bonus (prefer paths through important node types)
        type_bonus = 0.0
        for node_id in path:
            node_type = self.graph.nodes[node_id].get('node_type', '')
            if node_type == NodeType.TABLE.value:
                type_bonus += 0.5
            elif node_type == NodeType.METRIC.value:
                type_bonus += 0.3
        
        score += type_bonus
        
        return score
    
    def rank_paths(self, paths: List[List[str]]) -> List[Tuple[List[str], float]]:
        """Rank paths by relevance score"""
        scored_paths = [
            (path, self.calculate_path_score(path)) 
            for path in paths
        ]
        return sorted(scored_paths, key=lambda x: x[1], reverse=True)
    
    def generate_textual_path(self, path: List[str]) -> str:
        """Generate human-readable representation of a path"""
        if len(path) == 1:
            node_data = self.graph.nodes[path[0]]
            return node_data.get('text_chunks', [''])[0]
        
        segments = []
        
        for i in range(len(path)):
            node_data = self.graph.nodes[path[i]]
            node_text = f"{node_data['name']} ({node_data.get('node_type', 'unknown')})"
            
            if i < len(path) - 1:
                # Add edge information
                next_node = path[i + 1]
                if self.graph.has_edge(path[i], next_node):
                    edge_data = self.graph[path[i]][next_node]
                    relation = edge_data.get('relation_type', 'related_to')
                    node_text += f" --[{relation}]--> "
                elif self.graph.has_edge(next_node, path[i]):
                    edge_data = self.graph[next_node][path[i]]
                    relation = edge_data.get('relation_type', 'related_to')
                    node_text += f" <--[{relation}]-- "
            
            segments.append(node_text)
        
        return ''.join(segments)
    
    def extract_metadata_from_paths(self, ranked_paths: List[Tuple[List[str], float]]) -> Dict:
        """
        Extract structured metadata from top-ranked paths
        This is the key output for Text-to-SQL context
        """
        metadata = {
            "tables": set(),
            "columns": {},  # table -> [columns]
            "relationships": [],
            "metrics": [],
            "constraints": [],
            "textual_context": []
        }
        
        for path, score in ranked_paths[:self.top_k_paths]:
            # Generate textual representation
            textual_path = self.generate_textual_path(path)
            metadata["textual_context"].append({
                "path": textual_path,
                "score": score
            })
            
            # Extract structured metadata
            for node_id in path:
                node_data = self.graph.nodes[node_id]
                node_type = node_data.get('node_type')
                
                if node_type == NodeType.TABLE.value:
                    metadata["tables"].add(node_data['name'])
                
                elif node_type == NodeType.COLUMN.value:
                    table = node_data['properties'].get('table')
                    if table:
                        if table not in metadata["columns"]:
                            metadata["columns"][table] = []
                        
                        col_info = {
                            "name": node_data['name'],
                            "type": node_data['properties'].get('data_type'),
                            "description": node_data.get('description', ''),
                            "nullable": node_data['properties'].get('nullable'),
                            "primary_key": node_data['properties'].get('primary_key')
                        }
                        metadata["columns"][table].append(col_info)
                
                elif node_type == NodeType.METRIC.value:
                    metadata["metrics"].append({
                        "name": node_data['name'],
                        "description": node_data.get('description', ''),
                        "formula": node_data['properties'].get('formula', '')
                    })
            
            # Extract relationships along the path
            for i in range(len(path) - 1):
                source = path[i]
                target = path[i + 1]
                
                if self.graph.has_edge(source, target):
                    edge_data = self.graph[source][target]
                    if edge_data.get('relation_type') == 'foreign_key':
                        metadata["relationships"].append({
                            "from": self.graph.nodes[source]['name'],
                            "to": self.graph.nodes[target]['name'],
                            "type": "foreign_key"
                        })
        
        # Convert sets to lists for JSON serialization
        metadata["tables"] = list(metadata["tables"])
        
        return metadata
    
    def query(self, user_query: str) -> Dict:
        """
        Main query interface - returns optimal metadata for Text-to-SQL
        """
        logger.info(f"Processing query: {user_query}")
        
        # 1. Hybrid retrieval (BM25 + Vector)
        start_nodes = self.retrieve_initial_nodes_hybrid(user_query)
        
        # 2. Find candidate paths
        all_paths = self.find_all_paths_optimized(start_nodes)
        
        # 3. Rank paths
        ranked_paths = self.rank_paths(all_paths)
        
        # 4. Extract structured metadata
        metadata = self.extract_metadata_from_paths(ranked_paths)
        
        logger.info(f"Extracted metadata: {len(metadata['tables'])} tables, "
                   f"{sum(len(cols) for cols in metadata['columns'].values())} columns")
        
        return {
            "query": user_query,
            "metadata": metadata,
            "top_paths": [
                {
                    "path": [self.graph.nodes[n]['name'] for n in path],
                    "score": score,
                    "textual": self.generate_textual_path(path)
                }
                for path, score in ranked_paths[:self.top_k_paths]
            ]
        }


def create_sample_fintech_rdf():
    """Create sample FinTech RDF/Turtle file for demonstration"""
    
    turtle_content = """
@prefix db: <http://fintech.com/schema/> .
@prefix schema: <http://schema.org/> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .

# Tables
db:customers rdf:type db:Table ;
    rdfs:label "customers" ;
    db:schemaName "public" ;
    schema:description "Customer master data containing customer demographics and account information" .

db:transactions rdf:type db:Table ;
    rdfs:label "transactions" ;
    db:schemaName "public" ;
    schema:description "Transaction records including all customer financial transactions" .

db:accounts rdf:type db:Table ;
    rdfs:label "accounts" ;
    db:schemaName "public" ;
    schema:description "Account information including balance and account type details" .

db:loan_applications rdf:type db:Table ;
    rdfs:label "loan_applications" ;
    db:schemaName "public" ;
    schema:description "Loan application records with approval status and loan amounts" .

db:credit_scores rdf:type db:Table ;
    rdfs:label "credit_scores" ;
    db:schemaName "public" ;
    schema:description "Credit score history for customers" .

# Customer Columns
db:customer_id rdf:type db:Column ;
    rdfs:label "customer_id" ;
    db:belongsToTable db:customers ;
    db:dataType "INTEGER" ;
    db:primaryKey "true" ;
    db:nullable "false" ;
    schema:description "Unique identifier for each customer" .

db:customer_name rdf:type db:Column ;
    rdfs:label "customer_name" ;
    db:belongsToTable db:customers ;
    db:dataType "VARCHAR(255)" ;
    schema:description "Full name of the customer" .

db:customer_email rdf:type db:Column ;
    rdfs:label "email" ;
    db:belongsToTable db:customers ;
    db:dataType "VARCHAR(255)" ;
    schema:description "Customer email address for communication" .

db:customer_registration_date rdf:type db:Column ;
    rdfs:label "registration_date" ;
    db:belongsToTable db:customers ;
    db:dataType "DATE" ;
    schema:description "Date when customer registered with the bank" .

db:customer_status rdf:type db:Column ;
    rdfs:label "status" ;
    db:belongsToTable db:customers ;
    db:dataType "VARCHAR(50)" ;
    schema:description "Customer account status (active, inactive, suspended)" .

# Account Columns
db:account_id rdf:type db:Column ;
    rdfs:label "account_id" ;
    db:belongsToTable db:accounts ;
    db:dataType "INTEGER" ;
    db:primaryKey "true" ;
    schema:description "Unique account identifier" .

db:account_customer_id rdf:type db:Column ;
    rdfs:label "customer_id" ;
    db:belongsToTable db:accounts ;
    db:dataType "INTEGER" ;
    schema:description "Reference to customer who owns this account" .

db:account_type rdf:type db:Column ;
    rdfs:label "account_type" ;
    db:belongsToTable db:accounts ;
    db:dataType "VARCHAR(50)" ;
    schema:description "Type of account (checking, savings, investment)" .

db:account_balance rdf:type db:Column ;
    rdfs:label "balance" ;
    db:belongsToTable db:accounts ;
    db:dataType "DECIMAL(15,2)" ;
    schema:description "Current account balance in USD" .

db:account_opened_date rdf:type db:Column ;
    rdfs:label "opened_date" ;
    db:belongsToTable db:accounts ;
    db:dataType "DATE" ;
    schema:description "Date when account was opened" .

# Transaction Columns
db:transaction_id rdf:type db:Column ;
    rdfs:label "transaction_id" ;
    db:belongsToTable db:transactions ;
    db:dataType "INTEGER" ;
    db:primaryKey "true" ;
    schema:description "Unique transaction identifier" .

db:transaction_account_id rdf:type db:Column ;
    rdfs:label "account_id" ;
    db:belongsToTable db:transactions ;
    db:dataType "INTEGER" ;
    schema:description "Account involved in the transaction" .

db:transaction_amount rdf:type db:Column ;
    rdfs:label "amount" ;
    db:belongsToTable db:transactions ;
    db:dataType "DECIMAL(15,2)" ;
    schema:description "Transaction amount (positive for credit, negative for debit)" .

db:transaction_date rdf:type db:Column ;
    rdfs:label "transaction_date" ;
    db:belongsToTable db:transactions ;
    db:dataType "TIMESTAMP" ;
    schema:description "Timestamp when transaction occurred" .

db:transaction_type rdf:type db:Column ;
    rdfs:label "transaction_type" ;
    db:belongsToTable db:transactions ;
    db:dataType "VARCHAR(50)" ;
    schema:description "Type of transaction (deposit, withdrawal, transfer, payment)" .

db:transaction_description rdf:type db:Column ;
    rdfs:label "description" ;
    db:belongsToTable db:transactions ;
    db:dataType "TEXT" ;
    schema:description "Detailed description of the transaction" .

# Loan Application Columns
db:loan_id rdf:type db:Column ;
    rdfs:label "loan_id" ;
    db:belongsToTable db:loan_applications ;
    db:dataType "INTEGER" ;
    db:primaryKey "true" ;
    schema:description "Unique loan application identifier" .

db:loan_customer_id rdf:type db:Column ;
    rdfs:label "customer_id" ;
    db:belongsToTable db:loan_applications ;
    db:dataType "INTEGER" ;
    schema:description "Customer applying for the loan" .

db:loan_amount rdf:type db:Column ;
    rdfs:label "loan_amount" ;
    db:belongsToTable db:loan_applications ;
    db:dataType "DECIMAL(15,2)" ;
    schema:description "Requested loan amount in USD" .

db:loan_status rdf:type db:Column ;
    rdfs:label "status" ;
    db:belongsToTable db:loan_applications ;
    db:dataType "VARCHAR(50)" ;
    schema:description "Loan application status (pending, approved, rejected, disbursed)" .

db:loan_application_date rdf:type db:Column ;
    rdfs:label "application_date" ;
    db:belongsToTable db:loan_applications ;
    db:dataType "DATE" ;
    schema:description "Date when loan was applied for" .

db:loan_approval_date rdf:type db:Column ;
    rdfs:label "approval_date" ;
    db:belongsToTable db:loan_applications ;
    db:dataType "DATE" ;
    schema:description "Date when loan was approved (if applicable)" .

db:loan_interest_rate rdf:type db:Column ;
    rdfs:label "interest_rate" ;
    db:belongsToTable db:loan_applications ;
    db:dataType "DECIMAL(5,2)" ;
    schema:description "Annual interest rate percentage for the loan" .

# Credit Score Columns
db:score_id rdf:type db:Column ;
    rdfs:label "score_id" ;
    db:belongsToTable db:credit_scores ;
    db:dataType "INTEGER" ;
    db:primaryKey "true" ;
    schema:description "Unique credit score record identifier" .

db:score_customer_id rdf:type db:Column ;
    rdfs:label "customer_id" ;
    db:belongsToTable db:credit_scores ;
    db:dataType "INTEGER" ;
    schema:description "Customer whose credit score is recorded" .

db:credit_score_value rdf:type db:Column ;
    rdfs:label "score" ;
    db:belongsToTable db:credit_scores ;
    db:dataType "INTEGER" ;
    schema:description "Credit score value (300-850 range)" .

db:score_date rdf:type db:Column ;
    rdfs:label "score_date" ;
    db:belongsToTable db:credit_scores ;
    db:dataType "DATE" ;
    schema:description "Date when credit score was recorded" .

# Foreign Key Relationships
db:fk_accounts_customer rdf:type db:ForeignKey ;
    db:fromColumn db:account_customer_id ;
    db:toColumn db:customer_id ;
    schema:description "Links accounts to their owning customer" .

db:fk_transactions_account rdf:type db:ForeignKey ;
    db:fromColumn db:transaction_account_id ;
    db:toColumn db:account_id ;
    schema:description "Links transactions to the account they belong to" .

db:fk_loans_customer rdf:type db:ForeignKey ;
    db:fromColumn db:loan_customer_id ;
    db:toColumn db:customer_id ;
    schema:description "Links loan applications to customers" .

db:fk_credit_scores_customer rdf:type db:ForeignKey ;
    db:fromColumn db:score_customer_id ;
    db:toColumn db:customer_id ;
    schema:description "Links credit scores to customers" .

# Business Metrics
db:total_customer_balance rdf:type db:Metric ;
    rdfs:label "Total Customer Balance" ;
    schema:description "Sum of all account balances for a customer across all their accounts" ;
    db:formula "SUM(accounts.balance) GROUP BY accounts.customer_id" .

db:average_transaction_amount rdf:type db:Metric ;
    rdfs:label "Average Transaction Amount" ;
    schema:description "Average transaction amount for a customer over a time period" ;
    db:formula "AVG(transactions.amount) GROUP BY transactions.account_id" .

db:customer_lifetime_value rdf:type db:Metric ;
    rdfs:label "Customer Lifetime Value" ;
    schema:description "Total value of all transactions for a customer since registration" ;
    db:formula "SUM(transactions.amount) WHERE transaction_type IN ('deposit', 'transfer_in')" .

db:loan_approval_rate rdf:type db:Metric ;
    rdfs:label "Loan Approval Rate" ;
    schema:description "Percentage of loan applications that get approved" ;
    db:formula "COUNT(CASE WHEN status='approved' THEN 1 END) / COUNT(*) * 100" .

db:high_value_customer rdf:type db:Metric ;
    rdfs:label "High Value Customer" ;
    schema:description "Customer with total balance exceeding $100,000" ;
    db:formula "SUM(accounts.balance) > 100000 GROUP BY customer_id" .
"""
    
    # Write to file
    with open('fintech_schema.ttl', 'w') as f:
        f.write(turtle_content)
    
    logger.info("Created sample FinTech RDF file: fintech_schema.ttl")
    return 'fintech_schema.ttl'


def demonstrate_pathrag_advantages():
    """
    Demonstrate PathRAG advantages with real examples
    Shows edge cases where vector-only retrieval fails
    """
    
    print("=" * 80)
    print("PATHRAG DEMONSTRATION: FinTech Text-to-SQL Context Enhancement")
    print("=" * 80)
    print()
    
    # Create sample data
    rdf_file = create_sample_fintech_rdf()
    
    # Initialize PathRAG
    pathrag = OptimalPathRAG(
        embedding_model="all-MiniLM-L6-v2",
        max_path_length=4,
        top_k_nodes=8,
        top_k_paths=5
    )
    
    # Build graph
    print("Loading RDF knowledge graph...")
    pathrag.build_graph([rdf_file])
    print()
    
    # Test cases demonstrating PathRAG advantages
    test_queries = [
        {
            "query": "Show me all high-value customers with recent loan applications",
            "challenge": "Requires connecting customers → accounts (balance) → loans (application)",
            "vector_limitation": "Vector embeddings might retrieve 'customers' and 'loans' separately but miss the account balance context needed for 'high-value' determination"
        },
        {
            "query": "What is the average transaction amount for customers who have credit scores above 750?",
            "challenge": "Requires path: customers → credit_scores → accounts → transactions",
            "vector_limitation": "Vector search might find 'transactions' and 'credit_scores' but miss the relational path showing how to join these tables correctly"
        },
        {
            "query": "Find customers with declined loan applications but good credit history",
            "challenge": "Needs both loan_applications.status and credit_scores tables with proper joins",
            "vector_limitation": "May retrieve both tables but not understand that credit scores need to be filtered separately from loan status"
        }
    ]
    
    for i, test_case in enumerate(test_queries, 1):
        print(f"\n{'=' * 80}")
        print(f"EXAMPLE {i}")
        print(f"{'=' * 80}")
        print(f"\nQuery: {test_case['query']}")
        print(f"\nChallenge: {test_case['challenge']}")
        print(f"\nVector-Only Limitation: {test_case['vector_limitation']}")
        print(f"\n{'-' * 80}")
        print("PATHRAG RESULTS:")
        print(f"{'-' * 80}")
        
        # Execute query
        result = pathrag.query(test_case['query'])
        
        # Display top paths found
        print(f"\nTop {len(result['top_paths'])} Relational Paths Discovered:")
        for j, path_info in enumerate(result['top_paths'], 1):
            print(f"\n  Path {j} (Score: {path_info['score']:.3f}):")
            print(f"    Nodes: {' → '.join(path_info['path'])}")
            print(f"    Context: {path_info['textual'][:200]}...")
        
        # Display extracted metadata
        metadata = result['metadata']
        print(f"\n{'-' * 80}")
        print("EXTRACTED METADATA FOR TEXT-TO-SQL:")
        print(f"{'-' * 80}")
        
        print(f"\n📊 Tables Identified: {', '.join(metadata['tables'])}")
        
        print(f"\n📋 Columns by Table:")
        for table, columns in metadata['columns'].items():
            print(f"\n  {table}:")
            for col in columns[:5]:  # Show first 5 columns
                print(f"    - {col['name']} ({col['type']}): {col['description'][:60]}...")
        
        if metadata['relationships']:
            print(f"\n🔗 Relationships Found:")
            for rel in metadata['relationships']:
                print(f"    {rel['from']} --[{rel['type']}]--> {rel['to']}")
        
        if metadata['metrics']:
            print(f"\n📈 Relevant Business Metrics:")
            for metric in metadata['metrics']:
                print(f"    - {metric['name']}: {metric['description']}")
                print(f"      Formula: {metric['formula']}")
        
        print(f"\n{'-' * 80}")
        print("WHY PATHRAG WINS:")
        print(f"{'-' * 80}")
        
        advantages = []
        
        if len(metadata['tables']) > 1:
            advantages.append(f"✓ Identified {len(metadata['tables'])} related tables with proper join paths")
        
        if metadata['relationships']:
            advantages.append(f"✓ Discovered {len(metadata['relationships'])} foreign key relationships for accurate joins")
        
        if metadata['metrics']:
            advantages.append(f"✓ Found {len(metadata['metrics'])} business metrics with calculation formulas")
        
        path_lengths = [len(p['path']) for p in result['top_paths']]
        avg_length = sum(path_lengths) / len(path_lengths) if path_lengths else 0
        if avg_length > 2:
            advantages.append(f"✓ Traversed multi-hop paths (avg {avg_length:.1f} hops) that vector search would miss")
        
        advantages.append("✓ Provided relational context showing HOW tables connect, not just WHICH tables exist")
        
        for adv in advantages:
            print(f"  {adv}")
        
        # Generate sample SQL context
        print(f"\n{'-' * 80}")
        print("GENERATED CONTEXT FOR TEXT-TO-SQL AGENT:")
        print(f"{'-' * 80}")
        
        context = generate_sql_context(metadata)
        print(context)
        
        print("\n")
    
    print("=" * 80)
    print("DEMONSTRATION COMPLETE")
    print("=" * 80)


def generate_sql_context(metadata: Dict) -> str:
    """Generate structured context for Text-to-SQL agent"""
    
    context_parts = []
    
    context_parts.append("# Relevant Schema Context\n")
    
    # Tables and columns
    for table in metadata['tables']:
        if table in metadata['columns']:
            context_parts.append(f"\nTable: {table}")
            context_parts.append("Columns:")
            for col in metadata['columns'][table]:
                col_desc = f"  - {col['name']} ({col['type']})"
                if col.get('primary_key') == 'true':
                    col_desc += " [PRIMARY KEY]"
                if col.get('description'):
                    col_desc += f" - {col['description']}"
                context_parts.append(col_desc)
    
    # Relationships
    if metadata['relationships']:
        context_parts.append("\n# Table Relationships")
        for rel in metadata['relationships']:
            context_parts.append(f"  JOIN: {rel['from']} -> {rel['to']} ({rel['type']})")
    
    # Metrics
    if metadata['metrics']:
        context_parts.append("\n# Relevant Business Metrics")
        for metric in metadata['metrics']:
            context_parts.append(f"  {metric['name']}:")
            context_parts.append(f"    Description: {metric['description']}")
            context_parts.append(f"    Formula: {metric['formula']}")
    
    # Relational paths
    if metadata['textual_context']:
        context_parts.append("\n# Relational Context Paths")
        for i, path_ctx in enumerate(metadata['textual_context'][:3], 1):
            context_parts.append(f"  Path {i}: {path_ctx['path'][:150]}...")
    
    return '\n'.join(context_parts)


# Main execution
if __name__ == "__main__":
    demonstrate_pathrag_advantages()