# kubernetes-event-exporter

> This tool is presented at [KubeCon 2019 San Diego](https://kccncna19.sched.com/event/6aa61eca397e4ff2bdbb2845e5aebb81).

This tool allows exporting the often missed Kubernetes events to various outputs so that  they can be used for 
observability or alerting purposes. You won't believe what you are missing. 

## Deployment

Head on to `deploy/` folder and apply the YAMLs in the given filename order. Do not forget to modify the 
`deploy/01-config.yaml` file to your configuration needs. The additional information for configuration is as follows:

## Configuration

Configuration is done via a YAML file, when run in Kubernetes, it's in ConfigMap. The tool watches all the events and
user has to option to filter out some events, according to their properties. Critical events can be routed to alerting
tools such as Opsgenie, or all events can be dumped to an Elasticsearch instance. You can use namespaces, labels on
the related object to route some Pod related events to owners via Slack. The final routing is a tree which allows
flexibility. It generally looks like following:

```yaml
route:
  # Main route
  routes: 
    # This route allows dumping all events because it has no fields to match and no drop rules.
    - match:
      - receiver: dump
    # This starts another route, drops all the events in *test* namespaces and Normal events 
    # for capturing critical events 
    - drop:
      - namespace: "*test*"
      - type: "Normal"
      match: 
      - receiver: "critical-events-queue"
    # This a final route for user messages
    - match:
        kind: "Pod|Deployment|ReplicaSet"
        labels:
          version: "dev"    
        receiver: "slack"    
receivers:
# See below for configuring the receivers
```

* A `match` rule is exclusive, all conditions must be matched to the event.
* During processing a route, `drop` rules are executed first to filter out events.
* The `match` rules in a route are independent of each other. If an event matches a rule, it goes down it's subtree.  
* If all the `match` rules are matched, the event is passed to the `receiver`.
* A route can have many sub-routes, forming a tree. 
* Routing starts from the root route.


### Opsgenie

[Opsgenie](https://www.opsgenie.com) is an alerting and on-call management tool. kubernetes-event-exporter can push to 
events to Opsgenie so that you can notify the on-call when something critical happens. Alerting should be precise and 
actionable, so you should carefully design what kind of alerts you would like in Opsgenie. A good starting point might be
filtering out Normal type of events, while some additional filtering can help. Below is an example configuration.

```yaml
# ...
receivers:
  - name: "alerts"
    opsgenie:
      apiKey: xxx
      priority: "P3"
      message: "Event {{ .Reason }} for {{ .InvolvedObject.Namespace }}/{{ .InvolvedObject.Name }} on K8s cluster"
      alias: "{{ .UID }}"
      description: "<pre>{{ toPrettyJson . }}</pre>"
      tags:
        - "event"
        - "{{ .Reason }}"
        - "{{ .InvolvedObject.Kind }}"
        - "{{ .InvolvedObject.Name }}"
```

### Webhooks/HTTP

Webhooks are te easiest way of integrating this tool to external systems. It allows templating & custom headers
which allows you to push events to many possible sources out there. See [Customizing Payload] for more information.

```yaml
# ...
receivers:
  - name: "alerts"
    webhook:
      endpoint: "https://my-super-secret-service.com"
      headers:
        X-API-KEY: "123"
        User-Agent: kube-event-exporter 1.0
      layout: # Optional
```

### Elasticsearch

[Elasticsearch](https://www.elastic.co/) is a full-text, distributed search engine which can also do powerful aggregations.
You may decide to push all events to Elasticsearch and do some interesting queries over time to find out which images
are pulled, how often pod schedules happen etc. You can [watch the presentation](https://static.sched.com/hosted_files/kccncna19/d0/Exporting%20K8s%20Events.pdf) 
in Kubecon to see what else you can do with aggregation and reporting.

```yaml
# ...
receivers:
  - name: "dump"
    elasticsearch:
      hosts:
      - http://localhost:9200
      index: kube-events            
      username: # optional
      password: # optional
      cloudID: # optional
      apiKey: # optional
      # If set to true, it allows updating the same document in ES (might be useful handling count)
      useEventID: true|false
  	  layout: # Optional
```

### Slack

Slack is a cloud-based instant messaging platform where many people use it for integrations and getting notified by 
software such as Jira, Opsgenie, Google Calendar etc. and even some implement ChatOps on it. This tool also allows
exporting events to Slack channels or direct messages to persons. If your objects in Kubernetes, such as Pods, Deployments
have real owners, you can opt-in to notify them via important events by using the labels of the objects. If a Pod sandbox
changes and it's restarted, or it cannot find the Docker image, you can immediately notify the owner. 

```yaml
# ...
receivers:
  - name: "slack"
    slack:
      token: YOUR-API-TOKEN-HERE
      channel: "@{{ .InvolvedObject.Labels.owner }}"
      message: "{{ .Message}}"
      fields:
        namespace: "{{ .Namespace }}"
        reason: "{{ .Reason }}"
        object: "{{ .Namespace }}"
        
```

### Kinesis

Kinesis is an AWS service allows to collect high throughput messages and allow it to be used in stream processing.

```yaml
# ...
receivers:
  - name: "kinesis"
    kineis:
      streamName: "events-pipeline"
      region: us-west-2
      layout: # Optional
```

### SNS

SNS is an AWS service for highly durable pub/sub messaging system.

```yaml
# ...
receivers:
  - name: "sns"
    sns:
      topicARN: "arn:aws:sns:us-east-1:1234567890123456:mytopic"
      region: "us-west-2"
      layout: # Optional
```

### SQS

SQS is an AWS service for message queuing that allows high throughput messaging.

```yaml
# ...
receivers:
  - name: "file"
    sqs:
      queueName: "/tmp/dump"
      region: us-west-2
      layout: # Optional
```

### File

For some debugging purposes, you might want to push the events to files. Or you can already have a logging tool that can
ingest these files and it might be a good idea to just use plain old school files as an integration point.

```yaml
# ...
receivers:
  - name: "file"
    file:
      path: "/tmp/dump"
      layout: # Optional
```

### Customizing Payload

Some receivers allow customizing the payload. This can be useful to integrate it to external systems that require 
the data be in some format. It is designed to reduce the need for code writing. It allows mapping an event using
Go templates, with [sprig](github.com/Masterminds/sprig) library additions. It supports a recursive map definition,
so that you can create virtually any kind of JSON to be pushed to a webhook, a Kinesis stream, SQS queue etc.

```yaml
# ...
receivers:
  - name: pipe
    kinesis:
      region: us-west-2
      streamName: event-pipeline
      layout:
        region: "us-west-2"
        eventType: "kubernetes-event"
        createdAt: "{{ .GetTimestampMs }}"
        details:
          message: "{{ .Message }}"
          reason: "{{ .Reason }}"
          type: "{{ .Type }}"
          count: "{{ .Count }}"
          kind: "{{ .InvolvedObject.Kind }}"
          name: "{{ .InvolvedObject.Name }}"
          namespace: "{{ .Namespace }}"
          component: "{{ .Source.Component }}"
          host: "{{ .Source.Host }}"
          labels: "{{ toJson .InvolvedObject.Labels}}" 
```

### Planned Receivers

- Big Query
- PubSub
- AWS Firehose
- Splunk
- Kafka
- Redis
- Logstash
- Console