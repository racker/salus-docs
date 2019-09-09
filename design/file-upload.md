# Intro
Certain telegraf monitors' configuration allow specifying a auxiliary file to be used in the configuration process, the classic example being the specification of tls files for use with db connections.

With local monitors this is not a problem, because the users typically are typically able to upload files directly to the filesystem of the device being monitored.  With remote monitors, particularly on public pollers, this is not the case, as users likely don't have access to those filesystems.

Below, I outline an approach for associating files with monitors and resources, so they can be uploaded to the filesystems on which the corresponding bound monitors are run.

NOTE:  This is needed for remote public monitors, and the users can always work around this problem by using either private pollers or local monitors.  So that may reduce the urgency for implementing these changes.


# Design
## DB changes
Both monitors and resources can have a zip file associated with them.  The api will allow the upload of such files.  The db will store them as a mediumblob: https://wiki.ispirer.com/sqlways/mysql/data-types/mediumblob

The boundmonitor table will have fields that show the hash of each of these zip files if they exist.

## Ambassador-Envoy changes
When the ambassador generates the configuration op for a bound monitor, it will add the two hashes, (either in the "extraLabels" field or in a dedicated field.)

When the envoy receives the config instructions, it will check to see if the hashes, of the files it has, corresponds to the hashes in config op.  If not, it request the new files from the ambassador.  It won't restart the telegraph instance until it has recieved the new files, and unzipped them into the proper location, removing any previous files.

This will probably require a new "GetFile" rpc call on the server.  The envoy will pass back the id of the monitor and/or resource containing the zip file in question, and the ambassador will retrieve it and forward it.

To prevent collisions with files from other monitors/resources, the envoy will have to unzip this file in a directory whose name is based on the tenant/monitor/resource id.

## Template Variables
In addition, we will need a template variable that users can use in their configs to point to the directory, something like "{{MonitorDir}}", or "{{ResourceDir}}".  Then they can use that template when specifying their config.  For example, the mysql config has a field called "tls_ca".  In order to point to the unzipped, uploaded files, they could specify something like this:
```
"tls_ca"": "{{MonitorDir}}telegraf/ca.pem"

The content rendering code will need to be updated to support these new template variables.

## Zip file size
We will want to limit the size of these files.  I think a megabyte should be enough.  (I think grpc supports up to 4megs before it has to be a streaming call, so maybe we should just make it 4.)

## Deletion
When the configuration op is a delete, the envoy will delete the corresponding directories.
