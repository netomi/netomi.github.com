---
layout: post
title:  Minimizing shaded dependencies with gradle and R8
date:   2020-04-20 13:00:00
description: shrinking, gradle, r8
comments_id: 3
---

Sometimes you need or want to include 3rd party dependencies into your own jar file e.g. to provide your users with
an uber jar. These dependencies can additionally be repackaged to avoid any potential classpath issues. There exist
various plugins for build systems like gradle or maven to achieve this:

* [Android shade plugin for maven](http://maven.apache.org/plugins/maven-shade-plugin/index.html)
* [Shadow jar plugin for gradle](https://imperceptiblethoughts.com/shadow/)

Both plugins also offer a simple minimization feature, which tries to shrink the 3rd party dependencies as much as
possible. For this purpose a small and nice [library](https://github.com/tcurdt/jdependency) is being used that is
able to analyse transitive dependencies of class files.

This already leads to some good results. Lets take a simple example. In my project [Units of Mesurement](https://github.com/netomi/uom)
I have the need to include some code from other projects, namely [guava](https://github.com/google/guava) and [hibernate validator](https://github.com/hibernate/hibernate-validator).
These two projects contain some nice utility classes that I would like to use in my own library:

* [ConcurrentReferenceHashMap](https://github.com/hibernate/hibernate-validator/blob/master/engine/src/main/java/org/hibernate/validator/internal/util/ConcurrentReferenceHashMap.java)
* [Suppliers#memoize()](https://github.com/google/guava/blob/master/guava/src/com/google/common/base/Suppliers.java#L101)

In case only such a limited number of classes / methods are being used, copy&pasting the relevant code to your project is a viable option.
But I wanted to explore if the minimize feature of the shadow jar plugin can help me without copying the code which is tedious
and makes it more difficult to integrate potential bugfixes (apart from the testing coverage which might drop for your own code).

So I added a configuration like that to my project to add the 2 libraries as dependency, adding a relocate config,
and enable the standard minimize feature:

{% highlight shell %}
shadowJar {
    exclude 'META-INF/**'
    exclude '**/**.properties'
    exclude '**/*.swp'

    relocate 'com.google.common', 'org.netomi.uom.util.shadow.guava'
    relocate 'org.hibernate.validator.internal', 'org.netomi.uom.util.shadow.hibernate'
    minimize()
}

dependencies {
    compile('com.google.guava:guava:28.2-jre')
    compile('org.hibernate.validator:hibernate-validator:6.1.3.Final')

    ...
}
{% endhighlight %}

The resulting jar contains the following classes from the 2 libraries:

{% highlight shell %}
tn@proteus:~/workspace/netomi/uom/build/libs$ unzip -l uom-1.0-SNAPSHOT-all.jar  | grep shadow
        0  2020-04-20 15:43   org/netomi/uom/util/shadow/
        0  2020-04-20 15:43   org/netomi/uom/util/shadow/guava/
        0  2020-04-20 15:43   org/netomi/uom/util/shadow/guava/annotations/
      616  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/annotations/Beta.class
      670  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/annotations/GwtCompatible.class
      678  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/annotations/GwtIncompatible.class
      304  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/annotations/VisibleForTesting.class
        0  2020-04-20 15:43   org/netomi/uom/util/shadow/guava/base/
     4184  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/Absent.class
      851  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/AbstractIterator$1.class
     1388  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/AbstractIterator$State.class
     2381  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/AbstractIterator.class
     1068  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/CharMatcher$1.class
     2104  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/CharMatcher$And.class
....
{% endhighlight %}

in total 131 classes:

{% highlight shell %}
tn@proteus:~/workspace/netomi/uom/build/libs$ unzip -l uom-1.0-SNAPSHOT-all.jar  | grep shadow | wc -l
131
{% endhighlight %}

This would definitely be too much, and I would prefer to copy&paste the relevant classes to my own library and update
my NOTICE.txt file accordingly. I have worked for several years on a tool called [ProGuard](https://www.guardsquare.com/en/products/proguard),
so I was pretty sure that we can do better than that. Due to license problems (ProGuard is licensed under GPL) I decided to
use [R8](https://r8.googlesource.com/r8) to extend the shadow jar plugin to use a full blown shrinker to improve the results
of its minimization feature. R8 is the default minimization tool of the Android SDK and licensed under Apache v2.0 license.

The result of this proof-of-concept can be found in this [pull request](https://github.com/johnrengelman/shadow/pull/566) for the
shadow jar plugin. Using R8 instead of just simply collecting transitive class dependencies results in a shaded jar that 
contains only this number of shaded classes:

{% highlight shell %}
tn@proteus:~/workspace/netomi/uom/build/libs$ unzip -l uom-1.0-SNAPSHOT-all.jar  | grep shadow 
        0  2020-04-20 15:52   org/netomi/uom/util/shadow/
        0  2020-04-20 15:52   org/netomi/uom/util/shadow/guava/
        0  2020-04-20 15:52   org/netomi/uom/util/shadow/guava/base/
      408  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/Preconditions.class
      194  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/Supplier.class
     1760  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/Suppliers$MemoizingSupplier.class
     1812  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/Suppliers$NonSerializableMemoizingSupplier.class
      850  2019-12-26 22:08   org/netomi/uom/util/shadow/guava/base/Suppliers.class
        0  2020-04-20 15:52   org/netomi/uom/util/shadow/hibernate/
        0  2020-04-20 15:52   org/netomi/uom/util/shadow/hibernate/util/
      520  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$Option.class
     9488  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap.class
     2082  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$EntrySet.class
     1214  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$KeyIterator.class
     1689  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$EntryIterator.class
    10076  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$Segment.class
     1158  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$SoftKeyReference.class
     1225  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$SoftValueReference.class
      221  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$KeyReference.class
     3650  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$HashEntry.class
     2885  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$HashIterator.class
     2168  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$SimpleEntry.class
     1225  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$WeakValueReference.class
      624  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$ReferenceType.class
     1494  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$Values.class
     1395  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$WriteThroughEntry.class
     1222  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$ValueIterator.class
     1158  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$WeakKeyReference.class
     1676  2020-04-10 11:22   org/netomi/uom/util/shadow/hibernate/util/ConcurrentReferenceHashMap$KeySet.class

total 29
{% endhighlight %}

Most of them are inner classes of the _ConcurrentReferenceHashMap_ which are simply needed and can not be shrunk.
This result is certainly much better and would allow me to go the route of shading off-the-shelf dependencies into my
own library rather than copy&pasting code.

If you would like to see this feature in the shadow jar plugin, star this [issue](https://github.com/johnrengelman/shadow/issues/565).

The test that I made with my own library can be found [here](https://github.com/netomi/uom/tree/shadow-dependencies).

Note: I am of course perfectly aware that under normal circumstances it is not advised to shade any 3rd party dependency as
it might result in duplicated code being loaded by consumers of your library. In this specific use-case I think a technique like
this makes sense as I want to include a very small subset of a larger dependency without cutting the ties to upstream releases of
these dependencies (and potentially using more features).