
# Agent-Version Monitor Translation

## Background

Salus currently does a monitor conversion step in `MonitorConversionService` within `monitor-management`, which only tasked with converting our own type-safe Java data classes that define the parameters of local and remote monitors into an agent agnostic, JSON content blob. The JSON content blob is stored as a column of the persisted monitor entity. (MySQL technically can further query into JSON columns, but we don't yet make use of that feature.) That JSON content blob is also what serves as a "template" for the rendered JSON content that is persisted into bound monitors.

Today the Salus system is heavily leaning on telegraf as the primary agent managed by the Envoy. As such, the configuration file processing that Envoy does for telegraf makes a purposely simple implementation assumption that the JSON content blob can be field-for-field converted to TOML.

That becomes a risky assumption since a newer version of telegraf may decide to completely remove a deprecated field. We'd prefer insulate Salus end users from those deprecations especially since some monitor definitions could be baked into business level rules maintained by external organizations. All in all, the one-to-one mapping of Java data model at the REST API, JSON content blob in the monitor enties, and then to the final telegraf TOML configuration forces the structure of each plugin/details to be dictated by the final agent requirements.

## Design Proposal

> A [sequence diagram is provided](agent-version-monitor-translation-seq.puml), which might be more helpful to view after reading the proposal below

A given tenant might have several versions of telegraf in use across their fleet depending on how granular the agent install selectors were chosen. Therefore, assumptions cannot be made at a tenant-level about what version of a particular agent is in use. As such, monitor translation (which is different from "conversion", described above) is not practical at the time of monitor creation, updates, and bindings since the agent-install lifecycle transitions independently of the monitor lifecycle. 

Envoy is the only part of the current system that has a concrete need to comprehend telegraf and its configuration format. Logically that would be the place where the content blob should be translated into a version specific form of telegraf configuration; however, we need the Envoy to be a slowly changing codebase. Deploying new versions of Envoy across a large set of resources across all tenants is not expected nor desired to be a common activity.

Currently, the Ambassador has indirect knowledge of what agents and what version of those agents are installed on a currently attached Envoy since it is the one that forms an agent install instruction while consuming an agent install change event; however, it doesn't currently persist those details after the instruction is sent. The Ambassador could be enhanced to track the agent types and agent versions in the `EnvoyRegistry` much like the bound monitor tracking. It could then use that tracked knowledge to facilitate agent/version specific transformations of the monitor content while building the respective config instruction. So, Ambassador seems to be an ideal step in the [monitor transformation path](../design/flow/monitor-transformation-path.puml) to initiate the agent-version specific translation of monitor content.

Similar to the comment about not wanting to modify the Envoy codebase frequently we should likewise try to avoid writing hardcoded logic that translates the monitor JSON content blobs into an agent-version specific configuration blob. For example, if it were hardcoded into the Ambassadors, then each new version of telegraf introduced into the system would require a re-deployment of Ambassadors. While that is much more feasible than redeploying Envoys, it is not a scalable/prudent reason for continuous re-deployments.

So, rather than hardcoding the translation logic, a "table driven" approach should be used for the agent-version monitor content translation. Specifically, the table of translation operations should be stored in SQL such that they can be persisted and modified on the active system via an admin API. Translation operations would need to be query-able by either just agent type or agent type plus agent version. That would allow for generally applicable agent level translations to be combined with type and version specific deprecations.

Some translation operations known from looking at deprecation notices in telegraf 1.12:
- renaming of a field. _This is also useful to avoid exposing inconsistent word-break styles in plugins, such as the Telegraf CPU plugin's `percpu` vs `collect_cpu_time` fields._
    - `parse_data_dog_tags` of [statsd](https://github.com/influxdata/telegraf/blob/72c2ac964875202f1c7d5d3e751e1464af04d719/plugins/inputs/statsd/README.md)
- converting separate host and port fields into a URL field
    - `url` of [activemq](https://github.com/influxdata/telegraf/blob/130c5c5f12f85a62df7841f19632d3d8d2286a7b/plugins/inputs/activemq/README.md)
- converting a scalar string field into another field as array of strings
    - `directory` of [filecount](https://github.com/influxdata/telegraf/blob/9740e956ca3176774ae23a312e5d498f4c20ef5a/plugins/inputs/filecount/README.md)
    - `url` of [amqp_consumer](https://github.com/influxdata/telegraf/blob/e141518cf05177ef0f6a8efb3608ec0c1acbf49f/plugins/inputs/amqp_consumer/README.md)

Additional operations we would want:
- restrict allowed values of a field, such as only allowing "localhost" for SNMP input plugin
- populate fixed values
- mapping `MonitorType` name to telegraf input plugin name

**NOTE** while investigating deprecated telegraf config fields it was observed that metric fields are also deprecated at times. The occurrence of those seems to be very small, but non-zero. This is something that should be further considered at a later time.

Since each translation operation requires specific parameters and there will likely be future operations with varying requirements, the plan is encode and persist each operation's definition as a JSON object structure. Similar to `MonitorDetails` the operations can be a polymorphic data model using a `operation` field as a discriminator. Most of the operations would also require a `plugin` field to target the plugin configuration that requires that translation operation.

It is worth clarifying that each operation type would require a small amount of implementation -- this slightly contradicts the hardcoding avoidance stated earlier, but the alternative would require a heavier solution such as [jackson-jq](https://github.com/eiiches/jackson-jq) to fully script the operations. An implementation per operation type is a reasonable compromise since the addition of operation types would be infrequent.

The following are some **examples** of operation definitions in JSON form:

```json
{
  "operation": "rename_monitor_type",
  "monitor_type": "tls",
  "to_plugin": "x509_cert"
}
```

```json
{
  "operation": "rename_field",
  "plugin": "statsd",
  "from": "parse_data_dog_tags",
  "to": "datadog_extensions"
}
```

```json
{
  "operation": "convert_host_and_port_to_url",
  "plugin": "activemq",
  "host_field": "server",
  "port_field": "port",
  "to": "url",
  "scheme": "http"
}
```

The sections above decided agent-version translation should occur when the the Ambassador is forming a configuration instruction; however, it wasn't yet decided where the translation rules should be stored and what microservice should own the execution of those rules. It turns out that when the Ambassador handles a monitor bound event it queries Monitor Management for the bound monitor DTOs. The bound monitor **DTO** is already a flattened representation of a bound monitor entity and its associated monitor entity. The bound monitor DTO's content field is currently populated from the persisted, rendered content field of the bound monitor entity.

The current design proposes that when the Ambassador queries Monitor Management for the bound monitors, it can also indicate the agent type and version of the Envoy-Resource. The  processing of bound monitor DTOs can be enhanced to perform all applicable agent-version monitor translations. This would require changing `MonitorApiClient.getBoundMonitors` and the REST API invoked,  or introduced a new API operation focused on this particular combination of bound monitor retrieval and agent-version monitor translation. Ambassador is the called of `getBoundMonitors` so both ideas should probably be combined by renaming and extending the existing REST operation. 

The previous paragraph implies that Monitor Management becomes the microservice that owns management and persistence of the monitor translation operations. It also becomes the endpoint for admin APIs to manage the translation operations. The downside to this approach is that the Monitor Management microservice would need to take on even more responsibility and it already has many; however, at the broader system level, this approach avoids introducing any new dependencies, applications, or communication paths. Introductions to the bulk and complexity of the `MonitorManagement` Spring component could be minimized and a new Spring component can encapsulate a majority of the new translation logic.

With all of the pieces above, the Ambassador's processing of agent install events can be enhanced to handle potential re-translation of monitors previously bound to the attached envoy. That enhancement turns out to be quite simple since it can initiate the same processing flow as handling of a bound monitor event. Specifically, the same query for bound monitor DTOs would be invoked, this time with the newly installed agent type and version, the translated monitor content would come back, and the Ambassador's `EnvoyRegistry` can compute necessary configuration instructions as it does already.

## Open questions

What to do when an agent-version translation operation is CRUD'ed?
- Broadcast an event to be consumed by Ambassadors? Each Ambassador could look for its attached Envoys with the agent type and version referenced by the event, re-query for bound monitors, and process config instructions. This approach results in a multi-tenant, system-wide level of activity similar to policy management changes.
- Do nothing? Changes to agent-version operation definitions will most likely occur because we are about to introduce a new agent type or version that probably adds new plugin/monitor types. As such, previously configured monitors would not be affected by the change in translations. We'd purposely prepare for the new agent/version by modifying the translation operations, deploy the new code changes, and only when new monitors are created with the newly support agent/plugins/versions would the operations even be needed.

Should translation operation definitions be managed by policy management? At a very high level, they seem similar since they are system-wide definitions; however, the intention was not to override the operations per tenant or account type.  

## Alternate ideas and notes

_The following are random ideas and notes considered, but are only retained for discussion purposes._

One solution could be to introduce a step where Monitor Management queries the Agent Catalog Service or its repositories during content rendering of a bound monitor. Monitor Management would also need to consume agent install change events in order to re-render the content of bound monitors as the agent-version changes for a given envoy-resource. This has the downsides that a new repository retrieval would need to be needed during monitor binding and also introduces a dependency on the agent catalog data model.

An envoy-resource attach event, currently produced by the Ambassador, could include the indication of the installed version per agent type. With that information persisted as part of the `ResourceInfo`, the rendering of content for monitor binding could use those data points to facilitate monitor translation. As part of an Ambassador sending an agent install instruction, it could also produce a new event type, "attachment changed", which would include the new version of the installed agent type. Since Resource Management currently consumes the existing attach event, it follows that Resource Management would also consume this new event. In turn it would produce a resource event that Monitor Management could consume. Along with the existing resource change event handling the bound monitors can be re-evaluated and content re-rendered with the updated indicator of agent-version in use for that envoy-resource. This has the downside that it introduces more complexity into the already complex monitor binding logic and system-wide requires two new events and processing paths.

Introduce a new microservice that owns persistence and execution of monitor translation rules. It doesn't feel like a substantial enough reason to introduce a new microservice (code repo, devops burden, etc). Monitor translation is something that has to happen during the monitor lifecycle and/or agent installation lifecycle, so Agent Catalog Service, Ambassador, and Monitor Management are already situated in that flow.