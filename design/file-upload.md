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
In addition, we will need a template variable that users can use in their configs to point to the directory, something like "{{MonitorDir}}", or "{{ResourceDir}}".  They will use that template when specifying their config.  For example, the mysql config has a field called "tls_ca".  In order to point to the unzipped, uploaded files, they would specify something like:
```
"tls_ca"": "{{MonitorDir}}/telegraf/ca.pem"
```

The content rendering code will also need to be updated to support these new template variables.

## Zip file size
We will want to limit the size of these files.  I think a megabyte should be enough.  (I think grpc supports up to 4megs before it has to be a streaming call, so maybe we should just make it 4megs.)

## Deletion
When the configuration op is a delete, the envoy will delete the corresponding directories.


## Diagram
[Diagram](file-upload.puml).
