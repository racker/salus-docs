# Monitor-Plugin Data Models

Monitors in Salus are structured into effectively two layers:
- the top-level, universal attributes that apply to all types of monitors and agent implementations
- the inner details that are specific to the monitor type and ultimately the agent/plugin that implements the monitor type

For example, the name and label selectors of a monitor are applicable for every type of monitor. On the other hand, a disk monitor's mount point is an attribute that is utilized by Telegraf, which is the agent implementing that particular monitor type.

The Spring Web MVC controller handlers that service the Salus API to create or modify monitors map and validate JSON request bodies to the Java data model classes declared in `com.rackspace.salus.monitor_management.web.model`. The following diagram provides an overview of the relevant, "user facing" classes in that package:

![monitor-details-plugin-classes.puml](https://github.com/racker/salus-docs/raw/master/design/monitor-plugin-data-models/monitor-details-plugin-classes.puml)

## Local and Remote Model Grouping

As a first-level logical grouping, the plugin data model classes must extend one of two abstract base classes, `LocalPlugin` or `RemotePlugin`, to indicate the scope or "monitoring perspective" of the plugin. Local plugins are those that are run by an agent situated on the resource to be monitored and monitor an aspect of that resource. Remote plugins are run by an agent (aka "poller") with some kind of network access to the resource.

When the JSON parsing encounters one of these abstract classes it makes use of the following annotations to reconcile the concrete sub-class:

- `@JsonTypeInfo` to define a type discriminator property, called "type"
- `@JsonSubTypes` to declare the mapping of discriminator values to concrete sub-classes

## Associating plugin data model to agent and monitor type

After the user's JSON request content is parsed into a `DetailedMonitorInput` object structure, the request is ultimately converted into a `Monitor` entity for persistence and further processing. That conversion leverages annotations placed on each concrete plugin data model class:

- `@ApplicableAgentType` to declare the agent that implements the specific monitor
- `@ApplicableMonitorType` to map the plugin to the `MonitorType` enumeration to allow for type-safe references to monitors of a given type

The conversion process also identifies the local/remote scope of a monitor by the respective base class that the plugin data model class extends.

## Plugin data model classes organized by package

Since the set of monitoring agents supported by Salus may gradually grow over time, the plugin data model classes are organized by agent under a sub-package for each under `...web.model`. For example, all of the monitors implemented by [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) are located in the `...web.model.telegraf` package.

## Defining a new plugin data model class

1. Identify the agent that will/does implement the monitor and declare the class in the appropriate sub-package.
2. The name of the plugin data model class should match as closely as possible to the `MonitorType` enumeration entry and typically to the agent's naming of that monitor/plugin while still conforming to Java class naming guidelines.
3. All plugin classes should be annotated with Lombok's `@Data` to ensure consistent rendering of getters, setters, equals and hashCode methods.
4. Since the base class of each plugin is effectively a "marker", each plugin class should also be annotated with Lombok's `@EqualsAndHashCode(callSuper = false)`
5. It is somewhat redundant with the enclosing package, but the plugin class should be annotated with the Salus annotation `@ApplicableAgentType`, which is discussed in a previous section.
6. The mapping of the plugin class to `MonitorType` should be declared with the Salus annotation `@ApplicableMonitorType`
7. Each configurable aspect of the specific agent's implementation should be conveyed by one or more fields of the plugin class. In turn those fields can be processed by [monitor translation](https://github.com/racker/salus-docs/tree/master/design/monitor-translation); however, one to one mappings of fields are preferred when the name and meaning of the field makes sense for end users of Salus.
8. Apply validation constraints, JSON declarations, and/or JSON schema declarations on fields, as needed. Since the monitor request body is processed by [Spring's bean validation](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#validation-beanvalidation), the fields can be declared with a combination of 
   - "javax.validation" (preferred)
   - hibernate-specific annotations
   - Salus custom constraints implemented (or to be implemented) in the `...web.validator` package
9. Apply field defaults for cases where the "zero-value" is not the desired default value
   1. Declare a field initializer, such as `boolean totalcpu = true`
   2. Add `@JsonProperty(defaultValue = "...")`
   3. Similarly, add `@JsonSchemaDefault("...")` to satisfy the generated JSON schema
   
## Notes on JSON Schema Generation

The following notes were gathered from inspecting the JSON scheme generated via Monitor Management's `/schema/monitor-plugins` REST endpoint and comparing against the plugin data model classes. It is not intended to be an exhaustive catalog of schema generation strategies, so others may come into play as more plugin classes are implemented.

### Implicit defaulting rules

When a field uses the implicit, zero-value default the generated JSON schema does not explicitly declare a `default`, so following outlines the implicit defaults per JSON schema primitive:

- numeric defaults to 0
- boolean defaults to false
- string defaults to null, if not required, and empty string, if required

### Handled correctly

#### `@NotBlank` String

From
```
  @NotBlank
  String mount;
```

To
```
      "properties": {
        "mount": {
          "type": "string",
          "pattern": "^.*\\S+.*$",
          "minLength": 1
        }
      },
      "required": [
        "mount"
      ]
```

#### nullable/optional String

From
```
  String username;
```

To
```
        "username": {
          "type": "string"
        },
```
and absent from `required`

#### nullable/optional Duration

From
```
  Duration timeout;
```

To
```
        "timeout": {
          "type": "string",
          "format": "date-time"
        },
```
and absent from `required`

#### nullable / optional Boolean

From
```
  Boolean insecureSkipVerify;
```

To
```
        "insecureSkipVerify": {
          "type": "boolean"
        }
```
and absent from `required`

#### NotNull enum

From
```
  @NotNull
  Protocol network = Protocol.udp;
```

To
```
        "network": {
          "type": "string",
          "enum": [
            "udp",
            "tcp"
          ]
        },
```
and listed in `required`

#### NotNull Integer

From
```
  @NotNull
  Integer port = 53;
```

To
```
        "port": {
          "type": "integer"
        },
```
and in `required`

#### custom pattern but optional

From
```
  @Pattern(regexp = "GET|PUT|POST|DELETE|HEAD|OPTIONS|PATCH|TRACE", message = "invalid http method")
  String method = "GET";
```

To
```
        "method": {
          "type": "string",
          "pattern": "GET|PUT|POST|DELETE|HEAD|OPTIONS|PATCH|TRACE"
        },
```
and absent from `required`

#### optional `Map<String,String>`

From
```
  Map<String, String> headers;
```

To
```
        "headers": {
          "type": "object",
          "additionalProperties": {
            "type": "string"
          }
        }
```
and absent from `required`

#### require non-empty `List<String>`

From
```
  @NotEmpty
  List<@ValidLocalHost @Pattern(regexp = Mysql.REGEXP, message = Mysql.ERR_MESSAGE) String> servers;
```

To
```
        "servers": {
          "type": "array",
          "minItems": 1,
          "items": {
            "type": "string"
          }
        },
```
included in `required`

#### optional Integer

From
```
  Integer perfEventsStatementsDigestTextLimit;
```

To
```
        "perfEventsStatementsDigestTextLimit": {
          "type": "integer"
        },
```
and absent from `required`

#### optional, empty-able List

From
```
  List<String> tableSchemaDatabases;
```

To
```
        "tableSchemaDatabases": {
          "type": "array",
          "items": {
            "type": "string"
          }
        },
```
and absent from `required`

#### renamed json property

From
```
  @JsonProperty("interface")
  String monitoredInterface;
```

To
```
        "interface": {
          "type": "string"
        }
```

#### required Integer with range

From
```
  @NotNull
  @Min(1)
  @Max(65535)
  Integer port;
```

To
```
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        },
```
included in `required`

### Not handled, but acceptable

#### `@Pattern` on list entry

From
```
  @NotEmpty
  List<@ValidLocalHost @Pattern(regexp = Mysql.REGEXP, message = Mysql.ERR_MESSAGE) String> servers;
```

To
```
        "servers": {
          "type": "array",
          "minItems": 1,
          "items": {
            "type": "string"
          }
        },
```

#### custom validation

From
```
  @NotEmpty
  @ValidLocalHost
  @Pattern(regexp = Postgresql.REGEXP, message = Postgresql.ERR_MESSAGE)
  String address;
```

To
```
        "address": {
          "type": "string",
          "pattern": "^(postgres://.+)|(([^ ]+=[^ ]+ )*([^ ]+=[^ ]+))$",
          "minLength": 1
        },
```
include in `required`

`@ValidLocalHost` is programmatic validation and therefore isn't expected to be expressed via schema

### Not handled as originally expected

#### default value of non-optional boolean

From
```
  boolean collectCpuTime;
```

To
```
        "collectCpuTime": {
          "type": "boolean"
        },
```
and present in `required`
