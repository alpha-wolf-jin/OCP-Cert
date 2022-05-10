# OCP-Cert

**Create git**

```
echo "# OCP-Cert" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/OCP-Cert.git
git config --global credential.helper 'cache --timeout 7200'
git push -u origin main


git add . ; git commit -a -m "update README" ; git push -u origin main
```
**Purpose**

-    Inspect a new certificate signed by the classroom certificate authority.
-    Configure the ingress controller to use the new certificate.
-    Configure the master API to use the new certificate.
-    Validate the new certificate by accessing the web console and by running the oc login command.

**Important Things to Note**
-    The new certificate and key in PEM format.
-    The certificate must have a subjectAltName extension of *.apps.<OPENSHIFT-DOMAIN>, such as *.apps.ocp4.example.com, that enables using the certificate as a wildcard certificate for the .apps subdomain.
-    Changing the certificate used by the ingress controller operator does not affect certificates signed by the internal OpenShift certificate authority.

## Inspect the new certificate to view its subject, issuer, dates, and subject alternate name

```
$ openssl x509 -in wildcard-api.pem -noout -subject -issuer -ext 'subjectAltName' -dates
subject=C = US, ST = NC, L = Raleigh, O = "Red Hat, Inc.", OU = Training, CN = *.apps.ocp4.example.com
issuer=C = US, ST = NC, L = Raleigh, O = "Red Hat, Inc.", OU = Training, CN = GLS Training Classroom Certificate Authority
X509v3 Subject Alternative Name: 
    DNS:*.apps.ocp4.example.com, DNS:api.ocp4.example.com
notBefore=May 10 13:06:32 2022 GMT
notAfter=May  7 13:06:32 2032 GMT

```
> Note the CN and SAN. SAN has to include all FQDNs

> wildcard-api.pem is sgined by CA classroom-ca.pem

## Create an additional certificate combining the new certificate and the classroom CA certificate

```
$ cat wildcard-api.pem /etc/pki/tls/certs/classroom-ca.pem > combined-cert.pem


```

## Create a new TLS secret using the combined certificate

```
$ oc login -u admin -p redhat https://api.ocp4.example.com:6443

$ oc create secret tls custom-tls --cert combined-cert.pem --key wildcard-api-key.pem -n openshift-config

$ vim apiserver-cluster.yaml
apiVersion: config.openshift.io/v1
kind: APIServer
metadata:
  name: cluster
spec:
  servingCerts:
    namedCertificates:
    - names:
      - api.ocp4.example.com
      servingCertificate:
        name: custom-tls

```

>Ensure that the configuration map uses a data key of ca-bundle.crt.
**CN**
- *.apps.ocp4.example.com

**SAN**
- *.apps.ocp4.example.com
- api.ocp4.example.com
 
>Modify the cluster proxy so that it adds the new configuration map to the trusted certificate bundle

```
$ oc create configmap combined-certs --from-file ca-bundle.crt=combined-cert.pem -n openshift-config

$ vim proxy-cluster.yaml
apiVersion: config.openshift.io/v1
kind: Proxy
metadata:
  name: cluster
spec:
  trustedCA:
    name: combined-certs

$ oc apply -f proxy-cluster.yaml


```

## Create a TLS secret named custom-tls-bundle in the openshift-ingress namespace

```
$ oc create secret tls custom-tls-bundle --cert combined-cert.pem --key wildcard-api-key.pem  -n openshift-ingress

$ vim ingresscontrollers.yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  finalizers:
  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
  name: default
  namespace: openshift-ingress-operator
spec:
  defaultCertificate:
    name: custom-tls-bundle
  replicas: 2

$ oc apply -f ingresscontrollers.yaml

$ oc get pods -n openshift-ingress
NAME                              READY   STATUS        RESTARTS   AGE
router-default-56cd95c74c-nn4hc   0/1     Pending       0          38s
router-default-649b9b8b5b-542g6   0/1     Terminating   0          2m29s
router-default-649b9b8b5b-zgq26   1/1     Running       0          5m

[student@workstation certificates-enterprise-ca]$ oc get pods -n openshift-ingress
NAME                              READY   STATUS    RESTARTS   AGE
router-default-56cd95c74c-h8cgt   1/1     Running   0          92s
router-default-56cd95c74c-nn4hc   1/1     Running   0          2m36s

$ oc whoami --show-console
https://console-openshift-console.apps.ocp4.example.com

```
**Open the web console URL. Firefox displays a secure lock icon in the address bar**

- https://console-openshift-console.apps.ocp4.example.com

>If the previous router pods have not fully terminated, then accessing the OpenShift web console URL in the next step might still produce a certificate warning

## Configure the API server to use the custom-tls secret

```
$ oc apply -f apiserver-cluster.yaml

$ oc get clusteroperator/kube-apiserver
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
kube-apiserver   4.6.36    True        True          False      174d

$ oc get clusteroperator/kube-apiserver ; oc get pods -l app=openshift-kube-apiserver -n openshift-kube-apiserver
NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
kube-apiserver   4.6.36    True        True          False      174d
NAME                      READY   STATUS    RESTARTS   AGE
kube-apiserver-master01   5/5     Running   0          6m1s
kube-apiserver-master02   5/5     Running   0          5h55m
kube-apiserver-master03   5/5     Running   0          2m53s

$ watch "oc get clusteroperator/kube-apiserver ; oc get pods -l app=openshift-kube-apiserver -n openshift-kube-apiserver"
Every 2.0s: oc get clu...  workstation.lab.example.com: Tue May 10 09:49:00 2022

NAME             VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
kube-apiserver   4.6.36    True        False         False	174d
NAME                      READY   STATUS    RESTARTS   AGE
kube-apiserver-master01   5/5     Running   0          7m30s
kube-apiserver-master02   5/5     Running   0          70s
kube-apiserver-master03   5/5     Running   0          4m22s

$ oc logout

$ oc login -u admin -p redhat https://api.ocp4.example.com:6443

```

## edge route

take advantage of the new wildcard certificate by creating an edge route. While an edge route only secures traffic between the router and the user, it does not require making any modifications to the application container image.

```
$ oc create route edge <ROUTE-NAME> --service <SERVICE>

```
# Configuring Applications to Trust the Enterprise Certificate Authority

**Purpose**
-    Identify the configuration map used by the cluster proxy.
-    Verify the certificates in the cluster proxy configuration map.
-    Create a new configuration map that is injected with the trusted certificate bundle.
-    Mount your configuration map inside a pod so that it trusts certificates signed by your enterprise CA.

## configuration map used by the cluster proxy
```
$ oc get proxy/cluster -o yaml |tail -4
spec:
  trustedCA:
    name: combined-certs
status: {}

$  oc get configmap combined-certs -n openshift-config -o jsonpath='{.data.*}' | grep Classroom
# Classroom Wildcard & Master API Certificate  on top
# GLS Training Classroom Certificate Authority on bottom

```

## Create the certificates-app-trust project with two applications named hello1 and hello2

```
$ oc new-project certificates-app-trust

$ oc new-app --name hello1  --docker-image quay.io/redhattraining/hello-world-nginx:v1.0

$ oc create route edge --service hello1 --hostname hello1-trust.apps.ocp4.example.com

$ oc new-app --name hello2 --docker-image quay.io/redhattraining/hello-world-nginx:v1.0

$ oc create route edge --service hello2 --hostname hello2-trust.apps.ocp4.example.com

$ oc get routes
NAME     HOST/PORT                            PATH   SERVICES   PORT       TERMINATION   WILDCARD
hello1   hello1-trust.apps.ocp4.example.com          hello1     8080-tcp   edge          None
hello2   hello2-trust.apps.ocp4.example.com          hello2     8080-tcp   edge          None

$ curl https://hello1-trust.apps.ocp4.example.com
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>

$ curl https://hello2-trust.apps.ocp4.example.com
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>



```

Connect to the hello1 application pod and attempt to access the route for the hello2 application. This attempt fails because the hello1 pods do not trust certificates signed by the enterprise CA.

```
$ oc get deployments
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
hello1   1/1     1            1           7m2s
hello2   1/1     1            1           5m27s

$ oc exec -it deployment/hello1 -- /bin/bash
bash-4.4$ curl https://hello2-trust.apps.ocp4.example.com
curl: (60) SSL certificate problem: self signed certificate in certificate chain
More details here: https://curl.haxx.se/docs/sslcerts.html

bash-4.4$ exit:w

```

## Inject the trusted certificate bundle into a new ca-certs configuration map

### Create the ca-certs configuration map
```
$ oc create configmap ca-certs

```

### Label the configuration map with config.openshift.io/inject-trusted-cabundle=true
```
$ oc label configmap ca-certs config.openshift.io/inject-trusted-cabundle=true

```

### Display the contents of the configuration map

Although it was originally empty, the cluster network operator injected certificates into it. There are many certificates in the configuration map; the certificates from combined-certs were injected at the top.

```
$ oc get configmap ca-certs -o yaml | head -n 6
apiVersion: v1
data:
  ca-bundle.crt: |
    # Classroom Wildcard & Master API Certificate
    -----BEGIN CERTIFICATE-----
    MIIGEjCCA/qgAwIBAgIUHbAagJRT/So5GG5aFyEqrtVKloswDQYJKoZIhvcNAQEL

```

## Mount the ca-certs configuration map in the hello1 application pod

### Use the oc set volume command to mount the ca-certs configuration map into the hello1 deployment

```

$ oc set volume deployment/hello1 -t configmap  --name trusted-ca --add --read-only=true --mount-path /etc/pki/tls/certs --configmap-name ca-certs 

$ oc get deployment/hello1 -o yaml
...

    spec:
      containers:

        volumeMounts:
        - mountPath: /etc/pki/tls/certs
          name: trusted-ca
          readOnly: true

      volumes:
      - configMap:
          defaultMode: 420
          name: ca-certs
        name: trusted-ca

$ oc get pods -l deployment=hello1
NAME                      READY   STATUS    RESTARTS   AGE
hello1-7646448745-gjdjz   1/1     Running   0          5m45s

$ oc exec -it deployment/hello1 -- /bin/bash
bash-4.4$ grep Classroom /etc/pki/tls/certs/ca-bundle.crt
# Classroom Wildcard & Master API Certificate
# GLS Training Classroom Certificate Authority


bash-4.4$ curl https://hello2-trust.apps.ocp4.example.com
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>

```
