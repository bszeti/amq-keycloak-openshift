# amq-keycloak-openshift

This reopo is following kustomize structure that can be used with Argo CD to be deployed on an OpenShit cluster.

Certificates were generated with keytool:

```
keytool -genkey -keyalg RSA -keystore broker.ks -storepass secret -keypass secret -dname 'CN=my-broker'
keytool -export -keystore broker.ks -storepass secret -rfc -file broker.pem
keytool -import -keystore broker.ts -storepass changeit -file broker.pem
```