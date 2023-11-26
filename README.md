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

# #tl;dr
if it stinks, no ventilation. :)
