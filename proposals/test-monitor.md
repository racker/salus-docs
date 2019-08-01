
# Use Case

As a user, I would like to see what values would be collected when I configure a monitor a certain way for one or more of my existing resources.

## Functional Expectations

- The operation would be initiated by a REST POST operation very similar to a monitor create operation
- The REST operation would block until the results are available or a timeout is reached. The timeout can be pre-determined by the system.
- When completed normally, the response of the operation would be one or more sets of metrics (see ["Variants"](#variants), below)
- When some resources complete normally, but some do not, the status code of the response should indicate a failure but the response body should contain the normal metrics contents for the resources that succeeded and error indications for those that failed.
    - When the agent execution fails, the response body should include as much detail about the failure as possible. The goal would be to assist the user with correcting their monitor configuration to be valid for the selected resource(s).
- When timed out, the status code and response body would indicate which aspect of the operation timed out, such as the agent execution or a system level timeout

## Variants

- User provides a single, specific resource ID
- User provides label selectors and expects only one selected resource to be tested. In turn, the user expects a set of metrics along with the selected resource's ID.
- User provides label selectors and expects all selected resources to be tested. In turn they would expect the results to be a set of metrics associated with each resource that was selected. 

## System Assumptions

- The user provides a configuration for a monitor that supports test operations. For example, telegraf supports one-off monitor configuration runs, but filebeat doesn't. So only monitor types powered by telegraf would support test operations.

# Design Proposal

The following proposal focuses on the first variant discussed [above](#variants):

> User provides a single, specific resource ID

The design of the other variants could roughly extrapolate from the proposal below, but adds just enough complexity that would dilute the core design of the current proposal.

## Existing Challenges

### Envoy initiates gRPC calls

There is not a mechanism for request/response operations towards Envoys since they aren't the binding end of the Envoy-Ambassador relationship. What's confusing is that the Ambassador already "sends" instructions to Envoys; however, it does that by placing an asynchronous response in original request's response stream. 

The good news is that the concept of an "instruction" was purposely generalized. The proposal below will discuss how this can be extended to support test monitors.

### Ambassador doesn't have a REST API

Similarly, Ambassadors so far do not serve requests, but rather only initiate requests towards other microservices. As such, there is not a natural mechanism to invoke a synchronous operation, such as a test monitor, on a given Ambassador. The lack of incoming request serving is intentional since the deployment of Ambassadors is meant to be fluid as the capacity of the system needs to vary. 

Ambassadors already consume events, such as bound monitors, so that becomes a natural system flow to leverage for the proposal below.

## Telegraf support

As mentioned in the Jira issue, telegraf comes with a command-line option that is perfect for testing monitors, `--test`, which is tersely described [here in the Telegraf documentation](https://github.com/influxdata/telegraf#run-a-single-telegraf-collection-outputing-metrics-to-stdout).

For this design, the proposal is to have the Envoy support test monitors by starting a very short-lived telegraf process that uses the `--test` command-line option. What will be left as an implementation detail is whether a small config file should be temporarily created or follow the pattern of the the `TelegrafRunner` to pass a config server URL that points back to the Envoy.

## Proposed System Flow

The following diagram shows the end-to-end, request-response flow of a test-monitor operation against an existing resource specified by the user.

[Diagram](test-monitor-specific-resource.puml)

The continuous activation lifeline of the PublicAPI emphasizes that the REST API operation initiated by the user will be a "long running" and blocking call until the test-monitor results come back (or a timeout expires, which isn't depicted in the diagram). Spring Web supports request mappings that return `CompletableFuture`, which activates asynchronous servlet support. The advantage is that the HTTP REST request remains active, but doesn't consume a thread of the MonitorMgmt's embedded web server. The same would need to be applied to the PublicAPI proxy of the REST operation.

In contrast, the break in the activation lifelines of MonitorMgmt and Ambassador indicate that each of those services will process the request asynchronously by temporarily storing the state of the request and concluding the state when the downstream service replies (or a timeout expires). It is expected that each would store that state in memory, which seems durable enough given how short a test-monitor should take to process. 

Speaking of request state, the initiating MonitorMgmt will generate a UUID to be used a correlation ID that can be used to correlate the test-monitor response details to the originating request. That correlation ID can serve as a storage key for holding the state of the request. That same correlation ID will be passed down to the Ambassador and the Envoy to enable the final correlation of the response back to the originating REST operation.

An important piece of information MonitorMgmt will need to store in memory is the `CompletableFuture` that ties back to the REST operation. MonitorMgmt can wrap the operation in a timeout by using [completeOnTimeout](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html#completeOnTimeout(T,long,java.util.concurrent.TimeUnit)), but then complete the future normally by using the [complete](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html#complete(T)) method. 

## Event and Instruction Structures

### Event: test-monitor request

- correlation ID
- tenant ID
- resource ID
- envoy ID
- bound monitor DTO

**Note:** the inclusion of the bound monitor DTO within the event is purposely in contrast to the prevailing convention of state-less events. The inclusion is proposed as a lesser of two complexities. As an alternative, MonitorMgmt could store the test `Monitor` and `BoundMonitor` in the database with a new field in each to indicate that they are "test" variants. The addition of the new discriminator field is trivial, but would have possible implications on all queries of `Monitor` and `BoundMonitor`. Queries would need to exclude the test instances to ensure concurrent resource events would not accidentally retrieve and bind with the test monitor. 

### Instruction: test-monitor

This is a new "EnvoyInstruction" message that will be added to the Telemetry Edge protobuf schema. It borrows heavily from the existing structure of the `EnvoyInstructionConfigure` since it needs to convey the same level of detail to configure the telegraf plugin.

- correlation ID
- monitor config JSON

### Post test-monitor results

This is request body of a new gRPC method of the `TelemetryAmbassador` service schema:

- correlation ID
- error message, if not fully successful
- `Metric`, specifically a `NameTagValueMetric`

### Event: test-monitor results
- correlation ID
- tenant ID
- resource ID
- error message, if not fully successful
- unified metrics object, if successful