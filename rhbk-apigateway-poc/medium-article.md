# Securing AWS API Gateway with Red Hat Build of Keycloak (RHBK) on OpenShift — A Hybrid-Cloud POC

## The problem nobody talks about

You have APIs on AWS. Your identity platform runs on OpenShift. Now you need them to trust each other — across clouds, without cutting corners on security.

Most tutorials either assume everything lives in AWS (Cognito!) or hand-wave the cross-cloud piece. This article doesn't. We'll walk through a complete, working Proof of Concept that wires up **Red Hat Build of Keycloak (RHBK)** on OpenShift as the OAuth 2.0 identity provider for **AWS API Gateway's JWT Authorizer** — and prove it end-to-end with `curl`.

By the end, you'll have an architecture where:

- Keycloak issues signed JWTs from OpenShift
- AWS API Gateway validates those tokens in real time using Keycloak's JWKS endpoint
- Unauthorized requests get rejected at the gateway — before they ever touch your backend

---

## What We're Building

This POC uses the **OAuth 2.0 Client Credentials Grant** — the standard machine-to-machine flow. No user login screens, no browser redirects. Just services authenticating to services.

**The cast:**

| Component | Role |
|-----------|------|
| **Red Hat Build of Keycloak (RHBK)** | Identity Provider — issues and signs JWTs. Deployed via Operator on OpenShift. |
| **AWS API Gateway (HTTP API)** | API Gateway — validates JWTs using a built-in JWT Authorizer. |
| **httpbin.org/get** | Mock backend — proves that traffic made it through. |
| **PostgreSQL** | Keycloak's persistent datastore, running on OpenShift. |

### The Request Flow

Here's how a request moves through the system:

```
Client                  OpenShift (RHBK)              AWS
  |                          |                         |
  |---1. POST /token-------->|                         |
  |   (client_credentials)   |                         |
  |                          |                         |
  |<--2. Signed JWT----------|                         |
  |                          |                         |
  |---3. GET /test (Bearer JWT)--------------------->  |
  |                          |                         |
  |                    4. Fetch JWKS <--- JWT Authorizer|
  |                          |                         |
  |                    5. Validate token (sig, exp,     |
  |                       issuer, audience)             |
  |                          |                         |
  |<--7. 200 OK + JSON------- 6. Route to httpbin----->|
```

**Step by step:**

1. The client requests an access token from RHBK using the `client_credentials` grant
2. RHBK validates the credentials and returns a signed JWT with an `aud` (audience) claim
3. The client calls AWS API Gateway with the JWT in the `Authorization: Bearer` header
4. AWS JWT Authorizer fetches the public keys (JWKS) from Keycloak's OIDC discovery endpoint
5. The authorizer validates the token's signature, expiry, issuer, and audience
6. On success, API Gateway routes the request to the backend
7. Backend responds with `200 OK` — the response flows back to the client

---

## Prerequisites

Before you start, make sure you have:

- An **OpenShift cluster** with cluster-admin privileges
- An **AWS account** with permissions to create API Gateways
- The `oc` CLI and `curl` installed locally

---

## Step 1: Deploy PostgreSQL on OpenShift

RHBK needs a persistent database. We'll deploy PostgreSQL in the same namespace.

**Create the namespace:**

```bash
oc new-project rhbk
```

**Deploy PostgreSQL** using a StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
  namespace: rhbk
spec:
  serviceName: postgresql-db-service
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
        - name: postgresql-db
          image: postgres:latest
          volumeMounts:
            - mountPath: /data
              name: cache-volume
          env:
            - name: POSTGRES_PASSWORD
              value: keycloak
            - name: POSTGRES_USER
              value: keycloak
            - name: PGDATA
              value: /data/pgdata
            - name: POSTGRES_DB
              value: keycloak
      volumes:
        - name: cache-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: rhbk
spec:
  selector:
    app: postgresql-db
  type: ClusterIP
  ports:
    - port: 5432
      targetPort: 5432
```

**Create the database credentials secret** that the Keycloak CR will reference:

```bash
oc create secret generic keycloak-db-secret \
  --from-literal=username=keycloak \
  --from-literal=password=keycloak \
  -n rhbk
```

**Verify the pod is running:**

```bash
oc get pods -n rhbk -l app=postgresql-db
```

> **Production note:** This POC uses `emptyDir` for storage — acceptable for a demo, but in production you'll want a `PersistentVolumeClaim` or a managed database service like AWS RDS.

---

## Step 2: Deploy RHBK on OpenShift

**Install the Operator:**

1. Log into the OpenShift Web Console
2. Navigate to **OperatorHub**, search for **Red Hat Build of Keycloak**
3. Install the Operator with default settings

**Create the Keycloak instance** using this custom resource:

```yaml
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: rhbk-poc
  namespace: rhbk
spec:
  instances: 1
  db:
    vendor: postgres
    host: postgres-db
    usernameSecret:
      name: keycloak-db-secret
      key: username
    passwordSecret:
      name: keycloak-db-secret
      key: password
  hostname:
    hostname: <YOUR_OPENSHIFT_ROUTE_URL>
    strict: true
  http:
    httpEnabled: true
  ingress:
    className: openshift-default
    enabled: true
  additionalOptions:
    - name: proxy
      value: edge
    - name: proxy-headers
      value: xforwarded
    - name: hostname-url
      value: https://<YOUR_OPENSHIFT_ROUTE_URL>
```

Replace `<YOUR_OPENSHIFT_ROUTE_URL>` with your actual route hostname (e.g., `keycloak-rhbk.apps.mycluster.com`).

### Why the `additionalOptions` matter

This is the part that will save you hours of debugging. When OpenShift terminates TLS at the route (Edge termination), Keycloak thinks it's running on plain HTTP. Without the proxy settings, Keycloak generates `http://` URLs in its OIDC discovery document — and AWS API Gateway will refuse to fetch JWKS from an insecure endpoint.

The `proxy: edge` and `proxy-headers: xforwarded` options tell Keycloak: "Trust the `X-Forwarded-*` headers from the OpenShift router, and generate HTTPS URLs accordingly."

**Enable HTTPS on the route:**

In OpenShift, go to **Networking > Routes**, edit the Keycloak route, check **"Secure Route"**, and set TLS Termination to **Edge**.

**Retrieve admin credentials:**

```bash
oc extract secret/rhbk-poc-initial-admin --to=-
```

---

## Step 3: Configure Keycloak

Log into the Keycloak Admin Console at `https://<YOUR_OPENSHIFT_ROUTE_URL>`.

### Create a Realm

Create a new realm named `aws-poc-realm`. This isolates our POC configuration from the default `master` realm.

### Create a Client

- **Client ID:** `aws-api-client`
- **Client authentication:** ON
- **Service accounts roles:** ON — this is what enables the Client Credentials grant
- **Standard flow / Direct access:** OFF — we don't need browser-based flows for M2M

### Create the Audience Mapper (This Is Crucial)

This is the step most tutorials miss, and it's the one that will make or break your AWS integration.

By default, Keycloak does **not** include an `aud` (audience) claim in the access token that matches your client ID. AWS API Gateway's JWT Authorizer **requires** the audience claim to match. Without this mapper, you'll get `401 Unauthorized` responses even though the token is otherwise valid.

Here's how to set it up:

1. Go to the client's **Client scopes** tab
2. Click the `aws-api-client-dedicated` scope
3. Click **Add mapper > Configure a new mapper > Audience**
4. Configure it:
   - **Name:** `aws-audience`
   - **Included Client Audience:** Select `aws-api-client` from the dropdown
   - **Add to access token:** ON
5. Save

### Retrieve the Client Secret

Go to the client's **Credentials** tab and copy the Client Secret. You'll need it for testing.

---

## Step 4: Configure AWS API Gateway

Now we switch to the AWS side.

### Create the HTTP API

1. In the AWS Console, navigate to **API Gateway**
2. Click **Create API > HTTP API > Build**
3. **Add an integration:**
   - Type: HTTP
   - Method: `GET`
   - URL: `https://httpbin.org/get`
4. Name the API: `RHBK-POC-API`

### Configure the Route

- Method: `GET`
- Resource path: `/test`
- Attach the httpbin integration
- Click through to **Create**

### Attach the JWT Authorizer

This is where the magic happens. AWS will validate every incoming request against your Keycloak instance.

1. Go to **Authorization** in the left menu
2. Select the `GET /test` route
3. Click **Create and attach an authorizer**
4. Configure:
   - **Type:** JWT
   - **Identity source:** `$request.header.Authorization`
   - **Issuer URL:** `https://<YOUR_OPENSHIFT_ROUTE_URL>/realms/aws-poc-realm`
   - **Audience:** `aws-api-client`
5. Click **Create and attach**

> **Warning:** Do **NOT** include a trailing slash in the Issuer URL. AWS API Gateway will fail to resolve the OIDC discovery endpoint if you do. This is a common pitfall.

Copy your **Invoke URL** from the Stages or API overview page.

---

## Step 5: End-to-End Validation

This is the fun part. Three tests, three minutes.

### Test 1: Verify the Gateway Blocks Unauthorized Traffic

Hit the API with no token:

```bash
curl -i https://<YOUR_AWS_INVOKE_URL>/test
```

**Expected:** `HTTP/1.1 401 Unauthorized`

If you get this, the JWT Authorizer is active and blocking unauthenticated requests. Exactly what we want.

### Test 2: Get an Access Token from Keycloak

```bash
curl -s -X POST \
  'https://<YOUR_OPENSHIFT_ROUTE_URL>/realms/aws-poc-realm/protocol/openid-connect/token' \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=aws-api-client" \
  -d "client_secret=<YOUR_CLIENT_SECRET>" \
  -d "grant_type=client_credentials"
```

You'll get back a JSON response. Copy the value of the `access_token` field.

**Pro tip:** Pipe through `jq` to extract it cleanly:

```bash
curl -s -X POST \
  'https://<YOUR_OPENSHIFT_ROUTE_URL>/realms/aws-poc-realm/protocol/openid-connect/token' \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=aws-api-client" \
  -d "client_secret=<YOUR_CLIENT_SECRET>" \
  -d "grant_type=client_credentials" | jq -r '.access_token'
```

### Test 3: Access the API with a Valid Token

```bash
curl -i -H "Authorization: Bearer <YOUR_ACCESS_TOKEN>" \
  https://<YOUR_AWS_INVOKE_URL>/test
```

**Expected:** `HTTP/1.1 200 OK` with a JSON payload from httpbin showing your request headers.

That's it. AWS API Gateway fetched the JWKS from your OpenShift-hosted Keycloak, validated the token's signature, checked the expiry, verified the issuer and audience — and let the request through. Hybrid-cloud API security, working end-to-end.

---

## Troubleshooting

Here are the issues I hit during this POC, so you don't have to:

| Symptom | Cause | Fix |
|---------|-------|-----|
| `401 Unauthorized` even with a valid token | Missing audience (`aud`) claim in the JWT | Add the Audience Mapper in Step 3 |
| AWS can't reach Keycloak's JWKS endpoint | Keycloak generating `http://` URLs instead of `https://` | Add the `proxy: edge` and `proxy-headers` options in the Keycloak CR |
| `Issuer does not match` error | Trailing slash in the Issuer URL on AWS side | Remove the trailing slash from the Issuer URL |
| Token request fails with connection error | OpenShift route not configured for HTTPS | Enable Edge TLS termination on the Keycloak route |

---

## What This Means for Production

This POC validates the architecture, but production deployments will look different in a few ways:

**Token lifecycle automation:** No one is running `curl` in production. Backend services use HTTP interceptor libraries — Spring Security's `OAuth2RestTemplate`, Python's `requests-oauthlib`, or similar — to automatically obtain, cache, and refresh tokens before making API calls.

**Database durability:** Swap the `emptyDir` volume for a `PersistentVolumeClaim` or a managed database service like AWS RDS. Your realm configuration and client data need to survive pod restarts.

**High availability:** Scale the RHBK instances beyond `1` and configure the database for replication. Keycloak's Infinispan-based clustering works well on OpenShift.

**Certificate-based authentication:** For environments where shared secrets are unacceptable, you can replace `client_secret` with **mTLS client authentication** (`tls_client_auth`) using X.509 certificates. This eliminates the secret entirely — the private key never leaves the client. I've documented this variant separately as it involves additional certificate management and OpenShift route configuration changes.

---

## Wrapping Up

The hybrid-cloud pattern works. An identity provider running on OpenShift can secure APIs hosted on AWS without any proprietary glue — just standard OAuth 2.0 and OIDC protocols.

The key takeaways:

1. **The Audience Mapper is non-optional.** AWS API Gateway requires the `aud` claim. Keycloak doesn't include it by default. Miss this and you'll spend hours chasing phantom 401s.

2. **Proxy settings matter.** When OpenShift terminates TLS at the edge, you need to tell Keycloak explicitly. Otherwise, the OIDC discovery document advertises HTTP endpoints that AWS refuses to use.

3. **No trailing slash on the Issuer URL.** Small detail. Will cost you an hour if you miss it.

The full source — including YAML manifests, Keycloak CR, and the mTLS variant — is available on [GitHub](https://github.com/your-repo-here).

---

*If you found this useful, I'd appreciate a clap or a follow. I write about OpenShift, hybrid-cloud architecture, and the gaps between "it works in the tutorial" and "it works in production."*
