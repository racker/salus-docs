# Overview
This service is responsible for creating, updating, deleting, and distributing various default configurations.

It is purely for internal use (behind an admin api) and is for general business logic; not to be provided for customers to create their own policies.

## Initial Use Case
The first use case we will handle is the default monitor configurations for new accounts

Both Ping and SSH monitors should be configured at the global scope.

Both monitors will be configured with all customizable fields set to `null` so that policy metadata will be used.  Each tenant that the monitor applies to may require different metadata values to be put in place.  See the documentation for Metadata Policies for more details on that.

## High level explanation of initial use case actions
This documents the process performed to create the first ping monitor from the use case above.

1. We will use the Admin API to create one ping monitor
    1. It will be configured just as you would via the public api but the use of policy metadata fields will be maximized (by setting almost every field to `null`)
1. The monitor template will be stored in the same `monitors` table as account level monitors
    1. Except it will be stored using the tenant id of `_POLICY_`
1. We will then use the Admin API to create a monitor policy
    1. We will specify the following fields
        1. scope: global
        1. subscope: null (or just leave empty)
        1. type: monitor
        1. subtype: ping
        1. value: the monitor id of the monitor created in step 1
1. This will trigger a `MonitorPolicyEvent` to be sent to UMB for each tenant within the policy's scope
1. Monitor Mgmt will consume those events and handle each individually
1. Monitor Mgmt will query for all relevant monitor policies for that tenant
1. Monitor Mgmt will clone the monitor template to the tenant if the policy is applicable to the tenant
1. Monitor Mgmt will perform the standard binding of a monitor
1. `MonitorBoundEvent`s will be sent to UMB for consumption by the ambassador to process the new bindings.

# In-Scope Features
1. Default monitor configurations
1. Default task configurations
1. Default metadata
   * Timeout
   * Period/Frequency
   * Monitoring zones
   * Task thresholds
   * Consecutive count / bake time
   * Values stored would be things that we may want to change en-masse at some point
1. Ability to opt out of a global default
   * Some customers have agreements to not require Ping monitors

# Out of Scope Features
1. API Rate Limits
   * These could potentially be stored here, but handled by another service
1. Monitor Examples API
   * This will be monitor configurations that could be applied/cloned to an account
   * They will not be automatically applied via the global policy, but they will simplify the creation of new monitors for customers.
1. Account feature flags
   * This should be it's own policy type vs. being stored as metadata
   * This is not yet needed, but I'm sure it will at some point.
1. Notification Settings

# Terms
## Scope / Subscope
All policies will created within a given `Scope`.

Scopes are hierarchical and only the highest ranking scope relevant to the requesting tenant will be used.

The available scopes all relate to tenants and their properties.  Device level scopes are not required as that is already handled by the label matching capabilities of monitors & tasks.

For example, here are some possible scopes and their subscopes.

1. Global
1. Account Type
   * RS Cloud
   * Dedicated
   * FAWS
   * GCP
   * Azure
   * RPC
1. SLA
   * Managed
   * Intensive
   * Infrastructure
1. Tenant
   * ${tenantId}

A priority value will be assigned to each scope type, so that when a request for data is made it can be ordered and only one value will be used.

Some policies may only be set at the global scope, while others may be down to the SLA level.

## Object Types
### Monitor Policy
Used to manage default monitor configurations across all tenants.  These control what tenants a specific monitor template will be cloned and bound to.

For example, a monitor policy created with the global scope may lead to the creation of a monitor on every single tenant we know of and then bound to all resources on that tenant.

## Task Policy
Used to manage default task configurations across all tenants.

They act in the same way as monitor policies, but cause new tasks to be applied instead of monitors.  Typically we will have one task policy for each monitor policy.

## Metadata Policy
Used to define default values to be used in monitor and task configurations.

When a new monitor or task is created, any field with a `null` value will be replaced with the corresponding metadata policy value if one exists.  If that policy changes in the future, the monitors or tasks using that policy will also be updated to reflect the change.

See the Metadata Policy documentation for more details.

## Tenant Metadata
Used to provide tenant specific metadata.

One common use case will be the `AccountType` of the tenant, which will be used to narrow down the specific monitor/task/metadata policies relating to this tenant.

Another will be to override Monitor/Task policies to allow customers to opt out of the pre-defined policies.

# Database

The database stores `Policy` objects.  `Policy` is an abstract class containing `scope` and `subscope` fields and there will be numerous subclasses as defined below.

Each subclass may have its own unique fields.

## Monitor Policies
These are stored in the `monitor_policies` tables with `name` and `monitor_id` fields along with their `scope `and `subscope`.

  * `name` is a unique string to describe the monitor.
     * It is used to group similar monitor policies across different subscopes
     * e.g. the name "Ping" could be given to any Ping monitor policy so when looking up by tenant it is easy to pick the one effective policy
  * `monitor_id` will reference the `id` of a monitor template created under the `_POLICY_` tenant.

### Example
scope | subscope | name | monitor_id
--- | --- | --- | ---
Global | | Ping | 238a6013-aa09-40d9-a424-ac9429c85a4d
Global | | SSH | aa3d55ba-7cc9-4cf9-96fe-81c4e8d19188
AccountType | FAWS | AWS\_HTTP | 9c745f32-e0d8-4ac9-96d6-cb7515010b01
Tenant | 12345 | SSH | null

## Task Policies
These will be stored in the `task_policies` tables with `name` and `task_id` fields along with their `scope `and `subscope`.

The fields act in the same way as `Monitors` except they tie into the `tasks` table instead.

### Example
scope | subscope | name | task_id
--- | --- | --- | ---
Global | | Ping | 0d94f85d-8c8c-447f-a6e1-1b8278b4c157
Global | | SSH | d24f3d5c-a35e-4998-bfbd-d57d59166cc7
AccountType | FAWS | AWS\_HTTP | a749ba3c-8f74-4bb8-abe6-0931000916d4
Tenant | 12345 | SSH | null

## Tenant Metadata
These are stored in the `tenant_metadata` table.

The current known use-case involves us storing the account type of the tenant.

## Example

tenant_id | key | value
--- | --- | ---
hybrid:555433 | AccountType | Dedicated
43243 | AccountType | Cloud
56465 | AccountType | FAWS

# Scenarios

## Monitor Policy Creation / Deletion

A new monitor policy creation will simply clone the monitor template to the individual tenants effected by that policy and it will then act in the same way as any other monitor on their account.  The one exception is that these cloned monitors cannot be deleted from the tenant.  The method to remove a cloned monitor tied to a template is to have that customer opt-out of the policy.

If the policy is removed all cloned monitors and their corresponding bound monitors will be removed.

1. A policy create (or delete) will occur via the Admin API
1. This will cause a db operation within PolicyMgmt to save the new policy
1. PolicyMgmt will then query for all tenants in the policy's scope
1. PolicyMgmt will send out a MonitorPolicyEvent for each tenant
   1. It will contain the policy id, the monitor id, and a single tenant id
1. MonitorMgmt will consume the MonitorPolicyEvents
1. For each event it will
    1. Get the effective policies for the tenant
    1. If the policy in the MonitorPolicyEvent is in effect
        1. Clone the monitor template
        1. Set the customer tenant on the newly cloned monitor
        1. Refresh the metadata fields on the monitor to reflect the metadata policies applied to this tenant
        1. Link the cloned monitor to the monitor template id by setting the policyId and policyMonitorId fields  
        1. Save the cloned monitor to the customer tenant
    1. If the policy in the MonitorPolicyEvent is not in effect
        1. Get the customer monitor tied to the monitorId in the MonitorPolicyEvent
        1. Remove all bound monitors tied to that monitor
        1. Remove the customer monitor
    1. Send bind/unbind events to UMB for the ambassador to process
    
![](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/racker/salus-docs/master/design/policy-management/diagrams/monitor-policy-create.plantuml)

## A Monitor used within a Monitor Policy is updated

If the monitor template is updated it will not affect any customer currently using the policy but simply acts as an update to the template that will be used to clone any new monitors in future.

As metadata policies should be used within a monitor templates fields, no change to the monitor should be significant enough to require a push out to every customer using it.

In reality an update to a monitor template will probably never happen.

## A Monitor Policy is updated

Only the scope of a monitor policy can be updated, not the actual monitor id used.

This scenario documents the case where a monitor policy has already been in place and is applied to numerous resources, but then the scope of that policy is updated via the admin api.

1. Tenant's that were previously in-scope and are still in-scope will remain unchanged.
1. Tenant's that were not previously in-scope but are now will have the monitor template cloned to their account.
1. Tenant's that were in-scope but no longer are will have the cloned monitor and all related bound monitors removed for their account .

![](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/racker/salus-docs/master/design/policy-management/diagrams/monitor-policy-update.plantuml)

## New Resource Created
Due to monitor templates being cloned to individual tenant's where needed, a new resource coming online will be able to treat all monitors the same.  The monitor templates related to the account will be returned by label matching in exactly the same way as their own created monitors.

## Metadata Policy Change

Metadata policy changes will act the same for monitors tied to a policy as it will for those created by an individual tenant.

![](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/racker/salus-docs/master/design/policy-management/diagrams/metadata-policy.plantuml)

# Opting Out
Monitor Policies can be opted out of by creating a `null` policy of the same name for that tenant's scope.

Using the same example as above, assuming these policies are in place:

scope | subscope | name | monitor_id
--- | --- | --- | ---
Global | | Ping | 238a6013-aa09-40d9-a424-ac9429c85a4d
Global | | SSH | aa3d55ba-7cc9-4cf9-96fe-81c4e8d19188
AccountType | FAWS | AWS\_HTTP | 9c745f32-e0d8-4ac9-96d6-cb7515010b01
Tenant | 12345 | SSH | null

Tenant `12345` will never have a monitor cloned to them for the global ssh policy since they have a `null` policy in place with the same name.  Monitor Management may still receive a policy event with `tenantId: 12345, monitorId: aa3d55...` to process but due to the global policy not being in the list of effective policies for that tenant no action would be taken.

# Notes

## Monitor/Task Policy Deletions

When these policies are deleted the actual monitor/task under the `_POLICY_` tenant is not removed, only the policy entry and all cloned monitors tied to that policy. 

If a monitor template used within a policy is to be removed, it must be done so via the admin api / monitor management after the policy utilizing it has already been removed and the deletion of all cloned monitors has been completed.

## Other Dependencies
The account type will have to be populated into tenant metadata somehow, otherwise only the global policies will be used.  This may be able to utilize UMB's account/device info receivers.
