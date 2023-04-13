Openshift local, WildFly application secured with Keycloak SAML

* Using the Keycloak operator to deploy a secured Keycloak instance (TLS, strict hostname).
* Using WildFly Maven plugin to provision WildFly and Keycloak SAML Galleon feature-packs.
* Using Helm charts to build and deploy application in openshift

1- Setup CRC
Download an openshift local installation: https://console.redhat.com/openshift/create/local
Setup and start Openshift local: https://access.redhat.com/documentation/en-us/red_hat_openshift_local/2.16/html/getting_started_guide/index

Set limit ranges required to avoid pod to consume all memory.
oc apply -f limit-ranges.yaml 

2- Configure keycloak

* Configure keycloak operator from Openshift Console as documented in: https://www.keycloak.org/operator/installation

* Create a keycloak instance (https://www.keycloak.org/operator/basic-deployment) but with a special hostname in order for the route to be complete.

oc apply -f example-kc.yaml

* Create the Keycloak Realm and users required by this example

oc apply -f keycloak-saml-realm.yaml

* Create the SAML client keystore

keytool -genkeypair -alias saml-app -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore keystore.p12 -storepass password -dname "CN=saml-basic-auth,OU=EAP SAML Client,O=Red Hat EAP QE,L=MB,S=Milan,C=IT" -ext ku:c=dig,keyEncipherment -validity 365

keytool -importkeystore -deststorepass password -destkeystore keystore.jks -srckeystore keystore.p12 -srcstoretype PKCS12 -srcstorepass password'

* Create the keystore secret that will be mounted in deployment

oc create secret generic saml-app-secret --from-file=keystore.jks=./keystore.jks --type=opaque


3- Build and deploy the example

* Create the secret that contains the application env 

oc apply -f saml-secret.yaml

* Build and Deploy the application

helm install saml-app -f helm.yaml wildfly/wildfly

* Edit the `saml-secret.yaml` to add the env variables to configure the application route.

```yaml
stringData:
  ...
  HOSTNAME_HTTPS: <saml-app application route>
```

The value of `HOSTNAME_HTTPS` corresponds to the output of

```
echo $(oc get route saml-app --template='{{ .spec.host }}')
```

Then update the secret with `oc apply -f saml-secret.yaml`.

Let's redeploy the application to make sure it uses the new environment variables:

```
oc rollout restart deploy saml-app
```

4- Access the application

* In web browser https://$(oc get route saml-app --template='{{ .spec.host }}/saml-app')/saml-app
* Click on "Access Secured Servlet"
* Log as user, password user
* You will see the Prinicpal id displayed.
* Click on logout to logout
