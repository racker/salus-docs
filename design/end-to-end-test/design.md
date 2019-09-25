# Intro/assumptions
The end to end test, "e2et", is intended to quickly excercise the salus subsystems to confirm all are functioning.

To reduce the possibility of interference, each environment, (dev, staging, prod), will have a dedicated e2et tenant id, which will not be used elsewhere.

In addition, e2et will also require access to an admin tenant.  That one need not be dedicated.

An overview of the test is shown here: [Diagram](diagram.puml)

# Initialization
Note all of the following are done through the corresponding rest api's.
1. get tokens for regular and admin users from identity.
1. get all agent releases, if no valid one exists, create one.
1. delete all existing private zones for tenant; create a new private zone, with a random id.
1. remove all existing resources for the tenant.
1. remove all existing monitors/policy monitors for tenant
# Implementation
## Required implementation
1. start searching kafka for a new envoy event from presence monitor, (envoy will be created below.)
1. as a child process of e2et, start up envoy; configure it with private zone created above and a new random resource id.
1. confirm new envoy event from presence monitor.
1. delete all old agent installs for tenant; confirm removed from the corresponding envoy directory.
1. create new agent install; confirm added to envoy directory
1. create eventEngine task for http/tcp monitors on new resource id
1. start searching kafka for failed http/tcp events 
1. create new tcp monitor with the private zone, pointing to a non-responsive port
1. create account-specific http policy monitor using a public zone, pointing to a non-responsive port
1. confirm kafka failure event for both monitors, with timeout

# Configuration/helm charts
e2et will be configurable so as to run in any environment, local, dev, staging or prod.

helm charts will be created for the non local envs.

The build system will build a configurable container as is done for the other subsystems.  That build would create a container that contains the e2et and envoy exe's. The helm chart would specify how the container would be spun up; the corresponding values.yaml files would include env vars for all the various env specific urls etc that e2et needs. The envoy would be started and configured as needed by the e2et exe, so the container only needs to include the envoy exe, not start it.

The deployment process would use those helm charts to spin up an e2et service as needed. It would run, determine the status of the deploy and then exit.


# Not checking metrics
This design doesn't include checking for metrics in kafka, only events.  In prod, the volume of data is too great to allow checking metrics.  We could probably do it in staging/dev, however.  That might help debugging deploy failures, (by allowing us to decide if the problem is upstream or down stream of the metric topic.) For now, however, I decided it isn't worhtwhile.  It is easy enough to retrofit later if we decide to add it.

# Other notes
1. e2et will be implemented in golang in the salus-tools repo.
1. Dev environment will be updated to use staging identity.
