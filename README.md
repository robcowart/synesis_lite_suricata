# THIS PROJECT HAS BEEN ARCHIVED!

## I may revisit Suricata logs in the future, however I will likely focus on replacing the Filebeat and Logstash components with a more purpose built collector.

# sýnesis&trade; Lite for Suricata

sýnesis&trade; Lite for Suricata provides basic log analytics for Suricata IDS/IPS using the Elastic Stack. It is a solution for the collection and analysis of Suricata "eve" JSON logs. This includes alerts, flows, http, dns, statistics and other log types.

![synesis_lite_suricata](https://user-images.githubusercontent.com/10326954/58703990-c851e500-83aa-11e9-988a-2c0dea8f4cc4.png)

# Getting Started

sýnesis&trade; Lite for Suricata is built using the Elastic Stack, including Elasticsearch, Logstash and Kibana. To install and configure sýnesis&trade; Lite for Suricata, you must first have a working Elastic Stack environment. The latest release requires Elastic Stack version 6.2 or later.

Refer to the following compatibility chart to choose a release of sýnesis&trade; Lite for Suricata that is compatible with the version of the Elastic Stack you are using.

Elastic Stack | Release
:---:|:---:
7.1.x | &#10003; (v1.1.0)
7.0.x | &#10003; (v1.1.0)
6.x | &#10003; (v1.0.1)

## Suricata

Suggested configurations for the Suricata eve log output, and app_layer protocols is provided in the files found in the `suricata` directory.

## Setting up Elasticsearch

Previous versions of the Elastic Stack required no special configuration for Elasticsearch. However changes made to Elasticsearch 7.x, require that the following settings be made in `elasticsearch.yml`:

```text
indices.query.bool.max_clause_count: 8192
search.max_buckets: 250000
```

At high ingest rates (>5K logs/s), or for data redundancy and high availability, a multi-node cluster is recommended.

If you are new to the Elastic Stack, this video goes beyond a simple default installation of Elasticsearch and Kibana. It discusses real-world best practices for hardware sizing and configuration, providing production-level performance and reliability.

[![0003_es_install](https://user-images.githubusercontent.com/10326954/76195457-9ea2d580-61e8-11ea-8578-8fb39908afec.png)](https://www.youtube.com/watch?v=gZb7HpVOges)

Additionally local SSD storage should be considered as _*mandatory*_! For an in-depth look at how different storage options compare, and in particular how bad HDD-based storage is for Elasticsearch (even in multi-drive RAID0 configurations) you should watch this video...

[![0001_es_storage](https://user-images.githubusercontent.com/10326954/76195348-61d6de80-61e8-11ea-951d-1694d2e0392b.png)](https://www.youtube.com/watch?v=nKUpfJCBiS4)

## Filebeat

As Suricata is usually run on one or more Linux servers, the solution includes both [Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-overview.html) and [Logstash](https://www.elastic.co/guide/en/logstash/current/introduction.html). Filebeat is used to collect the log data on the system where Suricata is running, and ships it to Logstash via the [beats](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-beats.html) input. An example Filebeat log input configuration is included in `filebeat/filebeat.yml`.

## Setting up Logstash

The sýnesis&trade; Lite for Suricata Logstash pipeline is the heart of the solution. It is here that the raw flow data is collected, decoded, parsed, formatted and enriched. It is this processing that makes possible the analytics options provided by the Kibana [dashboards](#dashboards).

Follow these steps to ensure that Logstash and sýnesis&trade; Lite for Suricata are optimally configured to meet your needs. 

### 1. Set JVM heap size.

To increase performance, sýnesis&trade; Lite for Suricata takes advantage of the caching and queueing features available in many of the Logstash plugins. These features increase the consumption of the JVM heap. Additionally the size of the IP reputation dictionary `ip_rep_basic.yml` can also increase heap usage. The JVM heap space used by Logstash is configured in `jvm.options`. It is recommended that Logstash be given 4GB of JVM heap. This is configured in `jvm.options` as follows:

```text
-Xms4g
-Xmx4g
```

### 2. Add and Update Required Logstash plugins

Ensure that all Logstash plugins are up to date by executing the following command.

```shell
LS_HOME/bin/logstash-plugin update logstash-filter-dns
```

### 3. Copy the pipeline files to the Logstash configuration path.

There are four sets of configuration files provided within the `logstash/synlite_suricata` folder:

```text
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
SYNLITE_SURICATA_TEMPLATE_PATH | The path to where index templates are located | /etc/logstash/synlite_suricata/templates
SYNLITE_SURICATA_GEOIP_DB_PATH | The path where the GeoIP DBs are located | /etc/logstash/synlite_suricata/geoipdbs

### 4. Setup environment variable helper files

Rather than directly editing the pipeline configuration files for your environment, environment variables are used to provide a single location for most configuration options. These environment variables will be referred to in the remaining instructions. A [reference](#environment-variable-reference) of all environment variables can be found [here](#environment-variable-reference).

Depending on your environment there may be many ways to define environment variables. The files `profile.d/synlite_suricata.sh` and `logstash.service.d/synlite_suricata.conf` are provided to help you with this setup.

Recent versions of both RedHat/CentOS and Ubuntu use systemd to start background processes. When deploying sýnesis&trade; Lite for Suricata on a host where Logstash will be managed by systemd, copy `logstash.service.d/synlite_suricata.conf` to `/etc/systemd/system/logstash.service.d/synlite_suricata.conf`. Any configuration changes can then be made by editing this file.

> Remember that for your changes to take effect, you must issue the command `sudo systemctl daemon-reload`.

### 5. Add the sýnesis&trade; Lite for Suricata pipeline to pipelines.yml

Logstash 6.0 introduced the ability to run multiple pipelines from a single Logstash instance. The `pipelines.yml` file is where these pipelines are configured. While a single pipeline can be specified directly in `logstash.yml`, it is a good practice to use `pipelines.yml` for consistency across environments.

Edit `pipelines.yml` (usually located at `/etc/logstash/pipelines.yml`) and add the sýnesis&trade; Lite for Suricata pipeline (adjust the path as necessary).

```text
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

```text
tail -f /var/log/logstash/logstash-plain.log
```

Logstash takes a little time to start... BE PATIENT!

Logstash setup is now complete. If you are receiving data from Filebeat, you should have both `suricata-` and `suricata_stats-` daily indices in Elasticsearch.

## Setting up Kibana

The vizualizations and dashboards can be loaded into Kibana by importing the `synlite_suricata.kibana.7.1.x.json` file from within the Kibana UI. This is done in the Kibana `Management` app under `Saved Objects`.

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

![suricata_alerts_overview](https://user-images.githubusercontent.com/10326954/58702438-1d3f2c80-83a6-11e9-8bc0-9ad13fb8fad4.png)

### Alerts - Messages

![suricata_alerts_messages](https://user-images.githubusercontent.com/10326954/58702444-23350d80-83a6-11e9-87ab-d61392dfa4d6.png)

### Threats - Public Attackers

![suricata_threat_pub_attack](https://user-images.githubusercontent.com/10326954/58702567-64c5b880-83a6-11e9-9e3f-d2c850bbcfeb.png)

### Threats - At-Risk Servers

![suricata_threat_risk_servers](https://user-images.githubusercontent.com/10326954/58702573-67281280-83a6-11e9-8378-e8de1f4f5ad1.png)

### Threats - At-Risk Services

![suricata_threat_risk_services](https://user-images.githubusercontent.com/10326954/58702578-6a230300-83a6-11e9-94b0-2319c8e4715e.png)

### Threats - High-Risk Clients

![suricata_threat_risk_clients](https://user-images.githubusercontent.com/10326954/58702582-6c855d00-83a6-11e9-957b-1305cd0d6d79.png)

### Flows - Overview

![suricata_flows_overview](https://user-images.githubusercontent.com/10326954/58702448-262ffe00-83a6-11e9-9a47-92b68ca54727.png)

### Flows - Top Talkers

![suricata_flows_talkers](https://user-images.githubusercontent.com/10326954/58702453-29c38500-83a6-11e9-929b-b913f89fca51.png)

### Flows - Top Services

![suricata_flows_services](https://user-images.githubusercontent.com/10326954/58702454-2cbe7580-83a6-11e9-84c2-dca23084e946.png)

### Flows - Sankey

![suricata_flows_sankey](https://user-images.githubusercontent.com/10326954/58702461-2fb96600-83a6-11e9-8055-c6b9dd6c267a.png)

### Flows - Geo IP

![suricata_flows_geo](https://user-images.githubusercontent.com/10326954/58702468-334ced00-83a6-11e9-9980-4353d1198364.png)

### Flows - Messages

![suricata_flows_messages](https://user-images.githubusercontent.com/10326954/58702476-36e07400-83a6-11e9-84ba-8f870cd85c19.png)

### HTTP - Overview

![suricata_http_overview](https://user-images.githubusercontent.com/10326954/58702477-39db6480-83a6-11e9-9dc4-482658bec364.png)

### HTTP - Messages

![suricata_http_messages](https://user-images.githubusercontent.com/10326954/58702484-3cd65500-83a6-11e9-9e48-7a37413ca2af.png)

### DNS - Overview

![suricata_dns_overview](https://user-images.githubusercontent.com/10326954/58702492-3fd14580-83a6-11e9-994d-5fac29edec81.png)

### DNS - Messages

![suricata_dns_messages](https://user-images.githubusercontent.com/10326954/58702495-42cc3600-83a6-11e9-9f66-a4a09600e904.png)

### SSH - Overview

![suricata_ssh_overview](https://user-images.githubusercontent.com/10326954/58702499-45c72680-83a6-11e9-884c-0ff2e998c764.png)

### SSH - Messages

![suricata_ssh_messages](https://user-images.githubusercontent.com/10326954/58702506-48c21700-83a6-11e9-88aa-eba7ad875e0c.png)

### TLS - Overview

![suricata_tls_overview](https://user-images.githubusercontent.com/10326954/58702510-4c559e00-83a6-11e9-97ab-3ec43bc398de.png)

### TLS - Messages

![suricata_tls_messages](https://user-images.githubusercontent.com/10326954/58702514-4f508e80-83a6-11e9-9c16-1f3d3d304690.png)

### SMB - Overview

![suricata_smb_overview](https://user-images.githubusercontent.com/10326954/58702531-524b7f00-83a6-11e9-905c-2635b504e1a2.png)

### SMB - Messages

![suricata_smb_messages](https://user-images.githubusercontent.com/10326954/58702544-55df0600-83a6-11e9-8743-4cfa83d2c57c.png)

### NFS - Overview

![suricata_nfs_overview](https://user-images.githubusercontent.com/10326954/58702548-59728d00-83a6-11e9-9540-46140f14ef48.png)

### NFS - Messages

![suricata_nfs_messages](https://user-images.githubusercontent.com/10326954/58702553-5c6d7d80-83a6-11e9-9ada-417a209e2887.png)

### Raw Logs

![suricata_raw_logs](https://user-images.githubusercontent.com/10326954/58702560-5ecfd780-83a6-11e9-80a3-c181046cbe87.png)

### Statistics

![suricata_stats](https://user-images.githubusercontent.com/10326954/58702564-61cac800-83a6-11e9-90a8-c75e5f4efedc.png)

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

![timepicker:quickRanges](https://user-images.githubusercontent.com/10326954/57178139-9a8d8500-6e6c-11e9-8539-db61a81b321b.png)

```text
[
  {
    "from": "now-15m",
    "to": "now",
    "display": "Last 15 minutes"
  },
  {
    "from": "now-30m",
    "to": "now",
    "display": "Last 30 minutes"
  },
  {
    "from": "now-1h",
    "to": "now",
    "display": "Last 1 hour"
  },
  {
    "from": "now-2h",
    "to": "now",
    "display": "Last 2 hours"
  },
  {
    "from": "now-4h",
    "to": "now",
    "display": "Last 4 hours"
  },
  {
    "from": "now-12h",
    "to": "now",
    "display": "Last 12 hours"
  },
  {
    "from": "now-24h",
    "to": "now",
    "display": "Last 24 hours"
  },
  {
    "from": "now-48h",
    "to": "now",
    "display": "Last 48 hours"
  },
  {
    "from": "now-7d",
    "to": "now",
    "display": "Last 7 days"
  },
  {
    "from": "now-30d",
    "to": "now",
    "display": "Last 30 days"
  },
  {
    "from": "now-60d",
    "to": "now",
    "display": "Last 60 days"
  },
  {
    "from": "now-90d",
    "to": "now",
    "display": "Last 90 days"
  },
  {
    "from": "now/d",
    "to": "now/d",
    "display": "Today"
  },
  {
    "from": "now/w",
    "to": "now/w",
    "display": "This week"
  },
  {
    "from": "now/M",
    "to": "now/M",
    "display": "This month"
  },
  {
    "from": "now/d",
    "to": "now",
    "display": "Today so far"
  },
  {
    "from": "now/w",
    "to": "now",
    "display": "Week to date"
  },
  {
    "from": "now/M",
    "to": "now",
    "display": "Month to date"
  }
]
```

# Attribution

This product includes GeoLite data created by MaxMind, available from (http://www.maxmind.com)
