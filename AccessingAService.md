---
layout: default
title: Accessing a Service
description: Describes how to access an OSGi service programmatically.
date: 2016-04-17 12:00:00
commentIssue: 10
---

# Accessing a Service

A major benefit of using a framework is that you can reuse existing components in an architecturally well defined way. In this part I'm going to use the OSGi logging service to improve the logging behavior of our simple bundle.

The OSGi framework manages the available services and makes them available via the [BundleContext](https://osgi.org/javadoc/r6/core/org/osgi/framework/BundleContext.html) interface[^snbt]. An implementation of the `BundleContext` is passed to the `start` method of the `BundleActivator`. Maybe you had noticed this parameter already in our activator implementation:

[^snbt]: Note that according to an OSGi 
    [blog entry](https://blog.osgi.org/2013/07/real-men-dont-use-ds.html), this part and 
    the next part of the introduction should probably not be there 
    (or banished to some appendix).
    
    I don't share this view on software development, I prefer a "from the grounds up" approach when
    learning about a new technology. From my experience as software architect and project
    manager, one of the most reliable ways to make a project fail with respect to the delivery timeline
    is to use some technology that you don't really understand in combination with tools that 
    are supposed to compensate that lack of understanding. Of course, consultants love this
    approach and are prepared to come to the rescue -- which will result in the additional failure
    of the project regarding its budget constraints[^coc].

[^coc]: I definitely liked the response in this [blog](https://blogs.mulesoft.com/dev/news-dev/osgi-no-thanks/): "Ah so consulting is the answer to OSGi complexity. I’ll let everyone know".

```java
public class Activator implements BundleActivator {
    ...
    @Override
    public void start(BundleContext context) throws Exception {
        System.out.println("Hello World started.");
        helloWorld = new HelloWorld();
    }
    ...
}

```

The obvious choice for accessing a service is the `BundleContext.getService` method. It needs a `ServiceReference` parameter. Luckily, we can find a method `getServiceReference` right below the `getService` in the documentation. `ServiceReference` comes in two flavors. One takes the class as Java type, the other as a string. In both cases the class is presumably the main interface of the service that we want to use. The log service is part of all OSGi specifications, no matter whether they target residential or enterprise environments. [Download](https://www.osgi.org/developer/specifications/) either specification and have a look at the service description.

The full name of the log service interface is `org.osgi.service.log.LogService`. So let's try this code:

```java
...
import org.osgi.service.log.LogService;

public class Activator implements BundleActivator {
    ...
    @Override
    public void start(BundleContext context) throws Exception {
        LogService logService = context.getServiceReference(
                context.getServiceReference(LogService.class));
        ....
    }
    ....
}
```

You should see several error markers. First of all, `LogService` is unknown. That's okay, because up to now, we only needed the OSGi core and had therefore only a jar with the core API in the classpath. Go to the "Build" tab of the `bnd.bnd` editor and add "Bndtools Hub/osgi.cmpn"[^bndhub] to the "Build Path". Make sure to choose the (rather old) versions 4.2.0 for both osgi.core and osgi.cmpn, because I want you to become aware of something. Check your project's `bnd.bnd` for the proper versions on the (now augmented) buildpath:

```properties
-buildpath: \
	osgi.core;version=4.2.0,\
	osgi.cmpn;version=4.2.0
```

If the versions don't match, simply edit them. Save, rebuild (on the build tab), and Bndtools updates the Eclipse project's library path accordingly.

[^bndhub]: Since Bndtools 3.2.0 this will no longer be offered by default. 
    If you have created the "Bndtools OSGi Workspace" using the wizard (instead of 
    checking out the tutorial project), you have to add
    the following snippet to `cnf/build.bnd`:
    
    ```properties
	-plugin.6.bndtoolshub: \
		aQute.bnd.deployer.repository.FixedIndexedRepo; \
			name=Bndtools Hub; \
			locations=https://raw.githubusercontent.com/bndtools/bundle-hub/master/index.xml.gz
    ``` 
    
    This makes some older versions of the OSGi bundles available again.

Going back to the source, one error marker remains. The parameter of type `Class` is not accepted. Reading carefully through the method's [JavaDoc](https://osgi.org/javadoc/r6/core/org/osgi/framework/BundleContext.html#getServiceReference(java.lang.Class)), you will find that the flavor accepting this parameter only exists since version 1.6 of the API. This version was first included in the OSGi platform specification 4.3. So, if we want to use this method, we have to make sure that the environment provides at least this version. Managing application module's versions is one of the big topics of OSGi. If you come from building some simple Java application[^mv], this might surprise you. In an enterprise environment, however, this is definitely an issue. 

[^mv]: Rule of thumb: get the latest versions of all libraries and hope that they provide backward compatibility for parts of the application built with older version, right? Well, often this works surprisingly well... 

Add newer versions of the bundles (such as 4.3.0) in the Bndtools dialog (first, remove the bundles again, else they won't be offered)[^orSource]:

[^orSource]: Or change the version in the source tab. Editing the buildpath there
	allows the use of some very 
	[interesting options](https://github.com/bndtools/bnd/wiki/Buildpath-Versions)[^unbe].

[^unbe]: It's unbelievable where they sometimes hide interesting information. I mean,
	you would expect this information to be provided
	[here](https://bnd.bndtools.org/instructions/buildpath.html), wouldn't you?

![Choosing a version](images/Bndtools-version-dialog.png){: width="500px" }

Clicking "Add", saving and rebuilding results in a successful build[^ov]. Looking at the generated jar's `MANIFEST.MF`[^rl] you now see:<a name="version-range"></a>

```properties
Import-Package: org.osgi.framework;version="[1.6,2)"
```

[^ov]: There's some Bndtools magic happening here. The entry in `bnd.bnd` reflects exactly
    what you see in the GUI:
    
    ```
    -buildpath: \
	    osgi.residential;version=4.3,\
	    osgi.core;version=4.3
    ```
    
    And the continuous build executed by Bndtools works fine. When you try to build 
    the project using gradle, however, it fails with `error: type ServiceTracker does 
    not take parameters`. The reason is that the OSGi libraries provided for version 4.3.0 have been 
    compiled [with a special flag](https://blog.osgi.org/2012/10/43-companion-code-for-java-7.html) 
    that allows generics to be used in pre-Java 5 JVMs. Starting with Java 7, this 
    "trick" doesn't work anymore. Therefore the OSGi alliance has released jars built "in the
    ordinary way" (compatible with Java 7 and beyond) as version 4.3.1.
    
    As the 4.3.1 versions aren't offered in the `bnd.bnd` GUI dialog, you have to change 
    the version "manually" in the source view[^lv]:
    
    ```
    -buildpath: \
	    osgi.cmpn;version=4.3.1,\
	    osgi.core;version=4.3.1
    ```
    
    Using this configuration, the project builds both in Eclipse and "headless" using gradle.

[^lv]: Alternatively, you can choose version 6.0.0 for both jars. So why bother?
    The versions that you choose here result in a requirement for the runtime framework.
    If you choose 6.0.0, you'll have to use a framework that 
    [supports](https://en.wikipedia.org/wiki/OSGi_Specification_Implementations) this version
    of the OSGi specification.


[^rl]: Slightly simplified. If you have already added `osgi.residential` you actually see this:

    ```properties
    Import-Package: org.osgi.framework;version="[1.6,2)",org.osgi.service.log;version="[1.3,2)"
    ```

    Since `MANIFEST.MF` is basically a properties file, keys have to be unique. So if there are several imports, the value of the key `Import-Package` becomes an enumeration of imported packages.

The "version interval" uses the notation known from mathematics: at least version 1.6, higher versions (e.g. 1.6.1, 1.7) are okay, but the version must be less than 2.0[^ug].

[^ug]: The upper limit is an educated guess, of course. If there will ever be a version 2.x of the API, there is a chance that our code will still run. Usually, however, a change of the major version indicates some incompatible change of the API. So by excluding 2.0 and anything beyond, we're on the safe side. 

Now that we know how to easily run oder debug an OSGi application from Eclipse (using the "Run" or "Debug" buttons on the "Run" tab of the `bnd.bnd` editor), we can check the effect of our statement simply by setting a breakpoint after it. Do this and start the application in debug mode. You should get: 

```
Failed to start bundle SimpleBundle-logging-1.1.0, exception Unable to resolve 
SimpleBundle-logging [1](R 1.0): missing requirement [SimpleBundle-logging [1](R 1.0)] 
osgi.wiring.package; (&(osgi.wiring.package=org.osgi.service.log)(version>=1.3.0)
(!(version>=2.0.0))) Unresolved requirements: [[SimpleBundle-logging [1](R 1.0)] 
osgi.wiring.package; (&(osgi.wiring.package=org.osgi.service.log)(version>=1.3.0)
(!(version>=2.0.0)))]
```

Read the message carefully and try to understand it. You may encounter similar messages later when the cause is less obvious. Note how the `Import-Package` statement with version constraint is translated to a namespace/[filter](https://osgi.org/javadoc/r6/core/org/osgi/framework/Filter.html) pair that can be evaluated easily by the framework. The filter uses the quite intuitive prefix notation known from LDAP search filters. What you see here is the general form of a requirement. Such a requirement can also be used to specify a dependency on a specific Java version or some other so called "capability"[^WhyTranslate].

[^WhyTranslate]: Once, the `Import-Package` statement was the only kind of dependency. Others were added and finally a more general mechanism [was introduced](https://stackoverflow.com/questions/57414686/osgi-whats-the-difference-between-import-package-export-package-and-require-ca/57415720#57415720).

What the message comes down to is that there is no logging service in the runtime environment. Go back to the "Run" tab of the `bnd.bnd` editor. Add `org.apache.felix.log` and debug again.

![Adding felix log](images/Adding-felix-log.png){: width="400px" }

When the program stops, have a look at the variable `logService`. If you're lucky, it has been assigned a value like `org.apache.felix.log.LogServiceImpl@45afc369`. You have to be lucky, because activation of services happens in no specific order. The fact that our bundle states a dependency on `org.osgi.service.log` in its `MANIFEST.MF` does not imply that the OSGi framework constructs a dependency graph and activates services with dependencies only after the services that they depend on have been activated[^dss]. This would only solve part of the problem anyway, because -- as we well know from playing around with the Felix console -- bundles (and the services that they provide) can be started and stopped at any time[^sr].

[^dss]: But this does sound alluring, doesn't it? Well, that's why OSGi has added such a concept as "Declarative Services". But let's stick to the "mid level API" provided by the `ServiceTracker` utility class for now.

[^sr]: Bundles that provide a service usually register the service when they are started and unregister it when they are stopped.

What we need to do is track the registered services and run our service only if all required services (the ones that our service builds upon) are available.
 
---

