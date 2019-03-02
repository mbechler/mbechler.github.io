---
layout: post
title:  "Beware the Nashorn: ClassFilter gotchas"
date:   2019-03-02 13:41:24 +0100
categories: Java Nashorn JavaScript Sandbox CVE-2018-3183 CVE-2018-17191
---

> Nashorn's `ClassFilter` machanism alone is completely ineffective in preventing scripts from invoking arbitrary Java code.
> According to Oracle this is the intended behavior, so you probably want to avoid Nashorn for running untrusted scripts. 


Since Java 8 the Java runtime has been including the Nashorn JavaScript engine,
allowing developers to add JavaScript scripting to their applications.

A somewhat sane developer might assume that the `ClassFilter` mechanism can
be used to restrict access to the Java world from scripts and thereby
used for sandboxing of untrusted scripts. Enabling a `ClassFilter` even 
automatically disables all reflection functions to prevent the obvious 
bypass possible using them.
Well, assumptions are security's worst enemy. The secondary documentation 
(Nashorn User's Guide) as well as the original `ClassFilter` JEP (202) hint 
that this might not be enough, but do not really give details and sounds
more like it may be referring to a lack of control over Java code called into. 

> While the ClassFilter interface can prevent access to Java classes, it is not 
> enough to run untrusted scripts securely. The ClassFilter interface is not a 
> replacement for a security manager. Applications should still run with a security 
> manager before evaluating scripts from untrusted sources. Class filtering 
> provides finer control beyond what a security manager provides. 
> For example, an application that embeds Nashorn may prevent the spawning 
> of threads from scripts or other resource-intensive operations that may be
> allowed by security manager.


Having a somewhat closer look at Nashorn I had noticed that CVE-2018-3183 
was being patched in Java 8u191 with some additional restrictions  the 
_engine_ global. 
This property doesn't appear to be documented at all and provides access to
the native `NashornScriptEngine` instance running the script.

Extra points to the developers for the sneaky way the property accessor was 
introduced as an afterthought:

```
    public static Object __noSuchProperty__(final Object self, final Object name) {
	// [...]
        if ("context".equals(nameStr)) {
            return sctxt;
        } else if ("engine".equals(nameStr)) {
            // expose "engine" variable only when there is no security manager
            // or when no class filter is set.
            if (System.getSecurityManager() == null || global.getClassFilter() == null) {
                return global.engine;
            }
        }
```

This even ensures that all naive attempts to remove this property will fail horribly, as 
when overwritten, the property can simply be unset by a script to restore it's original value.
Mitigation appears to be only possible by removing the `__noSuchProperty__` handler as well.

Note that the logic, added in the security path, prevents access to _engine_ only when there 
is **BOTH** a `SecurityManager` and a `ClassFilter` present. 
Reporting this to Oracle, it was stated that this is the intended behavior.

So, even after the patch for CVE-2018-3183, if there is no `SecurityManager` active, the _engine_
property will be available to all scripts, even if there is a `ClassFilter` not allowing any access
to Java classes at all.

With access to the _engine_ property, it is pretty much game over for the sandbox:
```
this.engine.factory.scriptEngine.eval('java.lang.Runtime.getRuntime().exec("whatever")')
```
This breaks down to:
 * `this.engine`, global _engine_
 * `.factory`, `getFactory()` -> `NashornScriptEngineFactory` instance
 * `.scriptEngine`, `getScriptEngine()` creates a fresh engine instance with **default** (== no `ClassFilter`, etc..) configuration
 * `.eval([script])`, execute JavaScript code with full access to the Java world


Affected Nashorn usage resulting in sandbox bypass has been identified so far in:
 * Netbeans 9 Proxy PAC evaluation (CVE-2018-17191) 
 * delight-nashorn-sandbox (fix in 0.1.20+)

Given that Oracle does not "support" isolated/sandboxed engine configurations without a 
`SecurityManager` present and there could be similar issues introduced in the future, 
avoiding Nashorn for such applications is advised.
