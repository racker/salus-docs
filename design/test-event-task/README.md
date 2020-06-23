
# Testing Event Tasks Feature

## Problem statement

If users are setting up new monitors or new resources it can feel like a stab in the dark to create event tasks that evaluate as expected. The test-monitor feature helps to some extent since it reveals the measurement names and current values for the resource in question. What remains as an uncertainty is whether the threshold evaluation is correctly configured to produce the expected event level.

## Proposal Notes

- follows usage of MaaS' [`POST /entities/{entityId}/test-alarm`](https://developer.rackspace.com/docs/rackspace-monitoring/v1/api-reference/alarms-operations/#test-an-alarm)
    - POST a two part body
        - a measurement result from monitor management's [`POST /tenant/{tenantId}/test-monitor`](https://github.com/racker/salus-telemetry-monitor-management/blob/master/src/main/java/com/rackspace/salus/monitor_management/web/controller/TestMonitorApiController.java#L52)
        - event task definition to test
    - API call will be long-running, blocking using Spring MVC's `CompletableFuture` async handling
    - CompletableFuture's `orTimeout` mechanism will be used [like test-monitor implementation](https://github.com/racker/salus-telemetry-monitor-management/blob/master/src/main/java/com/rackspace/salus/monitor_management/services/TestMonitorService.java#L141-L142) to ensure that failure to see the resulting event will be handled and cleanup performed
- can leverage existing kapacitor instances due to special `.from()` designation, described below
- Event Engine Management (EEM) handling API request will select one of the kapacitor instances using existing discovery logic
    - Tenant ID from API request
    - Resource ID and measurement name from given metrics content to test
- test task is assigned a temporary UUID (`test-task-id`) for correlation with event results
- test task is given a special `.from()` designation to specifically limit the processing scope to the one task and the one metric fed into it
    - database = `test-tasks`
    - measurement = `{test-task-id}`
- test-task definition would be a "truncated" version of the given event task:
    - no use of state count/duration
    - no use of `stateChangesOnly`, which will ensure one metric in, one event out
    - ...basically eliminate any use of stateful/aggregated processing to ensure immediate response to ingested metric. 
      > The downside is that stateful behavior is not "tested"; however, it does still exercise eval expressions and threshold expressions
- EEM would use same InfluxDB ingest approach as event-engine-ingest where [it writes a metric to the designated kapacitor instance](https://github.com/racker/salus-event-engine-ingest/blob/master/src/main/java/com/rackspace/salus/event/ingest/services/IngestService.java#L131-L135)
- would involve provisioning [new event handler](https://docs.influxdata.com/kapacitor/v1.5/event_handlers/kafka/#example-handler-file) and [topic](https://docs.influxdata.com/kapacitor/v1.5/working/using_alert_topics/, such as "test-events"
    - provisioned kafka event handler would produce to a new "test-events results" topic
- EEM instances would all subscribe to "test-events results" topic
    - only act upon results that contain a `test-task-id` that they have pending
    - resolve the pending API request with the resulting event content, [similar to test-monitor implementation](https://github.com/racker/salus-telemetry-monitor-management/blob/master/src/main/java/com/rackspace/salus/monitor_management/services/TestMonitorService.java#L207)
    - delete the kapacitor task    
- An API can be implemented by Public API that chains a call to test-monitor with a call to test-event-task, [similar to monitoring.rackspace.net's](https://github.rackspace.com/monitoring-integration/rackspace-monitoring-gui/blob/master/static/js/views/checks_overview.js#L202-L255)  

## Example TICKscript for a tested event task

```tickscript
// database defined via task creation API
stream
  |from()
    .measurement('{{testTaskId}}')
  |eval(lambda: "error_count" / "total_count")
    .as('error_percent')
  |alert()
    .id('{{testTaskId}}')
    .crit(lambda: "error_precent" > 0.8)
    .topic('test-events')
```

## Sequence diagram

![](http://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/Rackspace-Segment-Support/salus-docs-internal/master/proposals/test-event-task/sequence.puml)