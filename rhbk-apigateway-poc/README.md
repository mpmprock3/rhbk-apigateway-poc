# End-to-End POC: Securing AWS API Gateway with Red Hat Build of Keycloak (RHBK) on OpenShift

This repository contains the documentation and steps for a hybrid-cloud Proof of Concept (POC). It demonstrates how to secure an AWS API Gateway (HTTP API) using JSON Web Tokens (JWT) issued by a Red Hat Build of Keycloak (RHBK) instance hosted on Red Hat OpenShift.

## Architecture Overview

This POC uses the **OAuth 2.0 Client Credentials Grant** (Machine-to-Machine) flow.

1. **Identity Provider:** Red Hat Build of Keycloak (RHBK) deployed via Operator on OpenShift.
2. **API Gateway:** AWS API Gateway (HTTP API) configured with a JWT Authorizer.
3. **Backend API:** `httpbin.org/get` acting as a mock backend to verify successful routing.

## Prerequisites

* An OpenShift cluster with cluster-admin privileges.
* An AWS Account with permissions to create API Gateways.
* The `oc` CLI and `curl` installed locally.

---

## Step 1: Deploy PostgreSQL Database on OpenShift

RHBK requires a persistent database backend. Deploy a PostgreSQL instance in the same namespace before creating the Keycloak CR.

1. **Create the `rhbk` namespace** (if it doesn't already exist):

```bash
oc new-project rhbk
```

2. **Deploy PostgreSQL** using the following StatefulSet and Service:

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

3. **Create the database credentials secret** that the Keycloak CR will reference:

```bash
oc create secret generic keycloak-db-secret \
  --from-literal=username=keycloak \
  --from-literal=password=keycloak \
  -n rhbk
```

4. **Verify the database pod is running:**

```bash
oc get pods -n rhbk -l app=postgresql-db
```

---

## Step 2: Deploy RHBK on OpenShift

1. Log into the OpenShift Web Console.
2. Navigate to **OperatorHub**, search for **Red Hat Build of Keycloak**, and install the Operator.
3. Once installed, create a `Keycloak` instance using the following custom resource (CR).
   *(Note: The `additionalOptions` are critical when using OpenShift Edge TLS termination so Keycloak generates HTTPS URLs for AWS. The `db` section connects RHBK to the PostgreSQL instance deployed in Step 1.)*

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
    hostname: <YOUR_OPENSHIFT_ROUTE_URL>  # e.g., keycloak-rhbk.apps.mycluster.com
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

4. **Enable HTTPS (Edge Termination):** In OpenShift, go to **Networking > Routes**, edit the Keycloak route, check **"Secure Route"**, and set TLS Termination to **Edge**.

5. **Get Admin Credentials:** Run `oc extract secret/rhbk-poc-initial-admin --to=-` to retrieve your Keycloak admin password.

---

## Step 3: Configure Keycloak

1. Log into the Keycloak Admin Console using the HTTPS URL and admin credentials.

2. **Create a Realm:** Name it `aws-poc-realm`.

3. **Create a Client:**
   - **Client ID:** `aws-api-client`
   - **Client authentication:** ON
   - **Service accounts roles:** ON (This enables the Client Credentials grant).
   - **Standard flow / Direct access:** OFF

4. **Create an Audience Mapper** (Crucial for AWS):
   - Go to the client's **Client scopes** tab.
   - Click the `aws-api-client-dedicated` scope.
   - Click **Add mapper > Configure a new mapper > Audience**.
   - **Name:** `aws-audience`
   - **Included Client Audience:** Select `aws-api-client` from the dropdown.
   - **Add to access token:** ON -> Save.

5. **Retrieve the Secret:** Go to the client's **Credentials** tab and copy the Client Secret.

---

## Step 4: Configure AWS API Gateway

1. Log into the AWS Console and navigate to **API Gateway**.

2. Click **Create API -> HTTP API -> Build**.

3. **Integrations:** Add an HTTP integration. Set Method to `GET` and URL to `https://httpbin.org/get`. Name the API `RHBK-POC-API`.

4. **Routes:** Configure the route Method as `GET` and Resource path as `/test`. Attach your integration. Click through to **Create**.

5. **Attach JWT Authorizer:**
   - Go to **Authorization** in the left menu.
   - Select the `GET /test` route and click **Create and attach an authorizer**.
   - **Type:** JWT
   - **Identity source:** `$request.header.Authorization`
   - **Issuer URL:** `https://<YOUR_OPENSHIFT_ROUTE_URL>/realms/aws-poc-realm`
     > **Warning:** Do NOT include a trailing slash!
   - **Audience:** `aws-api-client`
   - Click **Create and attach**.

6. Go to **Stages** or the API overview and copy your **Invoke URL**.

---

## Step 5: End-to-End Validation

### Test 1: Verify API Gateway Blocks Unauthorized Traffic

Run the following command against your AWS Invoke URL:

```bash
curl -i https://<YOUR_AWS_INVOKE_URL>/test
```

**Expected Result:** `HTTP/1.1 401 Unauthorized`

### Test 2: Generate Keycloak Access Token

Use your Client ID and Secret to request a token via the `client_credentials` grant:

```bash
curl -s -X POST 'https://<YOUR_OPENSHIFT_ROUTE_URL>/realms/aws-poc-realm/protocol/openid-connect/token' \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=aws-api-client" \
  -d "client_secret=<YOUR_CLIENT_SECRET>" \
  -d "grant_type=client_credentials"
```

Copy the string inside the `"access_token"` field.

### Test 3: Access the API Successfully

Pass the JWT as a Bearer token in the Authorization header:

```bash
curl -i -H "Authorization: Bearer <PASTE_YOUR_ACCESS_TOKEN_HERE>" https://<YOUR_AWS_INVOKE_URL>/test
```

**Expected Result:** `HTTP/1.1 200 OK` alongside a JSON payload from the `httpbin.org` mock backend, proving AWS validated the Keycloak signature and routed the traffic successfully.

---

## Implementation Notes for Production

- **Token Automation:** In a production scenario, backend services will not use manual `curl` commands. They will utilize standard HTTP interceptor libraries (e.g., Spring Security, Python `requests-oauthlib`) to automatically cache, inject, and refresh tokens before making calls to AWS.

- **Databases:** This POC uses an `emptyDir` volume for PostgreSQL. For production, replace it with a PersistentVolumeClaim (PVC) or use a managed database service (e.g., AWS RDS) for data durability.
