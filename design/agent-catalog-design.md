# Agent Catalog and Installation Design

The goal of this design is to show how we will migrate Salus from using etcd for
managing the agent releases installs to instead use the same SQL approach as
monitor and resource management.

## Terminology

### Agent Release

The declaration of a specific version of an agent, such as telegraf, that is available
for Envoy's to download and install. An agent release has a version and requires two
tags: os and architecture. 

### Agent Catalog

As such, the collective set of agent releases is known as the agent catalog since it 
provides a catalog of all possible agent versions that could be selected for installation.

Only Salus admins can delcare agent releases in the agent catalog; however, end users
can list agent releases so that they can pick a release for installing. 

### Agent Install

Envoys start off with a blank slate -- they don't know what agents to install/run
until they are told to do so. The agent install workflow is where an end user specifies
what specific version of an agent type should be installed based upon the usual label
selector mechanism.

For example, an end user might have only Linux servers, so they will pick telegraf
releases for Linux by using the `agent_discovered_linux=linux` label selector.
Furthermore, they might be paranoid and only want to install a newer release, say 1.11.0,
of Telegraf on their "dev" nodes and leave their "prod" nodes at an older version, 1.10.4.
In that case they would create two "agent installs" each picking the respective
agent release ID and using the appropriate label selector to differentiate "dev" from "prod". 

**NOTE** the modification of agent installs might later be moved from the public API
or restricted to only be invokable by Rackspace support personnel. 

Since an Envoy can only run one copy/version of a specific agent, such as telegraf,
then agent install binding needs to account for possible conflicts where multiple
agent installs have selected the same agent type, but with different versions.
To resolve, the conflict, [this SemVer library](https://github.com/zafarkhaja/jsemver#comparing-versions)
will be used to pick the newest selected version out of the selected agent releases.

### Agent Catalog Management (ACM)

This is the name of the new service that will be introduced and will operate alongside
its counterparts Resource Management and Monitor Management. Like those services,
this service is the single point of SQL access for entities related to the agent
catalog and installation tracking. 

It exposes a REST API to be used:
- internally by the Ambassador
- proxied by the public API for tenant operations, such as agent install selection
- proxied by the admin API for agent catalog manipulation

## Schema

[Diagram](agent-catalog-er.plantuml).

The schema loosely follows the pattern of resource, monitors, and monitors bound to
resources, where agent installs are like monitors in that they selector the agent
release and what envoy-resources get the installation by label selection.

The `AgentRelease` captures all of the details needed for a specific version of a specific
type of agent. This structure and its fields are the same as the object that was 
stored in etcd.

`AgentInstallSelector` also remains the same as what the API accepted before; however,
rather than the etcd label "fan out" the service did before it will be stored in SQL/JPA
as we have done with resources and monitors as an `@ElementCollection`.

The `BoundAgentInstall` is new to this design and follows in the pattern of the
`BoundMonitor` entity. These entities are created and deleted according to the
algorithm discussed below, but are basically the result of evaluating the label
selectors of the `AgentInstallSelector` instances against the tenant's resources and
reconciling multiple bindings of an agent type by version. When an Ambassador
queries the ACM on behalf of an Envoy-Resource attached to it, this entity is what 
primarily drives the query response.

## Catalog management

[Catalog Declaration Diagram](agent-catalog-declare.plantuml)

Management of the Agent Catalog itself remains quite similar to the etcd implementation
where it mostly consists of CRUD operations via JPA of the `AgentRelease` entities.

## Installation propagation

[Diagram](agent-catalog-install.plantuml)

The installation propagation is a very similar concept as monitors selecting resources;
however, it is a mostly simpler since it doesn't need to deal with local/remote permuations.
Agent installation is only concerned with binding to resources associated with Envoy and
furthermore only a single version of a given agent (such as telegraf) can be bound to
an Envoy-Resource at any point.

To summarize the input events from the diagram there are two categories of change 
in the system that trigger the installation propagation algorithm:
- Resource associated with Envoy changed
  - Envoy first attaches with resource
  - Envoy re-attaches with modified labels
  - End user modifies labels via API
  - End user deletes resource associated with detached Envoy
- Agent installation selection changed
  - End user create installation selection
  - End user removes installation selection

Even though there are several specific forms of resource change, all changes result in
a `ResourceEvent` to be produced. As such, the processing of that event becomes the
consolidated entry point for ACM to re-evaluate any existing agent installation selections.

ACM will expose REST operations for agent installation selection that in turn will be
proxied by the public API service. The API controller is the consolidated entry point for
ACM to evaluate the create/remove of the installation selection against pre-existing
resources associated with Envoys. ACM is query Resource Management for the resources
that match the agent installation label selectors.

Either category of change results in creation or removal of the `BoundAgentInstall` entities 
similar to monitor binding. The database changes are accompanied with ACM producing
`AgentInstallChangeEvent`s to a new kafka topic, `telemetry.installs.json`.  

There is one complication unique to ACM for reconciling those bindings. A user may have
created two agent install selections with the same label selectors or similar enough such
that the same resource is selected by both installs. Each install would reference 
distinct agent releases, but both releases might be for the same agent type, such as
telegraf. The remaining difference between the agent releases could then be the version
of the agent, such as 1.10.2 vs 1.11.0. The algorithm will reconcile this case of two or
more overlapping resource-agent bindings by choosing the agent release with the 
greatest/newest version according to 
[SemVer comparison rules](https://github.com/zafarkhaja/jsemver#comparing-versions).

Finally, it is the Ambassador's responsilbity to send install instructions to its
attached Envoys. Ambassadors will individually consume from the `telemetry.installs.json`
topic in order for the events to "broadcast" to all Ambassadors.
Upon consuming `AgentInstallChangeEvent`s each Ambassador will determine if the event
is applicable to the Envoys currently attached to that Ambassador. If so, the appropriate
instruction is constructed and sent via the attach-response stream of the Envoy.