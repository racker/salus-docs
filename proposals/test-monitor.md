
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

