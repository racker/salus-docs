Right now Salus does a monitor conversion step in `MonitorConversionService` within `monitor-management`. That already has the goal of converting our own type-safe Java data classes that define the parameters of local and remote monitors into an agent agnostic, JSON content blob.

Today the Salus system is heavily leaning on telegraf as the primary agent managed by the Envoy. As such, the configuration file processing that Envoy does for telegraf makes a purposely simple implementation assumption that the JSON content blob can be field-for-field converted to TOML.

That will break whenever telegraf removes a field they had previously marked as deprecated. We'd prefer those deprecations to not be imposed upon our end users especially since some monitor definitions might get baked into business level rules constrained by external organizational definition.

A given tenant might have several versions of telegraf in use across their fleet depending on how granular the agent install selectors were chosen. Therefore, assumptions cannot be made at a tenant-level about what version of a particular agent is in use. As such, that eliminates `monitor-management` as a viable point for agent-version specific content conversion. For example, the content might have been persisted target agent version 1.x, but the resources are later configured to install agent version 2.x -- the persisted content may no longer be compatible.

Envoy is the only part of the current system that has a concrete need to comprehend telegraf and its configuration format. Logically that would be the place where the content blob should be translated into a version specific form of telegraf configuration; however, we want and need Envoy to be a slowly changing codebase. Deploying new versions of Envoy across a large and disparate set of resources across tenants is not expected nor desired to be a common activity.

Ambassador has passing knowledge of what agents and what version of those agents are installed on a currently attached Envoy since it is the one that forms an agent install instruction while consuming an `AgentInstallChangeEvent`. Currently it opaquely populates the instruction from the event+DTO; however, it could be enhanced to track the agents and agent versions in the `EnvoyRegistry` much like the bound monitor tracking. It could then use that tracked knowledge to perform agent/version specific transformations of the monitor content while building the respective config instruction. So, Ambassador seems to be an ideal candidate for the execution of agent-version content translation.

Assuming the Ambassador is to perform the agent-version translation, then part of its existing processing agent install change events would also require re-evaluating current monitoring bindings tracked by the `EnvoyRegistry` and re-build/send config instructions to any affected Envoys.

Similar to the comment about not wanting to modify the Envoy codebase frequently we should likewise try to avoid writing hardcoded translation logic of agnostic content blobs into an agent-version specific configuration blobs. If it were hardcoded into the Ambassadors implementation, then each new version of telegraf, for example, introduced into the system would require a re-deployment of Ambassadors. While that is much more feasible than redeploying Envoys, it is not a scalable/prudent reason for continuous re-deployments.

As such, we should try to find a "table driven" approach to agent-version monitor content translation. Specifically, the table of translation rules/operations should be stored in SQL such that they can be persisted, yet easily modified on the active system via an admin API.

Some translation operations known from looking at deprecation notices in telegraf 1.12:
- renaming of a field
- converting separate host and port fields into a URL field
- converting a scalar string field into another field as array of strings
- converting a host:port field into two separate fields
  
**NOTE** while investigating deprecated telegraf config fields it was observed that metric fields are also deprecated at times. The occurrence of those seems to be very small, but non-zero.