---
title: Monitoring Docker with Grafana & Prometheus
date: 2024-08-01 15:16:35
tags:
- Kubernetes
- Grafana
- Prometheus
top: 99
---
*Using Grafana and Prometheus to monitoring our Docker service*
[reference1](https://medium.com/@ravipatel.it/building-a-monitoring-stack-with-prometheus-grafana-and-alerting-a-docker-compose-ef78127e4a19)
[reference2](https://github.com/880831ian/Prometheus-Grafana-Docker?tab=readme-ov-file)
For more code please visit my article repo:[Github](https://github.com/miyakiyu/DevOps-Exercise/tree/main/Monitoring)

<!--more-->
This is my file tree:
```bash
.
├── alertmanager
│   └── alertmanager.yml
├── docker-compose.yml
├── nginx
│   ├── Dockerfile
│   └── status.conf
└── prometheus
    ├── alerts.rules.yml
    └── prometheus.yml
```
Let's srart!

---
## Config setting file
### docker-compose
Let's create a docker-compose file include the service we need:
```bash
services:
  nginx:
    build: ./nginx/
    container_name: nginx
    ports:
      - 8080:8080
    restart: always
    
  nginx-prometheus-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx-prometheus-exporter
    command: -nginx.scrape-uri http://nginx:8080/stub_status
    ports:
      - 9113:9113
    depends_on:
      - nginx 

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports: 
      - 9090:9090
    volumes:
      - ./prometheus:/etc/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--web.enable-lifecycle'
    restart: always

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - 3000:3000
    depends_on:
      - prometheus
    restart: always

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    ports:
      - 9093:9093
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    restart: always
```
### Setting Nginx
Here is the Dockerfile, because we want to use this version and set our conf into container so:
```bash
FROM nginx:1.21.6 
COPY ./status.conf /etc/nginx/conf.d/status.conf 
```
Nginx in this version will start stub_status automatically.
And let's setting stub_status:
```bash
server {
    listen 8080;
    server_name  localhost;
    location /stub_status {
       stub_status on;
       access_log off;
    }
}
```
Start this setting.

### Setting Prometheus
Here is the setting for Prometheus:
```bash
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: "nginx"
    static_configs:
      - targets: ["localhost:8080"]
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "nginx_exporter"
    static_configs:
      - targets: ["nginx-prometheus-exporter:9113"]
alerting:
  alertmanagers:
    - scheme: http
    - static_configs:
        - targets: ["alertmanager:9093"]
```
Prometheus will scrape targets every 10 seconds. The scrape_configs section defines all the targets that need to be scraped.
And here is the Prometheus interface:
![Interface](/pic/monitor/prom.png)
If you see the green up that's mean your service yopu want to monitoring are up!

### Setting Grafana
#### Import Prometheus
Now we can visit Grafana with `localhost:3000/`.
Login with admin, and go to search bar and search `Data Sources`, and add one, we choose Prometheus.
After that set URL with below photo and click `Save&Test`
![config](/pic/monitor/config.png)

#### Setting Dashboard
As you done, now we add a new Dashboard with below steps:
Click Add visualize.
![step 1](/pic/monitor/1.png)
Choose Prometheus.
![step 2](/pic/monitor/2.png)
Config it as you like and don't forget to click Save!
![step 3](/pic/monitor/3.png)
And...Done! We got dashboard now.
![step 4](/pic/monitor/4.png)

### Config Granfana dashboard
Let's setting dashboard, import `ID:12708` dashboard made by other person, here is the result we got Nginx connection situation from Prometheus:
![dashboard](/pic/monitor/up.png)
When we visit `localhost:80` it will request to niginx and you could see the right side, the line is increase.
When we stop Nginx service:
![down](/pic/monitor/down.png)

---
## Config Alerting in Prometheus
We need to use PromQL for configure the alert rule and alertmanager.

### alert.yml
let's create `alert.yml`:
```bash
groups:
  - name: test
    rules:
      # Alert for any instance that is unreachable for >5 minutes.
      - alert: InstanceDown
        expr: up == 0
        for: 10s
        labels:
          severity: page
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."
```
This will let Alertmanager follow this rule to send the notification, and next let's setting it.

### config.yml
Let's setting Alertmanager:
```bash
route:
  receiver: "gmail-notifications"
receivers:
  - name: "gmail-notifications"
    email_configs:
      - to: (YOUR_EMAIL)
        from: (YOUR_EMAIL)
        smarthost: smtp.gmail.com:587
        auth_username: (YOUR_EMAIL)
        auth_identity: (YOUR_EMAIL)
        auth_password: ijox jecz tdit cipc
        send_resolved: true
```


### Add config to docker-compose
```bash
...
  prometheus:
    ...
      command:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--web.enable-lifecycle'
...
  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    ports:
      - 9093:9093
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
    restart: always
...
```
After set this, restart all the service by using `docker compose restart YOUR_CONTAINER_NAMES`, and wait a minute you will get the email:
![mail](/pic/monitor/mail.png)

---
## Conclusion
After doing that, I learn how to set up Prometheus and Grafana by using docker, next I hope I could doing this on Kuberbetes :/





