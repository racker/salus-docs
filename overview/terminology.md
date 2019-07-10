# Services

## Envoy
The process installed on a customer server that manages the installation of 3rd party monitoring agents (such as telegraf), the configuration of monitors to run, and the publishing of metrics gathered by those monitors/agents.
Envoys are used for three different methods:

### Public Poller
A cluster of envoys deployed in various regions/clouds to perform remote monitoring for publicly accessible customer resources

These are deployed on our own accounts / infrastructure

### Private Poller
A cluster of envoys deployed within a customer’s environment to perform remote monitoring on their internal resources

For example, RPC will deploy private pollers to monitor customer datacenter OpenStack infrastructure

### Agent
An envoy running on a device monitoring local system performance

This is what handles CPU, disk, memory etc.

## Ambassador
Envoys attach to this.  It accepts the envoy connections and handles all communication to/from the envoy.  This includes receiving metrics that are sent from the envoys and sending new monitor configurations to the envoys.

## Public API
The API that customers will interact with.  A token specific to the account the action is being performed on will be required.

## Internal API
The API that other Rackspace teams can interact with.  The operations available will likely be a superset of the Public API, but with likely only a few additions.  An Identity admin token could be used here to interact with any account.  Impersonation of a specific account will not be needed.

## Admin API
The API the engineering team can use to interact with various services.  Operations may include tweaking configuration properties in real-time to allow for easy scaling.

## Resource Management
A service that handles all CRUD operations for Resources.  It responds to requests from the APIs as well as reading events from Kafka whenever an envoy is attached.


# Data / Objects

## Resource
A resource typically represents an individual device.  i.e. a server, firewall, or load balancer.  Resources for these will automatically be created via automation or an envoy attachment.

Resources can manually be created to represent any other object that should be monitored.  e.g. you may create a Resource to represent an entire cloud deployment or even your house for home monitoring.

### Resource ID
The field in which an envoy or resource can be uniquely represented on an account.  Within an envoy config file a `resource_id` field can be specified which contains the value used to tie the envoy to a particular Resource.  For example, on a cloud server the `resource_id` should typically be set as “xen-id:123abc", where "123abc" is the xenId value of the server.  When the envoy connects, the Ambassador will see this value and determine whether a new Resource needs to be created or if an existing one should be updated with any new discovered labels.

### Labels
These are simple key/value mappings that can only consist of strings.  They are used to provide identifiable data that can be used to uniquely identify a device, such as `host: web0.example.com`, or they can be used as more generic identifiers to be able to form groups of devices: `type: web`.

See the Monitor Configuration section for how they are used.

### Metadata

These are key/value mappings containing arbitrary non-identifying metadata

These are similar to labels but have a few differences:

1. They cannot be used within a monitor's `labelSelector` to match on the resource
1. The value may contain either a single value or a list.


If a metadata field is used within a monitor's configuration, the templated value will be replaced with the corresponding metadata value.  For example, if a url monitor is configured to use `address: ${resource.metadata.urls}` it will lead to a new bound monitor being created for each `url` within the metadata's list.

## Monitor / Monitor Configuration
A monitor configuration declares the details of what agent plugin you wish to utilize.  It also contains a list of labels which determine what resources these metrics should be pulled from.  For example, you may create a monitor configured to pull mysql metrics from Resources with key “type” and value “database” or http metrics from any Resource which have a label with key “host” and value “web\*” (wildcards are allowed).

Whenever one or more resources matches the monitor configuration a monitor will be created for each resource and begin retrieving the relevant metrics for that service.

## Active Envoys
The etcd stored collection of envoys that are currently connected to the Ambassador.

## Expected Envoys / Resources
The mysql stored collection of resources that have had an envoy connected with the same `resourceId` at some point.  This data is stored in the same location as all Resources, but can easily be filtered based on whether it is relevant for envoys or not.


# Protocol Events

## Resource Event
A Kafka message to notify services of a change in a particular resource.  Minimal info will be provided in the event and the service consuming the events will have to query the Resource API to discover the details of the Resource.  This event could be caused by a create, update, or deletion.
