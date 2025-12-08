# Lecture 1: Elasticsearch Introduction

### What is ELK?

![ELK](https://www.guru99.com/images/tensorflow/082918_1504_ELKStackTut1.png)

Elastic Stack is an open-source collection of tools for data analysis, processing, storage, and visualization. It is known as the ELK Stack (Elasticsearch, Logstash, Kibana), which also includes Beats.

### What is an ELK Cluster?
![3+ Nodes System](../../media/nodes-diagram.svg)


![Cluster Sizing Requirements](../../media/cluster-sizing.svg)

## Kibana – data visualization
**Kibana** provides pie charts, line charts, histograms, and maps. It can visualize Elasticsearch data and format it as desired.

## Logstash – data processing
**Logstash** collects and processes data from various sources and sends it to Elasticsearch.

![Introduction to Elasticsearch Architecture](https://miro.medium.com/v2/resize:fit:720/format:webp/1*rfcn4XSrNI1KZFbq89l7Jw.png)

*Introduction to Elasticsearch Architecture*

*Source: [Medium](https://medium.com)*

![Elasticsearch Scaling Strategies](https://miro.medium.com/v2/resize:fit:720/format:webp/1*c-7HdcP9m17PEi2a0qLg_w.png)

*Elasticsearch Scaling Strategies*

*Source: [Medium](https://medium.com)*


## Beats – data sending
**Beats** are lightweight agents that send data from machines to Logstash or directly to Elasticsearch. For example: **Filebeat** monitors log files and imports them.

Elastic Stack centralizes data and provides powerful tools for analysis and visualization.

![Elasticsearch](https://www.bigdatawire.com/wp-content/uploads/2019/03/elasticsearch_logo.png)

![ELK TLS Docker Diagram](https://raw.githubusercontent.com/wiki/swimlane/elk-tls-docker/images/elk-tls-docker-diagram.png)

*ELK TLS Docker Architecture Diagram*

*Source: [Swimlane GitHub Wiki](https://github.com/swimlane/elk-tls-docker/wiki)*

### Beats Family Table

| **Beat**       | **Purpose**                                                                                                                                                                |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Filebeat**   | Lightweight shipper for logs and other data. Collects logs from various sources like security devices, cloud, containers, hosts, and OT, simplifying log forwarding. (7)   |
| **Metricbeat** | Lightweight shipper for metric data. Gathers system and service metrics, such as CPU, memory, Redis, NGINX, and more. Simplifies system and service monitoring. (8)       |
| **Packetbeat** | Lightweight shipper for network data, enabling real-time monitoring of network activity.                                                                                   |
| **Winlogbeat** | Lightweight shipper for Windows event logs, focused on centralizing Windows-specific log data.                                                                             |
| **Auditbeat**  | Lightweight shipper for audit data, designed for security auditing and compliance monitoring.                                                                              |
| **Heartbeat**  | Lightweight shipper for uptime monitoring. Automates anomaly detection and accelerates root cause analysis with AIOps. (9)                                                |

### What is Elasticsearch?

Elasticsearch is an open-source distributed search and analytics engine designed for horizontal scalability, real-time search, and high reliability. Being part of NoSQL databases, it is built on the Apache Lucene search engine library.

Elasticsearch is widely used for:
- Log and event data analysis
- Full-text search
- Business analytics
- Application monitoring

Elasticsearch stores data in a schema-free JSON format, making it flexible and adaptable to different data models and structures. Through a distributed RESTful API, developers can easily perform complex searches, aggregations, and analyses on large datasets.

### Core Functions of Elasticsearch:

| Feature | Description |
|---------|-------------|
| **Distributed & Scalable** | Built for distributed environments with horizontal scaling across nodes. Easy node addition for growing workloads. |
| **Real-time Search** | Near real-time querying and analytics for up-to-date insights on rapidly changing datasets. |
| **Full-text Search** | Advanced full-text search with fuzzy searches, wildcards, and proximity queries using DSL. |
| **Schema-free JSON** | Data stored as schema-free JSON documents allowing flexible data structures. |
| **High Reliability** | Built-in data replication and fault tolerance with distributed data across multiple nodes. |
| **RESTful API** | Intuitive API using standard HTTP methods (GET, POST, PUT, DELETE). |
| **Rich Querying** | Extensive tools for complex data analysis and visualization using metrics, histograms, and aggregations. |

---

### Elasticsearch's Role in DevOps:

| Area | Description | Examples |
|------|-------------|----------|
| **Log Management** | • Centralized logging across systems<br>• Real-time monitoring<br>• Automated alerts | • Application troubleshooting<br>• Infrastructure monitoring<br>• Slack/email alerts |
| **Data Search** | • Fast indexing and filtering<br>• Advanced analytics with Kibana | • E-commerce search<br>• Log analysis<br>• Sales pattern analysis |
| **ML/AI** | • Anomaly detection<br>• Custom ML models | • Fraud detection<br>• Cyber threat monitoring |
| **Data Centralization** | Unified platform for multiple data sources | Integrated ERP/CRM views |
| **Performance** | Real-time queries for large datasets | Processing millions of log events |
| **Security & Audit** | • Audit log management<br>• Security monitoring | • IT security audits<br>• IPS/IDS analysis |                      |

---

### Important Concepts: 

![Component Relation](https://www.elastic.co/guide/en/elasticsearch/reference/current/images/data_processing_flow.png)

### Key Concepts:
| Term | Description | Example |
|------|-------------|---------|
| **Node** | Single Elasticsearch instance running on a machine with a specific cluster role | Data node storing logs; coordinating node routing queries |
| **Cluster** | Collection of nodes working together for data storage and management | 3-node setup: 2 data nodes + 1 master-eligible node |
| **Index** | Logical namespace representing a collection of similar documents | Log index containing server errors and queries |
| **Shard** | Individual storage unit of an index distributed across nodes | "server_logs" split into 5 shards, each with a replica |
| **Replication** | Process of creating and maintaining shard copies across nodes | Each shard replicated once for failover protection |
| **Cluster State** | Global repository of cluster config and metadata, including index/shard distribution | Stores info about new index creation and shard allocation |
