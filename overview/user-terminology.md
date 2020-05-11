# Salus User Terminology
## Resource
A resource represents an individual device,  i.e. a server, firewall, or load balancer.  These types of resources are generally created automatically.

Resources can manually be created to represent any other object that should be monitored.  e.g. you may create a Resource to represent an entire cloud deployment or even your house for home monitoring.

### Resource Labels
These are simple key/value mappings that can only consist of strings.  They are used to provide identifiable data that can be used to uniquely identify a resource, such as `host: web0.example.com`, or they can be used as more generic identifiers to be able to form groups of resources: `type: web`.

See the Monitor Configuration section for how they are used.

## Monitor / Monitor Configuration
A monitor configuration declares the details of what type of monitoring is to be performed, and on which resources.  It contains a list of "label-selectors" which select the resources that will be monitored by this configuration.  For example, assume all the mysql resources have been defined to include the label "type: mysql".   Then a monitor can be configured to generate metrics for those Resources by defining the monitor with a label-selector "type: mysql".

Whenever one or more resources matches the monitor configuration, a monitor will be bound to each matching resource and begin retrieving the relevant metrics for that resource.  Such a monitor is referred to as a "bound monitor".

## Resource Metadata

These are key/value mappings containing arbitrary information of your choosing.  These can include any other data you might want to store with the resource, and can be used in monitors, event tasks, or any custom automation processes.

These are similar to labels but they are not used within a monitor's `labelSelector` to match on the resource.

A metadata field can be used within a monitor's configuration, where a templated value will be replaced with the corresponding metadata value.  For example, suppose a url monitor is configured to use `address: ${resource.metadata.my_url}`.  Then when new bound monitors are created for each matching resource, the value of `address` will be replaced with the `my_url` metadata value in the resource, each of which is likely to be different.

This abstraction facilitates the use of one monitor with many resources, eliminating the need to create a separate monitor for each.  

## Monitor Plugins
Different types of metrics can be retrieved using different "plugins".  For example cpu data is retrieved with the "cpu" plugin and mysql data with the "mysql" plugin.  The plugin must be specified when the monitor is configured.

The current list of plugins is: System, Postgresql, Net, Ping, Disk, SqlServer, Redis, Cpu, Smtp, DiskIo, Dns, Procstat, Mem, NetResponse, Mysql, X509Cert, Apache, HttpResponse

### Local vs Remote Plugins
Certain monitor plugins are "local".  This means they require a special agent called the "Envoy" running on the resource. The Envoy gathers the metrics produced for those monitors.  The following are local monitors: Apache, CPU, Disk, DiskIo, Mem,  Net, Procstat, Redis, System

Certain monitor plugins are remote.  This means they don't require an Envoy running on the resource.  The following are remote monitors: Dns, HttpResponse, NetResponse, Ping, Smtp, X509Cert

Finally, some monitors can be configured with either a local or remote plugin: Mysql, SqlServer, Postgresql

## REST API
The resources and monitors described above, (as well as many of the other constructs defined in this document,) can be created, updated, and deleted using an http based REST api.  Interacting with that api requires authentication with your Rackspace credentials.

## Envoy
An Envoy is the process installed on a resource that accepts the configuration of the monitors bound to it, and the publishing of metrics gathered by those monitors.

Before using the envoy, an envoy token must be generated.  It can be generated with the rest api.  It will need to be included in the envoy configuration when installing the envoy.


## Test Monitors
Monitors generate data which can take a few minutes to view.  While not long, it can be inconvenient waiting to confirm the expected data is being produced.  To bypass this wait, "test" monitors can be created which return immediately with the data generated.  Note that a test monitor is a one shot.  In other words, it only generates a bit of data for immediate viewing.

So if you're uncertain about the configuration of a particular monitor, you can create a corresponding test monitor, immediately confirm it's correctness, and then create a properly configured monitor.

## Events/Event Tasks
Monitors cause metrics to be generated that help describe the state of the resources they are bound to.  However, in order to generate an "event", (a notification of a bad state), an "event task" must be configured.  

At it's most basic, an event task is configured by specifying the field name of a metric, a comparison operator,(like "<" or ">"), and a threshold to be tested against.  So, for example, the cpu plugin generates a "usage_user" metric, which states the percent of the cpu being used by user processes.  Configuring an event task with field "usage_user", comparsion operator ">" and threshold "20" would cause an event to be generated whenever the user usage goes above 20%.

## Monitoring Zones
A public monitoring zone is a location from which Rackspace executes remote monitors.  It is an abstraction for a general location from which data is collected.  Examples of public monitoring zones are "public/us_east_1" and "public/eu_west_1".  Remote monitors are usually automatically configured to run from multiple public monitoring zones, for redundancy.

In addition, you can create private monitoring zones specific to your Rackspace account.  These are used in conjunction with private pollers as described in the next section.

## Pollers
Pollers are processes that execute remote monitors.  Public Pollers are the processes running in the public monitoring zones as described above.

However, public pollers may not be able to access the resources to be monitored behind your firewalls.  Rather than changing the firewall rules to allow access, some customers prefer to set up private pollers behind their firewalls.  These pollers are very similar to the Envoys described above but are configured to process remote monitors, rather than local ones.

These private pollers can be grouped into private monitoring zones, so as to provide the same level of redundancy, as exhibited by the public pollers that run on Rackspace infrastructure.

## Monitoring Policies
To minimize the work required in setting up monitoring on your infrastructure, Rackspace specificies some defaults that apply to most resources.  These defaults are specified as "Policies".  Generally, you don't need to consider them as they are managed for you automatically.  The only common exception is when you want to disable them.

In that case you would create your own policies that would allow you to opt out of them.

There are two specific kinds of policies, monitor and metadata.

### Monitor policies
These are monitors that are turned on by default, like ping and banner testing.

### Metadata policies
These are resource metadata that is created by default, like ip address.


## TODO
Here are a few other concepts I considered adding to this document but wasn't sure.  Do you think users need to know about the following:

```
telegraf
monitor translations
tenant metadata
acm/release/install
presence monitor
```
