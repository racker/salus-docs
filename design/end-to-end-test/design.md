# intro/assumptions
The end to end test, "e2et", is intended to quickly excercise the salus subsystems to confirm all are functioning.

To reduce the possibility of interference, each environment, (dev, staging, prod), will have a dedicated e2et tenant id, which will not be used elsewhere.

In addition, e2et will also require access to an admin tenant.  That one need not be dedicated.

An overview of the test is shown here: [Diagram](diagram.puml)

# initialization
Note all of the following are done through the corresponding rest api's.
1. get tokens for regular and admin users from identity.
1. get all agent releases, if no valid one exists, create one.
1. delete all existing private zones for tenant; create a new private zone, with a random id.
1. remove all existing resources for the tenant.
1. remove all existing monitors/policy monitors for tenant
# implementation
## required implementation
1. start searching kafka for a new envoy event from presence monitor, (envoy will be created below.)
1. as a child process of e2et, start up envoy; configure it with private zone created above and a new random resource id.
1. confirm new envoy event from presence monitor.
1. delete all old agent installs for tenant; confirm removed from the corresponding envoy directory.
1. create new agent install; confirm added to envoy directory
1. create eventEngine task for http/tcp monitors on new resource id
1. start searching kafka for failed http/tcp events 
1. create new tcp monitor with the private zone
1. create account-specific http policy monitor using a public zone
1. confirm kafka failure event for both monitors, with timeout

## optional extras
The *required implementation* described above should be sufficient to fully vet the system.  The following allow us to also see that the events reported above do change state.
1. start local webserver matching the http/tcp monitors above.
1. confirm kafka failure events goes away, with timeout
1. stop envoy and confirm kafka envoy missing event from presence monitor


# configuration/helm charts
e2et will be configurable so as to run in any environment, local, dev, staging or prod.

helm charts will be created for the non local envs.

The build system will build a configurable container as is done for the other subsystems.  That container should also contain the envoy exe so that it can be started by the e2et.

# not checking metrics
This design doesn't include checking for metrics in kafka, only events.  In prod, the volume of data is too great to allow checking metrics.  We could probably do it in staging/dev, however.  That might help debugging deploy failures, (by allowing us to decide if the problem is upstream or down stream of the metric topic.) For now, however, I decided it isn't worhtwhile.  It is easy enough to retrofit later if we decide to add it.

# questions
1. java or golang?  Seems like it would be slighty easier to do in golang, but I could go either way.
1. New repo? It doesn't quite fit into any of the repo's we have currently.  Does it go into a utility directory in the bundle repo? or a separate repo entirely.
1. our dev environment currently appears to be using prod identity.  is that what we want?
1. Each of the kafka event searches will require a timeout.  Not sure how long they should be; I guess we'll figure that out.
1. We talked about implementing a mock envoy to support some extra functionality as required.  So far, I haven't seen the need, so am not including a design for that.  Let me know if you see the need.