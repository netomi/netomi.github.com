---
layout: project
title: Units of Measurement
description: Orbit Exploration Toolkit
img: /img/logo-uom.png
link: https://github.com/netomi/uom
role: owner
license: Apache license 2.0
category: scientific
tags: java scientific_computing
---

This library is my take on JSR-385 to provide a useful API for representing physical measurements and perform
conversions between different units.

Its not a reference implementation of the Units of Measurement API due to technical limitations of the latter one.
Instead it takes many of the concepts of the released API and builds a full-fledged units of measurement library around
them to be able to represent physical measurements in different units with arbitrary precision.

The design goal of the library is to include:

* fully typed quantities:
{% highlight java %}
Length l1 = Length.of(1, SI.METER);
Length l2 = Length.ofMeter(2);
{% endhighlight %}

* transparent support for double and arbitrary decimal precision quantities (using BigDecimal):
{% highlight java %}
Length doubleLength  = Quantities.createQuantity(1, SI.METER, Length.class);
Length decimalLength = Quantities.createQuantity(BigDecimal.ONE, Imperial.YARD, Length.class);

Length l3 = l1.add(l2);
{% endhighlight %}

* support for generic quantities:
{% highlight java %}
Quantity<Speed> speed = Quantities.createGeneric(1, SI.METER_PER_SECOND);

System.out.println(speed.add(Speed.ofMeterPerSecond(2))); // -> prints 3 m/s
{% endhighlight %}

* support for quantity factories:
{% highlight java %}
QuantityFactory<Length> factory = DoubleQuantity.factory(Length.class);

Length l1 = factory.create(1, SI.METRE);

// default factories can be replaced
// use decimal precision with MathContext DECIMAL128 for every quantity of type Length:
Quantities.registerQuantityFactory(Length.class, DecimalQuantity.factory(MathContext.DECIMAL128, Length.class));

// quantity factories with pooling behavior can be registered
QuantityFactory<Length> myPoolFactory = ...;
Quantities.registerQuantityFactory(Length.class, myPoolFactory);
{% endhighlight %}

* support for the standard unit manipulations as defined by [JSR-385](https://www.jcp.org/en/jsr/detail?id=385) et al
* support for as many units as possible, the amazing [GNU units](https://www.gnu.org/software/units/) library is the reference to compare to
