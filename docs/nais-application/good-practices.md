This document describes the different properties a NAIS application should have.

## Set reasonable resource requests and limits

Setting reasonable resource `requests` and `limits` helps keep the clusters healthy, while not wasting resources.

A rule of thumb is to set the requested CPU and memory to what your applications uses under "normal" circumstances,
and set limits to what it is reasonable to allow the application to use at most.

Most of the time, the important aspect is `requests`, because this says how much Kubernetes should reserve for your application.
If you request too much, we are wasting resources.
If you request too little, you run the risk of your application being starved in a low-resource situation.

`limits` is mostly a mechanism to protect the cluster when something goes wrong.
A memory leak that results in your application using all available memory on the node is inconvenient.

## Handles termination gracefully

The application should make sure it listens to the `SIGTERM` signal, and prepare for shutdown \(closing connections etc.\) upon receival.

When running on NAIS \(or Kubernetes, actually\) your application must be able to handle being shut down at any given time. This is because the platform might have to reboot the node your application is running on \(e.g. because of a OS patch requiring restart\), and in that case will reschedule your application on a different node.

To best be able to handle this in your application, it helps to be aware of the relevant parts of the termination lifecycle.

1. Application \(pod\) gets status `TERMINATING`, and grace period starts \(default 30s\)
2. \(simultaneous with 1\) If the pod has a `preStop` hook defined, this is invoked
3. \(simultaneous with 1\) The pod is removed from the list of endpoints i.e. taken out of load balancing
4. \(simultaneous with 1, but after `preStop` if defined\) Container receives `SIGTERM`, and should prepare for shutdown
5. Grace period ends, and container receives `SIGKILL`
6. Pod disappears from the API, and is no longer visible for the client.

The platform will automatically add a `preStop`-hook that pauses the termination sufficiently that e.g. the ingress controller has time to update it's list of endpoints \(thus avoid sending traffic to a application while terminating\).

## Exposes relevant application metrics

The application should be instrumented using [Prometheus](https://prometheus.io/docs/instrumenting/clientlibs/), exposing the relevant application metrics. See the [metrics documentation](../observability/metrics.md) for more info.

## Writes structured logs to `stdout`

The application should emit `json`-formatted logs by writing directly to standard output. This will make it easier to index, view and search the logs later. See more details in the [logs documentation](../observability/logs/README.md).

## Crashes on fatal errors

If the application reaches an unrecoverable error, you should let it crash.
Instead, you should immediately exit the process and let the kubelet restart the container.

By restarting the container, you allow for the eventual readiness of other dependencies.

## Implements `readiness` and `liveness` endpoints

The `readiness`-probe is used by Kubernetes to determine if the application should receive traffic, while the `liveness`-probe lets Kubernetes know if your application is alive. If it's dead, Kubernetes will remove the pod and bring up a new one.
They should be implemented as separate services as they usually have different characteristics.

* `liveness`-probe should simply return `HTTP 200 OK` if main loop is running, and `HTTP 5xx` if not.
* `readiness`-probe returns `HTTP 200 OK` is able to process requests, and `HTTP 5xx` if not. If the application has dependencies to e.g. a database to serve traffic, it's a good idea to check if the database is available in the `readiness`-probe.

Useful resources on the topic:

* [https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
* [https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-setting-up-health-checks-with-readiness-and-liveness-probes)
* [https://medium.com/metrosystemsro/kubernetes-readiness-liveliness-probes-best-practices-86c3cd9f0b4a](https://medium.com/metrosystemsro/kubernetes-readiness-liveliness-probes-best-practices-86c3cd9f0b4a)
* [https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-revisited-how-to-avoid-shooting-yourself-in-the-other-foot/#letitcrash](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-revisited-how-to-avoid-shooting-yourself-in-the-other-foot/#letitcrash)


## 