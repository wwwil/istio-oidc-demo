# Istio OIDC Demo

:construction:
:warning:
**This demo is just a proof of concept and is currently under construction.**
:warning:
:construction:

Much of this demo is based on [this Jetstack blog post](https://www.blog.jetstack.io/blog/istio-oidc/) which I would highly recommend.
The purpose of this demo is to explore using OIDC with Istio in a way that avoids use of `EnvoyFilters`. 
It also allows both shared and per app OIDC auth. This means a cluster operator can provide an OIDC solution that tenants can use with minimal configuration, but they are also free to use their own OIDC provider and handle configuration themselves.

## TODO

- Dump Istio manifests and inject the oauth2-proxy, rather than using a static manifest.
- Consider using a separate LoadBalancer Service to point to the oauth2-proxy, and changing the Istio generated one to be type ClusterIP.
- Test with multiple replicas of the ingress gateway with oauth2-proxy sidecars.
- Finish trying GitHub as an alternative OAuth2 provider.

## Prerequisites

- `gcloud` set up and authenticated.
- `istioctl` version 1.8.
- A domain to create DNS records on.

## 1. Setup

Create a GKE cluster:

```bash
# Set your Google Cloud project ID
PROJECT_ID=""
ZONE="europe-west2-a"
gcloud container clusters create istio-oidc-demo --project=${PROJECT_ID} --zone=${ZONE}
```

Deploy Istio and enforce strict mutual TLS:

```bash
istioctl manifest install -y
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF
```

## 2. Deploy Demo App

Deploy `httpbin` as a demo app:

```bash
kubectl apply -f httpbin.yaml
```

Once this is running verify that it's accessible via the Istio ingress gateway.

## 3. Google OAuth2

Set up [Google OAuth2](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/oauth_provider#google-auth-provider). 

## 4. OIDC Ingress Gateway

Deploy another Istio ingress gateway for OIDC, and configure it to route traffic to the demo app:

```bash
# Set your client ID and secret from your Google OIDC provider setup
CLIENT_ID=""
CLIENT_SECRET=""
COOKIE_SECRET=$(openssl rand -hex 16)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: oauth2-proxy
  namespace: istio-system
stringData:
  OAUTH2_PROXY_CLIENT_ID: $CLIENT_ID
  OAUTH2_PROXY_CLIENT_SECRET: $CLIENT_SECRET
  OAUTH2_PROXY_COOKIE_SECRET: $COOKIE_SECRET
EOF
kubectl apply -f istio-ingress-oidc.yaml
kubectl apply -f httpbin-oidc.yaml
```

Google's OAuth2 process does not allow access directly by IP address, so create a DNS record to point at the external IP address of the OIDC ingress gateway.

Accessing the ingress gateway should now redirect to Google's OAuth2 consent screen, and on successful sign in should show the demo app.

## 5. GitHub OAuth2

TODO

## 6. App Specific OIDC

Deploy `oauth2-proxy` using the client ID and secret from GitHub OAuth2:

```bash
# Set your client ID and secret from your GitHub OIDC provider setup
CLIENT_ID=""
CLIENT_SECRET=""
COOKIE_SECRET=$(openssl rand -hex 16)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: oauth2-proxy
  namespace: httpbin
stringData:
  OAUTH2_PROXY_CLIENT_ID: $CLIENT_ID
  OAUTH2_PROXY_CLIENT_SECRET: $CLIENT_SECRET
  OAUTH2_PROXY_COOKIE_SECRET: $COOKIE_SECRET
EOF
kubectl apply -f oauth2-proxy.yaml
```

Then secure the demo app with Istio `RequestAuthentication` and `AuthorizationPolicy` resources, and change the `Gateway` and `VirtualService` resources used to configure the unsecured ingress gateway to point traffic at the `oauth2-proxy`:

```bash
kubectl apply -f httpbin-auth.yaml
```

Trying to access the unsecured gateway should now redirect to GitHub's OAuth2 consent screen, and on successful sign in should show the demo app.
 