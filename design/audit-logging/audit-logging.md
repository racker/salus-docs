# Requirements
Auditing information on each entity that customers and support personnel can alter (Monitors, Resources, etc...)
Information to be stored should be who updated (and if there was an impersonator who was doing the impersonation) and when.

Auditing information should only be on human interactable objects. Bound Monitors for instance will cause a storage explosion
and create paging issues with returning result sets.

Displaying information:
The result set of looking at all of the auditing information will be large. API's should be provided for looking 
at particular portions of the information (all monitors) or even filtering down to specifics (look at monitor with a specific ID).
This is particularly helpful when requests come in to validate information for a particular month such as:

"How many monitors were deleted on X account in August?"

"Were the IP's of any resources updated since the beginning of the month?"

# Audit Logging
Ultimately the code for this is very simple. I created a draft PR that gives support
for Monitors <https://github.com/racker/salus-telemetry-model/pull/153>. This draft 
describes the specifics of how to add audit logging. I will briefly go over that and 
then make some suggestions on a few ways that we might be able to customize and improve
over the out of the box solution.


# Overview
Each repository needs to extend `RevisionRepository`
Where JPA repositories are enabled we need to tell it to use the `EnversRevisionRepositoryFactoryBean.class`

On each entity we want audited we need to add the `@Audited` annotation (its also a parameter level annotation
and there is an annotation to ignore auditing certain fields)

The `@LastModifiedDate`, `@CreatedDate`, `@LastModifiedBy`, and `@CreatedBy` annotations are all JPA annotations
that tie into spring security and get populated based on the values provided. The field that is annotated with 
each is obviously the column that those values will get placed in. 

# Spring Security
We currently aren't passing through the spring security values to the microservices. This needs to be
fixed for JPA to auto inject the user who created or who updated the entity.

This tutorial [DZONE jpa auditing](https://dzone.com/articles/spring-data-jpa-auditing-automatically-the-good-stuff) 
suggests that I may have missed adding an `AuditorAware` class for grabbing the spring security 
information. The other tutorials I looked at seemed to indicate that it should happen automagically
if spring security were enabled. 

### Injecting Impersonators
Injecting the impersonator is going to be a bit more tricky.
I believe that we can get this working by implementing our own EntityListener.
On all of the `PrePersist`, `PreUpdate`, and `PreDelete` am pretty sure we can grab the 
security information with the impersonator inside of it and inject it into the object.
We can even just implement the EntityListener that is being used to inject the data and then call the super function
to extend the functionality. 

# Altering auditing database
<https://docs.jboss.org/envers/docs/#configuration> indicates a configuration property 
`org.hibernate.envers.default_catalog` that can be set to indicate the new database that the auditing
information should be placed in. It is not clear whether that allows a specification of a different
datasource or whether its limited to a different database within the same datasource as the entities 
being audited

# Accessing auditing information
As a `RevisionRepository` we can access the auditing information through the repository by utilizing 
these methods https://docs.spring.io/spring-data/envers/docs/2.3.4.RELEASE/api/

# Mitigations
The auditing is directly copying our entities. This will cause issues in the future when invariably 
we need to alter how entities are stored. The problem is going to be with the migration of the data 
and how long it takes MySQL to alter the data on disk. Depending on how much data there is this could 
take anywhere from minutes to days. 
* One solution would be to create a migration service to run the flyway migrations.
* Another solution that seems possible is to use the EntityListener to alter the output object by 
serializing it into JSON. It's not clear whether this could be made to work in this way or how much
work it would require.


# TODO
* Extend each repository for each entity we are going to audit.
* Add annotations and any new data (createdBy, CreatedAt, etc...) to each entity
* Export above audit tables into flyway migration scripts (generate them by adding `ddl-auto:true` to
one of the application-dev.yml and then pull them from the database)
* Add impersonator information (this pre-supposes that by above suggestion will work)
* Fix Spring security to pass through correct information OR add `AuditorAware` where needed
* Build cross service API to access auditing information for entire account.

