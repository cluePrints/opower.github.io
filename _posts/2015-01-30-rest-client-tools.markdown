--- 
author: chris.phillips 
layout: post 
title: Making REST Services Easy to Consume with rest-client-tools
---

Early in our SOA journey at Opower, we established the practice of providing
client jars for our REST services to save clients the hassle of implementing
repetitive client code and dealing with the error-prone concerns like
authentication, error handling, retries, resource management, message
serialization. Early client jars were based on the Resteasy Proxy Framework. The
way it works is you create a java interface that describes the REST endpoints
using the standard JAX-RS annotations. On the server side, you implement that
interface. On the client side, you pass that same interface to the Resteasy
Proxy Framework to generate a client proxy that will do all of the work needed
to call that endpoint (build and execute the http request, serialization /
deserialization of request and response bodies etc). As time goes on and the API
changes, this helps keep the client in sync with the API specification. 

In our case, the client jars, authored by service writers, would subclass a
BasicClient class that used the [Resteasy Proxy
Framework](http://docs.jboss.org/resteasy/docs/3.0.9.Final/userguide/html_single/index.html#d4e2143)
to generate the proxies. This approach worked well for a time, but we
encountered problems that required us to make changes. As soon as we had to
deploy service clients into legacy applications, we discovered that using the
Resteasy Proxy Framework in the same JVM as another JAX-RS implementation --
including other versions of Resteasy -- triggers runtime exceptions due to
implementation conflicts. The heart of the matter lies in how JAX-RS initializes
a javax.ws.rs.ext.RuntimeDelegate
[instance](http://docs.oracle.com/javaee/6/api/javax/ws/rs/ext/RuntimeDelegate.html#getInstance()).
Only one RuntimeDelegate is loaded per JVM and one version of Jersey or Resteasy
wins. Then when your Resteasy client proxy tries to use the Jersey
RuntimeDelegate you get Exceptions such as the following:

	  java.lang.ClassCastException: com.sun.jersey.server.impl.provider.RuntimeDelegateImpl
                 cannot be cast to org.jboss.resteasy.spi.ResteasyProviderFactory

Switching every existing application over to the same version of Resteasy was
not an option for us. Thus rest-client-tools was born. In essence, we started
with Resteasy and made several key changes to remove runtime conflicts and
ensure portability:

 * Extracted the client proxy generation code from Resteasy.
 * Refactored that code so that we no longer use the RuntimeDelegate when creating
   client proxies.
 * Heavily tested client proxies alongside different versions of Jersey and
   Resteasy at runtime to demonstrate compatibility.

We changed the client jars for our services to use this early version of
rest-client-tools instead of Resteasy for proxy generation and the runtime
conflicts went away. 

This solution worked well for a somewhat shorter period of time. The number of
services deployed grew at an accelerating pace. The graph of interconnected
services also grew more complex and that’s when we ran into our next problem:
dependency hell (the problem that SOA was supposed to prevent). The problem
appeared in applications that had to call multiple services meaning they
imported multiple client jars. The client jars were authored by service writers
and required several transitive dependencies. In some cases, writers of one
service had upgraded some of the dependencies required by their client jar. This
caused runtime conflicts with the dependencies from the client jar from another
team who had yet to make similar upgrades. To correct this problem we had to
make a few more changes both to the code in rest-client-tools and the
conventions we established for service writers and consumers. 

First we now require service writers to provide a jar with just the resource
interface and any dependant model objects. This is less work for service authors
since they only have to create the interface and model objects instead of a
full-blown client jar. We take special care to ensure that resource interfaces
don’t have extra transitive dependencies as well. You could be really extreme in
this and only allow JDK classes (0 transitive dependencies) in resource
interfaces. However, we have identified a few stable libraries that do a good
job with backwards compatibility such as joda-time or guava, and we allow
resource interfaces to use  them as dependencies. Without carefully controlling
the dependencies, we could end up with the same dependency hell as before. 

Additionally, we refactored the API of rest-client-tools to a set of fluent
builders. Instead of a subclass that implements the resource interface, you pass
the resource interface to the builder and get an instance of that interface
back. 

Finally, we integrated the [Hystrix](https://github.com/Netflix/Hystrix)
library. If you’re not familiar with Hystrix check out
[this](http://techblog.netflix.com/2012/11/hystrix.html) article from the
Netflix tech blog.

“In a distributed environment, failure of any given service is inevitable.
Hystrix is a library designed to control the interactions between these
distributed services providing greater tolerance of latency and failure. Hystrix
does this by isolating points of access between the services, stopping cascading
failures across them, and providing fallback options, all of which improve the
system's overall resiliency. “

rest-client-tools allows you to generate client proxies that automatically wrap
resource invocations in HystrixCommands. This added layer of resource isolation
can be quite valuable in a complicated graph of dependant services. 

Now, when you need to consume a REST service, you just import the resource
interface jar and rest-client-tools. Since rest-client-tools only appears on
your classpath once, you can consume any number of services in this manner
without conflict. The ensures that client applications:

 * have maximum control over what libraries appear on their classpath
 * benefit from the convenience of auto generated client proxies
 * run a relatively small risk of dependency hell

Internal services are not the only scenario where rest-client-tools is helpful.
We have used rest-client-tools to consume 3rd party APIs as well. All we have to
do is create a resource interface that describes the endpoints for the 3rd party
API, and client can use that to easily consume a service.

With rest-client-tools we feel we have struck a nice balance by providing a
useful client library while minimizing coupling when using it. Since both the
server and client use the same resource interface, API changes can be managed
more easily and client code can be upgraded easily. Client proxies created with
rest-client-tools can run alongside any JAX-RS implementation without conflict,
are more flexible and can be used in a broad set of scenarios.

For more information and examples checkout our github repo
[here](https://github.com/opower/rest-client-tools). 


