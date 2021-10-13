# amq-keycloak-openshift

This reopo is following kustomize structure that can be used with Argo CD to be deployed on an OpenShit cluster.

## Install OpenShift Gitops

Install Argo CD from OpenShift's OperatorHub via the webconsole or by adding this `Subsciption` (see _openshift-gitops-operator.yaml_):

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: preview
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

You should see several pods running in the `openshift-gitops` namespace after a few minutes.

Login to Argo CD via Route:

```
oc get route -ojsonpath='{$.spec.host}' -n openshift-gitops openshift-gitops-server
```

as `admin` using password:

```
oc extract -n openshift-gitops secret/openshift-gitops-cluster --to=- --keys=admin.password
```

## Deploy namespace

The directory structure in this repo follows the usual `kustomize` layout. The `base` directory contains the shared yamls for _AMQ Broker_ and _Keycloak_ utilizing operators. 

The files in `overlay/my-namespace` are prepared so they can be used with Argo CD to create the namespace and sync from this git repo. Just create an _Application_ like this (see _argocd-application.yaml_):

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-namespace
  namespace: openshift-gitops
spec:
  destination:
    namespace: my-namespace
    server: https://kubernetes.default.svc
  project: default
  source:
    path: overlay/my-namespace
    repoURL: https://bszeti@github.com/amq-keycloak-openshift
    targetRevision: HEAD
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Automatic sync is not enabled, but it can be added of course.

## Ordering

If we were installing our resources manually, the operators should be istalled first through OperatotHub and we should wait until the operator specific CRDs are created on the cluster. With ArgoCD we added `argocd.argoproj.io/sync-wave: "-1"` annotation on these resources, so they are installed first. Due to the asynchronous nature of Kubernetes, this may not be enough during the first _sync_ on a cluster where these CRDs don't exist yet. In this case we should run a partial sync first selecting only the _Namespace_, _OperatorGroup_ and _Subscription_ resources.

## Certificates

Certificates used in the AMQ secrets were generated with _keytool_:

```
keytool -genkey -keyalg RSA -keystore broker.ks -storepass secret -keypass secret -dname 'CN=my-broker'
keytool -export -keystore broker.ks -storepass secret -rfc -file broker.pem
keytool -import -keystore broker.ts -storepass changeit -file broker.pem
```

and added to the secrets like this:

```
oc create secret generic amq-broker-ssl-acceptor --from-file=broker.ks=broker.ks --from-file=client.ts=broker.ts --from-literal=trustStorePassword=changeit --from-literal=keyStorePassword=secret
```

## Test

Get exposed acceptor's hostname:

 `oc get route -ojsonpath='{.spec.host}' -n my-namespace my-broker-ssl-0-svc-rte`

Send a message:

```
bin/artemis producer \
  --url "tcp://$(oc get route -ojsonpath='{.spec.host}' -n my-namespace my-broker-ssl-0-svc-rte):443?verifyHost=false&sslEnabled=true&trustStorePath=broker.ts&trustStorePassword=changeit" \
  --user myproducer --password secret \
  --text-size 100 --message-count 1 \
  --destination queue://test
```

or using AMQP protocol (instead of CORE):

```
bin/artemis producer \
  --url "amqps://$(oc get route -ojsonpath='{.spec.host}' -n my-namespace my-broker-ssl-0-svc-rte):443?transport.verifyHost=false&transport.trustStoreLocation=broker.ts&transport.trustStorePassword=changeit" \
  --user myproducer --password secret \
  --text-size 100 --message-count 1 \
  --destination queue://amqp-test \
  --protocol amqp
```

Receive messages:

```
bin/artemis consumer \
  --url "tcp://$(oc get route -ojsonpath='{.spec.host}' -n my-namespace my-broker-ssl-0-svc-rte):443?verifyHost=false&sslEnabled=true&trustStorePath=broker.ts&trustStorePassword=changeit" \
  --user myconsumer --password secret \
  --message-count 1 \
  --destination queue://test
```

or through AMQP protocol:

```
bin/artemis consumer \
  --url "amqps://$(oc get route -ojsonpath='{.spec.host}' -n my-namespace my-broker-ssl-0-svc-rte):443?transport.verifyHost=false&transport.trustStoreLocation=broker.ts&transport.trustStorePassword=changeit" \
  --user myconsumer --password secret \
  --message-count 1 \
  --destination queue://amqp-test \
  --protocol amqp
```
