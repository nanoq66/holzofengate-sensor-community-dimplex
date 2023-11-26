# holzofengate-sensor-community-dimplex
Integration of sensor-community air sensor data with Prometheus &amp; Alertmanager to control centralised ventilation

# Motivation
The area I am living in is rather rural and many residents are historically (e.g. they own forests and are generating an income from selling fire wood) heating their houses with wood stoves. The pollution of these ovens, especially during initial ignition, can be rather high. Several factors (e.g. type of oven, ignition skills, wood type, dryness of wood) can play a role here, but the idea behind this project is not the political discussion around pro/con wood heating, air-pollution and #holzofengate, but a technical citizen science solution to mitigate the smoke issue for owners of modern houses with centralized ventilation.

# Architecture
[sensor-communitity air quality sensor](http://www.luftdaten.info/)  or https://github.com/Naesstrom/Airrohr-kit

exposes data via
    
[airrohr-prometheus-exporter](https://github.com/macbre/airrohr-prometheus-exporter)

which is scraped by a

[Prometheus TSDB](https://prometheus.io)

and thresholds are defined in rules for 

[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)

if an alert critera are met the alert triggers a

[Webhook](https://github.com/adnanh/webhook/)

which is switching of my Dimplex ZL 400 VF via Modbus (here customization might be needed if you don't have a Dimplex with Modbus interface).

If the air quality improves again the alert will be resolved and the ventilation will be reactivated.

and of course for monitoring & visualization we use [Grafana](https://grafana.com/grafana/)

# #tl;dr
if it stinks, no ventilation. :)

# Installation - how it's glued together
### Note: I won't dive into details how to setup linux basics like Python 3.x, apt or pip usage... please google or take use the manuals of the linked tools
## sensor-community air quality sensor
I have an SDS011 particulate matter sensor and a BME280 humidity/temperature sensor. But any particulate matter sensor will do, just the config of the integration has to be slightly adapted. The intake pipe of my ventilation is behind my house in a corner with my garage. So I placed the air quality sensor rougly 3m away and above the ventilation pipes, just be cause it was convenient and the position made sense, so the sensor can pick up any smoke before the ventilation sucks it in. Of course this is highly specific to my setting but I guess it might give a rough idea on how I placed the sensor. Also the setup could be easily be extended to multiple sensors if needed.

## airrohr-prometheus-exporter
The AQ sensor is actively pushing each measurement to the exporter see [docs](https://github.com/macbre/airrohr-prometheus-exporter#set-up-your-airrohr-sensors) for details. Especially homeserver, network, webserver and authentication setup is highly specific, so I just provide a simple setup for testing here. A simple webserver like gunicorn running on a port is sufficent for this purpose:

```
$ cat server.sh 
gunicorn --bind 0.0.0.0:"${HTTP_PORT:-8888}" --threads=1 --worker-class=gevent --worker-connections=1000 --access-logfile - app:app
$ nohup ./server.sh &
```

This gives us a prometheus endpoint. Each metric is a combination of a metric name, the value and a timestamp. This endpoint will now be configured as a scrape target in Prometheus.
```
$ curl localhost:8888/metrics
airrohr_last_measurement{sensor_id="esp8266-xxx"} 1.701024101e+09
# HELP airrohr_bme280_humidity bme280_humidity metric from airrohr.
# TYPE airrohr_bme280_humidity gauge
airrohr_bme280_humidity{sensor_id="esp8266-xxx"} 79.38 1701024101000
# HELP airrohr_bme280_pressure bme280_pressure metric from airrohr.
# TYPE airrohr_bme280_pressure gauge
airrohr_bme280_pressure{sensor_id="esp8266-xxx"} 94711.75 1701024101000
# HELP airrohr_bme280_temperature bme280_temperature metric from airrohr.
# TYPE airrohr_bme280_temperature gauge
airrohr_bme280_temperature{sensor_id="esp8266-xxx"} 3.64 1701024101000
(...)
# HELP airrohr_sds_p1 sds_p1 metric from airrohr.
# TYPE airrohr_sds_p1 gauge
airrohr_sds_p1{sensor_id="esp8266-xxx"} 3.5 1701024101000
# HELP airrohr_sds_p2 sds_p2 metric from airrohr.
# TYPE airrohr_sds_p2 gauge
airrohr_sds_p2{sensor_id="esp8266-xxx"} 2.17 1701024101000
```

## Prometheus & Alertmanager
Prometheus is a TSDB (Time Series Data Base), which is a highly optimized storage system for metrics with a temporal attribute. It can be setup highly-available and is very powerful for all kinds of applications, especially monitoring. E.g. you can setup a retention period of e.g. 1y to automatically purge the typically shortlived data.

Install Prometheus & Alertmanager by using the great [installation documentation](https://prometheus.io/docs/prometheus/latest/installation/) or some tutorial. Use your favourite service manager to run those as deamons. The default configuration of Prometheus and Alertmanager is usually ok, but can be customized to your liking. 

### Configure scrape target
What we need are some scrape configs in `prometheus.yml`:
```
global:
  scrape_interval: 1m

rule_files:
  - rules/rules.yml

# Alerting specifies settings related to the Alertmanager
alerting:
  alertmanagers:
    - static_configs:
      - targets:
        # Alertmanager's default port is 9093
        - localhost:9093
# Airrohr exporter
- job_name: airrohr
  scrape_interval: 30s
  static_configs:
  - targets:
     - localhost:8888 # if the airrohr exporter is running on the same host, otherwise use an ip
```
These few lines get our sensor data stored in Prometheus. The 30s are used to get a change of measurements as quickly as possible. The nice thing about Prometheus: duplicates with same value don't bloat up storage ;)

I have a few more scrape targets in here... e.g. I heavily use [mqtt2prometheus](https://github.com/hikhvar/mqtt2prometheus), also to get the data of my Dimplex.

### Alertmanager rules
This is where you define your rules and thresholds. Each alert rule consists of a name, expression and metadata. The expression and metadata is crucial to get right, because these are the triggers and are used in the next section to wire the alert to an actual action.

```for``` - the time until the alert is transitioning from pending to firing. prevents a constant flipping
```labels``` - used for routing in alertmanager.yml

```
$ /etc/prometheus/rules# cat rules.yml
groups:
- name: Feinstaub
  rules:
  - alert: PM25_above_threshold
    expr: airrohr_sds_p2{sensor_id="esp8266-xxx"} > 15
    for: 20s
    annotations: # used for Slack & Alertmanager UI
      title: 'PM2.5 above threshold'
      description: 'Theshold exceeded. Current value is: {{ .Value | humanize }}μg/m³'
    labels:
      severity: 'warning'
      type: 'feinstaub' 
  - alert: PM10_above_threshold
    expr: airrohr_sds_p1{sensor_id="esp8266-xxx"} > 20
    for: 30s
    annotations:
      title: 'PM10 above threshold'
      description: 'Threshold exceeded. Current value is: {{ .Value | humanize }}μg/m³'
    labels:
      severity: 'warning'
      type: 'feinstaub'
      target: 'dimplex' # <-- this is the actual trigger to switch of the ventilation
  - alert: PM10_above_WHO_threshold
    expr: airrohr_sds_p1{sensor_id="esp8266-xxx"} > 50
    for: 30s
    annotations:
      title: 'PM10 above WHO threshold'
      description: 'WMO Threshold of 50μg/m³ exeeded. Current value is: {{ .Value | humanize }}μg/m³'
    labels:
      severity: 'critical'
- name: Temperature # since we have the data and the alerting mechanism... we can also do other fun stuff
  rules:
  - alert: Freezing temperatures
    expr: airrohr_bme280_temperature{sensor_id="esp8266-xxx"} < 0
    for: 5m
    annotations:
      title: 'Freezing temperatures'
      description: 'Things might freeze. Temperatuere at: {{ .Value | humanize }}°C'
    labels:
      severity: 'warning'
```


### Alertmanager config
My alertmanager.yml config looks like this (I have also a Slack integration setup to notify me of my alerts). More or less you define a few globals variables, a route and receivers. So what happens here is: 
    - all alerts go to the global receiver 'slack-notifications', because we want all on slack (new alerts and also resolved). 
    - alerts with label ```target=dimplex``` will be routed to the receiver ```dimplex-alerting``` and that receiver has a ```webhook_config``` which triggers our webhook

```
global:
  resolve_timeout: 5m     # this resolves alerts if the threshold is no longer met. for me 5 minutes
  slack_api_url: 'https://hooks.slack.com/services/xxxxxxx (your token)

route:
  receiver: 'slack-notifications'
  group_wait: 10s
  repeat_interval: 30m
  routes:
    - matchers:
      - type="feinstaub" # German for pariculate matter
      - target="dimplex"  
      receiver: 'dimplex-alerting'

receivers:
- name: 'slack-notifications'
  slack_configs:
  - channel: '#alerts'
    send_resolved: true
    icon_url: https://sensor.community/favicon.png
    title: |-
     [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
    text: >-
     {{ range .Alerts -}}
       *Description:* {{ .Annotations.description }}
     {{ end }}
- name: 'dimplex-alerting'
  webhook_configs:
  - url: 'http://localhost:9000/hooks/dimplex-alerting' # ignore that for now... will be explained in webhook section
```

## Webhooks
I use https://github.com/adnanh/webhook/ because it is easy to use, powerful and stable. Install it and define your webhook(s). Plural because I use this to switch modes of my Dimplex ventilation remotely, but lets focus on the #holzofengate application:

in my ```webhook.conf``` I have the dimplex-alerting webhook defined, which is actually executing a Python script which implements the actual logic. For the logic to work the entire-payload of the alert will be passed on. The separation into this Python script allowed me easy testing without lowering thresholds or manually triggering alerts with cUrl calls.

```
- id: dimplex-alerting
  execute-command: "/opt/webhooks/dimplex_alerting.py"
  pass-arguments-to-command:
  - source: "entire-payload"
```

## Dimplex specific Modbus script
I guess the webhook script needs to be adapted for most users that would like to build something similar. My Dimplex has a ModBus extension, where I can retrieve some data (rpm of fans, temperatures) and also I have the option to switch between working states (off, auto, modes low, medium, max). I opted for this approach, because I didn't want to hard power off the device (e.g. with a shelly).

The ```dimplex-alerting.py``` script evaluates the workload of the alert and depending on the alert state it triggers a modbus call. ```firing``` switches to mode off, all other alert states (e.g. resolved or pending) switch or keep the device in auto mode. Some logging was added to actually see what was happening somewhere during the night ;)
```
#!/usr/bin/env python
import sys
import syslog
import json

from pyModbusTCP.client import ModbusClient

syslog.syslog("***** called dimplex-alerting.py *****")
syslog.syslog(' '.join(sys.argv[1:]))
c = ModbusClient(host="192.168.x.x the host of the ModBusTCP device hooked to the Dimplex", port=8080, timeout=10, unit_id=100, auto_open=True, auto_close=True)

payload = json.loads(sys.argv[1])
#syslog.syslog("payload: " + json.dumps(payload)) # debugging ;)
alerts = payload["alerts"][0]
syslog.syslog("alert status: "+alerts["status"])

if alerts["status"] == "firing":
    if c.write_single_register(1, 0):
        syslog.syslog("write ok - dimplex set to off due to alert")
    else:
        syslog.syslog("write error")
else:
    syslog.syslog(alerts["status"])
    if c.write_single_register(1, 1):
        syslog.syslog("write ok - dimplex set to AUTO")
    else:
        syslog.syslog("write error")

```

# Grafana
Install Grafana, setup your Prometheus instance as a datasource and create a new dashboard by importing the [dashboard.json](dashboard.json) in this repository.







