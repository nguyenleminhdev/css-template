# ------------------------ Install Elasticsearch -------------------------------
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg && \
sudo apt-get install apt-transport-https && \
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list && \
sudo apt-get update && \
sudo apt-get install elasticsearch -y && \
/usr/share/elasticsearch/bin/elasticsearch-plugin install discovery-ec2

# ------------------------- Edit config file -----------------------------------
vim /etc/elasticsearch/elasticsearch.yml

## --- only data
node.name: data-
node.roles: ["data"]

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: [_local_,_site_]
cluster.name: bbh-chatbox-ces1
discovery.ec2.tag.cluster_name: bbh-chatbox-ces1
discovery.seed_providers: ec2
discovery.ec2.endpoint: ec2.ap-southeast-1.amazonaws.com
cluster.routing.allocation.awareness.attributes: aws_availability_zone
logger.org.elasticsearch.discovery.ec2: "TRACE"
cloud.node.auto_attributes: true
xpack.security.enabled: false
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.transport.ssl.truststore.path: certs/transport.p12
## ---

## ---
node.name: master-1
node.roles: ["master"]

path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
cluster.initial_master_nodes: ["172.31.14.124"]
network.host: [_local_,_site_]
cluster.name: bbh-chatbox-ces1
discovery.ec2.tag.cluster_name: bbh-chatbox-ces1
discovery.seed_providers: ec2
discovery.ec2.endpoint: ec2.ap-southeast-1.amazonaws.com
cluster.routing.allocation.awareness.attributes: aws_availability_zone
logger.org.elasticsearch.discovery.ec2: "TRACE"
cloud.node.auto_attributes: true
xpack.security.enabled: false
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.transport.ssl.truststore.path: certs/transport.p12
## ---

# --------------------------- Authen AWS api -----------------------------------
/usr/share/elasticsearch/bin/elasticsearch-keystore add discovery.ec2.access_key
AKIARFCON2JNDZSJCXXA

/usr/share/elasticsearch/bin/elasticsearch-keystore add discovery.ec2.secret_key
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

# ----------------------- start elasticsearch ----------------------------------
sudo systemctl daemon-reload && \
sudo systemctl enable elasticsearch.service && \
sudo systemctl start elasticsearch.service

# ----------------------------- test command -----------------------------------
# ping
curl -XGET http://localhost:9200

# cluster info
curl -XGET http://localhost:9200/_cluster/health?pretty=true

# cluster node info
curl -XGET http://localhost:9200/_cluster/state/nodes?pretty=true

# get master node
curl -XGET http://localhost:9200/_cluster/state/master_node?pretty=true

# get all index
curl -XGET http://localhost:9200/_cat/indices

# get all shard
curl -XGET http://localhost:9200/_cat/shards

# ---------------------- Install node exporter ---------------------------------
wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz && \
tar -xf node_exporter-1.4.0.linux-amd64.tar.gz && \
sudo mv node_exporter-1.4.0.linux-amd64/node_exporter /usr/local/bin && \
rm -r node_exporter-1.4.0.linux-amd64* && \
sudo useradd -rs /bin/false node_exporter && \
sudo nano /etc/systemd/system/node_exporter.service

# ---
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
# ---

sudo systemctl daemon-reload && \
sudo systemctl enable node_exporter && \
sudo systemctl start node_exporter && \
sudo systemctl status node_exporter

# ---------------------- Uninstall Elasticsearch -------------------------------
sudo apt-get --purge autoremove elasticsearch

# ----------------------- reload elasticsearch ---------------------------------
sudo systemctl restart elasticsearch.service

# ---------------- Delete all data to make to is only master -------------------
/usr/share/elasticsearch/bin/elasticsearch-node repurpose

# ----------------------- stop elasticsearch -----------------------------------
sudo systemctl stop elasticsearch.service

# ----------------------------- Install kibana ---------------------------------
sudo apt-get update && sudo apt-get install kibana && \
sudo systemctl start kibana.service && \
sudo systemctl status kibana.service

# ----------------------------- Uninstall Kibana -------------------------------
sudo apt-get --purge autoremove kibana

# ------------------------ Install Nginx ---------------------------------------
sudo apt update -y && \
sudo apt install nginx -y && \
sudo apt install certbot python3-certbot-apache && \
systemctl status nginx