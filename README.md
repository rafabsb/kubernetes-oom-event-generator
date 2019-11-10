# kubernetes-oom-event-generator

[![Build Status](https://travis-ci.org/xing/kubernetes-oom-event-generator.svg?branch=master)](https://travis-ci.org/xing/kubernetes-oom-event-generator)

Generates Kubernetes Event when a container is starting and indicates that
it was previously out-of-memory killed.

## Design

The Controller listens to the Kubernetes API for new Events and changes to
Events. Every time a notification regarding an Event is received it checks
whether this Event refers to a "ContainerStarted" event, based on the `Reason`
for the Event and the `Kind` of the involved object. If this is the case
and the Event constitutes a change (meaning it is not a not-changing update,
which happens when the resync, that is executed every two minutes, is run) it checks
the underlying Pod resource. Should the `LastTerminationState` of the Pod refer to
an OOM kill the controller will emit a Kubernetes Event with a level of `Warning`
and a reason of `PreviousContainerWasOOMKilled`.

## Usage

    Usage:
      kubernetes-oom-event-generator [OPTIONS]

    Application Options:
      -v, --verbose= Show verbose debug information [$VERBOSE]
          --version  Print version information

    Help Options:
      -h, --help     Show this help message

Run the pre-built image [`xingse/kubernetes-oom-event-generator`] locally (with
local permission):

    echo VERBOSE=2 >> .env
    docker run --env-file=.env -v $HOME/.kube/config:/root/.kube/config xingse/kubernetes-oom-event-generator

## Deployment

Example Clusterrole:

    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRole
    metadata:
      name: xing:controller:kubernetes-oom-event-generator
    rules:
      - apiGroups:
          - ""
        resources:
          - pods
          - pods/status
        verbs:
          - get
          - list
          - watch
      - apiGroups:
          - ""
        resources:
          - events
        verbs:
          - create
          - patch

Run this controller on Kubernetes with the following commands:

    kubectl create serviceaccount kubernetes-oom-event-generator \
      --namespace=kube-system

    kubectl create -f path/to/example-clusterrole.yml
    # alternatively run: `cat | kubectl create -f -` and paste the above example, hit Ctrl+D afterwards.

    kubectl create clusterrolebinding xing:controller:kubernetes-oom-event-generator \
      --clusterrole=xing:controller:kubernetes-oom-event-generator \
      --serviceaccount=kube-system:kubernetes-oom-event-generator

    kubectl run kubernetes-oom-event-generator \
      --image=xingse/kubernetes-oom-event-generator \
      --env=VERBOSE=2 \
      --serviceaccount=kubernetes-oom-event-generator

## Alerting on OOM killed pods

When [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) is deployed
in the cluster and a [prometheus](https://prometheus.io) installation is scraping the metrics
you can alert on OOM-killed pods using the prometheus alert manager.

Example alert:

    alert: ComponentOutOfMemory
    expr: sum_over_time(kube_pod_container_status_terminated_reason{reason="OOMKilled"}[5m])
      > 0
    for: 10s
    labels:
      severity: warning
    annotations:
      description: Critical Pod {{$labels.namespace}}/{{$labels.pod}} was OOMKilled.

# Developing

You will need a working Go installation (1.11+) and the `make` program.  You will also
need to clone the project to a place outside you normal go code hierarchy (usually
`~/go`), as it uses the new [Go module system].

All build and install steps are managed in the central `Makefile`. `make test` will fetch
external dependencies, compile the code and run the tests. If all goes well, hack along
and submit a pull request. You might need to modify the `go.mod` to specify desired
constraints on dependencies.

Make sure to run `go mod tidy` before you check in after changing dependencies in any way.

[Go module system]: https://github.com/golang/go/wiki/Modules
[`xingse/kubernetes-oom-event-generator`]: https://hub.docker.com/r/xingse/kubernetes-oom-event-generator

## Releases

Releases are a two-step process, beginning with a manual step:

* Create a release commit
  * Increase the version number in [kubernetes-oom-event-generator.go/VERSION](kubernetes-oom-event-generator.go#20)
  * Adjust the [CHANGELOG](CHANGELOG.md)
* Run `make release`, which will create an image, retrieve the version from the
  binary, create a git tag and push both your commit and the tag

The Travis CI run will then realize that the current tag refers to the current master commit and
will tag the built docker image accordingly.
