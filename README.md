# Envoy Kubernetes GCP JWT Lab

To follow along with this lab you need a GCP project. You can sign up for a
[free tier here](https://cloud.google.com/free). Though, the focus is on people
that are GCP users and have been deploying their apps to GCP services such
as GKE. You'll also need a k8s cluster - Minikube(https://kubernetes.io/docs/tasks/tools/install-minikube/)
is fine for the tasks we are doing here.

## Goal

In this lab we are going to use Envoy as a reverse-proxy to secure a backend app
using JWT. Our tokens are going to be generated by Google service accounts and
validated against Google remote jwks.

## How it works

We configure an [Envoy HTTP Filter](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/jwt_authn/v3/config.proto)
to perform [JWT Authentication](https://jwt.io/introduction/) using [Google service accounts](https://cloud.google.com/endpoints/docs/openapi/service-account-authentication).

```yaml
- name: envoy.filters.http.jwt_authn
  typed_config:
    "@type": type.googleapis.com/envoy.config.filter.http.jwt_authn.v2alpha.JwtAuthentication
    providers:
      provider_jwt-example:
        issuer: jwt-example@your_gcp_project.iam.gserviceaccount.com
        audiences:
        - jwt-example.backend-app
        remote_jwks:
          http_uri:
            uri: https://www.googleapis.com/service_accounts/v1/jwk/jwt-example@your_gcp_project.iam.gserviceaccount.com
            cluster: provider_cluster
            timeout:
              seconds: 5
          cache_duration:
            seconds: 300
    rules:
    - match: { prefix: / }
    requires: { provider_name: provider_jwt-example }
```

In practice, it will allow securing service-to-service communication while leveraging
the Google infrastructure support. It can be an option to replace [ESPs](https://cloud.google.com/endpoints/docs/openapi/specify-proxy-startup-options).

## Exploring

Let's start deploying a backend app. Nothing special here, we just need it to
configure Envoy in front of something:

```bash
$ kubectl create ns backend
$ kubectl apply -f backend-app.yaml
```

Check it's working

```bash
$ kubectl get po -n backend

NAME                             READY   STATUS    RESTARTS   AGE
backend-app-5d8d977f96-78k2b     1/1     Running   0          10m
backend-app-5d8d977f96-q8jmw     1/1     Running   0          10m
backend-app-5d8d977f96-s6lrk     1/1     Running   0          10m
```

To configure the Envoy JWT provider we need the Google service account details
(issuer, remote jwks URL, etc). For this, we are going to generate a service
account:

```bash
$ gcloud iam service-accounts create jwt-example --display-name "Envoy JWT Example"
```

It will create a service account with the following address: `jwt-example@your_gcp_project.iam.gserviceaccount.com`

Let's generate a service account key that is going to be used later to generate/sign
the JWT:

```bash
$ gcloud iam service-accounts keys create ./key.json --iam-account jwt-example@your_gcp_project.iam.gserviceaccount.com
```

It saves the SA json key in a file called `key.json` in the current dir.

Let's now deploy Envoy. _Heads up_, this example won't work if you don't generate
the SA and update the [Envoy config](./envoy.yaml) replacing `your_gcp_project` 
by your actual project name.

```bash
$ kubectl apply -f envoy.yaml
```

Check the pod has started:

```bash
$ kubectl get po -n backend

NAME                             READY   STATUS    RESTARTS   AGE
backend-app-5d8d977f96-78k2b     1/1     Running   0          10m
backend-app-5d8d977f96-q8jmw     1/1     Running   0          10m
backend-app-5d8d977f96-s6lrk     1/1     Running   0          10m
backend-envoy-559c97f657-5wqsc   1/1     Running   0          7m
```

Now let's test this configuration. Just for the sake of simplicity we are not
using an external IP for this lab, so we are going to port-forward the pod
in order to be able to access it locally.

> In time, notice that this example won't make complete sense since any
application deployed inside the cluster can bypass Envoy and access your backend
service directly. If you want to protect intra-cluster communication one option is
to deploy Envoy as a sidecar in your backend app's pod.

```bash
$ kubectl --n backend port-forward backend-envoy-559c97f657-5wqsc 8081:80
```

Let's see if the JWT is blocking access:

```bash
$ curl -I --request GET --url 'http://localhost:8081'

HTTP/1.1 401 Unauthorized
content-length: 14
content-type: text/plain
date: Wed, 06 May 2020 13:48:41 GMT
server: envoy

$ curl -I --request GET --url 'http://localhost:8081' --header 'authorization: Bearer foo'

HTTP/1.1 401 Unauthorized
content-length: 50
content-type: text/plain
date: Wed, 06 May 2020 13:48:17 GMT
server: envoy
```

Ok, let's now generate/sign a valid JWT and check. I'll use the gcloud CLI
but you can use your preferred language. Here are some examples: [Go](https://github.com/GoogleCloudPlatform/golang-samples/blob/master/endpoints/getting-started/client/main.go), [Python](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/master/endpoints/getting-started/clients/google-jwt-client.py) and [Java](https://github.com/GoogleCloudPlatform/java-docs-samples/blob/master/endpoints/getting-started/clients/src/main/java/com/example/app/GoogleJwtClient.java).

In order to sign a JWT using a service account you need to give your user the
`iam.serviceAccounts.signJwt` permission. We'll use the existing
`roles/iam.serviceAccountTokenCreator` role that contains this permission.

```bash
gcloud projects add-iam-policy-binding your_gcp_project \
  --member user:your_user@your_domain.com \
  --role roles/iam.serviceAccountTokenCreator
```

Now generate and sign the JWT using the `key.json` file saved before. The JWT
will be stored in the file `output.jwt`

```bash
$ gcloud iam service-accounts sign-jwt --iam-account jwt-example@your_gcp_project.iam.gserviceaccount.com claims.json output.jwt
```

Cool, now let's call the backend service through Envoy and check the JWT is
working.

```bash
$ curl --request GET --url 'http://localhost:8081' --header "authorization: Bearer $(cat output.jwt)"

Hostname: backend-app-5d8d977f96-78k2b

[...]
```

* https://jwt.io/introduction
* https://cloud.google.com/endpoints/docs/openapi/service-account-authentication
* https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/jwt_authn/v3/config.proto
