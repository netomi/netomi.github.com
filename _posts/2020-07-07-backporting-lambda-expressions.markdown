---
layout: post
title:  Backporting lambda expressions
date:   2020-07-07 12:00:00
description: java 8, lambda expressions, backporting
comments_id: 6
---

Let's take a look into lambda expressions as part of the Java language specifiction, what they are, how they are represented,
and how existing bytecode can be backported to earlier class files versions without losing any functionality.

# Introduction

A lambda expressions is basically syntactic sugar that allows you to define inline anonymous classes that implement specific interfaces.
Take a look at the following example:

{% highlight java %}
...
Button button = (Button) findViewById(R.id.button);
button.setOnClickListener((view) -> Toast.makeText(this, "Hello World!", Toast.LENGTH_LONG).show());
... 
{% endhighlight %}

In this example we set a click listener for a specific button using a lambda expression. This is equivalent to writing this code:

{% highlight java %}
...
Button button = (Button) findViewById(R.id.button);
button.setOnClickListener(new OnClickListener() {
    @Override
    public void onClick(View view) {
        Toast.makeText(this, "Hello World!", Toast.LENGTH_LONG).show();
    } 
});
... 
{% endhighlight %}

The use of lambda expressions makes the whole code a bit more concise and less bloated. I dont want to go into more
details on how to write lambda expressions in the java language. There are many resources out there for this purpose. In this
blog post I would like to focus on how such lambda expressions are represented in byte code, and how it can be backported
to earlier class file versions, which is important when targeting platforms like Android.

# Representation

Let's analyse how the _javac_ compiler translates this java code into the corresponding class file format:

{% highlight java %}
  public void onCreate(android.os.Bundle);
    descriptor: (Landroid/os/Bundle;)V
    flags: (0x0001) ACC_PUBLIC
    Code:
        ...
        23: invokedynamic #29,  0             // InvokeDynamic #0:onClick:(Lxxx/yyy/HelloWorldActivity;)Landroid/view/View$OnClickListener;
        28: invokevirtual #33                 // Method android/widget/Button.setOnClickListener:(Landroid/view/View$OnClickListener;)V
        ...
{% endhighlight %}

We can see that an instruction of type _invokedynamic_ is executed which returns an object of type _OnClickListener_ and passes
this on to the _setOnClickListener_ method of the _Button_ object. The _invokedynamic_ instruction calls the _BootstrapMethod_ #0 as defined
in the class:
 
{% highlight java %}
BootstrapMethods:
  0: #71 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #78 (Landroid/view/View;)V
      #79 REF_invokeSpecial xxx/yyy/HelloWorldActivity.lambda$onCreate$0:(Landroid/view/View;)V
      #78 (Landroid/view/View;)V
{% endhighlight %}

The definition of this _BootstrapMethod_ contains information about the actual method to be called, its parameter and return types.
The JVM will then dynamically create at runtime a _CallSite_ targeting the specified target method and execute it. 

In our case the target method is also stored in the class file itself as private method (with the name as specified in the BootstrapMethod):
 
{% highlight java %}
  private void lambda$onCreate$0(android.view.View);
    descriptor: (Landroid/view/View;)V
    flags: (0x1002) ACC_PRIVATE, ACC_SYNTHETIC
    Code:
        ...
        aload_0
        ldc           #10                 // String: "Hello World!"
        iconst_1
        invokestatic  #41                 // Method android/widget/Toast.makeText:(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;
        invokevirtual #45                 // Method android/widget/Toast.show:()V
        return
{% endhighlight %}

Now this is the most simple example of a lambda expression. There are different cases to consider (e.g. capturing or non-capturing, references to constructors, ...).
I don't want to go into too much detail here, lets dive into ways to backport these constructs in a way that they can be represented in earlier class file versions.

# Backporting

There is a very popular and amazing tool called [retrolambda](https://github.com/luontola/retrolambda) which does exactly that, backport any
lambda expression to java version 5, 6 or 7. For various technical reasons, I had the need to implement such a mechanism myself as the product
I have been working on ([ProGuard / DexGuard](https://github.com/Guardsquare/proguard)) also had to work in full standalone mode and is generally
self-contained, thus has no external dependencies.

So the basic idea is to extract the generated private method into a separate lambda class that implements the necessary interface(s). Then replace
the _invokedynamic_ instruction calls in a way to create an instance of this lambda class.

The code to perform this is open-source and accessible at the [ProGuard github repo](https://github.com/Guardsquare/proguard/tree/master/base/src/proguard/backport).

Applied on the above example, the result would look like in the snippets below. An additional accessor method
has been added to avoid modifying the visibility of the lambda methods, but that is not a technical necessity
its rather a design choice.

{% highlight java %}
  static void accessor$HelloWorldActivity$lambda0(xxx.yyy.HelloWorldActivity, android.view.View);
    descriptor: (Lxxx/yyy/HelloWorldActivity;Landroid/view/View;)V
    flags: (0x1008) ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         aload_0
         aload_1
         invokespecial #15                 // Method lambda$onCreate$0:(Landroid/view/View;)V
         return

  private void lambda$onCreate$0(android.view.View);
    descriptor: (Landroid/view/View;)V
    flags: (0x1002) ACC_PRIVATE, ACC_SYNTHETIC
    Code:
      stack=3, locals=2, args_size=2
         aload_0
         ldc           #10                 // String: "Hello World!"
         iconst_1
         invokestatic  #28                 // Method android/widget/Toast.makeText:(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;
         invokevirtual #31                 // Method android/widget/Toast.show:()V
         return

  public void onCreate(android.os.Bundle);
    descriptor: (Landroid/os/Bundle;)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=4, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokespecial #35                 // Method android/app/Activity.onCreate:(Landroid/os/Bundle;)V
         5: aload_0
         6: ldc           #36                 // int 2130968576
         8: invokevirtual #40                 // Method setContentView:(I)V
        11: aload_0
        12: ldc           #41                 // int 2130903040
        14: invokevirtual #45                 // Method findViewById:(I)Landroid/view/View;
        17: checkcast     #47                 // class android/widget/Button
        20: new           #49                 // class xxx/yyy/HelloWorldActivity$$Lambda$0
        23: dup
        24: aload_0
        25: invokespecial #52                 // Method xxx/yyy/HelloWorldActivity$$Lambda$0."<init>":(Lxxx/yyy/HelloWorldActivity;)V
        28: invokevirtual #56                 // Method android/widget/Button.setOnClickListener:(Landroid/view/View$OnClickListener;)V
{% endhighlight %}

and the corresponding generated lambda class:

{% highlight java %}
  private final xxx.yyy.HelloWorldActivity arg$0;
    descriptor: Lxxx/yyy/HelloWorldActivity;
    flags: (0x0012) ACC_PRIVATE, ACC_FINAL

  public xxx.yyy.HelloWorldActivity$$Lambda$0(xxx.yyy.HelloWorldActivity);
    descriptor: (Lxxx/yyy/HelloWorldActivity;)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: invokespecial #13                 // Method java/lang/Object."<init>":()V
         4: aload_0
         5: aload_1
         6: putfield      #15                 // Field arg$0:Lxxx/yyy/HelloWorldActivity;
         9: return

  public void onClick(android.view.View);
    descriptor: (Landroid/view/View;)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: getfield      #15                 // Field arg$0:Lxxx/yyy/HelloWorldActivity;
         4: aload_1
         5: invokestatic  #24                 // Method xxx/yyy/HelloWorldActivity.accessor$HelloWorldActivity$lambda0:(Lxxx/yyy/HelloWorldActivity;Landroid/view/View;)V
         8: return
{% endhighlight %}

Of course, to fully support all possible variants of lambda expressions (and its sibling method references) some effort has to be
put forward, but the necessary code is quite limited, take a look at the referenced source code, especially the class _LambdaExpressionConverter_.
ProGuard offers a lot of functionality to make any kind of byte code engineering quite simple and effective.
 
# Addendum

In this blog post I am focusing on lambda expressions, but ProGuard is capable of backporting various other language features of Java, like
static interface methods, default methods, try-with-resources, string concatenations introduced in Java 9 and even the use of Java 8 APIs like the
popular stream and time APIs.

Take a look at the source code, its very clean and can easily be read and understood imho.