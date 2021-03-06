# Installing Elastic 6.x stack (ELK) on Centos 7
NOTE: You need at least 2 CPUSs and 2G RAM

## Prepare OS
```
sudo yum install -y epel-release wget
sudo yum -y update
```

## Install Java (OpenJDK)
```
 sudo yum install -y java-openjdk
```

## Install Elastic YUM repositories  
```
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

cat << EOF > /etc/yum.repos.d/elasticsearch.repo
[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
 
cat << EOF > /etc/yum.repos.d/logstash.repo
[logstash-6.x]
name=Elastic repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
 
cat << EOF > /etc/yum.repos.d/kibana.repo
[kibana-6.x]
name=Kibana repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

## Install Elasticsearch
``` 
sudo yum -y install elasticsearch
``` 

## Install Logstash 
``` 
sudo yum install -y logstash
``` 
## Install Kibana 
``` 
sudo yum install -y kibana
```
 
## Enable and start services for ELK stack 
``` 
sudo systemctl start elasticsearch.service
sudo systemctl enable elasticsearch.service
sudo systemctl start logstash.service
sudo systemctl enable logstash.service
sudo systemctl start kibana.service
sudo systemctl enable kibana.service
``` 
# Configuration

## Elasticsearch
```
sudo sed -i 's|#network.host: 192.168.0.1|network.host: 0.0.0.0|g' /etc/elasticsearch/elasticsearch.yml
```

## Kibana
```
sudo sed -i 's|#server.host: "localhost"|server.host: "0.0.0.0"|g' /etc/kibana/kibana.yml
```

## Restart services for ELK stack 
``` 
sudo systemctl restart elasticsearch.service
sudo systemctl restart logstash.service
sudo systemctl restart kibana.service
``` 

# Nginx Reverse Proxy with basic authentication
```
sudo yum install -y nginx httpd-tools
```
## Configuration for reverse proxy
```
cat <<EOF > /etc/nginx/conf.d/proxy.conf
proxy_redirect off;
proxy_set_header Host \$host;
proxy_set_header X-Real-IP \$remote_addr;
proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
client_max_body_size 10m;
client_body_buffer_size 128k;
proxy_connect_timeout 90;
proxy_send_timeout 90;
proxy_read_timeout 90;
proxy_buffers 32 4k;
EOF
```

## Virtual host configuration for Kibana with basic authentication
```
cat <<EOF > /etc/nginx/conf.d/kibana.conf
server {
    listen 80;
	
    server_name elk.example.com; # Change this to match your env

    location / {
        auth_basic "Restricted"; #For Basic Auth
        auth_basic_user_file /etc/nginx/.htpasswd; #For Basic Auth
        include conf.d/proxy.conf;
        proxy_pass http://localhost:5601;
    }
}
EOF

systemctl enable nginx
systemctl start nginx
```


Log in with kibanaadmin on http://elk.example.com
### Create admin user
```
sudo htpasswd -c /etc/nginx/.htpasswd kibanaadmin
```

Log in with kibanaadmin on http://elk.example.com

# Firewall settings
```
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --add-port=9200/tcp --permanent
sudo firewall-cmd --add-port=5000/tcp --permanent
sudo firewall-cmd --add-port=5044/tcp --permanent
sudo firewall-cmd --add-port=5601/tcp --permanent
sudo firewall-cmd --reload
```

# Housekeeping ELK

## Metricbeat
Manually insert indexes for Metricbeat into EL
metricbeat setup --template -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["localhost:9200"]'

## Hard drive FULL - 
If the disk space fills up (or gets higher than max value) Kibana might throw this at you
FORBIDDEN/12/index read-only / allow delete (api)
Elasticsearch is switching to read-only if it cannot index more documents because your hard drive is full. 
With this it ensures availability for read-only queries. Elasticsearch will not switch back automatically
To fix this run the following on your ELK stack
```
curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'
```

