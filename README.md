# Vpon Logging Best Practice

## The Purpose of Logging

Logging is an important function of any software and for Vpon, benefits the following groups of people in the manners described:
  - The service administrator — provides detailed information that lets them know if the service is running correctly
  - Operation / Devop — provides details and evidence of possible problems to help resolve incidents and issues
  - Developers — allows them to trace code execution without attaching a debugger

When you write any code, you should ask yourself what you would want to see, from the point of view of the above three people. If you were running a server, what would you want to be notified of? If you were handling a support case regarding some problem with the system, what information would you need? If you were tracing a bug in the code, what would you want to see logged?


## Log Levels

### ERROR

This level serves as the general error facility. It should be used whenever the software encounters an unexpected error which preventsfurther processing. The reporting component may attempt recovery which may affect system availability or performance for a period of time, or it may forcefully terminate if it is unable to recover. This level is always reported for all software components.

There are three examples when this level should be used:

 * Incorrectable internal state inconsistency, such as when a JVM reports an OutOfMemoryError. The toplevel handler would log this condition and force an JVM exit, as it is not capable of continuing execution.
 * Internal state inconsistency, which is correctable by flushing and re-creating the state. In this case the component would log an event, which would indicate what assertion has been violated and that the state has been flushed to recover.
 * Request-level processing error, e.g. the application encounters an error which is preventing a particular request from completing, but there is no indication of systematic failure which would prevent other requests from being successfully processed.

The primary audience are monitoring systems and system operators, as the events reported have impact on operational performance of the system.

### WARN

This level serves for events which indicate irregular circumstances, from which we have a clearly-defined recovery strategy which has no impact on system availability or performance as seen by the reporting component. Events on this level are usually reported.

A typical example for a candidate event is when a software component detects inconsistency within an external data feed and it performs a corrective action to compensate for it. Let's say we process a list of key/value pairs and encounter a duplicate key: we can either overwrite the old occurance, ignore the new occurance or abort. If we take any of the first two actions, we should report a WARN event. If we take the third, we should report an ERROR event.

The primary audience of these events are automated systems, operators and administrators, as this level of messages indicate non-optimal system operation (e.g. data feeds could use normalization) or may forebode a future failure.

### INFO

This level serves for events which constitute major state changes within a software component -- such as initialization, shutdown, persistent resource allocation, etc. -- which are part of normal operations. Events on this level are typically reported for non-library components.

Each software component should log at least four events on this level:

 * when it starts its initialization
 * when it becomes operational,
 * when it starts orderly shutdown,
 * just before it terminates normally

The primary audience of these events are operators and administrators, who use them to confirm major interactions (such as restarting components) have occurred within the system.

### DEBUG

This is the coarse diagnostic level, serving for events which indicate internal state transitions and detail what processing occurs. Events on this level are usually not reported, but are enabled when a subsystem's code flows need to be examined for troubleshooting purposes.

Placement and amount of events generated at this level are at the discretion of the development engineers, as both relate directly to the component logic. The amount of detail in these events has to be limited to a single line of information useful for pinning down a misbehaving software component in an seemingly normal system and should be directly cross-referencable to either previous DEBUG events or component configuration. A hard requirement is that there has to be at least one flow control statement (such as if, for, while) or a call to a different software component between two DEBUG event triggers.

Primary audience of these events are administrators and support personnel diagnosing operational issues, mainly in real-time as they occur on the system.

## Use parameterized logging

Using dynamically-constructed message strings constributes to major overhead as the message string has to be constructed before the call to logging method is performed, thus forcing overhead even if the constructed string is not used (for example DEBUG level is not enabled).

Another issue with dynamically-constructed message strings is that they cannot be easily extracted by static source code analysis -- a process critical for creating message catalogue of a particular software release, which in turn is needed for things like support knowledge bases, internationalization, etc.

While the former concern is addressed by Logger classes exposing methods like LOG.isDebugEnabled(), the second concern can only be alleviated by using explicit String literals when calling the Logger methods. The correct way to address both concerns is to use parameterized logging as described at http://www.slf4j.org/faq.html#logging_performance. The basic pattern to follow is this:

    // GOOD: string literal, no dynamic objects
    LOG.debug("Method called with arg {}", arg);

    // BAD: string varies with argument
    LOG.debug("Method called with arg " + arg);

    // BAD: code clutter
    if (LOG.isDebugEnabled()) {
        LOG.debug("Method called with arg {}", arg);
    }

There is one thing that needs to be noted in this style, which is that logging an exception is properly supported if you supply it as the last argumennt, but you have to MAKE SURE IT IS NOT HINTED TO IN THE MESSAGE STRING:


    // GOOD: note how there is no "{}" for ex
    try {
        doSomething(arg)

    } catch { case ex: SomeException =>
        LOG.warn("Failed to do something with {}, continuing”, arg, ex)
    }

    // BAD:
    // - exception is interpreted as an object
    // - exception chaining cause is lost
    // - stack trace is lost
    try {
        doSomething(arg)

    } catch { case ex: SomeException =>
        LOG.warn("Failed to do something with {} because {}, continuing", arg, ex)
    }

NOTE: while it is true that isDebugEnabled() can eliminate overhead associated with the variadic method call, the burden on the developer is not acceptable simply because there are much better methods of automatic control of this overhead, without having any impact on the source code (or even the class files). One of them is JIT-level optimizations stemming from the ability to inline calls to LOG.debug(). 

The other is the set of interfaces from java.lang.instrument package, which can be used to completely eliminate the call overhead by removing all calls to LOG.debug() from the class bytecode based on the logger configuration.

## Provide useful event context

Each logging call should provide useful context in which it occurred. This is not usually the case with a lot of Java-based software, notably even with some JVM implementations. Here are some typical anti-patterns which contribute to mitigated ability to diagnose problems when they happen:

    // VERY BAD:
    // - no context provided
    // - non-constant message string
    // - assumes useful toString()
    LOG.debug(arg.toString());

    // VERY BAD:
    // - no context provided
    LOG.debug("{}", arg);

    // COMPLETELY BAD:
    // - silently ignoring errors!!!
    try {
        doSomething(arg);
        ...
    } catch { case ex: SomeException =>
    }

    // EXTREMELY BAD:
    // - message is not constant
    // - no context is provided
    // - ex.getCause() is lost
    // - call stack is lost
    try {
        doSomething(arg);
        ...
    } catch { case ex: SomeException =>
        LOG.warn(ex.getMessage());
    }

    // EXTREMELY BAD:
    // - message is not constant
    // - no context is provided
    // - ex.getCause() is probably lost
    // - call stack is probably lost
    // - assumes useful toString()
    try {
        doSomething(arg);
        ...
    } catch { case ex: SomeException =>
        LOG.warn(ex.toString());
    }

    // VERY BAD:
    // - no useful context is provided
    // - ex.getCause() is probably lost
    // - call stack is probably lost
    // - administrators don't know what an Exception is!
    try {
        doSomething(arg);
        ...
    } catch { case ex: SomeException =>
        LOG.warn("Exception {}", ex);
    }

The proper fix for these anti-patterns is to always provide key information in the logging event:

 * what went wrong
 * how badly it went wrong
 * in case we recover, shortly describe how (especially on WARN level)

Here is some examples:

    // GOOD:
    // - string literal
    // - we explain what we tried to do
    // - we pass along information we have about the failure
    // - we explain that we recovered from the failure
    try {
        doSomething(arg)
        ...
    } catch { case ex: SomeException =>
        LOG.warn("Failed to do something with {}, ignoring it", arg, ex)
    }

    // GOOD:
    // - string literal
    // - we explain what we tried to do
    // - we pass along information we have about the failure
    // - we escalate the failure to our caller
    // - we also 'chain' the exception so it is not lost and can be
    // correlated
    try {
        doSomething(arg)
        ...
    } catch { case ex: SomeException =>
        LOG.error("Failed to do something with {}", arg, ex);
        throw new RuntimeException("Failed to do something", ex)
    }

