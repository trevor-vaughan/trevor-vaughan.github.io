---
title: "FIPS Enabling the FOSS Puppet Server"
date: 2024-01-30 19:41:00 -0500
toc: false
classes:
 - wide
categories:
  - Compliance
tags:
  - compliance
  - FIPS
  - NIST
  - DoD
  - Puppet
  - Automation
---

{::options parse_block_html="true" /}

# Intro

This is primarily a running log for anyone that needs to FIPS-enable the FOSS
version of [Puppet][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i>.
[Pull Requests][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i>
are most welcome if things change over time.

This article targets RHEL systems but should generally be suitable on any
platform.

<!-- more -->

# Initial Setup

1. [FIPS enable your system][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i>
   * Do **NOT** try to do this after you have installed and configured your
     system. Things tend to go poorly over time unless this is the first step.
   * If you are running in a `podman` hosted container on an already FIPS'd RHEL
     system, you just need to run `fips-mode-setup --enable`
2. [Install the puppet server][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i>
3. Run: `/opt/puppetlabs/bin/puppetserver ca setup`

# The Problem

At this point, if you run `/opt/puppetlabs/bin/puppetserver foreground`, you
will be presented with the following error:

```console
Execution error (NoSuchAlgorithmException) at java.security.KeyPairGenerator/getInstance (KeyPairGenerator.java:218).
RSA KeyPairGenerator not available
```

<details>
  <summary markdown='span'>Full stacktrace</summary>

```console
ERROR [async-dispatch-2] [p.t.internal] Error during service init!!!
java.security.NoSuchAlgorithmException: RSA KeyPairGenerator not available
        at java.security.KeyPairGenerator.getInstance(KeyPairGenerator.java:218)
        at com.puppetlabs.ssl_utils.SSLUtils.generateKeyPair(SSLUtils.java:154)
        at puppetlabs.ssl_utils.core$fn__21552$generate_key_pair__21561$fn__21564.invoke(core.clj:522)
        at puppetlabs.ssl_utils.core$fn__21552$generate_key_pair__21561.invoke(core.clj:517)
        at puppetlabs.puppetserver.certificate_authority$fn__37524$generate_ssl_files_BANG___37529$fn__37530.invoke(certificate_authority.clj:708)
        at puppetlabs.puppetserver.certificate_authority$fn__37524$generate_ssl_files_BANG___37529.invoke(certificate_authority.clj:695)
        at puppetlabs.puppetserver.certificate_authority$fn__38749$initialize_BANG___38754$fn__38755.invoke(certificate_authority.clj:1565)
        at puppetlabs.puppetserver.certificate_authority$fn__38749$initialize_BANG___38754.invoke(certificate_authority.clj:1544)
        at puppetlabs.services.ca.certificate_authority_service$reify__43837$service_fnk__5324__auto___positional$reify__43852.init(certificate_authority_service.clj:36)
        at puppetlabs.trapperkeeper.services$fn__5148$G__5140__5151.invoke(services.clj:9)
        at puppetlabs.trapperkeeper.services$fn__5148$G__5139__5155.invoke(services.clj:9)
        at puppetlabs.trapperkeeper.internal$fn__14387$run_lifecycle_fn_BANG___14394$fn__14395.invoke(internal.clj:196)
        at puppetlabs.trapperkeeper.internal$fn__14387$run_lifecycle_fn_BANG___14394.invoke(internal.clj:179)
        at puppetlabs.trapperkeeper.internal$fn__14416$run_lifecycle_fns__14421$fn__14422.invoke(internal.clj:229)
        at puppetlabs.trapperkeeper.internal$fn__14416$run_lifecycle_fns__14421.invoke(internal.clj:206)
        at puppetlabs.trapperkeeper.internal$fn__15041$build_app_STAR___15050$fn$reify__15062.init(internal.clj:614)
        at puppetlabs.trapperkeeper.internal$fn__15091$boot_services_for_app_STAR__STAR___15098$fn__15099$fn__15101.invoke(internal.clj:648)
        at puppetlabs.trapperkeeper.internal$fn__15091$boot_services_for_app_STAR__STAR___15098$fn__15099.invoke(internal.clj:647)
        at puppetlabs.trapperkeeper.internal$fn__15091$boot_services_for_app_STAR__STAR___15098.invoke(internal.clj:641)
        at clojure.core$partial$fn__5910.invoke(core.clj:2647)
        at puppetlabs.trapperkeeper.internal$fn__14461$initialize_lifecycle_worker__14472$fn__14473$fn__14636$state_machine__11695__auto____14661$fn__14664.invoke(internal.clj:249)
        at puppetlabs.trapperkeeper.internal$fn__14461$initialize_lifecycle_worker__14472$fn__14473$fn__14636$state_machine__11695__auto____14661.invoke(internal.clj:249)
        at clojure.core.async.impl.ioc_macros$run_state_machine.invokeStatic(ioc_macros.clj:978)
        at clojure.core.async.impl.ioc_macros$run_state_machine.invoke(ioc_macros.clj:977)
        at clojure.core.async.impl.ioc_macros$run_state_machine_wrapped.invokeStatic(ioc_macros.clj:982)
        at clojure.core.async.impl.ioc_macros$run_state_machine_wrapped.invoke(ioc_macros.clj:980)
        at clojure.core.async$ioc_alts_BANG_$fn__11924.invoke(async.clj:421)
        at clojure.core.async$do_alts$fn__11863$fn__11866.invoke(async.clj:288)
        at clojure.core.async.impl.channels.ManyToManyChannel$fn__6536$fn__6537.invoke(channels.clj:99)
        at clojure.lang.AFn.run(AFn.java:22)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at clojure.core.async.impl.concurrent$counted_thread_factory$reify__6439$fn__6440.invoke(concurrent.clj:29)
        at clojure.lang.AFn.run(AFn.java:22)
        at java.lang.Thread.run(Thread.java:750)
```

</details>

This is due to the fact that the `puppetserver` has been compiled against the
[BouncyCastle][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i>
libraries whereas the system is attempting to force the use of the RHEL-native
FIPS libraries.

# The Solution

To remedy the issue, we need to ensure that the `puppetserver` has access to the
[BouncyCastle FIPS][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i>
libraries and is properly configured to use them.

## Download the [BouncyCastle FIPS][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i> Libraries

Grab the latest versions available at the time and please
[donate][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i> to
the BouncyCastle project if you can!
<br/>
FOSS is expensive and time consuming to support!
{: .text-center }
{: .notice--warning }

* `cd /opt/puppetlabs/server/data/puppetserver/jars`
* `curl -LO https://downloads.bouncycastle.org/fips-java/bc-fips-1.0.2.4.jar`
* `curl -LO https://downloads.bouncycastle.org/fips-java/bcpkix-fips-1.0.7.jar`
* `curl -LO https://downloads.bouncycastle.org/fips-java/bctls-fips-1.0.18.jar`
* Verify [the checksums][]<i class="fas fa-external-link-alt fa-2xs" aria-hidden="true"></i>

At this point, the BouncyCastle FIPS libraries are loaded into the server's
classpath but you'll hit another issue when you try to start the server:

```console
Execution error (SecurityException) at java.net.URLClassLoader/getAndVerifyPackage (URLClassLoader.java:412).
sealing violation: can't seal package org.bouncycastle.crypto: already loaded
```

This means that there is a baked-in version of the BouncyCastle libraries in the
`puppetserver` uber-jar that we need to replace.

## :rocket: Surgery

Now to get our hands dirty and mod the `puppetserver` JAR itself.

> :bulb: You'll need the `jar` command for this so you may want to do it on a
> different system with `java-devel` installed.

* `cd /opt/puppetlabs/server/apps/puppetserver`
* `mkdir tmp`
* `cd tmp`
* `jar -xvf ../puppet-server-release.jar`
* `find . -name bouncycastle -exec rm -rf {} \;`
* `jar -cf ../puppet-server-release.jar *`
* `cd ..`
* `rm -rf tmp`

If you've done everything properly, now when you run
`/opt/puppetlabs/bin/puppetserver foreground`, you will get the following error:

```console
Execution error (NoSuchAlgorithmException) at sun.security.jca.GetInstance/getInstance (GetInstance.java:159).
BCFKS KeyStore not available
```

## Set Up a New Security Policy

Our issue is now that the `puppetserver` is still using the system
`java.security` policy without any overrides that tell it to use the
BouncyCastle FIPS library.

* `mkdir /etc/sysconfig/puppetserver-properties`
* Add the following to `/etc/sysconfig/puppetserver-properties/java.security`

  ```config
  security.provider.1=org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider
  security.provider.2=org.bouncycastle.jsse.provider.BouncyCastleJsseProvider BCFIPS
  security.provider.3=com.sun.net.ssl.internal.ssl.Provider BCFIPS
  security.provider.4=sun.security.provider.Sun
  security.provider.5=sun.security.ec.SunEC
  security.provider.6=

  fips.provider.1=org.bouncycastle.jcajce.provider.BouncyCastleFipsProvider
  fips.provider.2=org.bouncycastle.jsse.provider.BouncyCastleJsseProvider BCFIPS
  fips.provider.3=com.sun.net.ssl.internal.ssl.Provider BCFIPS
  fips.provider.4=sun.security.provider.Sun
  fips.provider.5=sun.security.ec.SunEC
  fips.provider.6=
  ```

  You may now see a warning that says `invalid entry for security.provider.6`.
  <br/>
  Everything is fine
  <br/>
  We're using this to ensure that no other providers are loaded afterwards.
  {: .text-center }
  {: .notice--warning }
* Add the following to the `JAVA_ARGS` line (inside the double quotes) in
  * `-Djava.security.properties=/etc/sysconfig/puppetserver-properties/java.security`
    * Points to the new `java.security` file
  * `-Dorg.bouncycastle.fips.approved_only=true`
    * Ensures that we limit BouncyCastle FIPS to only using approved algorithms

## Regenerate All Certificates

You'll need to regenerate all certificates to ensure that everything is
generated using the FIPS libraries.

* `rm -rf /etc/puppetlabs/puppetserver/ca`
* `rm -rf /etc/puppetlabs/puppet/ssl/*`
* `/opt/puppetlabs/bin/puppetserver ca setup`

## And You're Off :tada:

Simply run `systemctl restart puppetserver` and you should be good to go!

<!-- Link Refs -->

[BouncyCastle FIPS]: https://www.bouncycastle.org/fips_faq.html
[BouncyCastle]: https://www.bouncycastle.org/
[FIPS enable your system]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/assembly_installing-a-rhel-8-system-with-fips-mode-enabled_security-hardening
[Install the puppet server]: https://www.puppet.com/docs/puppet/latest/install_puppet.html
[Pull Requests]: https://github.com/trevor-vaughan/trevor-vaughan.github.io/pulls
[Puppet]: https://www.puppet.com/
[donate]: https://www.bouncycastle.org/donate/index.cgi
[the checksums]: https://www.bouncycastle.org/fips-java/BC-FJA-Checksums-1.0.2.csv

<!-- Footnotes -->
