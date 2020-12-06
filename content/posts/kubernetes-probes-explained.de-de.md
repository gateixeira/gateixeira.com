+++ 
date = "2020-12-06"
title = "An easy way to a highly available service with Kubernetes probes, Kotlin and Helm"
slug = "an-easy-way-to-a-highly-available-service-with-kubernete-probes-kotlin-and-helm"
tags = ["probes", "docker", "kubernetes", "kotlin", "helm"]
categories = []
images = ["/images/run.jpg"]
series = ["helm", "Kubernetes"]
+++

{{< figure src="/images/run.jpg" caption="Ready to run. From [unsplash](https://unsplash.com/photos/9HI8UJMSdZA) by [Braden Collum](https://unsplash.com/@bradencollum)" >}}

A simple explanation with examples of how Kubernetes probes work and help keeping your services always up.

Code is hosted in [Github](https://github.com/gateixeira/demo-spring-boot)

## Prerequisites (with the versions used in this tutorial):

- **Docker** (Engine 19.03.13)
- **Minikube**  (1.15.1)
- **Helm** (3.4.0)
- **Kubectl** (matching kubernetes version)

## About Probes

Probes (sometimes heartbeat or [health checks](https://microservices.io/patterns/observability/health-check-api.html)) are a fundamental feature of Microservices. **They perform constant checks to know the state of a service and enable recovery whenever things go sour.**

You can set up your probes as a really simple endpoint that returns a valid HTTP status code whenever the service is running or they can be more complex and look for things like the state of the database connection or whether the service can communicate with an external dependency, send messages to a queue, etc...

Kubernetes provides out-of-the-box options to set up those checks. If configured together with a multi-replica, multi-node environment, probes allow for a proactive error handling with important features to recover from failures.

They are divided in three categories:

### Liveness Probe

Liveness Probe is used to know when a pod/container should be restarted because it is running but not functional. It helps increasing availablity by bringing the service back to its original state.

Whenever the `initialDelaySeconds` value is passed, Kubernetes starts to perform the specified command and if the response is something other than 200-399 it fails. Unless `failureThreshold` value is changed, the pod will be restarted after 3 failures in a row.

### Readiness Probe

Readiness Probe is similar to the liveness probe in terms of options and implementation but has the function to inform Kubernetes when a pod is ready to receive traffic. Sometimes an application has to perform tasks on startup. Maybe load some data in memory, warm up a cache, process information before receiving new requests, you name it. In these scenarios, even if the service is actually running it isn't ready to process any request.

Kubernetes changes the pod condition to ready whenever the ready endpoint returns an HTTP code between 200-399. `successThreshold` value can be used to determine how many times the return has to be valid before the pod receives traffic.

### Startup Probe

Startup Probe provides a delay for both other probes to start. It's specially helpful when there's a slow starting container which guaranteed takes longer to finalize and make the liveness and readiness probe endpoints available. We will skip it in this tutorial.

## Implementation starting point

In the [last article](https://www.gateixeira.com/posts/spring-boot-kubernetes-kotlin-docker/) of this series we created a Kotlin app with Spring Boot and deployed on Kubernetes using Helm. Now we will extend it to include liveness and readiness probes.

## Writing probe endpoints

In the `Application.kt` file, let's create two new endpoints:

```kotlin
    @GetMapping("/health")
    @ResponseStatus(HttpStatus.OK)
    fun health(): String {
        println("Liveness Probe");
        return "Healthy";
    }

    @GetMapping("/ready")
    @ResponseStatus(HttpStatus.OK)
    fun ready(): String {
        println("Readiness Probe");
        return "Ready";
    }
```

Note that this will always return success, provided that the service is running. Later we will simulate failures.

## Extending Helm deployment template

Kubernetes probes are part of the deployment specification. So in your Helm's `deployment.yaml` template, add the following inside the `containers` spec:

```yaml
    livenessProbe:
        httpGet:
            path: /health
            port: 8080
        initialDelaySeconds: 10
        periodSeconds: 3
    readinessProbe:
        httpGet:
            path: /ready
            port: 8080
        initialDelaySeconds: 15
        periodSeconds: 5
```

In this case we are telling Kubernetes to start performing the liveness checks after 10 seconds and repeating it every 3 seconds, at the `/health` endpoint on the service port and readiness checks after 15 seconds, every 5 seconds on endpoint `/ready`.

### Build and run app

To test the configuration, just build and run the app on Kubernetes (instructions on how to do this with Minikube are available [here](https://www.gateixeira.com/posts/spring-boot-kubernetes-kotlin-docker/index.html#starting-minikube) or you can use the run.sh script on the root of the repository).

After running `kubectl get pods` shows that the pod is running and the logs are printing our messages:

```bash
~/code/demos/demo-spring-boot master >2 > kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
demo-spring-boot-7f7769cfbd-k747j   1/1     Running   0          19s
~/code/demos/demo-spring-boot master >2 > kubectl logs -f demo-spring-boot-7f7769cfbd-k747j

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.0)

2020-12-06 20:26:05.812  INFO 1 --- [           main] c.g.d.DemoSpringBootApplicationKt        : Starting DemoSpringBootApplicationKt using Java 15-ea on demo-spring-boot-7f7769cfbd-k747j with PID 1 (/app.jar started by root in /)
2020-12-06 20:26:05.816  INFO 1 --- [           main] c.g.d.DemoSpringBootApplicationKt        : No active profile set, falling back to default profiles: default
2020-12-06 20:26:09.290  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-12-06 20:26:09.372  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-12-06 20:26:09.372  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.39]
2020-12-06 20:26:09.510  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-12-06 20:26:09.510  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 3524 ms
2020-12-06 20:26:10.276  INFO 1 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-12-06 20:26:10.705  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-12-06 20:26:10.775  INFO 1 --- [           main] c.g.d.DemoSpringBootApplicationKt        : Started DemoSpringBootApplicationKt in 6.565 seconds (JVM running for 7.716)
2020-12-06 20:26:12.900  INFO 1 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-12-06 20:26:12.900  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-12-06 20:26:12.901  INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms
Liveness Probe
Liveness Probe
Readiness Probe
Liveness Probe
Readiness Probe
Liveness Probe
Liveness Probe
Readiness Probe
```

## Probe failures

Now in order to show how Kubernetes probes react in case of failures, let's simulate failures on our endpoints. I extended the Kotlin code to have a Counter object which increments a value by 1 whenever a health check happens. When the value reaches 10, our service will not be considered ready anymore because it will return a 503 Service Unavailable. With 15, health check will fail and the pod will be restarted:

```kotlin
package com.gateixeira.demospringboot.controller

import org.springframework.http.HttpStatus
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.http.ResponseEntity

@RestController
class Application {
    object Counter {
        public var value = 0
        fun count(): Int = value++
    }

    @GetMapping("/")
    fun home(): String {
        return "Hello World";
    }

    @GetMapping("/health")
    fun health(): ResponseEntity<String> {
        Counter.count();

        if (Counter.value > 15) {
            println("Service not healthy. Count: " + Counter.value);
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body("Not healthy");
        } else {
            println("Service is healthy. Count: " + Counter.value);
            return ResponseEntity.status(HttpStatus.OK).body("Healthy");
        }
    }

    @GetMapping("/ready")
    fun ready(): ResponseEntity<String> {

        if (Counter.value > 10) {
            println("Service not ready. Count: " + Counter.value);
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body("Not ready");
        } else {
            println("Service is ready. Count: " + Counter.value);
            return ResponseEntity.status(HttpStatus.OK).body("Ready");
        }
    }
}
```

I also added a couple new properties to the `deployment.yaml` template just to make sure that the thresholds are set to 1 and it ended up looking like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: {{ .Values.namespace }}
  name: {{ .Values.appName }}
  labels:
    app: {{ .Values.appName }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      containers:
        - name: "{{ .Values.appName }}"
          image: "{{ .Values.image.registry }}/{{ .Values.appName }}:{{ .Values.appVersion }}"
          imagePullPolicy: "{{ .Values.image.pullPolicy }}"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 3
            successThreshold: 1
            failureThreshold: 1
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 1
      restartPolicy: Always
```

Now let's build and start our service again and see the probes happening. The terminal window on top shows the pod going from `READY 0/1` to `READY 1/1` after a succeesful readiness probe:

{{< figure src="/images/turn-ready.gif" caption="Pod is ready after log 'Service is ready. Count 2'" >}}

After a couple seconds, pod goes back to `READY 0/1` because counter already passed 10 so it will not receive traffic anymore:

{{< figure src="/images/turn-not-ready.gif" caption="Pod is not ready after count is higher than 10" >}}

And finally, pod will be restarted with the failure of a liveness probe:

{{< figure src="/images/restart.gif" caption="Pod restarts when counter passes 15" >}}

All these checks caused our pod lifecycle to go from starting up when it's not ready yet to running and ready to receive traffic. Then not ready, to a complete failure causing it to restart so that it could finally be *ready* again. All within the timespan of a minute:

```bash

~/code/demos/demo-spring-boot master > kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
demo-spring-boot-f8bd8cdff-k8csl   0/1     Running   0          9s
~/code/demos/demo-spring-boot master > kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
demo-spring-boot-f8bd8cdff-k8csl   1/1     Running   0          19s
~/code/demos/demo-spring-boot master > kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
demo-spring-boot-f8bd8cdff-k8csl   0/1     Running   0          48s
~/code/demos/demo-spring-boot master > kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
demo-spring-boot-f8bd8cdff-k8csl   0/1     Running   1          60s
~/code/demos/demo-spring-boot master >1 !2 > kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
demo-spring-boot-f8bd8cdff-k8csl   1/1     Running   1          79s
```

Congratulations! You learned how Kubernetes probes work in practice and might have just won a "9" in your application up time.
