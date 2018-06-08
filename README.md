# sýnesis&trade; Lite for Suricata
sýnesis&trade; Lite for Suricata provides basic log analytics for Suricata IDS/IPS using the Elastic Stack. It is a solution for the collection and analysis of Suricata "eve" JSON logs. This includes alerts, flows, http, dns, statistics and other log types.

<img width="1163" alt="synesis_lite_suricata" src="https://user-images.githubusercontent.com/10326954/40805451-99e3f6e6-651e-11e8-9559-8e4535cd9b2c.png">

# Getting Started
sýnesis&trade; Lite for Suricata is built using the Elastic Stack, including Elasticsearch, Logstash and Kibana. To install and configure sýnesis&trade; Lite for Suricata, you must first have a working Elastic Stack environment. The latest release requires Elastic Stack version 6.2 or later.

Refer to the following compatibility chart to choose a release of sýnesis&trade; Lite for Suricata that is compatible with the version of the Elastic Stack you are using.

Elastic Stack |  v1.x
:---:|:---:
6.2 | &#10003;

## Suricata
An example configuration for the Suricata eve output is provided in `suricata/suricata.yml`.

## Setting up Elasticsearch
Currently there is no specific configuration required for Elasticsearch. As long as Kibana and Logstash can talk to your Elasticsearch cluster you should be ready to go. The index template required by Elasticsearch will be uploaded by Logstash.

At high ingest rates (>5K logs/s), or for data redundancy and high availability, a multi-node cluster is recommended.

## Filebeat
As Suricata is usually run on one or more Linux servers, the solution includes both [Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-overview.html) and [Logstash](https://www.elastic.co/guide/en/logstash/current/introduction.html). Filebeat is used to collect the log data on the system where Suricata is running, and ships it to Logstash via the Beats input. An example Filebeat prospector configuration is included in `filebeat/filebeat.yml`.

## Setting up Logstash
The sýnesis&trade; Lite for Suricata Logstash pipeline is the heart of the solution. It is here that the raw flow data is collected, decoded, parsed, formatted and enriched. It is this processing that makes possible the analytics options provided by the Kibana [dashboards](#dashboards).

Follow these steps to ensure that Logstash and sýnesis&trade; Lite for Suricata are optimally configured to meet your needs. 

### 1. Set JVM heap size.
To increase performance, sýnesis&trade; Lite for Suricata takes advantage of the caching and queueing features available in many of the Logstash plugins. These features increase the consumption of the JVM heap. The JVM heap space used by Logstash is configured in `jvm.options`. It is recommended that Logstash be given at least 2GB of JVM heap. This is configured in `jvm.options` as follows:

```
-Xms2g
-Xmx2g
```

### 2. Add and Update Required Logstash plugins
It is also recommended that you always use the latest version of the [DNS](https://www.elastic.co/guide/en/logstash/current/plugins-filters-dns.html) filter. This can achieved by running the following command:

```
LS_HOME/bin/logstash-plugin update logstash-filter-dns
```

### 3. Copy the pipeline files to the Logstash configuration path.
There are four sets of configuration files provided within the `logstash/synlite_suricata` folder:
```
logstash
  `- synlite_suricata
       |- conf.d  (contains the logstash pipeline)
       |- dictionaries (yaml files used to enrich the raw log data)
       |- geoipdbs  (contains GeoIP databases)
       `- templates  (contains index templates)
```

Copy the `synlite_suricata` directory to the location of your Logstash configuration files (e.g. on RedHat/CentOS or Ubuntu this would be `/etc/logstash/synlite_suricata` ). If you place the pipeline within a different path, you will need to modify the following environment variables to specify the correct location:

Environment Variable | Description | Default Value
--- | --- | ---
SYNLITE_SURICATA_DICT_PATH | The path where the dictionary files are located | /etc/logstash/synlite_suricata/dictionaries
SYNLITE_SURICATA__TEMPLATE_PATH | The path to where index templates are located | /etc/logstash/synlite_suricata/templates
SYNLITE_SURICATA_GEOIP_DB_PATH | The path where the GeoIP DBs are located | /etc/logstash/synlite_suricata/geoipdbs

### 4. Setup environment variable helper files
Rather than directly editing the pipeline configuration files for your environment, environment variables are used to provide a single location for most configuration options. These environment variables will be referred to in the remaining instructions. A [reference](#environment-variable-reference) of all environment variables can be found [here](#environment-variable-reference).

Depending on your environment there may be many ways to define environment variables. The files `profile.d/synlite_suricata.sh` and `logstash.service.d/synlite_suricata.conf` are provided to help you with this setup.

Recent versions of both RedHat/CentOS and Ubuntu use systemd to start background processes. When deploying sýnesis&trade; Lite for Suricata on a host where Logstash will be managed by systemd, copy `logstash.service.d/synlite_suricata.conf` to `/etc/systemd/system/logstash.service.d/synlite_suricata.conf`. Any configuration changes can then be made by editing this file.

> Remember that for your changes to take effect, you must issue the command `sudo systemctl daemon-reload`.

### 5. Add the sýnesis&trade; Lite for Suricata pipeline to pipelines.yml
Logstash 6.0 introduced the ability to run multiple pipelines from a single Logstash instance. The `pipelines.yml` file is where these pipelines are configured. While a single pipeline can be specified directly in `logstash.yml`, it is a good practice to use `pipelines.yml` for consistency across environments.

Edit `pipelines.yml` (usually located at `/etc/logstash/pipelines.yml`) and add the sýnesis&trade; Lite for Suricata pipeline (adjust the path as necessary).

```
- pipeline.id: synlite_suricata
  path.config: "/etc/logstash/synlite_suricata/conf.d/*.conf"
```

### 6. Configure inputs
By default Filebeat data will be recieved on all IPv4 addresses of the Logstash host using the default TCP port 5044. You can change both the IP and port used by modifying the following environment variables:

Environment Variable | Description | Default Value
--- | --- | ---
SYNLITE_SURICATA_BEATS_HOST | The IP address on which to listen for Filebeat messages | 0.0.0.0
SYNLITE_SURICATA_BEATS_PORT | The TCP port on which to listen for Filebeat messages | 5044

### 7. Configure Elasticsearch output
Obviously the data needs to land in Elasticsearch, so you need to tell Logstash where to send it. This is done by setting these environment variables:

Environment Variable | Description | Default Value
--- | --- | ---
SYNLITE_SURICATA_ES_HOST | The Elasticsearch host to which the output will send data | 127.0.0.1:9200
SYNLITE_SURICATA_ES_USER | The password for the connection to Elasticsearch | elastic
SYNLITE_SURICATA_ES_PASSWD | The username for the connection to Elasticsearch | changeme

> If you are only using the open-source version of Elasticsearch, it will ignore the username and password. In that case just leave the defaults.

### 8. Enable DNS name resolution (optional)
In the past it was recommended to avoid DNS queries as the latency costs of such lookups had a devastating effect on throughput. While the Logstash DNS filter provides a caching mechanism, its use was not recommended. When the cache was enabled all lookups were performed synchronously. If a name server failed to respond, all other queries were stuck waiting until the query timed out. The end result was even worse performance.

Fortunately these problems have been resolved. Release 3.0.8 of the DNS filter introduced an enhancement which caches timeouts as failures, in addition to normal NXDOMAIN responses. This was an important step as many domain owner intentionally setup their nameservers to ignore the reverse lookups needed to enrich flow data. In addition to this change, I submitted am enhancement which allows for concurrent queries when caching is enabled. The Logstash team approved this change, and it is included in 3.0.10 of the plugin.

With these changes I can finally give the green light for using DNS lookups to enrich the incoming data. You will see a little slow down in throughput until the cache warms up, but that usually lasts only a few minutes. Once the cache is warmed up, the overhead is minimal, and event rates averaging 10K/s and as high as 40K/s were observed in testing.

The key to good performance is setting up the cache appropriately. Most likely it will be DNS timeouts that are the source of most latency. So ensuring that a higher volume of such misses can be cached for longer periods of time is most important.

The DNS lookup features of sýnesis&trade; Lite for Suricata can be configured using the following environment variables:

Environment Variable | Description | Default Value
--- | --- | ---
SYNLITE_SURICATA_RESOLVE_IP2HOST | Enable/Disable DNS requests | false
SYNLITE_SURICATA_NAMESERVER | The DNS server to which the dns filter should send requests | 127.0.0.1
SYNLITE_SURICATA_DNS_HIT_CACHE_SIZE | The cache size for successful DNS queries | 25000
SYNLITE_SURICATA_DNS_HIT_CACHE_TTL | The time in seconds successful DNS queries are cached | 900
SYNLITE_SURICATA_DNS_FAILED_CACHE_SIZE | The cache size for failed DNS queries | 75000
SYNLITE_SURICATA_DNS_FAILED_CACHE_TTL | The time in seconds failed DNS queries are cached | 3600

### 9. Start Logstash
You should now be able to start Logstash and begin collecting network flow data. Assuming you are running a recent version of RedHat/CentOS or Ubuntu, and using systemd, complete these steps:
1. Run `systemctl daemon-reload` to ensure any changes to the environment variables are recognized.
2. Run `systemctl start logstash`
> NOTICE! Make sure that you have already setup the Logstash init files by running `LS_HOME/bin/system-install`. If the init files have not been setup you will receive an error.
To follow along as Logstash starts you can tail its log by running:
```
tail -f /var/log/logstash/logstash-plain.log
```
Logstash takes a little time to start... BE PATIENT!

Logstash setup is now complete. If you are receiving data from Filebeat, you should have both `suricata-` and `suricata-stats` daily indices in Elasticsearch.

## Setting up Kibana
An API (yet undocumented) is available to import and export Index Patterns. The JSON files which contains the Index Pattern configurations are `synlite_suricata.index_pattern.json` and `synlite_suricata_stats.index_pattern.json`. To setup the Index Patterns run the following commands:

```
curl -X POST -u USERNAME:PASSWORD http://KIBANASERVER:5601/api/saved_objects/index-pattern/suricata-* -H "Content-Type: application/json" -H "kbn-xsrf: true" -d @/PATH/TO/synlite_suricata.index_pattern.json

curl -X POST -u USERNAME:PASSWORD http://KIBANASERVER:5601/api/saved_objects/index-pattern/suricata_stats-* -H "Content-Type: application/json" -H "kbn-xsrf: true" -d @/PATH/TO/synlite_suricata_stats.index_pattern.json
```

Finally the vizualizations and dashboards can be loaded into Kibana by importing the `synlite_suricata.dashboards.json` file from within the Kibana UI. This is done in the Kibana `Management` app under `Saved Objects`.

### Recommended Kibana Advanced Settings
You may find that modifying a few of the Kibana advanced settings will produce a more user-friendly experience while using sýnesis&trade; Lite for Suricata . These settings are made in Kibana, under `Management -> Advanced Settings`.

Advanced Setting | Value | Why make the change?
--- | --- | ---
doc_table:highlight | false | There is a pretty big query performance penalty that comes with using the highlighting feature. As it isn't very useful for this use-case, it is better to just trun it off.
filters:pinnedByDefault | true | Pinning a filter will it allow it to persist when you are changing dashbaords. This is very useful when drill-down into something of interest and you want to change dashboards for a different perspective of the same data. This is the first setting I change whenever I am working with Kibana.
state:storeInSessionStorage | true | Kibana URLs can get pretty large. Especially when working with Vega visualizations. This will likely result in error messages for users of Internet Explorer. Using in-session storage will fix this issue for these users.
timepicker:quickRanges | [see below](#recommended-setting-for-timepicker:quickRanges) | The default options in the Time Picker are less than optimal, for most logging and monitoring use-cases. Fortunately Kibana no allows you to customize the time picker. Our recommended settings can be found [see below](#recommended-setting-for-timepicker:quickRanges).

## Dashboards
The following dashboards are provided.

> NOTE: The dashboards are optimized for a monitor resolution of 1920x1080.

### Alerts - Overview
![suricata_alerts_overview](https://user-images.githubusercontent.com/10326954/41160782-f4eee61e-6b30-11e8-90dc-b90d85553959.png)

### Alerts - Messages
![suricata_alerts_messages](https://user-images.githubusercontent.com/10326954/41160783-f53fcc3c-6b30-11e8-8375-9f254184a540.png)

### Threats - Public Attackers
![suricata_threat_pub_attack](https://user-images.githubusercontent.com/10326954/41160777-f3ff57a2-6b30-11e8-9ac0-a77ecf6c988c.png)

### Threats - At-Risk Servers
![suricata_threat_risk_servers](https://user-images.githubusercontent.com/10326954/41160723-dcd08e98-6b30-11e8-8793-5b2464dd0685.png)

### Threats - At-Risk Services
![suricata_threat_risk_services](https://user-images.githubusercontent.com/10326954/41160725-dcfb5d3a-6b30-11e8-8b35-fd0044130662.png)

### Threats - High-Risk Clients
![suricata_threat_risk_clients](https://user-images.githubusercontent.com/10326954/41160778-f43acd3c-6b30-11e8-8d42-5923b25db30a.png)

### Flows - Overview
![suricata_flows_overview](https://user-images.githubusercontent.com/10326954/41160766-f2431eda-6b30-11e8-8cfd-652c5902fa15.png)

### Flows - Top Talkers
![suricata_flows_talkers](https://user-images.githubusercontent.com/10326954/41160769-f2d7037a-6b30-11e8-9f0d-067ea6372296.png)

### Flows - Top Services
![suricata_flows_services](https://user-images.githubusercontent.com/10326954/41160768-f2a97982-6b30-11e8-84cc-527f52f822fd.png)

### Flows - Sankey
![suricata_flows_sankey](https://user-images.githubusercontent.com/10326954/41160767-f277a81c-6b30-11e8-8130-0625c70e53a7.png)

### Flows - Geo IP
![suricata_flows_geo](https://user-images.githubusercontent.com/10326954/41160764-f1e2fd16-6b30-11e8-8a92-a08d406b8d5e.png)

### Flows - Messages
![suricata_flows_messages](https://user-images.githubusercontent.com/10326954/41160765-f211f490-6b30-11e8-9f9b-189114be23bb.png)

### HTTP - Overview
![suricata_http_overview](https://user-images.githubusercontent.com/10326954/41160772-f35fadf6-6b30-11e8-9002-950dcb9ffc74.png)

### HTTP - Messages
![suricata_http_messages](https://user-images.githubusercontent.com/10326954/41160770-f327274c-6b30-11e8-9c30-33a6a552878c.png)

### HTTP - Overview
![suricata_dns_overview](https://user-images.githubusercontent.com/10326954/41160779-f47b45ec-6b30-11e8-872c-0bd1f16f0991.png)

### HTTP - Messages
![suricata_dns_messages](https://user-images.githubusercontent.com/10326954/41160781-f4b413b8-6b30-11e8-8e9b-1574873f99f2.png)

### HTTP - Raw Logs
![suricata_raw_logs](https://user-images.githubusercontent.com/10326954/41160775-f3963448-6b30-11e8-8d16-22cbe71b80e3.png)

### HTTP - Statistics
![suricata_stats](https://user-images.githubusercontent.com/10326954/41160776-f3ceb138-6b30-11e8-8baa-189b8c7cdee5.png)

# Environment Variable Reference
The supported environment variables are:

Environment Variable | Description | Default Value
--- | --- | ---
SYNLITE_SURICATA_DICT_PATH | The path where the dictionary files are located | /etc/logstash/synlite_suricata/dictionaries
SYNLITE_SURICATA_TEMPLATE_PATH | The path to where index templates are located | /etc/logstash/synlite_suricata/templates
SYNLITE_SURICATA_GEOIP_DB_PATH | The path where the GeoIP DBs are located | /etc/logstash/synlite_suricata/geoipdbs
SYNLITE_SURICATA_GEOIP_CACHE_SIZE | The size of the GeoIP query cache | 8192
SYNLITE_SURICATA_GEOIP_LOOKUP | Enable/Disable GeoIP lookups | true
SYNLITE_SURICATA_ASN_LOOKUP | Enable/Disable ASN lookups | true
SYNLITE_SURICATA_CLEANUP_SIGS | Enable this option to remove unneeded text from alert signatures. | false
SYNLITE_SURICATA_RESOLVE_IP2HOST | Enable/Disable DNS requests | false
SYNLITE_SURICATA_NAMESERVER | The DNS server to which the dns filter should send requests | 127.0.0.1
SYNLITE_SURICATA_DNS_HIT_CACHE_SIZE | The cache size for successful DNS queries | 25000
SYNLITE_SURICATA_DNS_HIT_CACHE_TTL | The time in seconds successful DNS queries are cached | 900
SYNLITE_SURICATA_DNS_FAILED_CACHE_SIZE | The cache size for failed DNS queries | 75000
SYNLITE_SURICATA_DNS_FAILED_CACHE_TTL | The time in seconds failed DNS queries are cached | 3600
SYNLITE_SURICATA_ES_HOST | The Elasticsearch host to which the output will send data | 127.0.0.1:9200
SYNLITE_SURICATA_ES_USER | The password for the connection to Elasticsearch | elastic
SYNLITE_SURICATA_ES_PASSWD | The username for the connection to Elasticsearch | changeme
SYNLITE_SURICATA_BEATS_HOST | The IP address on which to listen for Filebeat messages | 0.0.0.0
SYNLITE_SURICATA_BEATS_PORT | The TCP port on which to listen for Filebeat messages | 5044

# Recommended Setting for timepicker:quickRanges
I recommend configuring `timepicker:quickRanges` for the setting below. The result will look like this:

![screen shot 2018-05-17 at 19 57 03](https://user-images.githubusercontent.com/10326954/40195016-8d33cac4-5a0c-11e8-976f-cc6559e4439a.png)

```
[
  {
    "from": "now/d",
    "to": "now/d",
    "display": "Today",
    "section": 0
  },
  {
    "from": "now/w",
    "to": "now/w",
    "display": "This week",
    "section": 0
  },
  {
    "from": "now/M",
    "to": "now/M",
    "display": "This month",
    "section": 0
  },
  {
    "from": "now/d",
    "to": "now",
    "display": "Today so far",
    "section": 0
  },
  {
    "from": "now/w",
    "to": "now",
    "display": "Week to date",
    "section": 0
  },
  {
    "from": "now/M",
    "to": "now",
    "display": "Month to date",
    "section": 0
  },
  {
    "from": "now-15m",
    "to": "now",
    "display": "Last 15 minutes",
    "section": 1
  },
  {
    "from": "now-30m",
    "to": "now",
    "display": "Last 30 minutes",
    "section": 1
  },
  {
    "from": "now-1h",
    "to": "now",
    "display": "Last 1 hour",
    "section": 1
  },
  {
    "from": "now-2h",
    "to": "now",
    "display": "Last 2 hours",
    "section": 1
  },
  {
    "from": "now-4h",
    "to": "now",
    "display": "Last 4 hours",
    "section": 2
  },
  {
    "from": "now-12h",
    "to": "now",
    "display": "Last 12 hours",
    "section": 2
  },
  {
    "from": "now-24h",
    "to": "now",
    "display": "Last 24 hours",
    "section": 2
  },
  {
    "from": "now-48h",
    "to": "now",
    "display": "Last 48 hours",
    "section": 2
  },
  {
    "from": "now-7d",
    "to": "now",
    "display": "Last 7 days",
    "section": 3
  },
  {
    "from": "now-30d",
    "to": "now",
    "display": "Last 30 days",
    "section": 3
  },
  {
    "from": "now-60d",
    "to": "now",
    "display": "Last 60 days",
    "section": 3
  },
  {
    "from": "now-90d",
    "to": "now",
    "display": "Last 90 days",
    "section": 3
  }
]
```

# Attribution
This product includes GeoLite data created by MaxMind, available from (http://www.maxmind.com)
