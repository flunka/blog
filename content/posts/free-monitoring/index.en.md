---
title: "Free monitoring with Graphana Labs"
date: 2021-05-03T09:37:41+02:00
draft: false
author: "Flunka.en"
tags: ["Monitoring", "Ansible", "DevOps"]
categories: ["DevOps"]
theme: wide
toc:
  enable: true
summaryStyle:
  hiddenImage: false
  hiddenDescription: false
  hiddenTitle: true
  tags:
    theme: "image"
    color: "white"
    background: "black"
    transparency: 0.8
resources:
  - name: featured-image
    src: grafana-labs.png
---

## Free Grafana Cloud Plan

Typically, small IT projects don't have enough resources to monitor their assets.
Fortunately, at the beginning of this year, Grafana Labs introduced a free plan for its cloud solution.
This plan includes:

- 10,000 series for Prometheus or Graphite metrics
- 50 GB of logs
- 14 day retention for metrics and logs
- Access for up to 3 team members

For me it is good enough so I decided to try it out to monitor my VPS server (Centos 7).
First of all you need to create free accout at [Grafana](https://grafana.com/auth/sign-up/create-user?pg=prod-cloud-pricing&plcmt=free).
After that you are ready to set up your tool to collect logs, metrics and send alerts.

## Logs

### Promtail

We will be using Promtail to send logs form VPS to Loki logging service.
According to theier documetnation the simplest way to do it is to run Promtail with docker. I did't have docker istalled so I've created Ansible playbook to install docker on my VPS.

```yaml
---
- name: Add repository
  yum_repository:
    name: docker-ce
    description: Docker repo
    baseurl: https://download.docker.com/linux/centos/7/$basearch/stable
    gpgkey: https://download.docker.com/linux/centos/gpg
    state: present
  become: yes
- name: Install a packages for docker
  yum:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present
  become: yes
- name: Ensure docker is running and enabled
  systemd:
    name: docker
    state: started
    enabled: yes
  become: yes
```

First task adds docker repo. Docker documetation was missleading. Accoring to theri documentation you need to add repo `https://download.docker.com/linux/centos/docker-ce.repo`, but after that I've got following error:

`https://download.docker.com/linux/centos/docker-ce.repo/repodata/repomd.xml: [Errno 14] HTTPS Error 404 - Not Found`

After quick research, I've found the solution on [docker forum] (https://forums.docker.com/t/docker-ce-stable-x86-64-repo-not-available-https-error-404-not-found-https-download-docker-com-linux-centos-7server-x86-64-stable-repodata-repomd-xml/98965/6). All I had to do was change the base url and I was done.
The second task installs the required packages, and the third task starts and enables the docker at system startup.
Docker is up and running, but before we run the container with Promtail let's create basic configuration for Promtail in `/etc/promtail/config.yaml`. In order to do it we neet to generate API key to authenticate to Grafana Clound. Your key should have role **MetricPusher**.
{{< admonition type=danger open=true >}}
Do NOT share your API key!
{{< /admonition >}}

```yaml
server:
  http_listen_port: 0
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

client:
  url: https://39544:<Your Grafana.com API Key>@logs-prod-us-central1.grafana.net/api/prom/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

The above configuration will monitor all files in the `/var/log/` directory that end with `.log`.
To launch docker container with Promtail I've created following ansible playbook.

```yaml
---
- name: Create a directory for Promtail config
  file:
    path: /etc/promtail
    state: directory
    group: "{{ansible_user}}"
    mode: "0775"
  become: yes
- name: Copy Promtail config file
  copy:
    src: config.yaml
    dest: /etc/promtail/config.yaml
    mode: "0644"

- name: Ensure docker-py is installed
  pip:
    name: docker-py
    state: present
  become: yes

- name: Container present
  docker_container:
    name: promtail
    state: started
    container_default_behavior: compatibility
    image: grafana/promtail:master
    volumes:
      - /etc/promtail:/etc/promtail
      - /var/log:/var/log
    command: -config.file=/etc/promtail/config.yaml
```

At the begining of this playbook we create directory for Promtail configuration. Next we copy config to that directory. To run `docker_container` module on server we need Python package `docker` or `docker-py`. I had some problems with `docker` package so I installed `docker-py`. Last thing is to run the container. We use the `grafana/promtail` image. Importan note is to use volumes with Promtail config (/etc/promtail) and log directory (/var/log). To run the container we need to sepecify path to configuration file.

### Loki

After successful container lauching our logs should be send to Loki service. We should be able to view our logs on Grafana. All you need to do is:

1. Log into Grafana
2. Go to Explore
3. Select data source grafanacloud-xxx-logs, where xxx is your name (this data source should be configured already)
4. Specify some query to get logs eg. to get logs from file `/var/log/yum.log` you need to type `{filename="/var/log/yum.log"}`.

More info about creating query you can find ing [Grafana documentation](https://grafana.com/docs/grafana/latest/datasources/loki/#querying-logs)

{{< figure src="loki-logs.png" title="Example log view" >}}

## Metrics

### Node exporter

To publish metrcs from our server we need to install node exporter. In order to do so, we cane use ansible role `cloudalchemy.node_exporter`, but first you need to download that role using the command `ansible-galaxy install cloudalchemy.node_exporter`. Node exporter will listen on port 9100 by default.

### Prometheus

Next we can run local instance of Prometheus which will send collected metrics to Grafana Clound.
The configuration of Prometheus is as follows:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ["172.17.0.1:9100"]
    relabel_configs:
      - source_labels: [__address__]
        regex: ".*"
        target_label: instance
        replacement: "vps790031.ovh.net"

remote_write:
  - url: https://prometheus-blocks-prod-us-central1.grafana.net/api/prom/push
    basic_auth:
      username: <USERNAME>
      password: <PASSWORD>
```

`scrape_interval` defines how frequent metrics will be collected. In section `scrape_configs` we specify source of metrics. Since we run Prometheus in a container, the metrics are retrieved through the internal docker interface. In order for our monitoring to know where the metrics are coming from, it's a good idea to change the labels for this host, because otherwise we will see that the metrics are coming from `172.17.0.1` and not from our server address. That's why in the `relable_conifgs` section we specify the name under which our metrics should be visible. In the last section we configure where metrics should be sent.

Configuration for Prometheus is ready so we need to create asible role which will be push config to the server and runs Prometheus inside container.

```yaml
---
- name: Create a directory for Prometheus config
  file:
    path: /etc/prometheus
    state: directory
    group: "{{ansible_user}}"
    mode: "0775"
  become: yes
- name: Copy Prometheus config file
  copy:
    src: prometheus.yaml
    dest: /etc/prometheus/prometheus.yaml
    mode: "0644"
  register: prom_config

- name: Ensure docker-py is installed
  pip:
    name: docker-py
    state: present
  become: yes

- name: Container present
  docker_container:
    name: prometheus
    state: started
    container_default_behavior: compatibility
    restart: "{{prom_config.changed}}"
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - /etc/prometheus/prometheus.yaml:/etc/prometheus/prometheus.yml
```

### Grafana dashboard

If everything went according to plan the metrics should already be visible on Grafana. In order to vizualize metrics we can use sample dashboard for node exporter. To do this, click on `Create->Import`, as shown in the screenshot below.

{{< figure src="import-dashboard.png" title="Import dashboard" >}}

Next we neet to provice id `1860` and click load. After dashboard is loaded, we need to select data source and click import.
When import is finish you should see a view similar to the one below. Of cource you can adjust this dashboard to your needs or create new one that meets your requirements. In my opinion this one looks very nice.

{{< figure src="prometheus-metrics.png" title="Node exporter dashboard" >}}

## Alerts

{{< figure src="grafana-cloud-alerting.png" title="Grafana Cloud Alerting" >}}

Collecting logs and metrics is configured so we can set up some alerts based on collected data.
With Grafana Cloud Aletring, we can define rules for alerts. Rules can be created based on logs or metrics.
Here you can find example rule for high CPU usage

```yaml
alert: HighUsage
expr: irate(node_cpu_seconds_total{mode="idle"}[1m]) * 100 < 30
for: 60s
annotations:
  description: "{{ $labels.instance }} has a CPU idle (current value: {{
    $value }}s)"
  summary: High usage on {{ $labels.instance }}
```

This alert will be triggered if for last minute idle CPU was under 30%.
If rules for alerts are already created, you need specify the mail to which emails should be sent. By default mails are sent from Grafana's servers but you can sepecify for servers as well.

{{< figure src="example-alert.png" title="Alert example" >}}

## Summary

In my opinion Grafana Labs provides really useful toolset for monitoring. Their free plan is more than enough for small projects, so if you have one I recommend giving it a try. All code you can find on my [github page](https://github.com/flunka/vps_monitoring)
