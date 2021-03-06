# Knative and OpenShift Demo for KubeCon

This is a Knative on OpenShift demo as given at KubeCon. It walks you through on what Knative has to
offer:

1. It builds an application through Knative's Build component (which makes use of the existing OpenShift
Build mechanic underneath).
2. The built image is then deployed as a Knative Service, which means it scales automatically, even down
to nothing as we'll see.
3. We wire an EventSource emitting Kubernetes events to our application through Knative's Eventing capabilities.

## 0. Setting up an OpenShift cluster

Before we get started, let's setup an environment. This setup guide assumes the usage of `minishift` and the
setup scripts are tailored to facilitate that. To setup a fresh minishift cluster with OLM, Istio and all parts
we need from Knative installed, run the following script:

```bash
git clone git@github.com:openshift-cloud-functions/knative-operators.git
./knative-operators/etc/scripts/install.sh
```

> This script is destructive towards the 'knative' profile in minishift.

After this script exits without any errors, your cluster will be setup and ready to go. Next, let's make `oc`
(OpenShift's CLI) available in the current terminal and set the namespace to `myproject`:

```bash
eval $(minishift oc-env)
oc project myproject
```

There we are, let's get crackin'.

### Setting up access rights

To be able to access everything we need for the demo, we'll need to add certain rights to the `default` ServiceAccount
in our namespace. For the Build part it needs CRUD access to all OpenShift Build related entities (Build, BuildConfig,
ImageStream) and for the Eventing part, the ServiceAccount needs to be able to get events in the namespace.

To set those up, run:

```bash
oc apply -f build/000-rolebinding.yaml
oc apply -f eventing/000-rolebinding.yaml
```

## 1. The Build component

The Build component in Knative is not so much a utility to build images themselves. It rather provides primitives,
to be able to string together the tools you want to do your image build. In a sense, it's an abstraction layer above
all the tools out there to build an image. In our case, the most prominent example is OpenShift's own build capability,
so this will show you, how we can implement a Knative Build by the means of an OpenShift Build.

Knative provides a mechanism called **BuildTemplates**, where you define a blueprint for a build, which contains the
arguments that that build might need to do its job of building the image in the end. An example of such a template can
be seen below. This template allows building images through Knative Build based on OpenShift's Build capability, so you
can use the tools you're used to while taking full advantage of Knative Build on top of it.

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: BuildTemplate
metadata:
  name: openshift-builds
spec:
  parameters:
  - name: IMAGE
    description: The name of the image to push
  - name: NAME
    description: Build configuration name
  - name: IMAGE_STREAM
    description: The image stream to use as input for the build
  - name: TO_DOCKER
    description: Push the image to a Docker repository or not (true by default)
    default: "false"
  - name: DIRECTORY
    description: The directory containing the app
    default: /workspace
  - name: OC_BUILDER_IMAGE
    description: The name of the builder image to use
    default: docker.io/vdemeester/kobw-builder:0.1.0
  steps:
  - name: kobw-create-or-update
    image: "${OC_BUILDER_IMAGE}"
    args: ["create", "--name=${NAME}", "--image=${IMAGE}", "--image-stream=${IMAGE_STREAM}", "--to-docker=${TO_DOCKER}", "."]
    workingDir: "${DIRECTORY}"
    env:
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
  - name: kobw-run
    image: "${OC_BUILDER_IMAGE}"
    args: ["run", "--name=${NAME}"]
    env:
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
```

To achieve the desired effect, this template wraps an OpenShift Build entity to perform the build of the desired application.
To "install" that template, we need to `oc apply` it by:

```bash
oc apply -f build/000-build-template.yaml
```

This on its own will do nothing. It only defines a template for a build to reference. A **Build**, as seen below, then includes
such a references and provides it with the arguments needed to perform the build.

```yaml
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: oc-build-1
spec:
  source:
    git:
      url: https://github.com/openshift-cloud-functions/openshift-knative-application
      revision: master
  template:
    name: openshift-builds
    arguments:
    - name: IMAGE_STREAM
      value: golang:1.11
    - name: IMAGE
      value: "helloworld:latest"
    - name: NAME
      value: helloworld-build
```

In this particular case, the source code is taken from a git repository's master branch. The arguments to the template then define
that we want to build a Golang based application and want to name the image `helloworld:latest`.

Now we could go ahead and run this build on its own by applying it like the template above and using
`oc apply -f build/010-build.yaml`, but it's much more interesting to see how it's stringed together
with creating a deployment in the same step. After all, an image on it's own is worth nothing if its
not deployed.

## 2. The Serving component

The Serving component revolves around the concept of a **Service** (for clarity, it's called **KService**
throughout, because that term is vastly overloaded in the Kubernetes space). A KService is a higher level
construct that describes how an application is built and then how it's deployed in Knative.

A KService definition looks like this:

```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: helloworld-openshift
  namespace: myproject
spec:
  runLatest:
    configuration:
      build:
        source:
          git:
            url: https://github.com/openshift-cloud-functions/openshift-knative-application
            revision: master
        template:
          name: openshift-builds
          arguments:
          - name: IMAGE_STREAM
            value: golang:1.11
          - name: IMAGE
            value: "helloworld:latest"
          - name: NAME
            value: helloworld-build
      revisionTemplate:
        metadata:
          annotations:
            alpha.image.policy.openshift.io/resolve-names: "*"
        spec:
          containerConcurrency: 1
          container:
            imagePullPolicy: Always
            image: docker-registry.default.svc:5000/myproject/helloworld:latest
            env:
            - name: BAR
              value: "bar"
```

It's very apparent that the `spec.runLatest.configuration.build` part is a one-to-one copy of the Build
manifest we created above. If we apply this specification, Knative will go ahead and build the image
through the capabilities described above. Once that's done it'll go ahead and deploy a **Revision**,
an immutable snapshot of an application. Think of it as the **Configuration** of the application at a
specific point in time.

The `revisionTemplate` part of the KService specification describes how the Pods that will contain the
application will be created and deployed. In this case, it's going to pull the image that's built
from the OpenShift internal registry.

We create the KService by, you guessed it, applying the file through `oc`:

```bash
oc apply -f build/020-serving.yaml
```

Now the build starts running (show in Console, link TODO) and once it finishes (Wait for finish) Knative
will produce a plain Kubernetes deployment that contains the container we specified in the RevisionSpec
above. (Show pods in Console, link TODO)

Now, to see that the service is actually running, we're going to send a request against it. To do so,
we'll get the domain of the KService:

```bash
$ oc get kservice
NAME                   DOMAIN                                       LATESTCREATED                LATESTREADY                  READY     REASON
helloworld-openshift   helloworld-openshift.myproject.example.com   helloworld-openshift-00001   helloworld-openshift-00001   True
```

The *DOMAIN* part is what we're interested in. The actual request is then sent to the entrypoint of our
servicemesh, which is listening on `$(minishift ip):32380`). Stringed together we get the following
command:

```bash
$ curl -H "Host: helloworld-openshift.myproject.example.com" "http://$(minishift ip):32380/health"

                    888 888             888
                    888 888             888
                    888 888             888
888d888 .d88b.  .d88888 88888b.  8888b. 888888
888P"  d8P  Y8bd88" 888 888 "88b    "88b888
888    88888888888  888 888  888.d888888888
888    Y8b.    Y88b 888 888  888888  888Y88b.
888     "Y8888  "Y88888 888  888"Y888888 "Y888
```

It's alive!

## 3. The Eventing Component

The previous steps showed you how to invoke a KService from an HTTP request, in that case submitted via curl.
Knative Eventing is all about how you can invoke those applications in response to other events such as those received from message brokers or external applications.
In this part of the demo we are going to show how you can receive Kubernetes platform events and route those to a Knative Serving application.

Knative Eventing is built on top of three primitives:
* Event Sources
* Channels
* Subscriptions

**Event Sources** are the components that receive the external events and forward them onto **Sinks** which can be a **Channel**.
Out of the box we have Channels that are backed by Apache Kafka, GCPPubSub and a simple in-memory channel.
**Subscriptions** are used to connect Knative Serving application to a Channel so that it can respond to the events that the channel emits.

Let's take a look at some of those resources in more detail.

We have some yaml files prepared which describe the various resources, firstly the channel.

```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Channel
metadata:
  name: testchannel
spec:
  provisioner:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: ClusterChannelProvisioner
    name: in-memory-channel
```
Here we can see that we've got a channel named `testchannel` and it is an `in-memory-channel` which is what we're going to use for this demo - in production we would probably use Apache Kafka.
Let's deploy that so that we can use it in a later stage.

```bash
oc apply -f eventing/010-channel.yaml -n myproject
```

Next let's take a look at the EventSource.

```yaml
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: KubernetesEventSource
metadata:
  name: testevents
spec:
  namespace: myproject
  serviceAccountName: default
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: testchannel
```

This is starting to get a little bit more interesting.
This EventSource is of type `KubernetesEventSource`.
The `KubernetesEventSource` is available as an example EventSource from the Knative project.
It is going to receive Kubernetes platform events from a particular namespace, in this case `myproject`.
All of the messages received from Kubernetes are going to be routed to the `sink` that is specified as part of the EventSource.
Here we can see that the `testchannel` we defined earlier is specified as the `sink`.

If we apply that we will see a pod created which is an instance of the `KubernetesEventSource` that is configured with the Channel we defined.

```bash
oc apply -f eventing/020-k8s-event-source.yaml -n myproject
```

```bash
$ oc get pods
NAME                                               READY     STATUS    RESTARTS   AGE
testevents-dbclc-b6gv8-db66b4985-lqljv             2/2       Running   0          15s
```

Question: do you want to show the logs here?

The EventSource is up and running and the final piece of the Knative Eventing is how we wire everything together.
This is done via a Subscription.

```yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Subscription
metadata:
  name: testevents-subscription
  namespace: myproject
spec:
  channel:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: testchannel
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: helloworld-openshift
```

This Subscription again references the `testchannel` and also defines a `Subscriber`, in this case the Serving application `helloworld-openshift` we built earlier.
This basically means that we have subscribed the previously built application to the `testchannel`.
Once we apply the `Subscription` any Kubernetes platform events from the `myproject` namespace will be routed to the `helloworld-openshift` application.
Behind the scenes there is a `SubscriptionController` which is doing the wiring for us.

```bash
oc apply -f eventing/030-subscription.yaml -n myproject
```

It may take a few seconds for the application pod to become ready.
If we take a look at the logs of the application we may start to see some messages.
Those represent the Kubernetes events flowing into our application.

```bash
oc logs helloworld-openshift-00001-deployment-5f8dbfb49c-txg8j -c user-container
```

If there are no events present, we can generate some Kubernetes events by creating a pod that when starting will generate events such as `pod created` and `container started`.
We will create a new pod using the `busybox` image to show this.

```bash
kubectl run -i --tty busybox --image=busybox --restart=Never -- sh
```

If we look at the logs again we will see some Kubernetes platform events appearing which are wrapped in CloudEvents.
We can see the CloudEvents headers which contain some metadata about the event such as the source of the event, timestamp and Ids.
In the payload of the message we can see the specific details of the event such as the pod was starting.

> This will depend on the exact messages which show up in the demo.

So, what we've seen here is building a Knative Serving application using Knative Build but backed by the OpenShift build mechanism.
We've shown how this Serving application can respond to an HTTP request.
Finally we've shown how this application can be wired to an external event source, in this case Kubernetes platform events.
