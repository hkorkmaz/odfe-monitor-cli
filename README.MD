### OpenDistro for Elasticsearch Alerting CLI

odfe-monitor-cli enables you to manage and organize [Elasticsearch Alerting monitors](https://opendistro.github.io/for-elasticsearch-docs/docs/alerting/) through YAML file.

## Why would I want to manage my monitors in YAML files?

* You can store your monitors in source control so you can track changes, and perform code reviews.
* Enable through automated pipelines.
* It is very easy to accidentally make changes to monitors on UI or through API. You can always refer back on this and update existing monitors.
* You may have multiple stages / regions and would like have monitors running on each stage / region.

## Installation

From source:

```
$ go get github.com/emretanriverdi/odfe-monitor-cli/
```

From binary to `./bin/odfe-monitor-cli`:

```
$ curl -sfL https://raw.githubusercontent.com/emretanriverdi/odfe-monitor-cli/master/godownloader.sh | bash
```

From binary to `/usr/local/bin/odfe-monitor-cli`:

```
$ curl -sfL https://raw.githubusercontent.com/emretanriverdi/odfe-monitor-cli/master/godownloader.sh | bash -s -- -b /usr/local/bin
```

## Getting Started


### Sync

```
odfe-monitor-cli sync --destinations
```

This command will create auto-generated destinations file with names and destinationId , so that they're easy to refer inside monitors.

```
odfe-monitor-cli sync --monitors
```

This command will create `monitors.yaml` and write remote monitors to local files and you can start off managing your monitors.

### Diff

```
odfe-monitor-cli diff
```

This command will show difference between remote and local monitors.

### Push

```
odfe-monitor-cli -e https://localhost:9200 -u admin -p admin -r your/yaml/files/ push --submit
```

Publish local monitors to remote Elasticsearch cluster:

- will run and validate updated/new monitors.
- will create new monitors and update existing monitors if `--submit` flag is added. This flag could override your changes if you edited an existing monitor in Kibana (or by any other way).
- will not delete any monitors from remote. You must provide `--delete` along with `--submit` to delete all untracked monitors. Be careful---this can't be undone.

Sample monitor

```yaml
- name: 'Sample Alerting monitor'
  type: 'monitor'
  schedule:
    period:
      interval: 10
      unit: MINUTES
  enabled: true
  inputs:
    - search:
        indices:
          - log*
        query: # This block should be valid Elasticsearch query
          size: 0
          query:
            match_all: {
              boost: 1.0
            }
  triggers:
    - name: '500'
      severity: '2'
      condition: | #This is how you can create multiline strings
        // Performs some crude custom scoring and returns true if that score exceeds a certain value
        int score = 0;
        for (int i = 0; i < ctx.results[0].hits.hits.length; i++) {
          // Weighs 500 errors 10 times as heavily as 503 errors
          if (ctx.results[0].hits.hits[i]._source.http_status_code == "500") {
            score += 10;
          } else if (ctx.results[0].hits.hits[i]._source.http_status_code == "503") {
            score += 1;
          }
        }
        if (score > 99) {
          return true;
        } else {
          return true;
        }
      actions:
        - name: Sample Action
          destinationId: test_my_destination #This destination should be available in destinations.yaml file otherwise it will throw an error.
          subject: 'There is an error'
          message: |
            Monitor {{ctx.monitor.name}} just entered an alert state. Please investigate the issue.
            - Trigger: {{ctx.trigger.name}}
            - Severity: {{ctx.trigger.severity}}
            - Period start: {{ctx.periodStart}}
            - Period end: {{ctx.periodEnd}}
```
