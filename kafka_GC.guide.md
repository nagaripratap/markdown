# Guide: Visualizing Kafka GC Logs in Elasticsearch & Kibana

To visualize Garbage Collection (GC) logs as graphs, you must transform unstructured text logs into structured JSON data. For Kafka running on Kubernetes, the most efficient path is using **Filebeat** with an **Elasticsearch Ingest Pipeline** or **Logstash**.

---

## Phase 1: Data Parsing (The "Grok" Step)

A typical GC log entry looks like this:
`[2023-10-01T10:00:00.123+0000][12345] [info][gc] GC(10) Pause Young (Normal) (G1 Evacuation Pause) 100M->50M(512M) 10.123ms`

The goal is to extract the following variables:
*   **gc_type:** (e.g., Young, Full, Remark)
*   **heap_before:** 100
*   **heap_after:** 50
*   **gc_duration_ms:** 10.123

---

## Phase 2: Implementation Paths

### Option A: Filebeat "G1GC" Module (Easiest)
If you are using the **G1 Garbage Collector** (standard for Confluent/Kafka), Filebeat has a built-in module. 

1. **Enable the module** in your Filebeat configuration:
```yaml
filebeat.modules:
  - module: elasticsearch
    gc:
      enabled: true
      var.paths: ["/var/log/confluent/gc.log*"]
Use code with caution.

Deploy Filebeat as a DaemonSet. It will automatically parse and ship the data.
Option B: Logstash Pipeline (Most Flexible)
Use this if you have a custom log format or require advanced filtering.
ruby
filter {
  if [log][file][path] =~ "gc.log" {
    grok {
      match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\].*? %{NUMBER:heap_before:int}M->%{NUMBER:heap_after:int}M\(%{NUMBER:heap_total:int}M\) %{NUMBER:duration:float}ms" }
    }
  }
}
Use code with caution.

Phase 3: Kibana Visualization
Once the data is indexed, follow these steps:
Create Index Pattern: Navigate to Stack Management > Index Patterns and add filebeat-* or logstash-*.
Create Visualization: Go to Visualize Library > Create Visualization > Lens.
Recommended Dashboard Graphs
Graph Type	X-Axis	Y-Axis	Purpose
Area Chart	@timestamp	heap_before & heap_after	Tracks the "sawtooth" pattern and memory leaks.
Bar Chart	@timestamp	duration (Max/Avg)	Identifies "Stop the World" pauses causing latency.
Metric Card	N/A	Count of GC types	Monitors the frequency of dangerous Full GC events.
Alternative: The "GCViewer" Shortcut
If you need immediate results without setting up the Elastic Stack:
Extract the log to Windows:
powershell
kubectl cp -n confluent kafka-0:/path/to/gc.log ./gc.log -c kafka
Use code with caution.

Use GCViewer:
Download the JAR file.
Open your gc.log.

