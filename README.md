# Logback JSON encoder

Provides an encoder, layout, and several appenders to log from [logback](http://logback.qos.ch/) in JSON format.

Supports both regular logging events and access events (provided by [logback-access](http://logback.qos.ch/access.html)).

Originally written to support output in [logstash](http://logstash.net/)'s JSON format, but has evolved into a highly-configurable, general-purpose, JSON logging mechanism.  The structured of the JSON output, and the data it contains, is fully configurable.

## Including it in your project

Maven style:

```xml
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>3.6</version>
</dependency>
```

If you get `ClassNotFoundException`/`NoClassDefFoundError`/`NoSuchMethodError` at runtime, then ensure the required dependencies (and appropriate versions) as specified in the pom file from the maven repository exist on the runtime classpath.  Specifically, the following need to be available on the runtime classpath:

* jackson-databind / jackson-core / jackson-annotations
* logback-classic / logback-core / logback-access
* slf4j-api

Older versions than the ones specified in the pom file _might_ work, but the versions in the pom file are what testing has been performed against.

## Usage

To log using JSON format, you must configure logback to use either:

* an appender provided by the logstash-logback-encoder library, OR
* an appender provided by logback (or another library) with an encoder or layout provided by the logstash-logback-encoder library

Throughout this document:

* _LoggingEvent_ refers to standard logback events logged through a `Logger`, and
* _AccessEvent_ refers to events logged through [logback-access](http://logback.qos.ch/access.html).

The appenders, encoders, and layouts provided by the logstash-logback-encoder library are as follows:

| Format        | Protocol   | Function | LoggingEvent | AccessEvent
|---------------|------------|----------| ------------ | -----------
| Logstash JSON | Syslog/UDP | Appender | [`LogstashSocketAppender`](/src/main/java/net/logstash/logback/appender/LogstashSocketAppender.java) | n/a
| Logstash JSON | TCP        | Appender | [`LogstashTcpSocketAppender`](/src/main/java/net/logstash/logback/appender/LogstashTcpSocketAppender.java) | [`LogstashAccessTcpSocketAppender`](/src/main/java/net/logstash/logback/appender/LogstashAccessTcpSocketAppender.java)
| Logstash JSON | TCP/SSL    | Appender | [`SSLLogstashTcpSocketAppender`](/src/main/java/net/logstash/logback/appender/SSLLogstashTcpSocketAppender.java) | n/a
| Logstash JSON | any        | Encoder  | [`LogstashEncoder`](/src/main/java/net/logstash/logback/encoder/LogstashEncoder.java) | [`LogstashAccessEncoder`](/src/main/java/net/logstash/logback/encoder/LogstashAccessEncoder.java)
| Logstash JSON | any        | Layout   | [`LogstashLayout`](/src/main/java/net/logstash/logback/layout/LogstashLayout.java) | [`LogstashAccessLayout`](/src/main/java/net/logstash/logback/layout/LogstashAccessLayout.java)
| General JSON  | any        | Encoder  | [`LoggingEventCompositeJsonEncoder`](/src/main/java/net/logstash/logback/encoder/LoggingEventCompositeJsonEncoder.java) | [`AccessEventCompositeJsonEncoder`](/src/main/java/net/logstash/logback/encoder/AccessEventCompositeJsonEncoder.java)
| General JSON  | any        | Layout   | [`LoggingEventCompositeJsonLayout`](/src/main/java/net/logstash/logback/layout/LoggingEventCompositeJsonLayout.java) | [`AccessEventCompositeJsonLayout`](/src/main/java/net/logstash/logback/encoder/AccessEventCompositeJsonLayout.java)

These encoders/layouts can generally be used by any logback appender (such as `RollingFileAppender`).

The general _composite_ JSON encoders/layouts can be used to output any JSON format/data by configuring them with various JSON _providers_.  The Logstash encoders/layouts are really just extensions of the general composite JSON encoders/layouts with a pre-defined set of providers.

The logstash encoders/layouts are easier to configure if you want to use the standard output format.  Use the composite encoders/layouts if you want to heavily customize the output.



### Syslog UDP Socket Appender

To output JSON for LoggingEvents to a syslog/UDP channel, use the `LogstashSocketAppender` in your `logback.xml` like this:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <appender name="stash" class="net.logstash.logback.appender.LogstashSocketAppender">
    <host>MyAwesomeSyslogServer</host>
    <!-- port is optional (default value shown) -->
    <port>514</port>
  </appender>
  <root level="all">
    <appender-ref ref="stash" />
  </root>
</configuration>
```
Internally, the `LogstashSocketAppender` uses a `LogstashLayout` to perform the JSON formatting.   Therefore, by default, the output will be logstash-compatible.  You can further customize the JSON output of `LogstashSocketAppender` just like you can with a `LogstashLayout` as described in later sections.

There currently is no way to log AccessEvents over syslog/UDP.

To receive syslog/UDP input in logstash, configure a [`syslog`](www.logstash.net/docs/latest/inputs/syslog) or [`udp`](www.logstash.net/docs/latest/inputs/udp) input with the [`json`](www.logstash.net/docs/latest/codecs/json) codec in logstash's configuration like this:
```
input {
  syslog {
    codec => "json"
  }
}
```
 
### TCP Socket Appender

To output JSON for LoggingEvents over TCP use the `LogstashEncoder` along with the `LogstashTcpSocketAppender` or `SSLLogstashTcpSocketAppender`.

Example logging appender configuration in `logback.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <appender name="stash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
      <!-- remoteHost and port are optional (default values shown) -->
      <remoteHost>127.0.0.1</remoteHost>
      <port>4560</port>
  
      <!-- encoder is required -->
      <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
  </appender>
  
  <root level="DEBUG">
      <appender-ref ref="stash" />
  </root>
</configuration>
```

To output JSON for AccessEvents over TCP, use the `LogstashAccessEncoder` along with the `LogstashTcpSocketAppender`.  An SSL appender is not currently available for AccessEvents.

Example access appender in `logback-access.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <appender name="stash" class="net.logstash.logback.appender.LogstashAccessTcpSocketAppender">
      <!-- remoteHost and port are optional (default values shown) -->
      <remoteHost>127.0.0.1</remoteHost>
      <port>4560</port>
  
      <!-- encoder is required -->
      <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
  </appender>

  <appender-ref ref="stash" />
</configuration>
```

To receive TCP input in logstash, configure a [`tcp`](www.logstash.net/docs/latest/inputs/tcp) input with the [`json`](www.logstash.net/docs/latest/codecs/json) codec in logstash's configuration like this:

```
input {
    tcp {
        port => 4560
        codec => json
    }
}
```

### Encoder / Layout

You can use the any encoders/layouts provided by the logstash-logback-encoder library with other logback appenders.

For example, to output LoggingEvents to a file, use the `LogstashEncoder` with the `RollingFileAppender` in your `logback.xml` like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <appender name="stash" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
      <level>info</level>
    </filter>
    <file>/some/path/to/your/file.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>/some/path/to/your/file.log.%d{yyyy-MM-dd}</fileNamePattern>
      <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
  </appender>
  <root level="all">
    <appender-ref ref="stash" />
  </root>
</configuration>
```

To log AccessEvents to a file, configure your `logback-access.xml` like this:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <appender name="stash" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/some/path/to/your/file.log</file>
    <encoder class="net.logstash.logback.encoder.LogstashAccessEncoder" />
  </appender>

  <appender-ref ref="stash" />
</configuration>
```

To receive file input in logstash, configure a [`file`](www.logstash.net/docs/latest/inputs/file) input in logstash's configuration like this:

```
input {
  file {
    path => "/some/path/to/your/file.log"
    codec => "json"
  }
}
```


## LoggingEvent Fields

The fields included in the JSON output by default for LoggingEvents written by the `LogstashEncoder`, `LogstashLayout`, and the logstash appenders are as follows...

### Standard Fields

These fields will appear in every LoggingEvent unless otherwise noted.
The field names listed here are the default field names.
The field names can be customized (see below).

| Field         | Description
|---------------|------------
| `@timestamp`  | Time of the log event. (`yyyy-MM-dd'T'HH:mm:ss.SSSZZ`)  See below for customizing the timezone.
| `@version`    | Logstash format version (e.g. 1)
| `message`     | Formatted log message of the event 
| `logger_name` | Name of the logger that logged the event
| `thread_name` | Name of the thread that logged the event
| `level`       | String name of the level of the event
| `level_value` | Integer value of the level of the event
| `stack_trace` | (Only if a throwable was logged) The stacktrace of the throwable.  Stackframes are separated by line endings.
| `tags`        | (Only if tags are found) The names of any markers not explicitly handled.  (e.g. markers from `MarkerFactory.getMarker` will be included as tags, but the markers from [`Markers`](/src/main/java/net/logstash/logback/marker/Markers.java) will not.)

### MDC fields

By default, each entry in the Mapped Diagnostic Context (MDC) (`org.slf4j.MDC`)
will appear as a field in the LoggingEvent.

This can be fully disabled by specifying `<includeMdc>false</includeMdc>`,
in the encoder/layout/appender configuration.

You can also configure specific entries in the MDC to be included or excluded as follows:

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <includeMdcKeyName>key1ToInclude</includeMdcKeyName>
  <includeMdcKeyName>key2ToInclude</includeMdcKeyName>
</encoder>
```
or

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <excludeMdcKeyName>key1ToExclude</excludeMdcKeyName>
  <excludeMdcKeyName>key2ToExclude</excludeMdcKeyName>
</encoder>
```

When key names are specified for inclusion, then all other fields will be excluded.
When key names are specified for exclusion, then all other fields will be included.
It is a configuration error to specify both included and excluded key names.

### Context fields

By default, each property of Logback's Context (`ch.qos.logback.core.Context`)
will appear as a field in the LoggingEvent.
This can be disabled by specifying `<includeContext>false</includeContext>`
in the encoder/layout/appender configuration.

### Caller Info Fields
The encoder/layout/appender do not contain caller info by default. 
This can be costly to calculate and should be switched off for busy production environments.

To switch it on, add the `includeCallerInfo` property to the configuration.
```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <includeCallerInfo>true</includeCallerInfo>
</encoder>
```

OR

```xml
<appender name="stash" class="net.logstash.logback.appender.LogstashSocketAppender">
  <includeCallerInfo>true</includeCallerInfo>
</appender>
```

When switched on, the following fields will be included in the log event:

| Field                | Description
|----------------------|------------
| `caller_class_name`  | Fully qualified class name of the class that logged the event
| `caller_method_name` | Name of the method that logged the event
| `caller_file_name`   | Name of the file that logged the event
| `caller_line_number` | Line number of the file where the event was logged

### Custom Fields

In addition to the fields above, you can add other fields to the LoggingEvent either globally, or on an event-by-event basis.

#### Global Custom Fields

Add custom fields that will appear in every LoggingEvent like this : 
```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <customFields>{"appname":"myWebservice","roles":["customerorder","auth"],"buildinfo":{"version":"Version 0.1.0-SNAPSHOT","lastcommit":"75473700d5befa953c45f630c6d9105413c16fe1"}}</customFields>
</encoder>
```

OR

```xml
<appender name="stash" class="net.logstash.logback.appender.LogstashSocketAppender">
  <customFields>{"appname":"myWebservice","roles":["customerorder","auth"],"buildinfo":{"version":"Version 0.1.0-SNAPSHOT","lastcommit":"75473700d5befa953c45f630c6d9105413c16fe1"}}</customFields>
</appender>
```

#### Event-specific Custom Fields

When logging a message, you can specify additional fields to add to the LoggingEvent by using the markers provided by 
[`Markers`](/src/main/java/net/logstash/logback/marker/Markers.java).

For example:

```java
import static net.logstash.logback.marker.Markers.*

/*
 * Add "name":"value" to the json event.
 */
logger.info(append("name", "value"), "log message");

/*
 * Add "name1":"value1","name2":"value2" to the json event by using multiple markers.
 */
logger.info(append("name1", "value1").and(append("name2", "value2")), "log message");

/*
 * Add "name1":"value1","name2":"value2" to the json event by using a map.
 *
 * Note the values can be any object that can be serialized by Jackson's ObjectMapper
 * (e.g. other Maps, JsonNodes, numbers, arrays, etc)
 */
Map myMap = new HashMap();
myMap.put("name1", "value1");
myMap.put("name2", "value2");
logger.info(appendEntries(myMap), "log message");

/*
 * Add "array":[1,2,3] to the json event
 */
logger.info(appendArray("array", 1, 2, 3), "log message");

/*
 * Add "array":[1,2,3] to the json event by using raw json.
 * This allows you to use your own json seralization routine to construct the json output
 */
logger.info(appendRaw("array", "[1,2,3]"), "log message");

/*
 * Add any object that can be serialized by Jackson's ObjectMapper
 * (e.g. Maps, JsonNodes, numbers, arrays, etc)
 */
logger.info(append("object", myobject), "log message");

/*
 * Add fields of any object that can be unwrapped by Jackson's UnwrappableBeanSerializer.
 * i.e. The fields of an object can be written directly into the json output.
 * This is similar to the @JsonUnwrapped annotation.
 */
logger.info(appendFields(myobject), "log message");

```

See [DEPRECATED.md](DEPRECATED.md) for other deprecated ways of adding json to the output.


## AccessEvent Fields

The fields included in the JSON output by default for AccessEvents written by the `LogstashAccessEncoder`, `LogstashAccessLayout`, and the logstash access appenders are as follows...

### Standard Fields

These fields will appear in every AccessEvent unless otherwise noted.
The field names listed here are the default field names.
The field names can be customized (see below).

| Field         | Description
|---------------|------------
| `@timestamp`  | Time of the log event. (`yyyy-MM-dd'T'HH:mm:ss.SSSZZ`)  See below for customizing the timezone.
| `@version`    | Logstash format version (e.g. 1)
| `@message`     | Message in the form `${remoteHost} - ${remoteUser} [${timestamp}] "${requestUrl}" ${statusCode} ${contentLength}`
| `@fields.method` | HTTP method
| `@fields.protocol` | HTTP protocol
| `@fields.status_code` | HTTP status code
| `@fields.requested_url` | Request URL
| `@fields.requested_uri` | Request URI
| `@fields.remote_host` | Remote host
| `@fields.HOSTNAME` | another field for remote host (not sure why this is here honestly)
| `@fields.remote_user` | Remote user
| `@fields.content_length` | Content length
| `@fields.elapsed_time` | Elapsed time in millis

### Header Fields

Request and response headers are not logged by default, but can be enabled by specifying a field name for them, like this:

```xml
<encoder class="net.logstash.logback.encoder.LogstashAccessEncoder">
  <fieldNames>
    <fieldsRequestHeaders>@fields.request_headers</fieldsRequestHeaders>
    <fieldsResponseHeaders>@fields.response_headers</fieldsResponseHeaders>
  </fieldNames>
</encoder>
```

See _Customizing Standard Field Names_ for more details.


## Customizing Standard Field Names

The standard field names above for LoggingEvents and AccessEvents can be customized by using the `fieldNames`configuration element in the encoder or appender configuration.

For example:
```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <fieldNames>
    <timestamp>time</timestamp>
    <message>msg</message>
    ...
  </fieldNames>
</encoder>
```
Prevent a field from being output by setting the field name to `[ignore]`.

For LoggingEvents, see [`LogstashFieldNames`](/src/main/java/net/logstash/logback/fieldnames/LogstashFieldNames.java)
for all the field names that can be customized.  Additionally, a separate set of [shortened field names](/src/main/java/net/logstash/logback/fieldnames/ShortenedFieldNames.java) can be configured like this:
```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <fieldNames class="net.logstash.logback.fieldnames.ShortenedFieldNames"/>
</encoder>
```

For LoggingEvents, log the caller info, MDC properties, and context properties
in sub-objects within the JSON event by specifying field
names for `caller`, `mdc`, and `context`, respectively.

For AccessEvents, see [`LogstashAccessFieldNames`](/src/main/java/net/logstash/logback/fieldnames/LogstashAccessFieldNames.java)
for all the field names that can be customized
 
## Customizing TimeZone

By default, timestamps are logged in the default TimeZone of the host Java platform.
You can change the timezone like this:

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <timeZone>UTC</timeZone>
</encoder>
```

The value of the `timeZone` element can be any string accepted by java's  `TimeZone.getTimeZone(String id)` method.


## Customizing JSON Factory and Generator

The `JsonFactory` and `JsonGenerator` used to serialize output can be customized by creating
custom instances of [`JsonFactoryDecorator`](/src/main/java/net/logstash/logback/decorate/JsonFactoryDecorator.java)
or [`JsonGeneratorDecorator`](/src/main/java/net/logstash/logback/decorate/JsonGeneratorDecorator.java), respectively.

For example, you could enable pretty printing like this:
```java
public class PrettyPrintingDecorator implements JsonGeneratorDecorator {

    @Override
    public JsonGenerator decorate(JsonGenerator generator) {
        return generator.useDefaultPrettyPrinter();
    }

}
```

and then specify your decorator in the logback.xml file like this:

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <jsonGeneratorDecorator class="your.package.PrettyPrintingDecorator"/>
</encoder>
```

## Customizing Logger Name Length

For LoggingEvents, you can shorten the logger name field length similar to the normal pattern style of `%logger{36}`.
Examples of how it is shortened can be found [here](http://logback.qos.ch/manual/layouts.html#conversionWord)

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <shortenedLoggerNameLength>36</shortenedLoggerNameLength>
</encoder>
```

## Customizing Stack Traces

For LoggingEvents, stack traces are formatted using logback's `ExtendedThrowableProxyConverter` by default.
However, you can configure the encoder to use any `ThrowableHandlingConverter`
to format stacktraces.

A powerful [`ShortenedThrowableConverter`](/src/main/java/net/logstash/logback/stacktrace/ShortenedThrowableConverter.java)
is included in the logstash-logback-encoder library to format stacktraces by:

* Limiting the number of stackTraceElements per throwable (applies to each individual throwable.  e.g. caused-bys and suppressed)
* Limiting the total length in characters of the trace
* Abbreviating class names
* Filtering out consecutive unwanted stackTraceElements based on regular expressions.
* Using evaluators to determine if the stacktrace should be logged.
* Outputing in either 'normal' order (root-cause-last), or root-cause-first.

For example:

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
    <maxDepthPerThrowable>30</maxDepthPerThrowable>
    <maxLength>2048</maxLength>
    <shortenedClassNameLength>20</shortenedClassNameLength>
    <exclude>sun\.reflect\..*\.invoke.*</exclude>
    <exclude>net\.sf\.cglib\.proxy\.MethodProxy\.invoke</exclude>
    <evaluator class="myorg.MyCustomEvaluator"/>
    <rootCauseFirst>true</rootCauseFirst>
  </throwableConverter>
</encoder>
```

[`ShortenedThrowableConverter`](/src/main/java/net/logstash/logback/stacktrace/ShortenedThrowableConverter.java)
can even be used within a `PatternLayout` to format stacktraces in any non-JSON logs you may have.

## Composite Encoder/Layout

If you want greater flexibility in the JSON format and data included in LoggingEvents and AccessEvents, use the [`LoggingEventCompositeJsonEncoder`](/src/main/java/net/logstash/logback/encoder/LoggingEventCompositeJsonEncoder.java)  and  [`AccessEventCompositeJsonEncoder`](/src/main/java/net/logstash/logback/encoder/AccessEventCompositeJsonEncoder.java)  (or the corresponding layouts).

These encoders/layouts are composed of one or more JSON _providers_ that contribute to the JSON output.  No providers are configured by default in the composite encoders/layouts.  You must add the ones you want.

For example:

```xml
<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
  <providers>
    <mdc/>
    <pattern>
      <pattern>
        {
          "timestamp": "%date{ISO8601}"
          "myCustomField": "fieldValue",
          "relative": "#asLong{%relative}"
        }
      </pattern>
    </pattern>
    <stackTrace>
      <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
        <maxDepthPerThrowable>30</maxDepthPerThrowable>
        <maxLength>2048</maxLength>
        <shortenedClassNameLength>20</shortenedClassNameLength>
        <exclude>sun\.reflect\..*\.invoke.*</exclude>
        <exclude>net\.sf\.cglib\.proxy\.MethodProxy\.invoke</exclude>
        <evaluator class="myorg.MyCustomEvaluator"/>
        <rootCauseFirst>true</rootCauseFirst>
      </throwableConverter>
    </stackTrace>
  </providers>
</encoder>
```


The logstash-logback-encoder library contains many providers out-of-the-box,
and you can even plug-in your own.
Each provider has its own configuration options to further customize it. 

For LoggingEvents, the available providers and their configuration properties (defaults in parenthesis) are as follows:

<table>
  <tbody>
    <tr>
      <th>Provider</th>
      <th>Description/Properties</th>
      <th>Example Output</th>
    </tr>
    <tr>
      <td><tt>timestamp</tt></td>
      <td><p>Event timestamp</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>@timestamp</tt>)</li>
          <li><tt>pattern</tt> - Output format (<tt>yyyy-MM-dd'T'HH:mm:ss.SSSZZ</tt>)</li>
          <li><tt>timeZone</tt> - Timezone (local timezone)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>version</tt></td>
      <td><p>Logstash JSON format version</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>@version</tt>)</li>
          <li><tt>version</tt> - Output value (<tt>1</tt>)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>message</tt></td>
      <td><p>Formatted log event message</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>message</tt>)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>loggerName</tt></td>
      <td><p>Name of the logger that logged the message</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>logger_name</tt>)</li>          
          <li><tt>shortenedLoggerNameLength</tt> - Length to which the name will be attempted to be abbreviated (no abbreviation)</li>          
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>threadName</tt></td>
      <td><p>Name of the thread from which the event was logged</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>thread_name</tt>)</li>          
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>logLevel</tt></td>
      <td><p>Logger level text (INFO, WARN, etc)</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>level</tt>)</li>          
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>logLevelValue</tt></td>
      <td><p>Logger level numerical value </p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>level_value</tt>)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>callerData</tt></td>
      <td><p>Outputs data about from where the logger was called (class/method/file/line)
        <ul>
          <li><tt>fieldName</tt> - Sub-object field name (no sub-object)</li>
          <li><tt>classFieldName</tt> - Field name for class name (<tt>caller_class_name</tt>)</li>
          <li><tt>methodFieldName</tt> - Field name for method name (<tt>caller_method_name</tt>)</li>
          <li><tt>fileFieldName</tt> - Field name for file name (<tt>caller_file_name</tt>)</li>
          <li><tt>lineFieldName</tt> - Field name for lin number (<tt>caller_line_number</tt>)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>stackTrace</tt></td>
      <td><p>Stacktrace of any throwable logged with the event.  Stackframes are separated by newline chars.</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>stack_trace</tt>)</li>
          <li><tt>throwableConverter</tt> - The <tt>ThrowableHandlingConverter</tt> to use to format the stacktrace (<tt>stack_trace</tt>)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>context</tt></td>
      <td><p>Outputs entries from logback's context</p>
        <ul>
          <li><tt>fieldName</tt> - Sub-object field name (no sub-object)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>mdc</tt></td>
      <td>
        <p>Outputs entries from the Mapped Diagnostic Context (MDC).
           Will include all entries by default.
           When key names are specified for inclusion, then all other fields will be excluded.
           When key names are specified for exclusion, then all other fields will be included.
           It is a configuration error to specify both included and excluded key names.
        </p>
        <ul>
          <li><tt>fieldName</tt> - Sub-object field name (no sub-object)</li>
          <li><tt>includeMdcKeyName</tt> - Name of keys to include (all)</li>
          <li><tt>excludeMdcKeyName</tt> - Name of keys to include (none)</li>
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>tags</tt></td>
      <td><p>Outputs logback markers as a comma separated list</p>
        <ul>
          <li><tt>fieldName</tt> - Output field name (<tt>tags</tt>)</li>          
        </ul>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>logstashMarkers</tt></td>
      <td><p>Used to output Logstash Markers as specified in <em>Event-specific Custom Fields</em></p>
      </td>
      <td></td>
    </tr>
    <tr>
      <td><tt>pattern</tt></td>
      <td>
        <p>Outputs fields from a configured JSON Object string,
           while substituting patterns supported by logback's <tt>PatternLayout</tt>.
        </p>
        <p>
           See <em>Pattern JSON Provider</em>.
        </p>
        <ul>
          <li><tt>pattern</tt> - JSON object string (no default)</li>          
        </ul>
      </td>
      <td></td>
    </tr>
  </tbody>
</table>

### Pattern JSON Provider

When used with a composite JSON encoder/layout, the `pattern` JSON provider can be used to
define a template of JSON for the event.  The encoder/layout will populate values within the template.
Every value in the template is treated as a pattern for logback's standard `PatternLayout` so it can be a combination
of literal strings (for some constants) and various conversion specifiers (like `%d` for date).

This example...

```xml
<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
  <providers>
    <timestamp/>
    <version/>
    <pattern>
        { "level": "%level" }
    </pattern>
  </providers>
</encoder>
```
... will produce something like...

```
{
  "@timestamp":"...",
  "@version": 1,
  "level": "DEBUG"
}
```

The real power comes from the fact that there are lots of standard conversion specifiers so you
can customise what is logged and how. For example, you could log a single specific value from MDC with `%mdc{mykey}`.
Or, for access logs, you could log a single request header with `%i{User-Agent}`.

You can use nested objects and arrays in your pattern.

If you use a null, number, or a boolean constant in a pattern, it will keep its type in the
resulting JSON. However, only the text values are searched for conversion patterns.
And, as these patterns are sent through `PatternLayout`, the result is always a string
even for something which you may feel should be a number - like for `%b` (bytes sent, in access logs).

You can either deal with the type conversion on the logstash side or you may use special operations provided by this encoder.
The operations are:

* `#asLong{...}` - evaluates the pattern in curly braces and then converts resulting string to a long (or a null if conversion fails).
* `#asDouble{...}` - evaluates the pattern in curly braces and then converts resulting string to a double (or a null if conversion fails).

So this example...

```xml
<pattern>
  {
    "bytes_sent_str": "%b",
    "bytes_sent_long": "#asLong{%b}"
  }
</pattern>
```
...will produce something like...

```
{
  "bytes_sent_str": "1024",
  "bytes_sent_long": 1024
}
```

The value that is sent for `bytes_sent_long` is a number even though in your pattern it is a quoted text.

#### LoggingEvent patterns

For LoggingEvents, patterns from logback-classic's
[`PatternLayout`](http://logback.qos.ch/manual/layouts.html#conversionWord) are supported.

For example:

```xml
<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
  <providers>
    <timestamp/>
    <pattern>
        {
        "custom_constant": "123",
        "tags": ["one", "two"],
        "logger": "%logger",
        "level": "%level",
        "thread": "%thread",
        "message": "%message",
...
        }
    </pattern>
  </providers>
</encoder>
```

#### AccessEvent patterns

For AccessEvents, patterns from logback-access's
[`PatternLayout`](http://logback.qos.ch/xref/ch/qos/logback/access/PatternLayout.html) are supported.

For example:  

```xml
<encoder class="net.logstash.logback.encoder.AccessEventCompositeJsonEncoder">
  <providers>
    <pattern>
        {
        "custom_constant": "123",
        "tags": ["one", "two"],
        "remote_ip": "%a",
        "status_code": "%s",
        "elapsed_time": "%D",
        "user_agent": "%i{User-Agent}",
        "accept": "%i{Accept}",
        "referer": "%i{Referer}",
        "session": "%requestCookie{JSESSIONID}",
...
        }
    </pattern>
  </providers>
</encoder>
```

Note that the latest Logback (1.1.2 at the moment of writing), does not support deferred processing of
request attributes. And because `LogstashAccessTcpSocketAppender` defers processing of the event to a background thread,
you won't be able to use "%requestAttribute{name}" with TCP appender until this issue is fixed (or you build yourself
a custom logback). See http://jira.qos.ch/browse/LOGBACK-1033

There is also a special operation that can be used with this AccessEvents:

* `#nullNA{...}` - if the pattern in curly braces evaluates to a dash ("-"), it will be replaced with a null value.

You may want to use it because many of the `PatternLayout` conversion specifiers from logback-access will evaluate to "-"
for non-existent value (for example for a cookie, header or a request attribute).

So the following pattern...

```xml
<pattern>
  {
    "default_cookie": "%requestCookie{MISSING}",
    "filtered_cookie": "#nullNA{%requestCookie{MISSING}}"
  }
</pattern>
```

...will produce...

```
{
  "default_cookie": "-",
  "filtered_cookie": null
}
```

## Build status
[![Build Status](https://travis-ci.org/logstash/logstash-logback-encoder.svg?branch=master)](https://travis-ci.org/logstash/logstash-logback-encoder)

