# Salus User Terminology
## Resource
A resource represents an individual device,  i.e. a server, firewall, or load balancer.  These types of resources are generally created automatically.

Resources can manually be created to represent any other object that should be monitored.  e.g. you may create a Resource to represent an entire cloud deployment or even your house for home monitoring.

Resources are also created automatically for any server on which an Envoy runs.  (Envoys are described below.)

### Resource Labels
These are simple key/value mappings that can only consist of strings.  They are used to provide identifiable data that can be used to specify groups of resources, e.g.: `type: web`.

See the Monitor Configuration section for how they are used.

## Monitor / Monitor Configuration
A monitor configuration declares the details of what type of monitoring is to be performed, and on which resources.  It contains a list of "label-selectors" which select the resources that will be monitored by this configuration.  For example, assume all the mysql resources have been defined to include the label "type: mysql".   Then a single monitor can be configured to gather metrics for all those Resources by defining the monitor with a label-selector "type: mysql".

Whenever one or more resources matches the monitor configuration, a monitor will be bound to each matching resource and begin gathering the relevant metrics for that resource.  Such a monitor is referred to as a "bound monitor".

Salus supplies a "reasonable" set of defaults to the various monitor parameter fields so that in most cases only a few parameters need to be specified explicitly.

## Resource Metadata

These are key/value mappings containing arbitrary information of your choosing.  These can include any other data you might want to store with the resource, and can be used as template parameters in monitors.

These are similar to labels but they are not used within a monitor's `label-selector` to match on the resource.

A metadata field can be used within a monitor's configuration, where a templated value will be replaced with the corresponding metadata value.  For example, suppose an http monitor is configured to use `url: ${resource.metadata.my_url}`.  Then when new bound monitors are created for each matching resource, the value of `url` will be replaced with the `my_url` metadata value in the resource, each of which is likely to be different.

This abstraction facilitates the use of one monitor with many resources, eliminating the need to create a separate monitor for each.  

## Monitor Plugins
Different types of metrics can be retrieved using different "plugins".  For example cpu data is retrieved with the "cpu" plugin and mysql data with the "mysql" plugin.  The plugin must be specified when the monitor is configured.

Some of the common types are: Cpu, Disk, Memory, Ping, Http

### Local vs Remote Plugins
Certain monitor plugins are "local".  This means they require an agent manager called the "Envoy" running on the resource. Local plugins are implemented by managed agents running on the server resource.  The Envoy gathers the metrics produced for those monitors.  The following are example local monitors: CPU, Disk, Memory

Certain monitor plugins are remote.  This means they don't require an Envoy running on the resource.  The following are example remote monitors: Ping, Http

## Envoy
An Envoy is the process installed on a resource that accepts the configuration of the monitors bound to it, and the publishing of metrics gathered by those monitors.  It is responsible for managing the installation and execution of agents to gather the metrics according to the configured monitors.

The Envoy must be configured with a token, which is allocated from the Salus API. The Envoy uses that token to authenticate with the system and identify the owning tenant.

## Test Monitors
At times, you'll want to see exactly what metrics are gathered by a monitor.  The easiest way to quickly do this is to use a "test" monitor.  Unlike creating a standard monitor, which only returns success or failure depending on whether it is stored in the database, a test monitor immediately returns with sample of the metrics gathered for a particular resource.

So if you're uncertain about the configuration of a particular monitor, you can test a monitor's configuration, immediately confirm it's correctness and expected metrics, and then follow-up with the creation of a standard monitor.

## Events/Event Tasks
Monitors define the gathering of metrics that help to describe the ongoing state of the resources they are bound to. However, since it not feasible to always watch those metrics for state changes, tasks can be defined to produce an event when metrics are unhealthy.


At it's most basic, an event task is configured by specifying
- the field name of a metric,
- a comparison operator,(like "<" or ">"),
- and a threshold to be tested against.

So, for example, the cpu plugin gathers a "usage_user" metric, which states the percent of the cpu being used by user processes.  Configuring an event task with field "usage_user", comparsion operator ">" and threshold "20" would cause an event to be generated whenever the user usage goes above 20%.

## Monitoring Zones
A public monitoring zone is a location from which Rackspace executes remote monitors.  It represents a general location from which data is collected.  Examples of public monitoring zones are "public/us_east_1" and "public/eu_west_1".  Remote monitors, unless explicitly configured otherwise, will be executed from multiple public monitoring zones to provide redundancy.  The zones used will differ based on the location of the monitored resource.


In addition, you can create private monitoring zones specific to a Rackspace account.  These are used in conjunction with private pollers as described in the next section.

## Pollers
Pollers are processes that execute remote monitors.  Public Pollers are the processes running in the public monitoring zones as described above.  They operate as a special mode of an Envoy that manages the execution of remote plugins, and are configured within a specific monitoring zone.

However, public pollers may not be able to access the resources to be monitored behind a customers firewalls.  Rather than changing the firewall rules to allow access, some customers prefer to set up private pollers behind their firewalls.

These private pollers can be grouped into private monitoring zones, so as to provide the same level of redundancy, as exhibited by the public pollers that run on Rackspace infrastructure.

## Monitoring Policies
To minimize the work required in setting up monitoring on your infrastructure, Rackspace specificies some defaults that apply to most resources.  These defaults are specified as "Policies".  Generally, you don't need to consider them as they are managed automatically.  The only common exception is when customers want to disable them.

In that case you would create a customer specific policy that would allow them to opt out.

There are a number kinds of policies, the main one being "monitor" policies.

These are monitors that are configured on each account by default, such as ping monitoring across all resources containing an ip address.  These configurations may vary based on the type of account account has.
