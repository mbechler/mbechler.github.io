---
layout: post
title:  "PSA: Log4Shell and the current state of JNDI injection"
date:   2021-12-10 20:00:00 +0100
categories: Java JNDI LDAP RMI CVE-2021-44228 Log4Shell
---

> The "Log4Shell" vulnerability has triggered a lot of interest in JNDI Injection exploits.
> Unfortunately, regarding exploitability there seems to go a bit of misinformation around.
> TLDR: A current Java runtime version won't safe you. Do patch.


The Log4Shell vulnerability (CVE-2021-44228) ultimately is a quite simple JNDI Injection flaw, 
but in a really really bad place. Log4J will perform a JNDI lookup() while expanding placeholders 
in logging messages (or indirectly as parameters for formatted messages). 

In a default installation there are two "interesting" protocols supported by JNDI: RMI and LDAP.
In both cases a lookup() call is actually meant to return a Java object. This usually means 
serialized Java objects, however there is also a JNDI Reference mechanism for indirect construction
through a factory. This object and factory bytecode can potentially be loaded from a remote URL 
codebase (read: a webserver with .class files).

For a long time for both RMI and LDAP the specified reference factory codebase was not restricted 
at all, a lookup call to a attacker specified JNDI RMI or LDAP name was direct remote code execution.

Starting with Java 8u121 remote codebase were no longer permitted by default for RMI (but not LDAP).

Apparently there had been a prior patch (CVE-2009-1094) for LDAP, but that was completely
ineffective for the factory codebase. Therefore, LDAP names would still allow direct remote 
code execution for some time after the RMI patch. That "oversight" was only addressed later as 
CVE-2018-3149  in Java 8u191 (see <https://bugzilla.redhat.com/show_bug.cgi?id=1639834>). 

So, up to Java 8u191 there is a direct path from a controlled JNDI lookup to remote classloading
of arbitrary code. That Java version is about 3 years old.

However, exploitation does not end there. Turning back to RMI, References and object construction
with factories are still supported, just remote codebases are prohibited. 
Michael Stepankin in <https://www.veracode.com/blog/research/exploiting-jndi-injections-java> 
describes how the Apache XBean _BeanFactory_ can be used in a returned Reference to achieve
code execution. This class has to be locally available on the targeted system, however, it
is for example included in Apache Tomcat. If your application runs in Tomcat, bad luck. 
<https://github.com/veracode-research/rogue-jndi> also has another vector for WebSphere.

Also, RMI is inherently based on Java serialization and LDAP supports a special object class,
deserializing a Java object from the directory to return from the lookup(). In both cases, 
unless a global deserialization filter is applied, JNDI injection will result in deserialization 
of untrusted attacker provided data. While adding a bit more complexity to the attack,
in many cases this will still be exploitable for remote code execution.

Long story short: Do not rely on a current Java version to save you. Update Log4 (or remove the JNDI lookup). 
Disable the expansion (seems a pretty bad idea anyways).

Update (14.12): In the original post I forgot to mention CORBA/IIOP which is the third relevant JNDI protocol. Impact is very similar to RMI.

Previously on this blog:
- <https://mbechler.github.io/2018/11/01/Java-CVE-2018-3149/> - Oracle patching JNDI/LDAP
- <https://mbechler.github.io/2018/01/20/Java-CVE-2018-2633/> - Some JNDI Injection basics
