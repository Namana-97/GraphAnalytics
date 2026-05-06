# Real-Time Graph Analytics System with Neo4j and Kubernetes

**Institution:** Arizona State University  
**Dataset:** NYC Taxi Trip Data (March 2022, Bronx subset)  

---

## Overview

This project builds a real-time graph analytics pipeline on NYC taxi trip data using Neo4j as the graph database. Locations are modeled as nodes and trips as weighted relationships, enabling efficient execution of graph algorithms like PageRank and Breadth-First Search.

Phase 1 establishes a Dockerized Neo4j environment with data ingestion and algorithm execution. Phase 2 extends this into a fully distributed streaming pipeline using Kubernetes and Kafka, capable of processing continuous data flows with end-to-end latency under 2 seconds.

---

## Repository Structure

```
.
├── README.md
├── Phase1/
│   ├── Dockerfile          # Neo4j + GDS + Python environment
│   ├── data_loader.py      # Parquet to CSV conversion and batch ingestion
│   └── interface.py        # PageRank and BFS query interface
│
└── Phase2/
    ├── zookeeper-setup.yaml         # Zookeeper deployment and service
    ├── kafka-setup.yaml             # Kafka broker with dual-listener config
    ├── neo4j-values.yaml            # Neo4j Helm chart values with GDS sidecar
    └── kafka-neo4j-connector.yaml   # Kafka Connect to Neo4j pipeline
```

---

## Phase 1: Docker and Neo4j

### Architecture

```
Host Machine
    |
    v
Docker Container
    |-- Neo4j 5.5.0 (GDS Plugin 2.21.0)
    |-- OpenJDK 21
    |-- Python (neo4j, pandas, pyarrow)
    |
    |-- data_loader.py  -->  LOAD CSV (batched transactions)  -->  Neo4j
    |-- interface.py    -->  PageRank / BFS queries           -->  Results
```

### Graph Schema

- **Nodes:** `Location` with integer ID property
- **Relationships:** `TRIP` with properties `distance`, `fare`, `pickup_dt`, `dropoff_dt`
- **Scale:** 261 location nodes, 7.2 million trip relationships

### Algorithms

| Algorithm | Configuration | Purpose |
|---|---|---|
| PageRank | Damping factor 0.85, 20 iterations, distance-weighted | Identify central hubs in the taxi network |
| BFS / Dijkstra | GDS shortest path, distance-weighted | Compute shortest paths between locations |

### Setup

**Prerequisites:** Docker

```bash
# Build the image
cd Phase1
docker build -t neo4j-graph-analytics .

# Run the container
docker run -d \
  --name neo4j-graph-analytics \
  -p 7474:7474 -p 7687:7687 \
  neo4j-graph-analytics

# Load data
docker exec neo4j-graph-analytics python data_loader.py

# Run queries
docker exec neo4j-graph-analytics python interface.py
```

---

## Phase 2: Kubernetes and Kafka

### Architecture

```
data_producer.py
      |
      v
   Kafka (localhost:9092 / kafka-service:29092)
      |
      v
 Kafka Connect (kafka-neo4j-connector)
      |
      v
   Neo4j (Helm, GDS Plugin, 5Gi PV)
      |
      v
 PageRank / BFS on streamed data
```

### Infrastructure

| Component | Deployment | Config File |
|---|---|---|
| Zookeeper | Kubernetes Deployment, port 2181 | `zookeeper-setup.yaml` |
| Kafka | Kubernetes Deployment, dual listeners | `kafka-setup.yaml` |
| Neo4j | Helm chart, standalone community edition | `neo4j-values.yaml` |
| Kafka-Neo4j Connector | Kubernetes Deployment, pre-built image | `kafka-neo4j-connector.yaml` |

### Kafka Listener Configuration

Kafka is configured with two listeners to handle both external and in-cluster access:

| Listener | Address | Used By |
|---|---|---|
| PLAINTEXT | `localhost:9092` | External clients (data producer on host) |
| INTERNAL | `kafka-service:29092` | In-cluster pods (Kafka Connect, Neo4j connector) |

### Setup

**Prerequisites:** Minikube, Helm, kubectl

```bash
# Start Minikube with sufficient resources
minikube start --memory=10240 --cpus=8

# Deploy Zookeeper
kubectl apply -f Phase2/zookeeper-setup.yaml

# Deploy Kafka
kubectl apply -f Phase2/kafka-setup.yaml

# Deploy Neo4j via Helm
helm repo add neo4j https://helm.neo4j.com/neo4j
helm install neo4j neo4j/neo4j -f Phase2/neo4j-values.yaml

# Deploy the Kafka-Neo4j Connector
kubectl apply -f Phase2/kafka-neo4j-connector.yaml

# Verify all pods are running
kubectl get pods
```

---

## Results

### Phase 1

- Data loading created 261 location nodes and 7.2 million trip relationships.
- PageRank successfully identified central hubs and peripheral locations in the Bronx taxi network.
- BFS  correctly computed shortest paths across all five test cases, including routes such as 159 to 212 and 3 to 240, returning full path information with intermediate stops.

### Phase 2

| Component | Score |
|---|---|
| Zookeeper | 10/10 |
| Kafka infrastructure and connectivity | 20/20 |
| Neo4j deployment and connectivity | 15/15 |
| Kafka-Neo4j connector | 15/15 |
| Data loading | 20/20 |
| End-to-end pipeline validation | 20/20 |

End-to-end latency from message send to queryable data in Neo4j was under 2 seconds. The pipeline processed thousands of records per minute with no data loss.

---

## Tech Stack

| Technology | Purpose |
|---|---|
| Neo4j 5.x | Graph database |
| Neo4j GDS Plugin | PageRank and shortest path algorithms |
| Docker | Phase 1 containerization |
| Apache Kafka | Message broker for real-time streaming |
| Apache ZooKeeper | Kafka coordination |
| Kafka Connect | Stream processing pipeline to Neo4j |
| Kubernetes (Minikube) | Container orchestration for Phase 2 |
| Helm | Neo4j deployment management |
| Python (neo4j, pandas, pyarrow) | Data processing and query interface |
