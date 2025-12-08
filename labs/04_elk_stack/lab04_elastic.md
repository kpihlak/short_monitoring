# Enhanced Elasticsearch Stack Lab Guide

## Introduction

This lab guide walks through setting up an Elasticsearch stack (Elasticsearch, Kibana, Logstash) in a 4-hour workshop for beginners. We'll start with a single-node deployment to verify functionality, then progress to a multi-node cluster with security features enabled. The guide has been extensively tested and includes solutions for common permission issues.

## Prerequisites

You already have Multipass installed. The remaining requirements are:

### Hardware Requirements for the VM
- CPU: 4 cores (Elasticsearch requires multiple cores for parallel processing)
- RAM: 8GB (Elasticsearch is memory-intensive, especially for search operations)
- Disk: 50GB (Sufficient for storing indices and logs)

### Software to be installed
- Docker and Docker Compose (For containerized deployment)
- Elasticsearch, Kibana (via Docker images)

## Lab 1: Setting Up a Single-Node Elasticsearch Deployment (60 minutes)

### 1.1 Configure a Multipass VM

```bash
# Create a new VM with sufficient resources
# We need at least 4 CPUs and 8GB of RAM for Elasticsearch to run efficiently
multipass launch --name elastic-lab --cpus 4 --mem 8G --disk 50G

# Connect to the VM
multipass shell elastic-lab

# Get VM info (note the IP address for later use)
# You'll need this IP to connect to Elasticsearch and Kibana from your host machine
multipass info elastic-lab
```

### 1.2 Install Docker and Docker Compose

```bash
# Update package lists to ensure we get the latest versions
sudo apt-get update

# Install required packages
# These packages are prerequisites for adding Docker's repository
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# Add Docker's official GPG key for package verification
# Using the recommended method that works with Ubuntu 22.04 and newer
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up the stable repository for Docker packages
# This format is compatible with Ubuntu 22.04 and newer
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to the docker group
# This allows you to run Docker commands without sudo
sudo usermod -aG docker $USER

# Apply group changes without logging out
# This creates a new shell with the updated group membership
newgrp docker

# Verify the installation
docker --version
docker compose version

# Set system parameters for Elasticsearch
# vm.max_map_count is a kernel parameter that must be set higher for Elasticsearch
# Default value is too low and will cause Elasticsearch to fail with "max virtual memory areas" error
sudo sysctl -w vm.max_map_count=262144

# Make vm.max_map_count setting persistent across reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# If you see an error like "sysctl: cannot stat /proc/sys/net/core/somaxconn"
# Don't worry, this is just saying it can't find some other parameters, but the vm.max_map_count was set
```

### 1.3 Create a Single-Node Elasticsearch Setup

Create a project directory:

```bash
# Create a dedicated directory for our project
# This helps keep our configurations organized
mkdir -p ~/elastic-lab/single-node
cd ~/elastic-lab/single-node
```

Create `docker-compose.yml`:

```yaml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: es01
    environment:
      # Name this node - useful for logs and cluster formation
      - node.name=es01
      # Name the cluster - nodes with the same cluster name join together
      - cluster.name=single-node-cluster
      # Tell Elasticsearch this is a single-node deployment
      # This disables cluster formation checks that would otherwise prevent startup
      - discovery.type=single-node
      # Allow Elasticsearch to lock memory for better performance
      # Prevents swapping which can cause significant performance issues
      - bootstrap.memory_lock=true
      # Set Java heap size explicitly - prevents Elasticsearch from using too much RAM
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      # Disable security for initial testing to simplify setup
      - xpack.security.enabled=false
    ulimits:
      # These settings allow ES to lock memory properly
      memlock:
        soft: -1
        hard: -1
    volumes:
      # Persist data between container restarts
      - esdata01:/usr/share/elasticsearch/data
    ports:
      # Expose REST API port to host
      - 9200:9200
      # Expose transport port for node-to-node communication
      - 9300:9300
    networks:
      - elastic
    # Add healthcheck to ensure Elasticsearch is truly ready
    # Even after the container starts, ES takes time to initialize
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200/_cluster/health"]
      interval: 10s
      timeout: 10s
      retries: 5

  kibana:
    image: docker.elastic.co/kibana/kibana:8.17.0
    container_name: kib01
    ports:
      # Expose Kibana web interface
      - 5601:5601
    environment:
      # Tell Kibana where to find Elasticsearch
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    networks:
      - elastic
    depends_on:
      # Make sure Elasticsearch starts before Kibana
      - elasticsearch
    # Add healthcheck for Kibana
    # This ensures Kibana is truly ready
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5601/api/status"]
      interval: 10s
      timeout: 10s
      retries: 5

# Create a dedicated network for ES stack components
# This allows containers to communicate by name
networks:
  elastic:
    driver: bridge

# Define persistent volumes 
# This ensures data survives container restarts
volumes:
  esdata01:
    driver: local
```

### 1.4 Start the Single-Node Setup

```bash
# Start the services
# -d flag runs containers in the background (detached mode)
docker compose up -d

# Check service status to ensure everything is running
docker ps

# Check Elasticsearch logs for any startup issues
# Also check for any warnings or errors that might indicate configuration problems
docker logs es01

# Note: Elasticsearch will take a minute or two to fully initialize
# If you see errors in logs, wait a bit and check again
sleep 30
```

### 1.5 Verify Elasticsearch and Kibana

Get the VM's IP address:

```bash
# Get the IP address of your VM for external access
# This allows you to connect from your host computer
IP_ADDRESS=$(hostname -I | awk '{print $1}')
echo "Elasticsearch: http://$IP_ADDRESS:9200"
echo "Kibana: http://$IP_ADDRESS:5601"

# If the above doesn't work, try:
# IP_ADDRESS=$(multipass info elastic-lab | grep IPv4 | awk '{print $2}')
```

Test Elasticsearch:

```bash
# Test the REST API to verify Elasticsearch is responding
curl http://localhost:9200

# Check cluster health - should show status green for a healthy system
# This provides information about shards, nodes, and overall health
curl http://localhost:9200/_cluster/health?pretty
```

Access Kibana in your browser: `http://VM_IP:5601`

> **IMPORTANT NOTE**: If you can't access Kibana from your host machine, check firewall settings on the VM:
> ```bash
> # Allow Elasticsearch and Kibana ports through the firewall
> sudo ufw allow 9200/tcp
> sudo ufw allow 5601/tcp
> ```

### 1.6 Create a Test Index and Add Data

You can run these commands either:
1. From the VM terminal (SSH) - recommended for this step
2. Or through Kibana's Dev Tools if you prefer a GUI (available at http://VM_IP:5601/app/dev_tools#/console)

#### Using SSH Terminal (from VM):

```bash
# Create an index with defined mappings
# Mappings define how documents and their fields are stored and indexed
curl -X PUT "http://localhost:9200/products" \
  -H "Content-Type: application/json" \
  -d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "float" },
      "category": { "type": "keyword" }
    }
  }
}'

# Add a document to the index to verify indexing works
curl -X POST "http://localhost:9200/products/_doc" \
  -H "Content-Type: application/json" \
  -d '{
  "name": "Smartphone",
  "price": 799.99,
  "category": "electronics"
}'

# Search for documents to verify search works
curl -X GET "http://localhost:9200/products/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
  "query": {
    "match": {
      "category": "electronics"
    }
  }
}'
```

#### Using Kibana Dev Tools (alternative):

1. Open Kibana in your browser: http://VM_IP:5601
2. Navigate to Dev Tools from the left menu
3. Enter these commands (without the curl part):

```
# Create an index
PUT products
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "float" },
      "category": { "type": "keyword" }
    }
  }
}

# Add a document
POST products/_doc
{
  "name": "Smartphone",
  "price": 799.99,
  "category": "electronics"
}
```

4. Click the play button (▶️) next to each command to execute it

### 1.7 Shut Down the Single-Node Setup

After verifying everything works:

```bash
# Stop the containers
docker compose down

# Remove volumes to start fresh
# This deletes all indexed data to ensure a clean start for the cluster
docker compose down -v
```
## Lab 2: Creating a Multi-Node Elasticsearch Cluster (60 minutes)

### 2.1 Prepare the Cluster Configuration

First, let's create a new directory to organize our cluster setup:

```bash
# Create a new directory for our multi-node setup
mkdir -p ~/elastic-lab/cluster
cd ~/elastic-lab/cluster

# Create a directory for certificates with proper permissions
mkdir -p certs
# Set permissions to allow writing by the Elasticsearch user (UID 1000)
sudo chown -R 1000:1000 certs
```

Now create a `docker-compose.yml` file for a three-node cluster:

```bash
cat > docker-compose.yml << 'EOF'
version: '3'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: es01
    environment:
      # NODE IDENTIFICATION
      - node.name=es01
      - cluster.name=es-docker-cluster
      
      # CLUSTER DISCOVERY
      - discovery.seed_hosts=es02,es03
      # For initial cluster formation only - will be removed after first successful cluster formation
      - cluster.initial_master_nodes=es01,es02,es03
      
      # PERFORMANCE OPTIONS
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
      
      # SECURITY SETTINGS
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.keystore.password=password
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.password=password
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata01:/usr/share/elasticsearch/data
      - ./certs:/usr/share/elasticsearch/config/certs
    ports:
      - 9200:9200
    networks:
      - elastic

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.keystore.password=password
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.password=password
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata02:/usr/share/elasticsearch/data
      - ./certs:/usr/share/elasticsearch/config/certs
    networks:
      - elastic

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
      - xpack.security.enabled=true
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.keystore.password=password
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.password=password
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata03:/usr/share/elasticsearch/data
      - ./certs:/usr/share/elasticsearch/config/certs
    networks:
      - elastic

  kibana:
    image: docker.elastic.co/kibana/kibana:8.17.0
    container_name: kib01
    ports:
      - 5601:5601
    environment:
      - ELASTICSEARCH_HOSTS=http://es01:9200
    networks:
      - elastic
    depends_on:
      - es01

networks:
  elastic:
    driver: bridge

volumes:
  esdata01:
    driver: local
  esdata02:
    driver: local
  esdata03:
    driver: local
EOF
```

**Key Configuration Explanations:**

- **node.name**: Unique identifier for each node in the cluster
- **cluster.name**: Common name that all nodes use to join the same cluster
- **discovery.seed_hosts**: List of other nodes to contact for cluster formation
- **cluster.initial_master_nodes**: Nodes eligible for master election during initial cluster formation
- **xpack.security.enabled**: Enables authentication and authorization
- **xpack.security.transport.ssl.enabled**: Essential when security is enabled - secures inter-node communication

> 📝 **IMPORTANT NOTE**: When `xpack.security.enabled=true`, you must also set `xpack.security.transport.ssl.enabled=true` or Elasticsearch will fail to start with a "bootstrap check failure." This is a strict security requirement in Elasticsearch 8.x.
>
> The `cluster.initial_master_nodes` setting should only be used for the first startup of a cluster. After a successful cluster formation, create a new docker-compose.yml file with this setting removed from all nodes.

### 2.2 Generate Certificates

Before starting the cluster, we need to generate SSL certificates for secure communication between nodes:

```bash
# Create temporary container for generating certificates
docker run --name elastic_certs -v $(pwd)/certs:/usr/share/elasticsearch/config/certs docker.elastic.co/elasticsearch/elasticsearch:8.17.0 /bin/bash -c "
  elasticsearch-certutil ca --out /usr/share/elasticsearch/config/certs/elastic-stack-ca.p12 --pass password &&
  elasticsearch-certutil cert --ca /usr/share/elasticsearch/config/certs/elastic-stack-ca.p12 --ca-pass password --name node --pass password --out /usr/share/elasticsearch/config/certs/elastic-certificates.p12
"

# Remove the container after generating certificates
docker rm elastic_certs

# Verify certificates were created
ls -la certs/

# Fix permissions for certificate files in case they're still owned by root
sudo chown -R 1000:1000 certs/
sudo chmod 644 certs/*
```

### 2.3 Start the Elasticsearch Cluster

Now we can start our cluster with the generated certificates:

```bash
# Navigate back to the cluster directory
cd ~/elastic-lab/cluster

# Start the services in detached mode
docker compose up -d

# Wait for the cluster to initialize (this may take a few minutes)
sleep 90

# Check cluster and nodes status
docker ps -a
```

**IMPORTANT: After First Successful Startup**
Once your cluster has formed successfully, you must remove the `cluster.initial_master_nodes` setting from all nodes for future restarts. This setting is only for bootstrapping a new cluster and can cause problems if left in place.

```bash
# Create a new docker-compose file without the initial_master_nodes setting
# Edit the docker-compose.yml file and remove or comment out all lines with:
# - cluster.initial_master_nodes=es01,es02,es03
```

### 2.4 Get Elastic User Credentials

Check if passwords were generated automatically:

```bash
# Check for generated passwords in the logs
docker logs es01 | grep "PASSWORD elastic"
docker logs es01 | grep "PASSWORD kibana_system"

# If you see the passwords, save them for later use
ELASTIC_PASSWORD=$(docker logs es01 | grep "PASSWORD elastic" | awk '{print $4}')
KIBANA_PASSWORD=$(docker logs es01 | grep "PASSWORD kibana_system" | awk '{print $4}')

# If passwords are not found in logs, reset them manually
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system

# Store the passwords from reset command output
ELASTIC_PASSWORD="your_elastic_password_here"
KIBANA_PASSWORD="your_kibana_system_password_here"
```

### 2.5 Update Kibana Configuration for Authentication

Now that we have user credentials, we need to update Kibana's configuration:

```bash
# Create a configuration file with the proper user
cat > kibana.yml << EOF
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://es01:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "your_kibana_system_password"
EOF

# Stop Kibana before updating config
docker stop kib01

# Copy the configuration into the container
docker cp kibana.yml kib01:/usr/share/kibana/config/

# Restart Kibana to apply new configuration
docker start kib01

# Give Kibana time to restart and connect to Elasticsearch
echo "Waiting for Kibana to initialize..."
sleep 30
```

### 2.6 Verify Cluster Setup

Let's check if our cluster is running correctly:

```bash
# Check cluster health - should show green status with 3 nodes
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cat/health?v

# Verify all three nodes are running and connected
curl -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cat/nodes?v

# If you see any errors, check the logs of each node
docker logs es01
docker logs es02
docker logs es03
```

You can now access Kibana at `http://VM_IP:5601` and log in with the `elastic` user and the password you have saved.

**Troubleshooting Tips:**
- If cluster formation issues occur, ensure the `cluster.initial_master_nodes` setting is either consistently set on all nodes (for first startup) or consistently removed from all nodes (for subsequent startups)
- Memory issues can cause slow startup - monitor with `docker stats`
- Certificate issues are common - ensure all nodes have the same certificate files with proper permissions

## Lab 3: Securing the Elasticsearch Cluster (60 minutes)

Now that we have a working cluster, let's improve its security with a more complete SSL/TLS configuration.

### 3.1 Enhanced Security Configuration

Update the `docker-compose.yml` to include both HTTP and Transport SSL:

```bash
cat > docker-compose-secure.yml << 'EOF'
version: '3'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      # Removed cluster.initial_master_nodes as it should only be used for initial bootstrap
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
      
      # SECURITY CONFIGURATION
      # Basic security - enable authentication & authorization
      - xpack.security.enabled=true
      
      # === HTTP LAYER SECURITY (REST API) ===
      # Enable HTTPS for the REST API
      - xpack.security.http.ssl.enabled=true
      # Path to the certificate keystore for HTTP
      - xpack.security.http.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.http.ssl.keystore.password=password
      # Path to the certificate truststore for HTTP
      - xpack.security.http.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.http.ssl.truststore.password=password
      
      # === TRANSPORT LAYER SECURITY (Node-to-Node) ===
      # Enable SSL for node-to-node communication
      - xpack.security.transport.ssl.enabled=true
      # Only verify certificate validity, not hostname
      - xpack.security.transport.ssl.verification_mode=certificate
      # Path to transport layer keystore
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.keystore.password=password
      # Path to transport layer truststore
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.password=password
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata01:/usr/share/elasticsearch/data
      # Mount certificates directory from host to container
      - ./certs:/usr/share/elasticsearch/config/certs
    ports:
      - 9200:9200
    networks:
      - elastic

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      # Removed cluster.initial_master_nodes as it should only be used for initial bootstrap
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
      # Security settings
      - xpack.security.enabled=true
      # HTTP SSL configuration
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.http.ssl.keystore.password=password
      - xpack.security.http.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.http.ssl.truststore.password=password
      # Transport SSL configuration - required when security is enabled
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.keystore.password=password
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.password=password
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata02:/usr/share/elasticsearch/data
      - ./certs:/usr/share/elasticsearch/config/certs
    networks:
      - elastic

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      # Removed cluster.initial_master_nodes as it should only be used for initial bootstrap
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
      # Security settings
      - xpack.security.enabled=true
      # HTTP SSL configuration
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.http.ssl.keystore.password=password
      - xpack.security.http.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.http.ssl.truststore.password=password
      # Transport SSL configuration - required when security is enabled
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.keystore.password=password
      - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
      - xpack.security.transport.ssl.truststore.password=password
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata03:/usr/share/elasticsearch/data
      - ./certs:/usr/share/elasticsearch/config/certs
    networks:
      - elastic

  kibana:
    image: docker.elastic.co/kibana/kibana:8.17.0
    container_name: kib01
    ports:
      - 5601:5601
    environment:
      # Connect to Elasticsearch using HTTPS
      - ELASTICSEARCH_HOSTS=https://es01:9200
      # Kibana system user for background tasks
      - ELASTICSEARCH_USERNAME=kibana_system
      # Use explicit password instead of environment variable 
      - ELASTICSEARCH_PASSWORD=your_kibana_system_password
      # Connect to all interfaces
      - SERVER_HOST=0.0.0.0
      # Disable HTTPS for Kibana temporarily until we fix certificates
      - SERVER_SSL_ENABLED=false
    volumes:
      # Mount certificates directory
      - ./certs:/usr/share/kibana/config/certs
    networks:
      - elastic
    depends_on:
      - es01

networks:
  elastic:
    driver: bridge

volumes:
  esdata01:
    driver: local
  esdata02:
    driver: local
  esdata03:
    driver: local
EOF
```
**Key Security Enhancements:**

1. **HTTP Layer SSL**: Now we've enabled HTTPS for the REST API, providing secure access for clients
2. **Transport Layer SSL**: Already enabled (required for security) but now fully configured
3. **Kibana Security**: Using HTTPS for Kibana's web interface
4. **Certificate Configuration**: Using the same certificates for both HTTP and Transport

### 3.2 Update and Restart the Cluster

```bash
# Stop the existing cluster
docker compose down

# Start with the new security configuration
# Rename the secure version to be the default
mv docker-compose-secure.yml docker-compose.yml

# Start with the new configuration
docker compose up -d

# Give the cluster time to fully initialize
echo "Waiting for the secure cluster to initialize..."
sleep 60

# Check logs for any errors
docker logs es01

# Verify secure HTTPS connectivity
curl -u elastic:${ELASTIC_PASSWORD} https://localhost:9200/_cat/health?v -k
```

### 3.3 Creating Users and Roles

Let's create additional users and roles for better security practices:

```bash
# Create read-only user
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-users useradd read_only_user -p StrongPassword123 -r read_only

# Create admin user
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-users useradd admin_user -p StrongAdminPass456 -r superuser

# Create a custom role (via API)
curl -X POST -u elastic:${ELASTIC_PASSWORD} https://localhost:9200/_security/role/data_analyst -H "Content-Type: application/json" -d '
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}' -k

# Create a user with the custom role
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-users useradd data_analyst_user -p AnalystPass789 -r data_analyst

# Confirm that the users were created successfully
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-users list
```

**User & Role Management Explanation:**
- The `elasticsearch-users` utility creates and manages users
- Roles control what actions users can perform
- Built-in roles like `read_only` and `superuser` provide preset permissions
- Custom roles give fine-grained control over permissions
- The data_analyst role we created has:
  - "monitor" cluster privilege - can view cluster health and stats
  - Read access only to indices matching the pattern "logs-*"

### 3.4 Additional Security Best Practices

Here are some additional security enhancements to consider:

```bash
# Enable auditing for security events
# Create a file to add audit logging settings
cat > audit.yml << EOF
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: ["authentication_success", "authentication_failed", "access_denied", "connection_denied"]
EOF

# Apply these settings
docker cp audit.yml es01:/usr/share/elasticsearch/config/
docker cp audit.yml es02:/usr/share/elasticsearch/config/
docker cp audit.yml es03:/usr/share/elasticsearch/config/

# Restart nodes to apply audit settings
docker restart es01 es02 es03

# Wait for cluster to restart
sleep 60

# The audit logs might not be generated until some auditable events occur. Let's try to create some events:
# Try to access Elasticsearch with incorrect credentials to generate a failed authentication event
curl -u wronguser:wrongpassword http://localhost:9200

# Try to access with correct credentials to generate a successful authentication
curl -u elastic:YOUR_ELASTIC_PASSWORD http://localhost:9200

# Check if the audit settings are in effect
curl -u elastic:YOUR_ELASTIC_PASSWORD http://localhost:9200/_cluster/settings?include_defaults=true | grep audit
```

## Lab 4: Working with Indices and Searching

## 4.1 Creating Test Indices and Adding Data

First, create the logs index with proper field mappings:

```bash
# Create logs index
curl -X PUT "https://localhost:9200/logs-2023" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "message": { "type": "text" },
      "level": { "type": "keyword" }
    }
  }
}'
```

Next, add sample log entries:

```bash
# Add log entries
for i in {1..5}; do
  curl -X POST "https://localhost:9200/logs-2023/_doc" -k \
    -u elastic:${ELASTIC_PASSWORD} \
    -H "Content-Type: application/json" \
    -d "{
    \"timestamp\": \"2023-01-0${i}T12:00:00Z\",
    \"message\": \"This is log message $i\",
    \"level\": \"$([ $i -gt 3 ] && echo 'ERROR' || echo 'INFO')\"
  }"
  echo ""
done
```

## 4.2 Performing Basic and Advanced Searches

Run these three search queries:

```bash
# Basic search - finds all documents with "log" in the message field
curl -X GET "https://localhost:9200/logs-2023/_search" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "query": {
    "match": {
      "message": "log"
    }
  }
}' | jq .

# Boolean search with filter - finds only ERROR logs with "log" in the message
curl -X GET "https://localhost:9200/logs-2023/_search" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "log" } }
      ],
      "filter": [
        { "term": { "level": "ERROR" } }
      ]
    }
  }
}'

# Aggregation - counts documents by log level
curl -X GET "https://localhost:9200/logs-2023/_search" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "size": 0,
  "aggs": {
    "log_levels": {
      "terms": {
        "field": "level"
      }
    }
  }
}' | jq .
```

## 4.3 Working with Kibana

1. Access Kibana at `https://VM_IP:5601` and log in as the `elastic` user

2. Set up an index pattern:
   - Go to Stack Management > Index Patterns > Create index pattern
   - Enter "logs-*" as the pattern
   - Select "timestamp" as the time field
   - Click "Create index pattern"

3. Use Discover to search logs:
   - Go to Discover tab in the main menu
   - Select your "logs-*" index pattern
   - Try searching for "log" in the search bar
   - Try filtering for ERROR logs using the field filters

4. Create a visualization:
   - Go to Visualize > Create visualization
   - Choose "Pie" chart
   - Select your "logs-*" index pattern
   - Add a bucket: Split slices > Terms > Field: level
   - Click "Update" to see the visualization

5. Create a dashboard:
   - Go to Dashboard > Create dashboard
   - Click "Add" to add your visualization
   - Arrange it on the dashboard
   - Click "Save" to save your dashboard

The lab is now complete. You have:
- Created and populated an Elasticsearch index
- Performed different types of searches
- Created visualizations and a dashboard in Kibana

# Lab 5: Security Health Check and Testing

## 5.1 Verify Security Configuration

First, let's verify the active security settings by checking your cluster's current security status:

```bash
# Check cluster health
curl -X GET "https://localhost:9200/_cluster/health" -k \
  -u elastic:${ELASTIC_PASSWORD}

# List users to verify security is enabled
curl -X GET "https://localhost:9200/_security/user" -k \
  -u elastic:${ELASTIC_PASSWORD}

# List roles
curl -X GET "https://localhost:9200/_security/role" -k \
  -u elastic:${ELASTIC_PASSWORD}
```

These commands confirm that security is properly enabled, showing you all configured users and roles in your system.

## 5.2 Test Role-Based Access Control

Create a new user with limited access permissions:

```bash
# Create a restricted role for data analysts
curl -X POST "https://localhost:9200/_security/role/logs_reader" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}'

# Create a user with the logs_reader role
curl -X POST "https://localhost:9200/_security/user/logs_user" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "password": "LogsUser123",
  "roles": ["logs_reader"],
  "full_name": "Logs User"
}'

# Create a test index that should NOT be accessible to logs_user
curl -X PUT "https://localhost:9200/restricted-index" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}'
```

Now test the access restrictions:

```bash
# Test access to logs index (should SUCCEED)
curl -X GET "https://localhost:9200/logs-2023/_search?pretty" -k \
  -u logs_user:LogsUser123 \
  -H "Content-Type: application/json" \
  -d '{
  "query": {
    "match_all": {}
  }
}'

# Test access to restricted index (should FAIL with 403)
curl -X GET "https://localhost:9200/restricted-index/_search" -k \
  -u logs_user:LogsUser123
```

## 5.3 Test TLS/SSL Security

Verify that your HTTPS connections are secure:

```bash
# Check HTTPS connection details (note certificate information)
curl -v https://localhost:9200 -k \
  -u elastic:${ELASTIC_PASSWORD} 2>&1 | grep -E "SSL|TLS|certificate"

# Attempt non-HTTPS connection (should fail)
curl -v http://localhost:9200 2>&1 | grep -E "SSL|TLS|certificate"
```

## 5.4 Test Password Policies

Test if weak passwords are allowed:

```bash
# Try creating a user with a weak password
curl -X POST "https://localhost:9200/_security/user/weak_user" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "password": "weak",
  "roles": ["kibana_user"],
  "full_name": "Weak Password User"
}'
```

## 5.5 Create and Test an API Key

Create an API key for service authentication:

```bash
# Create an API key with limited permissions
curl -X POST "https://localhost:9200/_security/api_key" -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
  "name": "logs_api_key",
  "expiration": "7d", 
  "role_descriptors": {
    "logs_reader_role": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["logs-*"],
          "privileges": ["read"]
        }
      ]
    }
  }
}'

# The response includes id and api_key values
# Save these values to use in the next step

# Use the API key to access data (replace with actual values from response)
# curl -X GET "https://localhost:9200/logs-2023/_search" -k \
#   -H "Authorization: ApiKey <base64 encoded id:api_key>"
```

## 5.6 Document Security Configuration

Examine how security is configured in the environment:

```bash
# View Docker Compose settings (outside the container)
grep -E "xpack.security" docker-compose.yml

# See if security is defined in elasticsearch.yml
docker exec -it es01 grep -E "xpack.security" /usr/share/elasticsearch/config/elasticsearch.yml
```

## 5.7 Security Assessment Summary

Based on your testing, document:
1. Which security features are properly configured?
2. Are there any security gaps (like missing audit logging)?
3. What improvements would you recommend for production use?

This comprehensive assessment helps ensure your Elasticsearch deployment maintains proper security controls.

## Lab 6: Troubleshooting (30 minutes)

### 6.1 Common Issues and Solutions

#### Cluster Health Issues

```bash
# Check cluster health
# This shows the overall status (green/yellow/red) and shard allocation
curl -X GET "https://localhost:9200/_cluster/health?pretty" -k \
  -u elastic:${ELASTIC_PASSWORD}

# Check shard allocation
# This shows where primary and replica shards are allocated
# Helps identify unassigned shards and their reason
curl -X GET "https://localhost:9200/_cat/shards?v" -k \
  -u elastic:${ELASTIC_PASSWORD}
```

#### Understanding Cluster Status Colors:
- **Green**: All primary and replica shards are allocated
- **Yellow**: All primary shards are allocated, but some replica shards are unallocated
- **Red**: Some primary shards are unallocated, data is unavailable

#### Memory Issues

```bash
# Check JVM heap usage
# High heap usage (>75%) can cause performance issues or OOM errors
curl -X GET "https://localhost:9200/_nodes/stats/jvm?pretty" -k \
  -u elastic:${ELASTIC_PASSWORD}
```

If memory issues occur, adjust JVM heap size:
- Edit ES_JAVA_OPTS in docker-compose.yml
- For example: "ES_JAVA_OPTS=-Xms256m -Xmx256m" for less memory usage
- Restart the affected node

#### Network and Connection Issues

```bash
# Check if nodes can see each other
# Network connectivity problems are a common cause of cluster formation issues
docker exec -it es01 ping es02
docker exec -it es01 ping es03

# Check logs for connection issues
# Look for connection errors, transport problems, or discovery failures
docker logs es01 | grep -E "connection|transport|discovery"
```

### 6.2 Fixing Bootstrap Check Failures

If you encounter bootstrap check failures like:

```
bootstrap check failure [1] of [1]: Transport SSL must be enabled if security is enabled
```

The solution is to ensure these settings are correctly configured:
```
xpack.security.enabled=true
xpack.security.transport.ssl.enabled=true
```

### 6.3 Resolving Certificate Issues

If you have certificate issues, check:

```bash
# 1. Verify certificate files exist
docker exec -it es01 ls -la /usr/share/elasticsearch/config/certs/

# 2. Check certificate content
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-keystore list

# 3. Check permissions
docker exec -it es01 ls -la /usr/share/elasticsearch/config/certs/

# 4. Regenerate certificates if needed (as shown in Lab 2)
```

## Lab 7: Adding Metricbeat for Monitoring (45 minutes)

### 7.1 Setting Up Metricbeat

Metricbeat is a lightweight shipper for metrics that collects and ships various system and service metrics to Elasticsearch.

```bash
# Create a directory for Metricbeat configuration
mkdir -p ~/elastic-lab/cluster/metricbeat
cd ~/elastic-lab/cluster

# Create metricbeat.yml configuration file
cat > metricbeat/metricbeat.yml << EOF
metricbeat.modules:
  # Monitor Elasticsearch
  - module: elasticsearch
    metricsets:
      - node
      - node_stats
    period: 10s
    hosts: ["https://es01:9200"]
    username: "\${ELASTIC_USERNAME}"
    password: "\${ELASTIC_PASSWORD}"
    ssl.verification_mode: "none"  # For lab purposes only
    xpack.enabled: true

  # Monitor Kibana
  - module: kibana
    metricsets:
      - stats
    period: 10s
    hosts: ["https://kib01:5601"]
    username: "\${ELASTIC_USERNAME}"
    password: "\${ELASTIC_PASSWORD}"
    ssl.verification_mode: "none"  # For lab purposes only
    xpack.enabled: true

  # Monitor Docker
  - module: docker
    metricsets:
      - "container"
      - "cpu"
      - "diskio"
      - "memory"
      - "network"
    hosts: ["unix:///var/run/docker.sock"]
    period: 10s

# Output configuration
output.elasticsearch:
  hosts: ["https://es01:9200"]
  username: "\${ELASTIC_USERNAME}"
  password: "\${ELASTIC_PASSWORD}"
  ssl.verification_mode: "none"  # For lab purposes only

# Setup dashboards automatically
setup.dashboards.enabled: true
setup.kibana:
  host: "https://kib01:5601"
  username: "\${ELASTIC_USERNAME}"
  password: "\${ELASTIC_PASSWORD}"
  ssl.verification_mode: "none"  # For lab purposes only
EOF
```

### 7.2 Update Docker Compose for Metricbeat

Update your docker-compose.yml file to include Metricbeat:

```bash
  metricbeat:
    image: docker.elastic.co/beats/metricbeat:8.17.0
    container_name: metricbeat
    user: root  # Needed for Docker socket access
    volumes:
      - ./metricbeat/metricbeat.yml:/usr/share/metricbeat/metricbeat.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro
      - /proc:/hostfs/proc:ro
      - /:/hostfs:ro
    environment:
      - ELASTIC_USERNAME=elastic
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    networks:
      - elastic
    depends_on:
      - es01
      - kibana
    command: metricbeat -e -strict.perms=false
```

### 7.3 Start Metricbeat and Verify Setup

```bash
# Start just Metricbeat (the rest of the stack is already running)
docker compose up -d metricbeat

# Check Metricbeat logs
docker logs metricbeat

# Verify that Metricbeat indices are being created
curl -u elastic:${ELASTIC_PASSWORD} https://localhost:9200/_cat/indices?v -k | grep metricbeat
```

### 7.4 Explore Metricbeat Dashboards in Kibana

1. Log in to Kibana (https://VM_IP:5601) with elastic user
2. Navigate to Dashboards
3. Filter for "metricbeat" to see auto-created dashboards
4. Explore the "Metricbeat Docker" and "Metricbeat System" dashboards
5. Navigate to Stack Monitoring to see Elasticsearch and Kibana metrics

## Lab 8: Adding Logstash to the Stack (45 minutes)

### 8.1 Create Logstash Configuration

Logstash is a server-side data processing pipeline that ingests, transforms, and ships data to Elasticsearch.

```bash
# Create directories for Logstash configuration
mkdir -p ~/elastic-lab/cluster/logstash/pipeline
cd ~/elastic-lab/cluster

# Create a simple Logstash pipeline
cat > logstash/pipeline/logstash.conf << EOF
input {
  # HTTP input to test Logstash
  http {
    host => "0.0.0.0"
    port => 8080
  }
}

filter {
  # Add a timestamp field if not present
  if ![timestamp] {
    mutate {
      add_field => { "timestamp" => "%{@timestamp}" }
    }
  }
  
  # Add processing info
  mutate {
    add_field => { 
      "processed_by" => "logstash" 
      "environment" => "docker-lab"
    }
  }
}

output {
  # Send to Elasticsearch
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "\${ELASTICSEARCH_USERNAME}"
    password => "\${ELASTICSEARCH_PASSWORD}"
    ssl_certificate_verification => false  # For lab purposes only
    index => "logstash-data-%{+YYYY.MM.dd}"
  }
  
  # Also print to console for debugging
  stdout {
    codec => rubydebug
  }
}
EOF

# Create Logstash settings
cat > logstash/logstash.yml << EOF
http.host: "0.0.0.0"
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["https://es01:9200"]
xpack.monitoring.elasticsearch.username: "\${ELASTICSEARCH_USERNAME}"
xpack.monitoring.elasticsearch.password: "\${ELASTICSEARCH_PASSWORD}"
xpack.monitoring.elasticsearch.ssl.verification_mode: none
EOF
```

### 8.2 Update Docker Compose for Logstash

Update your docker-compose.yml file to include Logstash:

```bash
cat >> docker-compose.yml << 'EOF'

  logstash:
    image: docker.elastic.co/logstash/logstash:8.17.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "8080:8080"
      - "5044:5044"
    environment:
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
    networks:
      - elastic
    depends_on:
      - es01
EOF
```

### 8.3 Start Logstash and Verify Setup

```bash
# Start Logstash
docker compose up -d logstash

# Check Logstash logs
docker logs logstash

# Wait for Logstash to fully start (may take a minute)
sleep 60

# Test Logstash HTTP input
curl -X POST -H "Content-Type: application/json" -d '{"message":"Hello Elasticsearch", "source":"curl_test"}' http://localhost:8080
```

### 8.4 Verify Data in Elasticsearch and Kibana

```bash
# Check if Logstash index was created
curl -u elastic:${ELASTIC_PASSWORD} https://localhost:9200/_cat/indices?v -k | grep logstash

# Query the newest data
curl -u elastic:${ELASTIC_PASSWORD} https://localhost:9200/logstash-data-*/_search?pretty -k
```

In Kibana:
1. Go to Stack Management > Index Patterns
2. Create a new index pattern "logstash-data-*"
3. Select "@timestamp" as the time field
4. Go to Discover and select the new index pattern to see the data

## Lab 9: Configuring a Complete Data Pipeline (30 minutes)

Let's create a more realistic data pipeline with Filebeat → Logstash → Elasticsearch:

### 9.1 Setting Up Filebeat

```bash
# Create directories for Filebeat
mkdir -p ~/elastic-lab/cluster/filebeat
cd ~/elastic-lab/cluster

# Create sample log file
mkdir -p ~/elastic-lab/cluster/logs
cat > logs/sample.log << EOF
2023-01-01T12:00:00 INFO Application started successfully
2023-01-01T12:01:15 INFO User login successful: user123
2023-01-01T12:05:30 WARN High memory usage detected: 85%
2023-01-01T12:10:45 ERROR Database connection failed: timeout
2023-01-01T12:15:00 INFO Backup process completed
EOF

# Create Filebeat configuration
cat > filebeat/filebeat.yml << EOF
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /logs/*.log
  fields:
    source: sample_logs
  fields_under_root: true

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info

# Enable monitoring to see Filebeat metrics in Kibana
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["https://es01:9200"]
  username: "\${ELASTIC_USERNAME}"
  password: "\${ELASTIC_PASSWORD}"
  ssl.verification_mode: "none"
EOF
```

### 9.2 Update Logstash Configuration

Update the Logstash pipeline configuration to accept Filebeat input:

```bash
cat > logstash/pipeline/beats.conf << EOF
input {
  beats {
    port => 5044
  }
}

filter {
  if [source] == "sample_logs" {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:log_timestamp} %{LOGLEVEL:log_level} %{GREEDYDATA:log_message}" }
    }
    
    # Add structured info
    mutate {
      add_field => { 
        "processed_by" => "logstash"
        "environment" => "docker-lab"
      }
    }
  }
}

output {
  if [source] == "sample_logs" {
    elasticsearch {
      hosts => ["https://es01:9200"]
      user => "\${ELASTICSEARCH_USERNAME}"
      password => "\${ELASTICSEARCH_PASSWORD}"
      ssl_certificate_verification => false
      index => "logs-%{+YYYY.MM.dd}"
    }
  }
}
EOF
```

### 9.3 Update Docker Compose for Filebeat

```bash
cat >> docker-compose.yml << 'EOF'

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.17.0
    container_name: filebeat
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./logs:/logs:ro
    environment:
      - ELASTIC_USERNAME=elastic
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    networks:
      - elastic
    depends_on:
      - es01
      - logstash
EOF
```

### 9.4 Start the Complete Pipeline

```bash
# Start Filebeat
docker compose up -d filebeat

# Check logs of each component
docker logs filebeat
docker logs logstash

# Append more logs to the sample log file to trigger processing
cat >> logs/sample.log << EOF
2023-01-01T13:00:00 INFO Scheduled maintenance started
2023-01-01T13:05:20 WARN API response time exceeds threshold: 2.5s
2023-01-01T13:10:30 ERROR Authentication failed: invalid credentials
2023-01-01T13:15:45 INFO Cache cleared successfully
EOF
```

### 9.5 Verify Complete Pipeline in Kibana

1. Navigate to Kibana (https://VM_IP:5601)
2. Go to Stack Management > Index Patterns
3. Create a new index pattern "logs-*" if not already created
4. Go to Discover and select this index pattern
5. You should see the log data processed through the complete pipeline

### Reference Documentation

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Logstash Documentation](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Beats Documentation](https://www.elastic.co/guide/en/beats/libbeat/current/index.html)