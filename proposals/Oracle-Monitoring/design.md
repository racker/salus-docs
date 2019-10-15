#Oracle Monitoring
Rackspace appears to only have Managed Oracle customers, which puts a completely different spin on our approach to this.
Since we can rely on managed Oracle hosting we can make similar requirements of the DBA's to what EM7 has had.

We are going to approach this from a different paradigm to the rest of our system.
EM7 has alarm criteria in the configuration happening on the server itself. We don't need to send true/false
through our system but it does make sense in some of these cases to massage the data a bit more than normal
on the server side (otherwise part of this turns into log monitoring).

To accomplish this the proposal is that we create a new Monitoring Agent that handles the Oracle specific
monitoring that we want to do. 


##RMAN backup monitoring:
DBA’s run this RMAN backup via some scheduler on a per database basis.
Each database gets its own log file generated and each log file could potentially get large.

Note that the EM7 configurations list stores the alarming criteria in its configuration

###EM7 Configuration Information:
1. DB Name
2. log file location
3. log file age
4. exclusion list of error codes


###Proposed Salus monitoring
We would send one metric that is just the log file age.
Another metric we would send would be a sort of log file monitoring. We would read in the log file 
(while storing the position we are in the log file) and search for any error codes of the form 
RMAN-##### or ORA-##### while ignoring any of the codes in the exclusion list. This metric would end 
up being a boolean. True means there were no errors and false means it found an error. 

Monitor:
   * List of DB Name's
   * List of error codes we can safely ignore
   * log file path


Alarm:
   * log file age
   * true/false for error codes


##Tablespace Usage
DBA's setup an export script for this that outputs the following: 
File contents are currently in this format
TABLESPACE_NAME : CURRENT USAGE % : WARNING% : CRITICAL%  

For a particular database each table will have its own line in the log file.

Ultimately we dont need the WARNING% or CRITICAL% in the output file.

###EM7 configuration 
DATABASE : tablespace : reportfile : maxage : default warning%, default critical% 


###Proposed Salus monitoring
Monitor:
   * Database
   * tablespace
   * reportfile
    
Alarmm:
   * max age
   * warning %
   * critical %
    
    
##Dataguard Replication 
Dataguard replication for EM7 is an output file with a single line.

For Salus I think that we can look at what the DBA's have setup and directly poll this information from the database

###EM7 configuration file
DATABASE : CODE : REPORT FILE : ? : WARNING : CRITICAL 
    
If report file does not exist, error 
 * Open report file and read current value. 
 * If current value > warning, raise warning 
 * If current value > critical, raise critical. 


###Proposed Salus monitoring

Monitor:
    database

Event:
    Warning
    Critical
    
    
###Oracle ASM Volume
This was just EM7 doing SNMP polling to gather CPU, Disk, memory, etc... monitoring
This should just be replaced by our normal monitoring through Telegraf. 

###Oracle Listener
This monitor is just making sure that Oracle is accepting a tcp connection
This monitor should just migrate to a remote TCP connection
