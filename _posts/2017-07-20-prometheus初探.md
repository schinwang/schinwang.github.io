---
bg: "superman.jpg"
layout: post
title: "prometheus初探"
summary: ""
tags: ['ops']
categories: ops
---

[1 模块](#module)

[2 使用](#usage)

[3 配置文件示例](#config)

​	[3.1 Prometheus配置](#prometheus_config)

​	[3.2 alertmanager 配置](#alert_config)



Prometheus监控系统是类似于[Google Borgmon](http://flacro.me/google-borgmon/)的一个开源监控系统，使用go语言实现. Prometheus是一个白盒监控系统，监控Server采用pull方式，通过对监控模块发起HTTP请求获取应用内部的服务状态(如请求数，状态，goroutine等)，同时提供了一个push_gateway模块满足对定时任务(或批处理任务，一般为非连续性的) 的监控数据采集.

#### 1 模块 {#module}  

Prometheus系统包括几个主要模块:

- **prometheus**本身是一个server模块，负责监控指标配置，监控数据收集，告警规则设置，server本身提供了一个UI界面，可以查看相关的配置及监控指标数据
  ![prometheus](/assets/images/posts/0720/prometheus.png)

- **[grafana](https://grafana.com/*)**是一个开源的指标量监测和可视化工具. 对Prometheus查询语言做了接入兼容, 可以对监控指标做友好展示

  ![grafana](/assets/images/posts/0720/grafana.png)

- **alertmanager**模块是一个独立的告警模块，通过配置告警人, 告警分组, 告警方式可以按照不同的规则发送给不同的模块负责人，alertmanager支持email, slack，等告警方式, 也可以通过webhook接入钉钉等国内IM工具.

  prometheus接入alertmanager只需启动时通过-alertmanager.url参数指定告警模块地址即可

- **pushgatewey**模块提供了一种push方式上报监控指标的方法，通过将脚本或其他任务的采集数据上报给push gateway，然后Prometheus server定期去push gateway pull 数据

- **node_exporter**是基础监控模块，采集cpu/disk/mem/io等信息

#### 2 使用 {#usage}

Prometheus的使用门槛在于其配置和自己的查询语言
比如，查询cpu idle，node_exporter中并不是易于理解的百分比数值，需要通过

```shell
avg(irate(node_cpu{mode="idle"}[5m])*100)  来获取5min内的平均IDLE值
```

再比如，配置一个延时告警，也需要通过在配置文件中通过指令配置

```shell
{% raw %}
# Alert for any instance that have a median request latency >1s.
ALERT APIHighRequestLatency
  IF api_http_request_latencies_second{quantile="0.5"} > 1
  FOR 1m
  ANNOTATIONS {
    summary = "High request latency on {{ $labels.instance {{",
    description = "{{ $labels.instance {{ has a median request latency above 1s (current value: {{ $value {{s)",
  }
{% endraw %}
```

由于配置是需要不定期更新的，所以Prometheus支持对配置文件的热加载，通过发送HUP信号给进程即可重新加载配置文件

#### 3 配置文件示例 {#config}

##### 3.1 Prometheus配置 {#prometheus_config}

prometheus.yml

```shell
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  scrape_timeout:      5s # How long until a scrape request times out, default 10s.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'tiantian'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
#告警规则
rule_files:
  - "rules/*.rule"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
#监控任务
  - job_name: 'prometheus'
    static_configs:
      - targets: ['*.*.*.*:9090', '*.*.*.*:9100']
  # relay's monitoring scripts
  - job_name: 'relay'
    static_configs:
      - targets: ['*.*.*.*:9091','*.*.*.*:9100']
  - job_name: 'php'
    file_sd_configs:
      - files:
        - "targets/php.json"
#告警服务
alerting:
  alert_relabel_configs:
  alertmanagers:
    - static_configs:
      - targets: ['127.0.0.1:9093']        
```

targets/php.json

```shell
[
  {
    "targets": ["10.0.0.2:9100", "10.0.0.4:9100", "10.0.0.3:9100", "10.0.0.5:9100", "10.0.0.24:9100", "10.0.0.25:9100"]
  }
]
```

rules/basic.rule

```shell
{% raw %}
ALERT cpu_idle
  IF avg(irate(node_cpu{mode="idle",job="go-relation"}[5m])*100) < 20
  FOR 1m
  ANNOTATIONS {
    summary = "{{ $labels.instance {{: cpu使用率超80%, current_$value: {{ $value {{"
  }

ALERT cpu_steal
  IF avg(irate(node_cpu {mode="steal"}[5m])*100) > 10
  FOR 1m
  ANNOTATIONS {
    summary = "{{ $labels.instance {{: cpu争抢严重, current_$value: {{ $value {{"
  }

ALERT iowait
  IF avg(irate(node_cpu{mode="iowait"}[5m])*100) > 20
  FOR 1m
  ANNOTATIONS {
    summary = "{{ $labels.instance {{: iowait率偏高, current_$value: {{ $value {{"
  }

ALERT mem_free
  IF (node_memory_MemFree+node_memory_Cached)/node_memory_MemTotal*100 < 20
  FOR 1m
  ANNOTATIONs {
    summary = "{{$labels.instance{{: 空闲内存占比<20%, current$value:{{$value{{"
  }

ALERT sys_disk_usage
  IF node_filesystem_free{mountpoint="/"}/node_filesystem_size{mountpoint="/"}*100 < 20
  FOR 1m
  ANNOTATIONS {
    summary = "{{ $labels.instance {{: 系统盘使用率超80%, current_$value: {{ $value {{"
  }

ALERT data_disk_usage
  IF node_filesystem_free{mountpoint="/data"} / node_filesystem_size{mountpoint="/data"}*100 < 20
  FOR 1m
  ANNOTATIONS {
    summary = "{{ $labels.instance {{: 数据盘使用率超80%, current_$value: {{ $value {{"
  }
{% endraw %}
```

##### 3.2 alertmanager 配置 {#alert_config}

```shell
global:
  # The smarthost and SMTP sender used for mail notifications.
  smtp_smarthost: 'smtp.qq.com:587'
  smtp_from: '***********'
  smtp_auth_username: '************'
  smtp_auth_password: '*************'
  resolve_timeout: 1m
  # The auth token for Hipchat.
  #hipchat_auth_token: '1234556789'
  # Alternative host for Hipchat.
  #hipchat_url: 'https://hipchat.foobar.org/'

# The directory from which notification templates are read.
templates:
- '/etc/alertmanager/template/*.tmpl'

# The root route on which each incoming alert enters.
route:
  # The labels by which incoming alerts are grouped together. For example,
  # multiple alerts coming in for cluster=A and alertname=LatencyHigh would
  # be batched into a single group.
  group_by: ['alertname', 'job']

  # When a new group of alerts is created by an incoming alert, wait at
  # least 'group_wait' to send the initial notification.
  # This way ensures that you get multiple alerts for the same group that start
  # firing shortly after another are batched together on the first 
  # notification.
  group_wait: 30s

  # When the first notification was sent, wait 'group_interval' to send a batch
  # of new alerts that started firing for that group.
  group_interval: 1m

  # If an alert has successfully been sent, wait 'repeat_interval' to
  # resend them.
  repeat_interval: 1m

  # A default receiver
  receiver: RD

  # All the above attributes are inherited by all child routes and can 
  # overwritten on each.

  # The child route trees.
  routes:
  # This routes performs a regular expression match on alert labels to
  # catch alerts that are related to a list of services.
  - match_re:
    #service: ^(foo1|foo2|baz)$
    #severity: critical
    receiver: RD
    # The service has a sub-route for critical alerts, any alerts
    # that do not match, i.e. severity != critical, fall-back to the
    # parent node and are sent to 'team-X-mails'
# already critical.
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  equal: ['alertname', 'cluster', 'service']

receivers:
- name: 'RD'
  email_configs:
  - to: '**********'
  webhook_configs:
  - url: 'http://yourhost/dingding'
```

告警用钉钉，因为钉钉webhook的pos格式t固定，所以需要单独起一个server用来转换alertmanager的告警内容(可以用flask起一个python server)

```python
@main.route('/dingding',methods=['POST'])
def send():
  if request.method == 'POST':
     post_data = request.get_data()
     return alert_data(post_data)
  return
def alert_data(data):
  import json
  data = json.loads(post_data)
  #print data
  data = data['alerts'][0]
  content = {
    'status': data['status'],
    'module': data['labels']['job'],
    'annotations': data['annotations'],
    'url': 'http://prometheus_url/alerts'
  }
  import requests
  url = '钉钉机器人webhook'
  send_data = {
    "msgtype": "text",
    "text": {
      "content": content
    }
  }
  headers = {
    'Content-Type':'application/json'
  }
  r =  requests.post(url,headers=headers,data=json.dumps(send_data))
  return jsonify({'status':r.ok,'mesg':r.text})

```

告警内容类似

```shell
{"module":"relay","status":"firing","annotations":{"summary":"10.0.0.2:9100: 系统盘使用率超80%, current_$value: 39.616372037291725"},"url":"http://prometheus_url/alerts"}
```

