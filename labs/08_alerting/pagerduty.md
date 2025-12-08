# 📊 Lab: End-to-End Monitoring with Prometheus, Grafana, and PagerDuty

Welcome! In this lab, you’ll learn how to monitor a system using:
- **Prometheus** for collecting metrics
- **Grafana** for creating dashboards
- **PagerDuty** for sending alerts

![Pagerdury flow](https://files.readme.io/1a278bb-quick_start-guide-flow-chart-22x.png)

Based on the GitHub project:  
🔗 https://github.com/ibraheemcisse/End-to-End-Monitoring-Setup-Prometheus-Grafana-and-PagerDuty

---

## 🖥️ Step 0: Prepare Your Virtual Machine (VM)

Before starting the lab, create a Linux VM with the following specs:

- **OS**: Ubuntu 22.04 LTS (preferred) or Debian-based
- **CPU**: 2 cores minimum
- **RAM**: 2 GB (4 GB recommended)
- **Disk**: 10+ GB
- **Network**: Must have internet access
- **Pre-installed tools**: `wget`, `tar`, `curl`, `nano`, `systemctl`

### Recommended Platforms:
- Local: VirtualBox, VMware Workstation Player
- Cloud: AWS EC2 (t2.micro), Google Cloud (e2-micro), Azure (B1s)

> 🔐 If using cloud, open ports `3000` and `9090` to access Grafana and Prometheus.

---

## ⚠️ Network & Access Tips

- Local VM: Use `http://localhost:9090` and `http://localhost:3000`
- Cloud VM: Use `http://<your-vm-ip>:9090` and `http://<your-vm-ip>:3000`
- Open these ports in your cloud firewall settings.
- ✅ PagerDuty works without needing public URLs — alerts are sent *outbound*.

---

## 📁 Step 1: Clone the Project

```bash
git clone https://github.com/ibraheemcisse/End-to-End-Monitoring-Setup-Prometheus-Grafana-and-PagerDuty.git
cd End-to-End-Monitoring-Setup-Prometheus-Grafana-and-PagerDuty
```

---

## 📦 Step 2: Install Prometheus

```bash
wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-*.linux-amd64.tar.gz
tar -xvzf prometheus-*.linux-amd64.tar.gz
cd prometheus-*.linux-amd64/
```

Edit `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Run Prometheus:

```bash
./prometheus --config.file=prometheus.yml
```

Visit [http://localhost:9090](http://localhost:9090)

---

## 📊 Step 3: Install Grafana

```bash
wget https://dl.grafana.com/oss/release/grafana-10.0.0.linux-amd64.tar.gz
tar -zxvf grafana-10.0.0.linux-amd64.tar.gz
cd grafana-10.0.0/
./bin/grafana-server
```

Visit [http://localhost:3000](http://localhost:3000) — login: `admin`/`admin`

---

## 🔌 Step 4: Connect Grafana to Prometheus

1. Settings > Data Sources > Add Data Source
2. Choose Prometheus
3. URL: `http://localhost:9090` (or cloud IP)
4. Click Save & Test

---

## 🧭 Step 5: Create Dashboards

Create → Dashboard → Add panel

**CPU Query:**
```prometheus
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Memory Query:**
```prometheus
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Disk Query:**
```prometheus
100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)
```

---

## 🚨 Step 6: Set Up PagerDuty

### What is PagerDuty?

PagerDuty is a **real-time incident response platform** used by IT teams to detect, escalate, and resolve issues rapidly.  
Founded in 2009, it has become a standard in **DevOps**, **SRE**, and **IT Operations**.  
Trusted by thousands of companies, it integrates with tools like Grafana, Slack, AWS, and more.

### Create a PagerDuty Service

1. Sign up at [https://www.pagerduty.com](https://www.pagerduty.com)
2. Go to **Services > + New Service**
3. Integration Type: **Events API v2**
4. Copy the Integration Key

---

## 📢 Step 7: Connect PagerDuty to Grafana

1. Grafana > Alerting > Notification Channels > Add Channel
2. Type: PagerDuty
3. Paste the Integration Key
4. Name: `PagerDuty Alerts`
5. Save

---

## ⚙️ Step 8: Create Alert Rule

Grafana > Alerting > Alert Rules > + Create

Query:

```prometheus
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Condition: last value > 90 for 1 minute → Notify: PagerDuty Alerts

---

## 🧪 Step 9: Test Alerts

1. Use "Test Rule" in Grafana
2. Simulate high CPU or edit value temporarily
3. Check PagerDuty dashboard for alert

---

## 🧪 Step 10: Test the Alert Delivery (Get Email/SMS/Call)

### ✅ Before You Start

Make sure:
- You created a PagerDuty **user account**
- You added your **email and phone number** under **User Settings**
- Your **notification rules** are set (e.g., "SMS immediately", "Call me after 1 min")

### 🧪 Option A: Simulate CPU Spike (Real)

Run this in your VM:

```bash
yes > /dev/null &
```

This will use 100% CPU and trigger the alert.  
To stop it, run:

```bash
killall yes
```

### 🧪 Option B: Use Grafana Test Button (Quick)

1. Go to **Alerting > Alert Rules**
2. Click the rule (e.g., "High CPU")
3. Click **Test Rule**
4. You should receive an alert via email, SMS, or call

### 📱 Optional: Use the PagerDuty Mobile App

- Install from your app store
- Log in and allow push notifications

---

## 🧹 Step 11: Clean Up (Prevent Billing!)

PagerDuty trial accounts may convert to paid if not canceled.  
To avoid charges:

1. Go to your PagerDuty account settings
2. **Cancel/delete the account** before your trial ends
3. Or downgrade to the free tier (if available)

> ⚠️ Failure to cancel may result in **unexpected billing** after the trial period.

---

## 🎉 Lab Complete

✅ Metrics collected with Prometheus  
✅ Dashboards created with Grafana  
✅ Alerts delivered using PagerDuty

---

## 📚 Resources

- [Prometheus Docs](https://prometheus.io/docs/introduction/overview/)
- [Grafana Docs](https://grafana.com/docs/)
- [PagerDuty Docs](https://support.pagerduty.com/)

Happy Monitoring! 🚀
