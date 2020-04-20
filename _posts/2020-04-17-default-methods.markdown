---
layout: post
title:  handling default methods with dynamic proxies in java
date:   2020-04-17 10:00:00
description: default methods, dynamic proxies, java 8
comments_id: 1
---

Java has a very powerful feature to create [dynamic proxies](https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html) for
an object implementing a given set of interfaces at runtime. This can be used to modify some implementations or adding some aspects the the
object, like logging, access checks or caching. The dynamic proxy pattern is used by a number of popular libraries to implement the core of
their functionality. Some well-known libraries in this regard are:

* spring (for its aspect oriented features)
* retrofit

There is a runtime cost associated with the use of dynamic proxies, but it is usually not that bad, several benachmarks
have been performed to prove their suitability for practical use-cases:

* [Debunking myths: proxies impact performance](https://spring.io/blog/2007/07/19/debunking-myths-proxies-impact-performance/)
* [Benchmarking the cost of dynamic proxies](http://ordinaryjava.blogspot.com/2008/08/benchmarking-cost-of-dynamic-proxies.html)

As a rule of thumb and based on my own experiments a dynamic proxy impacts performance such that an invocation of a proxied method
is roughly 1.6 times slower compared to direct invocation of the same method. Which is not too bad when you consider what happens
under the hood to make this working and comparing it with the performance of general reflection calls.

Let's get to some code. In order to create a dynamic proxy for a given interface, one has to code the following:

{% highlight java %}
Object delegate = ...
MyInterface impl =
    (MyInterface) Proxy.newProxyInstance(MyInterface.class.getClassLoader(), new Class[] { MyInterface.class }, new InvocationHandler() {
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            try {
                return method.invoke(delegate, args);
            } catch (InvocationTargetException ex) {
                // this is needed to throw the original exception to the caller.
                throw ex.getCause();
            }
        }
    });
{% endhighlight %}

This will create a dynamic proxy for the interface _MyInterface_ and delegate all the method calls it receives to the Object _delegate_.
To make this work, the _delegate_ object must be able to process the delegated calls, i.e. it must implement the requested interface.

Now we do not want to go into more details of dynamic proxies and what you can do with them, but rather take a look into a
problem introduced with some new feature in Java 8. This version of java added support for default methods in interfaces. But what has this to
do with dynamic proxies? If you take the example from above and use it with an interface containing default methods, you will
get a runtime exception like this:

{% highlight java %}
Exception in thread "main" java.lang.IllegalArgumentException: object is not an instance of declaring class
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
	at org.netomi.uom.util.Proxies$1.invoke(Proxies.java:52)
	at com.sun.proxy.$Proxy0.getSystemUnit(Unknown Source)
	...
{% endhighlight %}

This happens because the default method will be delegated to the actual object which usually does not have an implementation
for this method (unless it has overridden it of course). In order to support such interfaces with dynamic proxies there is a
workaround available for Java 8:

{% highlight java %}
import java.lang.invoke.MethodHandles;
...

Constructor<MethodHandles.Lookup> constructor = MethodHandles.Lookup.class.getDeclaredConstructor(Class.class);
if (!constructor.isAccessible()) {
    constructor.setAccessible(true);
}

Object result =
    constructor.newInstance(method.getDeclaringClass())
        .unreflectSpecial(method, method.getDeclaringClass())
        .bindTo(proxy)
        .invokeWithArguments(args);
...
{% endhighlight %}

This will access a private constructor of the _MethodHandles.Lookup_ class via reflection and use it to get a _MethodHandle_
object for the default method, bind it to the actual proxy instance and invoke the method. A bit hacky but the only way
to support that in Java 8.

Unfortunately, this approach fails with Java 9, but the good news is that there is supported API that we can use for the same
purpose and does not required the use of reflection:

{% highlight java %}
MethodHandles.Lookup lookup =
    MethodHandles.privateLookupIn(method.getDeclaringClass(), MethodHandles.lookup())

MethodHandle handle = null;
MethodType methodType =
    MethodType.methodType(method.getReturnType(), method.getParameterTypes());
      
if (Modifier.isStatic(method.getModifiers())) {
    handle = lookup.findStatic(method.getDeclaringClass(), method.getName(), methodType);
} else {        
    handle = lookup.findSpecial(method.getDeclaringClass(), method.getName(), methodType, method.getDeclaringClass());
}

Object result = handle.bindTo(proxy).invokeWithArguments(args);
...
{% endhighlight %}

This is a bit more complex but it is supported with all versions since Java 9+. To wrap things up we have to modify
our initial code for a delegating proxy to something like that:

{% highlight java %}
Object delegate = ...
MyInterface impl =
    (MyInterface) Proxy.newProxyInstance(MyInterface.class.getClassLoader(), new Class[] { MyInterface.class }, new InvocationHandler() {
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (!method.isDefault()) {
                try {
                    return method.invoke(delegate, args);
                } catch (InvocationTargetException ex) {
                    // this is needed to throw the original exception to the caller.
                    throw ex.getCause();
                }
            }
     
            return DefaultMethodHandler.getMethodHandle(method).bindTo(proxy).invokeWithArguments(args);
        }
    });
{% endhighlight %}

The class _DefaultMethodHandler_ wraps up the different mechanisms to get a method handle for a default method
depending on the current JVM, the code can be found [here](https://github.com/netomi/uom/blob/master/src/main/java/org/netomi/uom/util/Proxies.java#L63).
It is a derived work from the [spring framework](https://github.com/spring-projects/spring-framework), released under Apache v2.0 license.

Taking default methods correctly into account opens up a lot of possibilities for creating dynamic proxies. In my own project
[Units of Measurement](https://github.com/netomi/uom) I use proxies to create concrete quantity implementations for different datatypes (double, BigDecimal)
at runtime. This is a very flexible approach which can also easily be extended in the future with only a minimal performance impact.