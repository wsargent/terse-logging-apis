[<img src="https://img.shields.io/travis/tersesystems/terse-logback.svg"/>](https://travis-ci.org/tersesystems/terse-logback) 
[ ![Download](https://api.bintray.com/packages/tersesystems/maven/terse-logback/images/download.svg?version=0.1.6) ](https://bintray.com/tersesystems/maven/terse-logback/0.1.6/link)

# Structured Logging Example with Logback

This is a Java project that shows how to use [Logback](https://logback.qos.ch/manual/index.html) effectively for structured logging.  It should show how you configure Logback, and how you can reduce the amount of complexity in your end projects by packaging your logging appenders and configurators in a distinct project.

## Project Setup

The project is configured into several modules.  The most relevant ones to start with are `structured-config` and `example`.

The `structured-config` module contains all the logback code and the appenders, and is intended to be deployed as a small helper library for your other projects, managed through Maven and an artifact manager, or just by packaging the JAR.  The `example` project depends on `structured-config`, and contains the "end user" experience where log levels are adjusted and JSON can be pretty printed or not.

Notably, the `example` project cannot touch the appenders directly, and has no control over the format of the JSON appender -- console and text patterns can be overridden for developer convenience.  By enforcing a [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) between **logger configuration** and **logging levels**, it is easy and simple to manage appenders in one place, e.g. going from file appenders to TCP appenders, adding filters for sensitive information, or collapsing repeated log information.

The `guice-example` shows a logback factory that is exposed through a `Provider` in Guice.

This is not intended to be a drop in replacement or a straight library dependency.  You will want to modify this to your own tastes.

## What is Structured Logging?

It's logging in JSON.  Technically, you could be logging in another structure like XML or JSON, but almost everyone uses JSON.  It's been around for [a while](https://www.kartar.net/2015/12/structured-logging/).  Technically, since there are several JSON objects all in one file / stream, this is called "newline delimited JSON" or [NDJSON](http://ndjson.org/) or [jsonlines](http://jsonlines.org/).  In this project, both text and JSON formats are rendered independently, but if you only output JSON it's not a huge deal, because you can read JSON logs as text with a special log viewer such as [jl](https://github.com/koenbollen/jl/blob/master/README.md).

Semantically, a log entry typically has multiple pieces of information associated with it, described as "high cardinality" by observability geeks.  Structured logging means that the cardinality goes from "closed" -- you can only log things that you have defined fields for -- to "open", where you can add arbitrary fields and objects to your log entry as long as it's JSON.

Structured logging means that you can add more context to logs and do more with them without having to do regexes.  As [Honeycomb](https://honeycomb.io) [describes it](
https://www.honeycomb.io/blog/you-could-have-invented-structured-logging/):

> Structured logging is really all about giving yourself — and your team — a logging API to help you provide consistent context in events. An unstructured logger accepts strings. A structured logger accepts a map, hash, or dictionary that describes all the attributes you can think of for an event.

Logs are [different from events](https://www.honeycomb.io/blog/how-are-structured-logs-different-from-events/).  All events can be represented as logs, but not all logs are events.  Many logs are only portions of events.  An event is a conceptual abstraction and a log is one possible representation of that abstraction.

Logs are also different from metrics.  A metric represents a single number.  You can extract metrics from logs, but it's a very expensive way of going about it.

## Adding Context

There is a question of what you want to add when you log.  This is a matter of taste, but in general you should log so that you [create a consistent narrative](https://www.honeycomb.io/blog/event-foo-constructing-a-coherent-narrative/).  As previously mentioned, a log may indicate a portion of an event, so you want to log where doing so would help tell a story of what happened afterwards.

There are some things you should [always add to an event](https://www.honeycomb.io/blog/event-foo-what-should-i-add-to-an-event/), such as who is talking to your service, what they're asking, business relevant fields, additional context around your service / environment, response time and particulars. You should add units to your field names when you measure a quantity, i.e. `response_time_ms`, and add a "human readable" version of internal information if available.

You should add [context to your logs](https://www.honeycomb.io/blog/event-foo-moar-context-better-events/) that helps differentiate it from its peers, so you never have to guess where the source of a log is coming from.

Adding a [correlation id](https://blog.rapid7.com/2016/12/23/the-value-of-correlation-ids/) helps you [design for results](https://www.honeycomb.io/blog/event-foo-designing-for-results/) and tie your logs into a coherent event.  You don't need to use a UUID: a [flake id](https://github.com/boundary/flake) will probably be better for you.  I'm using [idem](https://github.com/mguenther/idem/) here, but most things will work.

So, we know what structured logging is now.  What does it look like in SLF4J?

## Adding Structure to Logging

SLF4J doesn't have specific support for structured logging, but [logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder/tree/logstash-logback-encoder-5.2#logback-json-encoder) does.  It's complete and comprehensive, but buried in a section called [Event specific custom fields](https://github.com/logstash/logstash-logback-encoder/tree/logstash-logback-encoder-5.2#event-specific-custom-fields).

Event specific custom fields are implemented in two ways: through [`net.logstash.logback.argument.StructuredArguments`](https://github.com/logstash/logstash-logback-encoder/blob/logstash-logback-encoder-5.2/src/main/java/net/logstash/logback/argument/StructuredArguments.java), which adds structured information through parameters, and [`net.logstash.logback.marker.Markers`](https://github.com/logstash/logstash-logback-encoder/blob/logstash-logback-encoder-5.2/src/main/java/net/logstash/logback/marker/Markers.java), which adds structured information through the `org.slf4j.Marker` API.

### StructuredArguments

`StructuredArguments` write out both to the text appenders and to the JSON appenders.  There is extra "key information" added to the JSON, and you see the value show up in the message.

```java
package example;

import org.slf4j.Logger;

import static net.logstash.logback.argument.StructuredArguments.*;
import static org.slf4j.LoggerFactory.*;

public class ClassWithStructuredArguments {
    private final Logger logger = getLogger(getClass());

    public void logValue(String correlationId) {
        if (logger.isInfoEnabled()) {
            logger.info("id is {}", value("correlationId", correlationId));
        }
    }

    public void logNameAndValue(String correlationId) {
        logger.info("id is {}", keyValue("correlationId", correlationId));
    }

    public void logNameAndValueWithFormat(String correlationId) {
        logger.info("id is {}", keyValue("correlationId", correlationId, "{0}=[{1}]"));
    }

    public void doThings(String correlationId) {
        logValue(correlationId);
        logNameAndValue(correlationId);
        logNameAndValueWithFormat(correlationId);
    }

    public static void main(String[] args) {
        String correlationId = IdGenerator.getInstance().generateCorrelationId();
        ClassWithStructuredArguments classWithStructuredArguments = new ClassWithStructuredArguments();
        classWithStructuredArguments.doThings(correlationId);
    }
}
```

This produces the following output in text:

```text
2019-01-20T23:24:40.004+0000 [INFO ] example.ClassWithStructuredArguments in main - id is FXtylIyzDbj9rfs7BRCAAA
2019-01-20T23:24:40.006+0000 [INFO ] example.ClassWithStructuredArguments in main - id is correlationId=FXtylIyzDbj9rfs7BRCAAA
2019-01-20T23:24:40.006+0000 [INFO ] example.ClassWithStructuredArguments in main - id is correlationId=[FXtylIyzDbj9rfs7BRCAAA]
```

and in JSON:

```json
{"@timestamp":"2019-01-20T23:24:40.004+00:00","@version":"1","message":"id is FXtylIyzDbj9rfs7BRCAAA","logger_name":"example.ClassWithStructuredArguments","thread_name":"main","level":"INFO","level_value":20000,"correlationId":"FXtylIyzDbj9rfs7BRCAAA"}
{"@timestamp":"2019-01-20T23:24:40.006+00:00","@version":"1","message":"id is correlationId=FXtylIyzDbj9rfs7BRCAAA","logger_name":"example.ClassWithStructuredArguments","thread_name":"main","level":"INFO","level_value":20000,"correlationId":"FXtylIyzDbj9rfs7BRCAAA"}
{"@timestamp":"2019-01-20T23:24:40.006+00:00","@version":"1","message":"id is correlationId=[FXtylIyzDbj9rfs7BRCAAA]","logger_name":"example.ClassWithStructuredArguments","thread_name":"main","level":"INFO","level_value":20000,"correlationId":"FXtylIyzDbj9rfs7BRCAAA"}
```

### Markers

If you want to add more context and don't want it to show up in the message, you can use [`net.logstash.logback.marker.Markers`](https://github.com/logstash/logstash-logback-encoder/blob/logstash-logback-encoder-5.2/src/main/java/net/logstash/logback/marker/Markers.java) instead.

```java
package example;

import net.logstash.logback.marker.LogstashMarker;
import net.logstash.logback.marker.Markers;
import org.slf4j.Logger;

import static org.slf4j.LoggerFactory.getLogger;

public class ClassWithMarkers {
    private final Logger logger = getLogger(getClass());

    public void doThingsWithMarker(String correlationId) {
        LogstashMarker logstashMarker = Markers.append("correlationId", correlationId);
        logger.info(logstashMarker, "log with marker explicitly");
    }

    public static void main(String[] args) {
        String correlationId = IdGenerator.getInstance().generateCorrelationId();
        ClassWithMarkers classWithMarkers = new ClassWithMarkers();
        classWithMarkers.doThingsWithMarker(correlationId);
    }
}
```

This produces the following text:

```text
2019-01-20T23:26:50.351+0000 [INFO ] example.ClassWithMarkers in main - log with marker explicitly
```

and the following JSON:

```json
{"@timestamp":"2019-01-20T23:26:50.351+00:00","@version":"1","message":"log with marker explicitly","logger_name":"example.ClassWithMarkers","thread_name":"main","level":"INFO","level_value":20000,"correlationId":"FXtylIy0T878gCNIdfWAAA"}
```

If you don't want to pass through anything at all, and instead use a proxy logger, you can use `com.tersesystems.logback.context.logstash.LogstashLogger`, which applies it under the hood.  Adding state to the logger is one of those useful tricks that can make life easier, as long as you implement `org.slf4j.Logger` and don't expose your logger to the world.  This is discussed in the "Logging with Injected Context" section.

## Controlling Logging

The SLF4J API code is built around the `org.slf4j.Logger` interface.  This is extremely useful, because it means that you can put different implementations behind that interface, and the core codebase will work the same.  In particular, we've seen that we can extend code around that concept.  There is one impediment, which is that the `Logger` API has a bunch of boilerplate related to handling the same logic at different levels.

Internally, the project uses [Javapoet](https://github.com/square/javapoet) to handle the boilerplate and generate code for all the levels so that writing implementation only has to be done once, as opposed to for `trace`, `debug`, `warn`, `info`, and `error` levels.  The code is available under the `slf4j-gen` project.

### LazyLogger, ConditionalLogger

There's two APIs that extend on top of `org.slf4j.Logger`, the `LazyLogger` and the `ConditionalLogger`.

The LazyLogger API looks like this:

```java
public interface LazyLogger {
        void debug(Consumer<LoggerStatement> lc);
        
        Optional<LoggerStatement> debug();
    
        void debug(Marker marker, Consumer<LoggerStatement> lc);
        
        Optional<LoggerStatement> debug(Marker marker);
        
        // ...
}
```

The ConditionalLogger API looks like this, where a condition is provided as a function to the method:

```java
public interface ConditionalLogger {
    
    void ifTrace(Supplier<Boolean> condition, Consumer<LoggerStatement> lc);

    void ifTrace(Marker marker, Supplier<Boolean> condition, Consumer<LoggerStatement> lc);

    Optional<LoggerStatement> ifTrace(Supplier<Boolean> condition);

    Optional<LoggerStatement> ifTrace(Marker marker, Supplier<Boolean> condition);
    
    // ...
}
```

There are two implementations of these APIs, one based on a straight proxy, and another based on predicates.

### Proxy Loggers

The proxy implementation takes an underlying logger, and passes the behavior through.

The lazy logger is implemented as follows:

```java
public interface ProxyLazyLogger extends LazyLogger {

    Logger logger();

    default void trace(Consumer<LoggerStatement> lc) {
        if (logger().isTraceEnabled()) {
            LoggerStatement stmt = new LoggerStatement.Trace(logger());
            lc.accept(stmt);
        }
    }

    default Optional<LoggerStatement> trace() {
        if (logger().isTraceEnabled()) {
            LoggerStatement stmt = new LoggerStatement.Trace(logger());
            return Optional.of(stmt);
        } else {
            return Optional.empty();
        }
    }
    
    // ...
}
```

and the conditional logger implementation is similar:

```java
public interface ProxyConditionalLogger extends ConditionalLogger {

    Logger logger();

    @Override
    default void ifTrace(Supplier<Boolean> condition, Consumer<LoggerStatement> lc) {
        if (logger().isTraceEnabled() && condition.get() ) {
            lc.accept(new LoggerStatement.Trace(logger()));
        }
    }

    @Override
    default void ifTrace(Marker marker, Supplier<Boolean> condition, Consumer<LoggerStatement> lc) {
        if (logger().isTraceEnabled(marker) && condition.get() ) {
            lc.accept(new LoggerStatement.Trace(logger()));
        }
    }
    
    // ...
}
```

Usually you will put all of them together:

```java
public class MyLogger implements ProxyLogger, ProxyLazyLogger, ProxyConditionalLogger {
    private Logger logger;

    public MyLogger(Logger logger) {
        this.logger = logger;
    }

    @Override
    public Logger logger() {
        return this.logger;
    }
}
```

### Predicate Loggers

There are reasons why you would not want to log information you may normally log.

The historical reason for not logging is that there is a construction cost involved in creating parameters.  This is still true in a way today -- CPU and memory are not typically constraints for logging statements, but there are storage costs involved in producing logs.  Accumulated logs must be parsed and searched, making queries slower.

In the same way that there's a `ProxyConditionalLogger`, there's a `PredicateConditionalLogger` class that will apply preconditions to loggers, and so the logging will only happen when the preconditions are met:

```java
public class MyPredicateLogger implements PredicateLogger, PredicateLazyLogger, PredicateConditionalLogger {

    private Predicate<Level> predicate;
    private Logger logger;

    public MyPredicateLogger(Predicate<Level> predicate, Logger logger) {
        this.predicate = predicate;
        this.logger = logger;
    }

    @Override
    public Predicate<Level> predicate() {
        return this.predicate;
    }

    @Override
    public Logger logger() {
        return this.logger;
    }
}
```

This lets you apply predicates at the class level, and combine them with conditions at the method level:

```java
package example;

import com.tersesystems.logback.ext.*;
import net.logstash.logback.marker.LogstashMarker;
import net.logstash.logback.marker.Markers;
import org.slf4j.Logger;
import org.slf4j.event.Level;

import static org.slf4j.LoggerFactory.getLogger;

public class ClassWithConditionalLogger {

    private ClassWithConditionalLogger() {
    }

    private final Logger logger = getLogger(getClass());

    private void doStuff() {
        // Set up conditional logger to only log if this is my machine:
        final ConditionalLogger conditionalLogger = new MyPredicateLogger(this::isDevelopmentEnvironment, logger);

        String correlationId = IdGenerator.getInstance().generateCorrelationId();
        LogstashMarker context = Markers.append("correlationId", correlationId);

        // ProxyConditionalLogger will only log if this is my machine
        Logger conditionalLoggerAsNormalLogger = (Logger) conditionalLogger;
        conditionalLoggerAsNormalLogger.info("This will still only log if it's my machine");

        // Log only if the level is info and the above conditions are met AND it's tuesday
        conditionalLogger.ifInfo(this::objectIsNotTooLargeToLog, stmt -> {
            // Log very large thing in here...
            stmt.apply(context, "log if INFO && isDevelopmentEnvironment && objectIsNotTooLargeToLog()");
        });
    }

    private Boolean objectIsNotTooLargeToLog() {
        return true; // object is not too big
    }

    private Boolean isDevelopmentEnvironment(Level level) {
        return "wsargent".equals(System.getProperty("user.name"));
    }

    public static void main(String[] args) {
        ClassWithConditionalLogger classWithExtendedLoggers = new ClassWithConditionalLogger();
        classWithExtendedLoggers.doStuff();
    }
}
```

## Logging with Injected Context

When you're using structured logging, you'll inevitably have to pass around the `LogstashMarker` or `StructuredArgument` with it so that you can add context to your logging.  In the past, the recommended way to do this was MDC.

Avoid [Mapped Diagnostic Context](https://logback.qos.ch/manual/mdc.html).  MDC is a well known way of adding context to logging, but there are several things that make it problematic.  

MDC does not deal well with multi-threaded applications which may pass execution between several threads.  Code that uses `CompletableFuture` and `ExecutorService` may not work reliably with MDC.  A child thread does not automatically inherit a copy of the mapped diagnostic context of its parent.  MDC also breaks silently: when MDC assumptions are violated, there is no indication that the wrong contextual information is being displayed.

There are numerous workarounds, but it's safer and easier to use an explicit context as a field or parameter.  If you don't want to manage this in your logger directly, then the safest way is to handle it through injection, also known as using constructor parameters.

When you create an instance, you can pass in a single `org.slf4j.ILoggerFactory` instance that will create your loggers for you.  

```java
package example;

import com.tersesystems.logback.context.logstash.LogstashContext;
import com.tersesystems.logback.context.logstash.LogstashLoggerFactory;
import org.slf4j.ILoggerFactory;
import org.slf4j.Logger;

public class ClassWithContext {

    static class ObliviousToContext {
        private final Logger logger;

        public ObliviousToContext(ILoggerFactory lf) {
            this.logger = lf.getLogger(this.getClass().getName());
        }

        public void doStuff() {
            logger.info("hello world!");
        }
    }

    public static void main(String[] args) {
        // You can create objects that are oblivious to context, and just use the base
        // logstash markers...
        String correlationId = IdGenerator.getInstance().generateCorrelationId();
        LogstashContext context = LogstashContext.create("correlationId", correlationId);
        LogstashLoggerFactory loggerFactory = LogstashLoggerFactory.create(context);
        ObliviousToContext obliviousToContext = new ObliviousToContext(loggerFactory);
        obliviousToContext.doStuff();

        // Or you can create your own context and futzs with it.
        // Here we create an AppContext / AppLogger / AppLoggerFactory that lets us
        // set domain specific attributes on the context.
        AppContext appContext = AppContext.create().withCorrelationId(correlationId);
        AwareOfContext awareOfContext = new AwareOfContext(appContext);
        awareOfContext.doStuff();
    }

    private static class AwareOfContext {
        private final AppContext appContext;
        private final AppLogger logger;

        public AwareOfContext(AppContext appContext) {
            this.appContext = appContext;
            this.logger = AppLoggerFactory.create().getLogger(getClass()).withContext(appContext);
        }

        public void doStuff() {
            logger.info("My correlation id is {}", appContext.getCorrelationId().orElse("null"));
        }
    }
}
```

In the second example, an `AppContext` / `AppLogger` is used -- this is an example of domain specific methods and fields being added to the context.

```java
package example;

import com.tersesystems.logback.context.TracerFactory;
import com.tersesystems.logback.context.Context;
import com.tersesystems.logback.context.logstash.AbstractLogstashContext;
import com.tersesystems.logback.context.logstash.AbstractLogstashLoggerFactory;
import com.tersesystems.logback.context.logstash.AbstractLogstashLogger;
import net.logstash.logback.marker.LogstashMarker;
import net.logstash.logback.marker.Markers;
import org.slf4j.ILoggerFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.Marker;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Stream;


class AppContext extends AbstractLogstashContext<AppContext> {

    public static final String CORRELATION_ID = "correlationId";
    private final boolean tracer;

    protected AppContext(Map<?, ?> entries, boolean tracer) {
        super(entries);
        this.tracer = tracer;
    }

    public static AppContext create() {
        return new AppContext(Collections.emptyMap(), false);
    }

    public static AppContext create(Object key, Object value) {
        return new AppContext(Collections.singletonMap(key, value), false);
    }

    public Optional<String> getCorrelationId() {
        return Stream.of(entries().get(CORRELATION_ID))
                .map(cid -> (String) cid)
                .findFirst();
    }

    public AppContext withCorrelationId(String correlationId) {
        return and(AppContext.create(CORRELATION_ID, correlationId));
    }

    @Override
    public AppContext withTracer() {
        return create(entries(), true);
    }

    public boolean isTracingEnabled() {
        return tracer;
    }

    @Override
    public LogstashMarker asMarker() {
        if (isTracingEnabled()) {
            return Markers.appendEntries(entries()).and(TracerFactory.getInstance().createTracer());
        } else {
            return Markers.appendEntries(entries());
        }
    }

    @Override
    public AppContext and(Context<? extends Marker, ?> otherContext) {
        boolean otherTracing = (otherContext instanceof AppContext) && ((AppContext) otherContext).isTracingEnabled();
        // XXX Same as LogstashContext -- is there a way to access this directly?
        Map<Object, Object> mergedEntries = new HashMap<>(this.entries());
        mergedEntries.putAll(otherContext.entries());
        return new AppContext(mergedEntries, this.isTracingEnabled() || otherTracing);
    }

}

class AppLogger extends AbstractLogstashLogger<AppContext, Logger, AppLogger> {

    public AppLogger(AppContext context, Logger logger) {
        super(context, logger);
    }

    @Override
    public AppLogger withContext(AppContext otherContext) {
        return new AppLogger(this.context.and(otherContext), this.logger);
    }
}

class AppLoggerFactory extends AbstractLogstashLoggerFactory<AppContext, AppLogger, ILoggerFactory, AppLoggerFactory> {

    protected AppLoggerFactory(AppContext context, ILoggerFactory loggerFactory) {
        super(context, loggerFactory);
    }

    @Override
    public AppLoggerFactory withContext(AppContext context) {
        return new AppLoggerFactory(getContext().and(context), getILoggerFactory());
    }

    @Override
    public AppLogger getLogger(String name) {
        return new AppLogger(AppContext.create(), getILoggerFactory().getLogger(name));
    }

    public static AppLoggerFactory create() {
        return create(AppContext.create());
    }

    public static AppLoggerFactory create(AppContext context) {
        return new AppLoggerFactory(context, LoggerFactory.getILoggerFactory());
    }

}
```

This style of programming does assume that you can control the instantiation of your objects, and it doesn't go into some of the details such as accumulating extra context.  Keeping a context object around so you can accumulate more context may be a good idea in some circumstances.

## Tracer Bullet Logging

The `AppLogger` makes reference to a tracer, but doesn't go into detail.  

Using a context also allows you the option to do "tracing bullet" logging, where some extra context, such as a query parameter in an HTTP request, could cause a logger to log at a lower level than it would normally do to a special marker.  You can use this for debugging on the fly without changing logger levels, or use it for random sampling of some number of operations.

Defining the following turbo filter in `logback.xml`:

```xml
<turboFilter class="ch.qos.logback.classic.turbo.MarkerFilter">
  <Name>TRACER_FILTER</Name>
  <Marker>TRACER</Marker>
  <OnMatch>ACCEPT</OnMatch>
</turboFilter>
```

and adding it to an existing marker and wrapping it in a context logger, you can get:

```java
package example;

public class ClassWithTracer {

    // Add tracer to the context, and return a logger that covers over the context.
    private AppLogger getContextLogger(Request request) {
        final AppContext context;
        if (request.queryStringContains("trace")) {
            context = request.context().withTracer();
        } else {
            context = request.context();
        }
        return AppLoggerFactory.create(context).getLogger(getClass());
    }

    public void doThings(Request request) {
        AppLogger logger = getContextLogger(request);

        // This class is not logged at a TRACE level, so this should not show under
        // normal circumstances...
        if (logger.isTraceEnabled()) {
            logger.trace("This log message is only shown if the request has trace in the query string!");
        }
    }

    public static void main(String[] args) {
        ClassWithTracer classWithTracer = new ClassWithTracer();

        // run it without the trace flag
        Request request = new Request("foo=bar");
        classWithTracer.doThings(request);

        // run it WITH the trace flag
        Request requestWithTrace = new Request("foo=bar&trace=on");
        classWithTracer.doThings(requestWithTrace);
    }
}
```

which gives the following output:

```text
2019-01-26T18:40:39.088+0000 [TRACE] example.ClassWithTracer in main - This log message is only shown if the request has trace in the query string!
```

```json
{"@timestamp":"2019-01-26T18:40:39.088+00:00","@version":"1","message":"This log message is only shown if the request has trace in the query string!","logger_name":"example.ClassWithTracer","thread_name":"main","level":"TRACE","level_value":5000,"tags":["TRACER"],"correlationId":"FX1UlmU3VfqlX0qxArsAAA"}
```

## Dependency Injection with Guice

Finally, if you're using a DI framework like Guice, you can leverage some of the contextual code in [Sangria](https://tavianator.com/announcing-sangria/) to do some of the gruntwork for you.  For example, here's how you configure a `Logger` instance in Guice:

```java
package example;

import com.google.inject.AbstractModule;
import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.spi.InjectionPoint;
import com.tavianator.sangria.contextual.ContextSensitiveBinder;
import com.tavianator.sangria.contextual.ContextSensitiveProvider;
import com.tersesystems.logback.context.logstash.LogstashContext;
import com.tersesystems.logback.context.logstash.LogstashLoggerFactory;
import org.slf4j.ILoggerFactory;
import org.slf4j.Logger;

import javax.inject.Inject;
import javax.inject.Provider;
import javax.inject.Singleton;

public class GuiceAssistedLogging {

    public static class MyClass {
        private final Logger logger;

        @Inject
        MyClass(Logger logger) {
            this.logger = logger;
        }

        public void doStuff() {
            logger.info("hello world!");
        }
    }

    @Singleton
    static class Slf4jLoggerProvider implements ContextSensitiveProvider<Logger> {
        private final ILoggerFactory loggerFactory;

        @Inject
        Slf4jLoggerProvider(ILoggerFactory loggerFactory) {
            this.loggerFactory = loggerFactory;
        }

        @Override
        public Logger getInContext(InjectionPoint injectionPoint) {
            return loggerFactory.getLogger(injectionPoint.getDeclaringType().getRawType().getName());
        }

        @Override
        public Logger getInUnknownContext() {
            return loggerFactory.getLogger(Logger.ROOT_LOGGER_NAME);
        }
    }

    static class ILoggerFactoryProvider implements Provider<ILoggerFactory> {
        @Override
        public ILoggerFactory get() {
            // This would be hooked up to @RequestScoped in a real application
            LogstashContext context = LogstashContext.create("threadName", Thread.currentThread().getName());
            return LogstashLoggerFactory.create().withContext(context);
        }
    }

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new AbstractModule() {
            @Override
            protected void configure() {
                install(new AbstractModule() {
                    @Override
                    protected void configure() {
                        bind(ILoggerFactory.class).toProvider(ILoggerFactoryProvider.class);
                        ContextSensitiveBinder.create(binder())
                                .bind(Logger.class)
                                .toContextSensitiveProvider(Slf4jLoggerProvider.class);
                    }
                });
            }
        });

        MyClass instance = injector.getInstance(MyClass.class);
        // Assume this is running in an HTTP request that is @RequestScoped
        instance.doStuff();
    }

}
```

which yields:

```json
{"@timestamp":"2019-01-27T00:19:08.628+00:00","@version":"1","message":"hello world!","logger_name":"example.GuiceAssistedLogging$MyClass","thread_name":"main","level":"INFO","level_value":20000,"threadName":"main"}
```

If you are using a Servlet based API, then you can piggyback of Guice's [servlet extensions](https://github.com/google/guice/wiki/Servlets) and then integrate the logging context as part of the [`CDI / JSR 299 / @RequestScoped`](https://docs.oracle.com/javaee/6/tutorial/doc/gjbbk.html).  I have not tried this myself.
