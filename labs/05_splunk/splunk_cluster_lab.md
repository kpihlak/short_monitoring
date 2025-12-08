# **Lab Manual: Setting Up a Splunk Cluster on a Single Docker Node**

## **Introduction**
This lab demonstrates how to configure a Splunk cluster on a single Docker node. The setup includes the following components:
- **Master Node**: Manages the cluster configuration.
- **Peer Node**: Handles data indexing and replication.
- **Search Head**: Provides the search interface for querying indexed data.
- **Application Container**: Generates logs to be sent to Splunk for analysis.

---

## **Prerequisites**
1. **Software Requirements**:
   - Install Docker: [Docker Installation Guide](https://docs.docker.com/get-docker/).
   - Install Docker Compose: [Docker Compose Installation Guide](https://docs.docker.com/compose/install/).

2. **Hardware Requirements**:
   - Minimum 8 GB of RAM.
   - At least 2 CPUs.
   - Sufficient disk space for Splunk data.

---

## **Step 1: Create the Docker Compose File**

1. **Set up the working directory**:
   ```bash
   mkdir splunk-cluster && cd splunk-cluster
   ```

2. **Create the `docker-compose.yml` file**:
   ```bash
   nano docker-compose.yml
   ```

3. **Add the following content**:
   ```yaml
   version: '3.8'

   services:
     master:
       image: splunk/splunk
       container_name: splunk_master
       ports:
         - "8000:8000"   # Web interface for Master Node
         - "8089:8089"   # Management port
         - "9997:9997"   # Data forwarding port
       environment:
         - SPLUNK_START_ARGS=--accept-license
         - SPLUNK_PASSWORD=YourSecurePassword
       networks:
         - splunk-net

     peer:
       image: splunk/splunk
       container_name: splunk_peer
       ports:
         - "8001:8000"   # Web interface for Peer Node
         - "8088:8088"   # HTTP Event Collector port
         - "9998:9997"   # Replication port for Peer Node
       environment:
         - SPLUNK_START_ARGS=--accept-license
         - SPLUNK_PASSWORD=YourSecurePassword
       networks:
         - splunk-net

     search-head:
       image: splunk/splunk
       container_name: splunk_search_head
       ports:
         - "8002:8000"   # Web interface for Search Head
       environment:
         - SPLUNK_START_ARGS=--accept-license
         - SPLUNK_PASSWORD=YourSecurePassword
       networks:
         - splunk-net

   networks:
     splunk-net:
       driver: bridge
   ```

4. Save and close the file.

---

## **Step 2: Deploy the Containers**

1. **Start all the containers**:
   ```bash
   docker-compose up -d
   ```

2. **Verify running containers**:
   ```bash
   docker ps
   ```
   You should see `splunk_master`, `splunk_peer`, and `splunk_search_head` running.

---

## **Step 3: Configure Splunk Components**

### **Master Node Configuration**
1. Open the Master Node interface: `http://<your-node-ip>:8000`.
2. Log in with:
   - **Username**: `admin`
   - **Password**: `YourSecurePassword`.
3. Navigate to `Settings > Indexer Clustering`.
4. Click **Enable Clustering** and choose the role as **Master Node**.
5. Configure:
   - **Replication Factor**: 1 (for simplicity).
   - **Search Factor**: 1 (for simplicity).
6. Save the settings and restart the container when prompted.

---

### **Peer Node Configuration**
1. Open the Peer Node interface: `http://<your-node-ip>:8001`.
2. Log in with the same credentials as the Master Node.
3. Navigate to `Settings > Indexer Clustering`.
4. Click **Enable Clustering** and choose the role as **Peer Node**.
5. Enter the **Master URI**: `https://splunk_master:8089`.
6. Configure:
   - **Replication Port**: `9997`.
   - Use the same security key as the Master Node.
7. Save the settings and restart the container.

---

### **Search Head Configuration**
1. Open the Search Head interface: `http://<your-node-ip>:8002`.
2. Log in with the same credentials as the Master Node.
3. Navigate to `Settings > Indexer Clustering`.
4. Click **Enable Clustering** and choose the role as **Search Head**.
5. Enter the **Master URI**: `https://splunk_master:8089`.
6. Save the settings and restart the container.

---

## **Step 4: Configure an Index**

1. Open the Peer Node interface: `http://<your-node-ip>:8001`.
2. Navigate to `Settings > Indexes`.
3. Click **New Index** and configure:
   - **Index Name**: `tutorial_index`.
   - Use default settings for simplicity.
4. Save the new index.

---

## **Step 5: Set Up the HTTP Event Collector (HEC)**

The HTTP Event Collector (HEC) allows applications and services to send data directly to Splunk.

1. **Access the Peer Node**:
   Open `http://<your-node-ip>:8001` in a browser and log in.

2. **Navigate to HEC Settings**:
   - Go to `Settings > Data Inputs > HTTP Event Collector`.
   - Click **Add New** to create a new HEC.

3. **Create the HEC**:
   - **Name**: Enter a name like `docker_logs_collector`.
   - **Index**: Select `tutorial_index` (created in Step 4).
   - Click **Next** and then **Submit**.

4. **Retrieve the Token**:
   - After submission, a page with the generated token will appear.
   - Copy the **token value**; this is your `Your-HEC-Token`.

5. **Enable the HEC**:
   - Go back to `Settings > Data Inputs > HTTP Event Collector`.
   - Click **Global Settings**.
   - Enable the HEC and choose `All Tokens` under "Allowed Indexes."
   - Save the changes.

---

## **Step 6: Send Logs from an Application Container**

Use the Nginx container to generate logs and send them to Splunk via the HEC.

1. **Run the Nginx Container**:
   Use the following command to start the Nginx container, replacing `<Your-HEC-Token>` with the token from Step 5 and `<peer-node-ip>` with the IP of the Peer Node:
   ```bash
   docker run -p 80:80 \
   --log-driver=splunk \
   --log-opt splunk-token=<Your-HEC-Token> \
   --log-opt splunk-url=https://<peer-node-ip>:8088 \
   --log-opt splunk-insecureskipverify=true \
   nginx
   ```

   **Explanation of the command**:
   - The `--log-driver=splunk` sets Splunk as the logging driver.
   - `--log-opt splunk-token=<Your-HEC-Token>` uses the token for authentication with Splunk.
   - `--log-opt splunk-url=https://<peer-node-ip>:8088` specifies the Peer Node as the log collector.
   - `--log-opt splunk-insecureskipverify=true` skips SSL certificate verification for simplicity.

2. **Access the Nginx Application**:
   - Open a browser and go to `http://<your-node-ip>:80`.
   - Perform actions (e.g., refresh the page) to generate logs.

3. **Verify Logs in Splunk**:
   - Open the Search Head at `http://<your-node-ip>:8002`.
   - Navigate to `Search & Reporting`.
   - Query for logs:
     ```text
     index="tutorial_index"
     ```
   - Check for Nginx logs to confirm the setup.

---

## **Ports Used**

| Port  | Description                                   |
|-------|-----------------------------------------------|
| 8000  | Web interface for Splunk components          |
| 8089  | Management port for internal communication    |
| 9997  | Data forwarding from universal forwarders     |
| 8088  | HTTP Event Collector for receiving logs       |
| 80    | Default web server port for testing (e.g., Nginx)|

---

## **Conclusion**

You have successfully set up a Splunk cluster on a single Docker node. Use this setup to explore Splunk's capabilities for log analysis and monitoring. You can expand this configuration by adding more nodes or integrating additional data sources.

Happy Splunking!

