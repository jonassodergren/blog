---
title: "DevOps - High availability load balancing"
date: 2020-01-09T17:02:08+01:00
draft: false
---

In order to further improve the availability of Jobtech services we
are exploring load balancing over multiple Openshift clusters.
![OpenShift load balancing](../../openshift.jpeg)
An Openshift cluster in itself is highly available, and with the
deployment we use we can loose an entire worker node and the workloads
will evacuate to other worker nodes in order to regain redundancy. In
normal cases this means no service interuptions will be noticeable,
since workloads are usually deployed redundantly.
### Problem
However, if the entire cluster fails, we are out of luck. This is not
a hypothetical scenario, it happened once because of certificate
expirations that went unnoticed.

> Deploying new Openshift clusters at other cloud providers is quite
> feasible since Openshift 4.2 where the installer was much improved
> since the previous versions we have been using, 3.10 and 3.11.

Deploying our services reduntantly over several clusters is also
something we have explored using the skopeo command to copy images
between clusters.

The remaining problem, then, is how to load balance between several
different clusters.

Several options where evaluated:

+ DNS round robin. This is simple, but not reliable. DNS record TTL
makes it difficult to fail-over between clusters reliably
+ Client based routing. This will increase complexity at the client
side. This is reliable and gives the client and opportunity to decide
what to do during backend outtages.
+ External load balancer service that spans over cloud providers.

### Solution
We believe using an external load balancer service such as Cloudflare
will make it easy to implement our API:s, and it can co-exist with
client based routing for organizations that need greater control.

**We will explore Cloudflare next.**

First we need a hello-world service in each cluster we are going to be
load balancing between.

*test.services.jtech.se* and *jave.services.jtech.se clusters* will be
used. The endpoint will be called *hello.platsbanka.nu*

*Platsbanka.nu* is a debug domain at Loopia that was used for this test. We also
made a lbjobtechdev.se domain at AWS route53, but there were problems
propagating the DNS SOA and NS record changes needed to use
Cloudflare, so we couldn't use this domain for these tests.

In each cluster do:

```
oc new-project hello
oc new-app openshift/hello-openshift
oc expose svc/hello-openshift
```

Then set the environment variable RESPONSE in the deployment to
something that identifies the cluster and create the route.
```
oc expose svc/hello-openshift --hostname=hello.platsbanka.nu --name=hello-openshift-cloudflare-pool
oc set env dc/hello-openshift RESPONSE="this is the JAVE/TEST cluster"
```
Create a load balancer service in Cloudflare, and create a pool,
hello-pool. Origin servers will be hello.jave.services.jtech.se and
hello.test.services.jtech.se. These DNS names will resolve to the
respecive clusters. Each cluster must then route hello.platsbanka.nu
to the correct service when a corresponding http request arrives from Cloudflare.

If you reload "hello.platsbanka.nu" repeatedly, Cloudflare will
round robin between the two clusters, the message will vary depending on
which cluster answers. Cloudflare will decide which cluster to route
to according to its own algorithm, and the weight assigned to each
cluster in the load balancer.

If you are too lazy to press reload in a browser you can of course use
curl.

while true;  do curl hello.platsbanka.nu;done
this is the JAVE cluster
this is the test cluster
this is the test cluster
this is the test cluster
this is the JAVE cluster
this is the test cluster

When you get tired of this, press ctrl-c.
