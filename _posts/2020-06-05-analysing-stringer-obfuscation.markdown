---
layout: post
title:  analysing string encryption of stringer
date:   2020-06-05 16:00:00
description: byte code engineering, deobfuscation
comments_id: 5
---

[Stringer](https://jfxstore.com/stringer/) is one of many commercial tools to apply obfuscation on java byte code level.
It supports various obfuscation techniques, but in this blog post I would like to analyse its ability to encrypt strings
and explore ways to automatically deobfuscate jars obfuscated by __stringer__.

Let's take a look at how the actual string encryption applied by __stringer__ looks like. The following snippets are
disassembled bytecode in __jasmin__ notation:

{% highlight java %}
    ldc "\u5d5a\ub1c7\u0712"
    invokestatic xxx/yy/z(Ljava/lang/Object;)Ljava/lang/String;
{% endhighlight %}

As you can see, an encrypted string, is pushed onto the stack and a method with a revealing signature is called, returning
the decrypted string a runtime. My first attempt is to call the decrypt method myself with the same argument in the hope
that it returns the decrypted string.

Unfortunately, this does not lead to a properly decrypted string, so there must be some additional mechanisms in place
in order to prevent code lifting and to easily decrypt encrypted strings inside the jar file.

Looking at the decrypt method (_xxx/yy/z_), I can see the following instructions:

{% highlight java %}
    invokestatic java/lang/Thread/currentThread()Ljava/lang/Thread;
    invokevirtual java/lang/Thread/getStackTrace()[Ljava/lang/StackTraceElement;
    ...
    invokestatic sun/misc/SharedSecrets/getJavaLangAccess()Lsun/misc/JavaLangAccess;
    aload 1
    iconst_2
    aaload
    invokevirtual java/lang/StackTraceElement/getClassName()Ljava/lang/String;
    invokestatic java/lang/Class/forName(Ljava/lang/String;)Ljava/lang/Class;
    invokeinterface sun/misc/JavaLangAccess/getConstantPool(Ljava/lang/Class;)Lsun/reflect/ConstantPool; 1
    invokevirtual sun/reflect/ConstantPool/getSize()I
    ...
    invokevirtual java/lang/StackTraceElement/getClassName()Ljava/lang/String;
    invokevirtual java/lang/StringBuilder/append(Ljava/lang/String;)Ljava/lang/StringBuilder;
    ...
    invokevirtual java/lang/String/hashCode()I
{% endhighlight %}

It looks like the decrypt method analyses the calling stack and uses information from this class/method
to setup its decryption key. Translating the same snippet to java code makes it quite clear:

{% highlight java %}
    StringBuilder sb = new StringBuilder();
    StackTraceElement[] stackTraceElements = Thread.currentThread().getStackTrace();
    JavaLangAccess javaLangAccess = sun.misc.SharedSecrets.getJavaLangAccess();
    int constantPoolSize = javaLangAccess.getConstantPool(Class.forName(stackTraceElements[2].getClassName())).getSize();
    sb.append(constantPoolSize);
    ...
    sb.append(stackTraceElements[2].getClassName());
    ...
    sb.append(stackTraceElements[2].getMethodName());
    ...
    int key = sb.hashCode();
    ...
{% endhighlight %}

So various information from the calling class/method is being used to generate a key that is needed for the actual 
decryption. If the key does not match the expected values, only garbage is returned, making code lifting more challenging.

Let's try to use [proguard-core](https://github.com/Guardsquare/proguard-core) to modify the byte code in a way that we can
easily execute it and get the decrypted strings during static analysis. For readability I will not post the full source code,
but the relevant pieces of code to understand how this can be done.

We know that the decrypt method deduces its decryption key from the calling method. Now lets try to find all places in the jar
file that seems to decrypt a string:

{% highlight java %}
    private static final String DECRYPT_METHOD_TYPE = "(Ljava/lang/Object;)Ljava/lang/String;";

    private final Constant[] CONSTANTS = new Constant[]
        {
            new MethodrefConstant(1, 2, null, null),
            new ClassConstant(X, null),
            new NameAndTypeConstant(Y, 3),
            new Utf8Constant(DECRYPT_METHOD_TYPE),
        };
        
    private final Instruction[] INSTRUCTIONS = new Instruction[]
        {
            new ConstantInstruction(Instruction.OP_LDC, Z),
            new ConstantInstruction(Instruction.OP_INVOKESTATIC, 0),
        };

    private final InstructionSequenceMatcher matcher =
         new InstructionSequenceMatcher(CONSTANTS, INSTRUCTIONS);

    ...

    public void visitAnyInstruction(Clazz clazz, Method method, CodeAttribute codeAttribute, int offset, Instruction instruction) {
    
        instruction.accept(clazz, method, codeAttribute, offset, matcher);
        
        // did we find a match?
        if (matcher.isMatching()) {
            InstructionSequenceMatcher matcher = matcher1.isMatching() ? matcher1 : matcher2;
        
            int classIndex = matcher.matchedConstantIndex(X);
            int nameIndex = matcher.matchedConstantIndex(Y);
            int stringIndex = matcher.matchedConstantIndex(Z);
            
            ...
        }
        
        ...
{% endhighlight %}

With the code snippet above, we look for code patterns that load a constant string onto the stack and invoke a
static method which takes an __Object__ as input and return a __String__. In the analysed samples of jars obfuscated
with __stringer__ all encrypted strings were encrypted using the same pattern, and its also quite uncommon for generic code
to use similar patterns, thus we can easily find such encrypted strings in this case.

Now that we know which classes and methods contain encrypted strings, we can prepare the referenced decrypt method in such
a way that the protection mechanism is not effective. For this purpose, we replace the aforementioned code in the decrypt
method with the values of the actual calling method:

{% highlight java %}
    InstructionSequenceBuilder ____ = new InstructionSequenceBuilder();
            
    instructions = new Instruction[][][]
        {
            {
                ____.invokestatic("sun/misc/SharedSecrets", "getJavaLangAccess", "()Lsun/misc/JavaLangAccess;")
                    .aload(1)
                    .iconst_2()
                    .aaload()
                    .invokevirtual("java/lang/StackTraceElement", "getClassName", "()Ljava/lang/String;")
                    .invokestatic("java/lang/Class", "forName", "(Ljava/lang/String;)Ljava/lang/Class;")
                    .invokeinterface("sun/misc/JavaLangAccess", "getConstantPool", "(Ljava/lang/Class;)Lsun/reflect/ConstantPool;")
                    .invokevirtual("sun/reflect/ConstantPool", "getSize", "()I")
                    .invokevirtual("java/lang/StringBuilder", "append", "(I)Ljava/lang/StringBuilder;").__(),

                ____.pushInt(constantPoolSize)
                    .invokevirtual("java/lang/StringBuilder", "append", "(I)Ljava/lang/StringBuilder;").__(),
            },

    ....
    
    public void visitCodeAttribute(Clazz clazz, Method method, CodeAttribute codeAttribute) {
        CodeAttributeEditor codeAttributeEditor = new CodeAttributeEditor();

        clazz.accept(
            new AllMethodVisitor(
            new AllAttributeVisitor(
            new PeepholeEditor(codeAttributeEditor,
            new InstructionSequencesReplacer(constants,
                                             instructions,
                                             null,
                                             codeAttributeEditor,
                                             new InstructionCounter())))));
    }
{% endhighlight %}

As you can see in this code snippet, we replace certain instruction patterns with known values so that the decryption
works regardless from which method it is being called. Now the only thing left to do is to copy the original code to
a new class, modify it as described, load the generated class and execute the method with the given encrypted string:

{% highlight java %}
    ProgramClass duplicatedClass =
        new ProgramClass(originalClass.u4version,
                         1,
                         new Constant[1],
                         originalClass.u2accessFlags,
                         0,
                         0);

    ConstantPoolEditor constantPoolEditor = new ConstantPoolEditor(duplicatedClass);

    duplicatedClass.u2thisClass =
        constantPoolEditor.addClassConstant(originalClass.getName(), null);
    duplicatedClass.u2superClass =
        constantPoolEditor.addClassConstant(ClassConstants.NAME_JAVA_LANG_OBJECT, null);

    // Copy over the class members.
    MemberAdder memberAdder = new MemberAdder(duplicatedClass);

    originalClass.fieldsAccept(memberAdder);
    originalClass.methodsAccept(memberAdder);
    
    modifiedClass.accept(
        new ProtectionRemover(ClassUtil.externalClassName(clazz.getName()),
                              method.getName(clazz),
                              constantPoolLength));

    modifiedClass.accept(new ProgramClassWriter(os))
    byte[] content = os.toByteArray();
    ...
    
    // load the class contents with a custom ClassLoader which effectively calls defineClass.
    ClassLoader classLoader =
        new ByteClassLoader(new URL[] { inputURL  },
                            CodeLifter.class.getClassLoader(),
                            Collections.singletonMap(externalClassName, content));

    Class<?> loadedClass = Class.forName(externalClassName, true, classLoader);

    ...
{% endhighlight %}

Now that we have the modified decrypt method, we can just call it via reflection using the encrypted string
as parameter. The returned result can be used to replace the original code in the obfuscated jar with a simple ldc instruction.

After we have implemented all this logic we see that the decrypted string still results in garbage, so there must be
another protection mechanism against code lifting. Some more debugging reveals that the __CodeSource__ of the
[ProtectionDomain](https://docs.oracle.com/javase/7/docs/api/java/security/ProtectionDomain.html) of the class containing the
decrypt method is also verified. If this does not return the expected value, the decryption is unsuccessful. Luckily this
protection mechanism can easily be avoided by modifying the __CodeSource__ of a given class using reflection:

{% highlight java %}
    URL inputURL = ... // original file name of the obfuscated jar
    Class<?> modifiedClass = ....

    CodeSource codeSource = new CodeSource(inputURL, (CodeSigner[]) null);
    ProtectionDomain domain = modifiedClass.getProtectionDomain();

    java.lang.reflect.Field field = domain.getClass().getDeclaredField("codesource");
    field.setAccessible(true);
    field.set(domain, codeSource);
{% endhighlight %}

So we change the __CodeSource__ of the modified class to the one that the decrypt method expects, and finally, we can
decrypt any strings in the obfuscated jar simply by the means of static code analysis.

The full source code to decrypt jars obfuscated by __stringer__ will be made available at a later time. More to come!