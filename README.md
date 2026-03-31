# Grafana-Prometheus-Cadvisor Observability
## *Project Overview*
Observability in a Docker environment acts as a high-resolution lens into a highly dynamic and fragmented system. Since containers are ephemeral and share host resources, observability is essential to identify "noisy neighbors" stealing CPU or memory and to perform "post-mortem" debugging on containers that have already crashed and disappeared. Ultimately, it shifts your strategy from simply knowing a service is down to understanding the root cause of complex bottlenecks across microservices.
## *Problem to be solved*
Implementing an observability stack like <i>cAdvisor + Prometheus + Grafana</i> solves several critical "blind spots" that occur when running containerized applications.
1. <b>The "Vanishing Data" Problem</b><br>
Containers are ephemeral; they are designed to be stopped, deleted, and replaced instantly. When a Docker container crashes, its internal logs and performance data usually disappear with it.<br>
<b>The Solution</b>: This stack pulls data out of the container in real-time and stores it externally (in Prometheus), ensuring you have a "black box" recording to analyze even after the container is gone.
2. <b>The "Resource Blindness" Problem</b><br>
Standard Linux monitoring tools (like top or htop) often show the host's overall health but struggle to attribute specific resource spikes to individual containers accurately.<br>
<b>The Solution</b>: cAdvisor digs into the kernel's cgroups to provide granular, per-container metrics. It solves the "Noisy Neighbor" mystery by showing exactly which container is stealing CPU cycles or leaking memory.
3. <b>The "Manual Correlation" Problem</b><br>
Without a centralized system, a DevOps engineer would have to manually log into different servers, run Docker commands, and try to mental-map how a CPU spike at 2:00 PM relates to a slow database query.<br>
<b>The Sample Solution</b>:
	- <i>Prometheus</i> automates the data collection (no manual SSH-ing).<br>
	- <i>Grafana</i> overlays different metrics on one timeline, solving the problem of correlation. You can visually see that a spike in "Web Traffic" directly caused a spike in "Database Latency."
## *Business Leverage*
<b>Business Impact</b>
| Business Metrics | Without Observability | With Observabilty |
| :--- | :--- | :--- |
|Operational Risk|High (Guesswork during outages)|Low (Data-driven resolution)|
|Infrastructure Spend|High (Wasted resources)|Optimized (Pay for what you use)|
|Engineering Focus|"Firefighting" constant bugs|Building new revenue-generating features|
|Customer Trust|Damaged by slow/broken services|High due to consistent performance|

## *Project Prerequisites*
1. Docker installed on your system (<i>cachyos</i>)
	```bash
	sudo pacman -S docker docker-compose docker-buildx
	
	docker --version
	
	Docker version 29.3.1, build c2be9ccfc3
	```
2. Text editor installed (<i>neovim</i>)
	```bash
	sudo pacman -S neovim
	
	nvim --version
	
	NVIM v0.11.7
	Build type: RelWithDebInfo
	LuaJIT 2.1.1774638290
	Run "nvim -V1 -v" for more info
	```
3. Supported Web Browser (<i>Brave</i>)
	```bash
	curl -fsS https://dl.brave.com/install.sh | sh
	```
## Project Workflow
1. Build <i>docker-compose.yml</i><br>
The docker-compose.yml file serves as a single blueprint to manage your entire observability stack as one unit.
	```neovim
	services:
  		cadvisor:
   		image: gcr.io/cadvisor/cadvisor:latest
    	...
  		prometheus:
    	image: prom/prometheus:latest
    	...
  		grafana:
    	image: grafana/grafana:latest
    	...
	volumes:
  		prometheus_data:
  		grafana_data:
	```
2. Build <i>prometheus.yml</i><br>
The prometheus.yml file is the central configuration heart of the Prometheus server. While the binary runs the database, this YAML file tells Prometheus what to monitor, how often to collect data, and where to send alerts.
	```neovim
	global:
  	scrape_interval: 15s # Default frequency to pull data

	scrape_configs:
  	- job_name: 'docker'
    	static_configs:
      	- targets: ['cadvisor:8080'] # This points to the cAdvisor service
      ...
	```
3. Build <i>cadvisor-prometheus-grafana</i> docker containers
	```bash
	docker compose up -d
	
	# after wait for a moment
	> docker compose ps
	NAME                            IMAGE                             COMMAND                  SERVICE         CREATED        STATUS                 PORTS
	observability-cadvisor-1        gcr.io/cadvisor/cadvisor:latest   "/usr/bin/cadvisor -…"   cadvisor        23 hours ago   Up 2 hours (healthy)   0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
	observability-grafana-1         grafana/grafana                   "/run.sh"                grafana         23 hours ago   Up 2 hours             0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
	observability-node_exporter-1   prom/node-exporter:latest         "/bin/node_exporter"     node_exporter   23 hours ago   Up 2 hours             0.0.0.0:9100->9100/tcp, [::]:9100->9100/tcp
	observability-prometheus-1      prom/prometheus                   "/bin/prometheus --c…"   prometheus      23 hours ago   Up 2 hours             0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp
	```
4. Access docker container through web browser<br>
	| docker container | web access | User/Password | 
	| :--- | :--- | :--- |
	| cAdvisor|http://localhost:8080|None| 
	| prometheus|http://localhost:9090|None|
	| grafana|http://localhost:3000|admin/admin|
5. Grafana Dashboard

	| Metric | PromQL | Chart | Description |
	| :--- | :--- | :--- | :---|
	| container|count(container_last_seen{id=~"/system.slice/docker-.*.scope"})|Stat|Count docker container actually in our system|
	| memory|sum(container_memory_usage_bytes{id=~"/system.slice/docker-.*.scope"})|Stat|Measure memory usage|
	| cpu|sum(rate(container_cpu_usage_seconds_total[5m])) by (image)|Stat|Measure total CPU usage|
	| network receive|rate(container_network_receive_bytes_total[5m])|Gauge|Monitor received data|
	| network transmit|rate(container_network_transmit_bytes_total[5m])|Gauge|Monitor transmited data|
	| memory|container_memory_usage_bytes{id=~"/system.slice/docker-.*.scope"}|Time Series|Monitor memory graphically|
	| cpu|rate(container_cpu_usage_seconds_total{id=~"/system.slice/docker-.*.scope"}[5m])|Time Series|Monitor CPU graphically|
	| network receive|rate(container_network_receive_bytes_total[5m])|Time Series|Monitor received data graphically|
	| network transmit| rate(container_network_transmit_bytes_total[5m])|Time Series|Monitor transmited data graphically|