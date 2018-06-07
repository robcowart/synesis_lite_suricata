# sýnesis&trade; Lite for Suricata
sýnesis&trade; Lite for Suricata provides basic log analytics for Suricata IDS/IPS using the Elastic Stack.

<img width="1163" alt="synesis_lite_suricata" src="https://user-images.githubusercontent.com/10326954/40805451-99e3f6e6-651e-11e8-9559-8e4535cd9b2c.png">

# Your feedback is welcome.

> This readme will be expanded upon, however it follows the same deployment method as [ElastiFlow&trade;](https://github.com/robcowart/elastiflow). If you review that readme, you should be able deploy this solution.

sýnesis&trade; Lite for Suricata is a solution for the collection and analysis of Suricata "eve" JSON logs. This includes alerts, flows, http, dns, statistics and other log types.

## Suricata
An example configuration for the Suricata eve output is provided in `suricata/suricata.yml`.

## Filebeat
As Suricata is usually run on one or more Linux servers, the solution includes both [Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-overview.html) and [Logstash](https://www.elastic.co/guide/en/logstash/current/introduction.html). Filebeat is used to collect the log data on the system where Suricata is running, and ships it to Logstash via the Beats input. An example Filebeat prospector configuration is included in `filebeat/filebeat.yml`.

## Logstash
The Logstash pipeline is complete and fully functional. (Setup is similar to [ElastiFlow&trade;](https://github.com/robcowart/elastiflow))

## Kibana
Dashboards for alerts, flow, DNS and HTTP logs are also complete and functional. Only the dashbaords for the statistics logs remains on the to-do list.

Kibana index patterns can be imported with the following commands:

```
curl -X POST -u USERNAME:PASSWORD http://KIBANASERVER:5601/api/saved_objects/index-pattern/elastiflow-* -H "Content-Type: application/json" -H "kbn-xsrf: true" -d @/PATH/TO/synlite_suricata.index_pattern.json

curl -X POST -u USERNAME:PASSWORD http://KIBANASERVER:5601/api/saved_objects/index-pattern/elastiflow-* -H "Content-Type: application/json" -H "kbn-xsrf: true" -d @/PATH/TO/synlite_suricata_stats.index_pattern.json
```

Kibana dashboards can be imported in the Kibana `Management` app under `Saved Objects`.

**If you have any feedback, please open an issue and share it.**

# Attribution
This product includes GeoLite data created by MaxMind, available from (http://www.maxmind.com)
