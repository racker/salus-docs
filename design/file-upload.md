# Intro
Certain telegraf monitors' configuration allow specifying an auxiliary file to be used in the configuration process, the classic example being the specification of tls files for use with network connections.

With local monitors this is not usually a problem, because the users can upload files directly to the device being monitored.  With remote monitors, particularly on public pollers, this is not the case, as users likely don't have access to those pollers

Below, I outline an approach for associating files with monitors and resources, so they can be uploaded to the devices on which the corresponding bound monitors are run.

NOTE:  This functionality is needed for remote public monitors.  Since users can always work around this problem by using either private pollers or local monitors, these changes may not be very urgent.


# Design
## DB changes
Both monitors and resources can have a zip file associated with them.  The api will allow the upload of such files.  The db will store them as a mediumblob: https://wiki.ispirer.com/sqlways/mysql/data-types/mediumblob

The boundmonitor table will have fields that show the hash of each of these zip files if they exist.  These hashes will need to change to stay in sync with the zip files as the users upload new ones.

## Ambassador-Envoy changes
When the ambassador generates the envoy configuration op for a bound monitor, it will add the two hashes, (either in the "extraLabels" field or in a newly created, dedicated field in the config op.)

When the envoy receives the config instructions, it will check the files it currently has to see if the hashes of those files match the incoming hashes.  If not, the envoy requests the new files from the ambassador.  The envoy won't restart the telegraph instance for the new config until it has recieved the new files for each of the bound monitors in the config, and unzipped them into the proper location, removing any previous files.

This will probably require a new "GetFile" rpc call on the server.  This new call will allow the envoy to pass the id of the monitor and/or resource containing the zip file in question, and the ambassador will retrieve it and forward it.

To prevent collisions with files from other monitors/resources, the envoy will have to unzip these files in directories whose names are based on the tenant/monitor/resource id.  

## Template Variables
In addition, we will need a template variable that users can use in their configs to point to the directory, something like "{{monitor-data-dir}}", or "{{resource-data-dir}}".  They will use that template when specifying their config.  For example, the mysql config has a field called "tls_ca".  In order to point to the unzipped, uploaded files, they would specify something like:
```
"tls_ca"": "{{monitor-data-dir}}/telegraf/ca.pem"
```

The content rendering code will also need to be updated to support these new template variables.

Note that the Envoy sets the working directory of the agent (such as telegraf) to be a specific directory here:

https://github.com/racker/salus-telemetry-envoy/blob/4b5d657611926c3ca06f45da63ea0c93de44d42e/agents/agents_router.go#L57-L60

so the {{monitor-data-dir}}, etc can be reliably computed as a relative directory at content rendering time.  The layout should be something like './monitor-files/<monitor-id>/'



## Zip file size
We will want to limit the size of these files to 1 mb; the api should validate this.

## Deletion
When the configuration op is a delete, the envoy will delete the corresponding monitor-data-dir, if no longer in use.  NOTE: there may be other boundmonitors still using the files, so only delete the files when the last corresponding boundmonitor is gone.


## Diagram
[Diagram](file-upload.puml).
