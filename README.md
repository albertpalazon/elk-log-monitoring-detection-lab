# ELK Data Pipeline and Log Monitoring Lab

## Overview

This project demonstrates the design and configuration of a complete ELK pipeline for two practical use cases:

1. Large dataset ingestion and transformation  
2. Structured Linux syslog monitoring and visualization  

The objective was to focus on data structuring, index design, and security-oriented visibility rather than simple stack installation.

---

## Architecture

The pipeline processes two different data sources:

- A large CSV dataset (≈160MB)
- Linux system logs (`/var/log/syslog`)

Data flow:

Data Source → Logstash → Elasticsearch → Kibana

<img width="1088" height="391" alt="image" src="https://github.com/user-attachments/assets/f1c42dfc-7e39-422d-9b19-0834f11332d6" />


---

# Part 1 – Large Dataset Ingestion (CSV ETL)

## Objective

Simulate ingestion of a large dataset and transform it into a structured, queryable index with geospatial visualization capabilities.

## Logstash ETL Configuration

Configuration file: `logstash/csv-cities-etl.conf`

```conf
input {
  file {
    path => "/path/worldcitiespop.csv"
    start_position => beginning
  }
}

filter {
  csv {
    columns => ["country","city","accentCity","region","population"]
  }

  mutate {
    add_field => { "location" => "%{column6},%{column7}" }

    remove_field => [
      "message",
      "@version",
      "@timestamp",
      "host",
      "path",
      "column6",
      "column7"
    ]
  }
}

output {
  stdout { }

  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "ciudades"
  }
}
```

## Key Technical Decisions

- Created a custom index before ingestion to control field mapping.
- Removed default Logstash metadata fields to reduce unnecessary index noise.
- Built a `location` field compatible with Kibana Maps.
- Tested ingestion performance using a large dataset (~160MB).
- Avoided uncontrolled dynamic mapping in Elasticsearch.

## Kibana Map Visualization

<img width="1160" height="527" alt="csv-map" src="https://github.com/user-attachments/assets/9ca2d97e-0d6a-4c60-a391-bcbffa7acd85" />

---

# Part 2 – Linux Syslog Monitoring

## Objective

Parse raw syslog entries into structured fields suitable for monitoring and behavioral analysis.

## Logstash Monitoring Pipeline

Configuration file: `logstash/syslog-pipeline.conf`

```conf
input {
  file {
    path => "/var/log/syslog"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    ignore_older => 0
  }
}

filter {
  grok {
    match => {
      "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\\[%{POSINT:syslog_pid}\\])?: %{GREEDYDATA:syslog_message}"
    }
    overwrite => ["message"]
  }

  date {
    match => ["syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss"]
  }

  metrics {
    meter => "events"
    add_tag => "metric"
  }
}

output {
  stdout {
    codec => rubydebug
  }

  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "syslogs"
  }

  if "metric" in [tags] {
    stdout {
      codec => line {
        format => "Total events processed: %{[events][count]}"
      }
    }
  }
}
```

## Extracted Fields

- `syslog_timestamp`
- `syslog_hostname`
- `syslog_program`
- `syslog_pid`
- `syslog_message`

## Parsed Events in Kibana Discover

<img width="1035" height="592" alt="syslog-discover" src="https://github.com/user-attachments/assets/1f171f81-6990-4e7c-9fe1-38ce17e26605" />

---

# Dashboards and Visualizations

The final dashboard includes:

- Top processes by log frequency
- Event volume over time
- PID distribution
- Keyword-based filtering
- Total event counter

## Full Dashboard Overview

<img width="886" height="459" alt="image" src="https://github.com/user-attachments/assets/5e36f5b6-b1f0-4196-9bb5-55ddcce43ebb" />
<img width="883" height="321" alt="image" src="https://github.com/user-attachments/assets/4aa78770-41e0-4cbb-b74a-301c69bf2c1e" />



## Top Programs by Log Frequency

<img width="886" height="695" alt="image" src="https://github.com/user-attachments/assets/8cf5cd93-740c-4e42-a835-52ba5affef0d" />

<img width="886" height="473" alt="image" src="https://github.com/user-attachments/assets/9e70cf33-699f-4470-82f8-6937519f2786" />



## Events Over Time

<img width="775" height="394" alt="image" src="https://github.com/user-attachments/assets/ab492386-74e2-459e-a942-dc47e3c155e0" />

---

## How to Run (Lab Setup)

### Requirements

- Elasticsearch
- Logstash
- Kibana
- Java
- Linux environment

Ensure version compatibility between ELK components.

### Start Services

```bash
sudo systemctl start elasticsearch
sudo systemctl start kibana
```

### Run CSV ETL Pipeline

```bash
/usr/share/logstash/bin/logstash -f logstash/csv-cities-etl.conf
```

### Run Syslog Monitoring Pipeline

```bash
/usr/share/logstash/bin/logstash -f logstash/syslog-pipeline.conf
```

### Verification

In Kibana:

- Create index pattern `ciudades`
- Create index pattern `syslogs`
- Validate structured fields in Discover
- Build visualizations

Note: `sincedb_path => "/dev/null"` is used for lab reprocessing.

---

# Security Perspective

This project demonstrates:

- Log pipeline architecture design
- Field normalization and structured parsing
- Index management
- Visualization strategy for monitoring
- Baseline behavior analysis
- Process-based visibility

Understanding how logs are structured and visualized is fundamental for both defensive monitoring and understanding detection logic from an offensive perspective.

---

# Skills Demonstrated

- Elasticsearch
- Logstash
- Grok parsing
- Kibana visualization
- ETL pipeline design
- Log structuring
- Monitoring fundamentals

---

# Future Improvements

- Threshold-based alerts
- Failed authentication monitoring
- Suspicious process detection rules
- Sigma rule mapping
- Threat intelligence enrichment
