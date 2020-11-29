+++ 
date = "2020-11-23" 
title = "Deploying a Spring Boot Kotlin app on Kubernetes with Docker and Helm"
slug = "spring-boot-kubernetes-kotlin-docker" 
tags = ["spring boot", "docker", "kubernetes", "kotlin", "helm"]
categories = []
images = ["spring-initializr.png"]
series = ["helm", "Kubernetes"]
+++

{{< figure src="/images/boots.jpeg" caption="Boots and flowers from [unsplash](https://unsplash.com/photos/szHQOklNOL8) by [Sarvenaz Sorour](https://unsplash.com/@sarvenazsorour)" >}}

The easiest hello-world application deployed on Kubernetes with Helm that you can put together.

Code is hosted in [Github](https://github.com/gateixeira/demo-spring-boot)

## Prerequisites (with the versions used in this tutorial):

- **Docker** (Engine 19.03.13)
- **Minikube**  (1.15.1)
- **Helm** (3.4.0)
- **Kubectl** (matching kubernetes version)


## Spring-boot app setup

On the [spring initializr](https://start.spring.io/), create your app project. In this case we go with Gradle, Kotlin and Spring Boot 2.4.0 for Java 15 packaged in a JAR file. For the dependencies, let's select Spring Web so that we can initialize a Hello World REST API:

{{< figure src="/images/spring-initializr.png" caption="" >}}

After generating it, you should have your project folder, which is the root to our repository.

## Writing a hello-world REST API

for a simple REST API, just create a controller folder on the same folder as your `DemoSpringBootApplication.kt`, create an `Application.kt` file inside and add this content:

```kotlin
package com.gateixeira.demospringboot.controller

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class Application {
    @GetMapping("/")
    fun home(): String {
        return "Hello World";
    }
}
```

### Build and run app

The generated zip file comes with an embedded Gradle executable, so in order to build our app and make sure that everything is working just do a `./gradlew build`:

```bash
~/code/demos/demo-spring-boot master > ./gradlew build
Starting a Gradle Daemon (subsequent builds will be faster)

> Task :test
2020-11-29 20:44:55.227  INFO 43582 --- [extShutdownHook] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'

BUILD SUCCESSFUL in 8s
7 actionable tasks: 3 executed, 4 up-to-date
```

This process generates a jar file under `build/libs`. This is our executable. You can run it with `java -jar build/libs/<your-app>`. Your terminal will show the Spring banner and the following message:

`DemoSpringBootApplicationKt        : Started DemoSpringBootApplicationKt in 1.492 seconds (JVM running for 1.852)`

Now if you go to `localhost:8080` you see "Hello World".


## Creating docker image

Now let's write a very simple dockerfile to generate our image. We are going to use [Open JDK's](https://openjdk.java.net/) alpine version as our base image, with Java 15 to match what we chose above.

```Dockerfile
FROM openjdk:15-jdk-alpine
ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

It simply copies the generated JAR file to the container as app.jar and uses this as entrypoint for a Java application.

## Writing Helm chart

Helm will be used to manage our application lifecycle inside Kubernetes. This will probably be the simplest chart possible but can easily be extended.

First, create a `charts` folder on your project's root. Then inside, create a `Chart.yaml` that specifies basic chart information:

```yaml
apiVersion: v2
name: helm
description: A Helm chart for our demo-spring-boot application
type: application
version: 0.1.0
appVersion: latest
```

In the charts folder, create a `templates` folder and add a deployment.yaml inside:

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
      restartPolicy: Always
```

Everything defined as `{{ variable name }}`  is a placeholder and will be set in the values file.

This is a very simple deployment file that specifies some metadata, labels match and selectors as well as the name of the container, it's image repository address and pull policy.

Now in the same folder, create a `service.yaml` file:

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: {{ .Values.namespace }}
  name: {{ .Values.appName }}
  labels:
    app: {{ .Values.appName }}
spec:
  type: LoadBalancer
  loadBalancerIP: 172.16.0.1
  sessionAffinity: None
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: {{ .Values.appName }}
```

This file defines the LoadBalancer configuration for the application, with an IP Address as entry point and the respective port in which the service is exposed (80) and the app is running (8080). Selector specifies the deployment to which this service relates.

Now in order to connect everything and replace the placeholders, let's create a `values.yaml` file in the charts folder:

```yaml
appName: demo-spring-boot
namespace: default
appVersion: latest
replicaCount: 1
image:
  registry: gateixeira
  pullPolicy: Never
```

Our app will be on the default namespace, have a `latest` version and just 1 single replica. The container image will be composed of the tag that was added on the `docker build command`. In this case: `gateixeira` as the container registry, the app name as image name and concatenated with the version. `pullPolicy` is set to `never` since we are building a local image.

## Deploying to Kubernetes

### Starting Minikube

Our Kubernetes will be running locally with [Minikube](https://minikube.sigs.k8s.io/):

```bash
~/code/demos/demo-spring-boot master > minikube start --memory 2048 --cpus 2 --disk-size 10g --kubernetes-version v1.18.8
😄  minikube v1.15.1 on Darwin 10.15.6
❗  Both driver=docker and vm-driver=virtualbox have been set.

    Since vm-driver is deprecated, minikube will default to driver=docker.

    If vm-driver is set in the global config, please run "minikube config unset vm-driver" to resolve this warning.

✨  Using the docker driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🔥  Creating docker container (CPUs=2, Memory=2048MB) ...
🐳  Preparing Kubernetes v1.18.8 on Docker 19.03.13 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass

❗  /usr/local/bin/kubectl is version 1.16.7, which may have incompatibilites with Kubernetes 1.18.8.
    ▪ Want kubectl v1.18.8? Try 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

Now our app should be ready to be deployed on Kubernetes. First, let's build the docker image.

Start by switching the docker environment to use Minikube's daemon, otherwise Minikube can't find the image:

`eval $(minikube docker-env)`

For those running on Windows, the powershell equivalent is 

`minikube docker-env | Invoke-Expression`

Then build the image with `docker build -t <image_name>:<image_tag> .`

```bash
~/code/demos/demo-spring-boot master > docker build -t gateixeira/demo-spring-boot:latest .                                                                                     4s
Sending build context to Docker daemon  23.16MB
Step 1/4 : FROM openjdk:15-jdk-alpine
 ---> f02adfce91a2
Step 2/4 : ARG JAR_FILE=build/libs/*.jar
 ---> Using cache
 ---> 9bb20485adba
Step 3/4 : COPY ${JAR_FILE} app.jar
 ---> e651311f698a
Step 4/4 : ENTRYPOINT ["java", "-jar", "/app.jar"]
 ---> Running in 0ed363cc6d8b
Removing intermediate container 0ed363cc6d8b
 ---> 0033b968a8a9
Successfully built 0033b968a8a9
Successfully tagged gateixeira/demo-spring-boot:latest
```

To install your application on Minikube, run:
`helm upgrade --install demo-spring-boot charts --values charts/values.yaml`. Where `charts` is your charts folder.

```bash
~/code/demos/demo-spring-boot master > helm upgrade --install demo-spring-boot charts --values charts/values.yaml                                               INT kube minikube
WARNING: "kubernetes-charts.storage.googleapis.com" is deprecated for "stable" and will be deleted Nov. 13, 2020.
WARNING: You should switch to "https://charts.helm.sh/stable"
Release "demo-spring-boot" does not exist. Installing it now.
NAME: demo-spring-boot
LAST DEPLOYED: Sun Nov 29 21:25:39 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

You can check your application is running with `kubectl get pods`:

```bash
~/code/demos/demo-spring-boot master !1 > kubectl get pods                                                                                                          kube minikube
NAME                               READY   STATUS    RESTARTS   AGE
demo-spring-boot-684cc98cc-zxgfv   1/1     Running   0          101s
```

With your app running, Minikube can tunnel your application and provide a URL to access it:

```bash
~/code/demos/demo-spring-boot master !1 > minikube service demo-spring-boot --url
🏃  Starting tunnel for service demo-spring-boot.
|-----------|------------------|-------------|------------------------|
| NAMESPACE |       NAME       | TARGET PORT |          URL           |
|-----------|------------------|-------------|------------------------|
| default   | demo-spring-boot |             | http://127.0.0.1:61460 |
|-----------|------------------|-------------|------------------------|
http://127.0.0.1:61460
```

Congratulations! :smile:

If you open your browser on `http://127.0.0.1:61460` you should see `Hello World`.
