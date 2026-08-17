# Client Certificate Rotation for CockroachDB Advanced

## Overview

This guide describes how to rotate customer-managed SQL client certificates for a CockroachDB Advanced cluster. It is a companion to `readme.md`, which explains how to create and upload the initial client Certificate Authority (CA) and issue a certificate for a SQL user.

There are two different rotation procedures:

1. **Rotate a client certificate:** Issue a new certificate and private key for a SQL user using the existing client CA. This is the normal renewal procedure and requires no change to the cluster's trust store.
2. **Rotate the client CA:** Create a new CA, temporarily configure the cluster to trust both the old and new CAs, migrate every client, and then remove the old CA.

This guide does not rotate the CockroachDB Cloud server CA used by `sslrootcert`. Cockroach Labs manages that CA separately. The CA discussed here is the customer-managed client CA that CockroachDB uses to authenticate SQL clients.

> **Important:** The private CA keys and client private keys must never be uploaded to CockroachDB Cloud. Upload only public CA certificates.

## Choosing the rotation type

Use client certificate rotation when:

- A client certificate is approaching expiration.
- You are following a routine short-lived certificate renewal schedule.
- The existing client CA and its private key remain valid and trusted.

Use client CA rotation when:

- The client CA is approaching expiration.
- The CA private key may have been compromised.
- Your security policy requires periodic CA replacement.
- You are migrating to a different PKI or secrets-management system.

The safe client CA sequence is:

```text
Trust old CA  ->  Trust old and new CAs  ->  Trust new CA only
```

The overlap allows old and new client certificates to authenticate during the migration.

## Prerequisites

- The files and OpenSSL configuration created by `readme.md`.
- `openssl`, `jq`, `curl`, and the `cockroach` SQL CLI.
- `Cluster Admin` or `Organization Admin` access to update the client CA.
- A CockroachDB Cloud service-account API key.
- The target cluster ID, SQL hostname, and Cloud server CA path.
- A record of every application and user that authenticates with a client certificate.

Configure the Cloud API variables:

```bash
export COCKROACH_SERVER="https://cockroachlabs.cloud"
export CLUSTER_ID="<cluster-id>"
export API_KEY="<service-account-api-key>"
```

Configure connection variables for validation. Replace the example values with values from the target cluster's **Connect** dialog:

```bash
export CLUSTER_HOST="<cluster-hostname>"
export SERVER_CA="<path-to-cockroach-cloud-server-ca.crt>"
export SQL_USERNAME="ron"
```

Confirm that `CLUSTER_ID` and `CLUSTER_HOST` identify the same cluster before changing its client CA.

## Check the current client CA state

Query the client CA resource before making a change:

```bash
curl --silent --show-error --fail-with-body \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}" |
jq '{status, x509_pem_cert}'
```

Interpret the status as follows:

| Status | Meaning | Action |
| --- | --- | --- |
| `NOT_SET` | No client CA is configured. | This is initial setup, not rotation. Follow `readme.md` and use `POST`. |
| `IS_SET` | One or more client CAs are configured. | Use `PATCH` to update the trust bundle. |
| `PENDING` | A client CA operation is running. | Wait and query again. Do not submit another update. |

Save the currently configured public CA certificate or bundle before starting a client CA rotation:

```bash
curl --silent --show-error --fail-with-body \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}" |
jq -r '.x509_pem_cert' \
  > ~/crdb-cert-lab/certs/client-ca-current.pem
```

Inspect the saved certificates:

```bash
grep -c -- 'BEGIN CERTIFICATE' \
  ~/crdb-cert-lab/certs/client-ca-current.pem

openssl crl2pkcs7 \
  -nocrl \
  -certfile ~/crdb-cert-lab/certs/client-ca-current.pem |
openssl pkcs7 -print_certs -noout
```

## Procedure A - Rotate a client certificate using the existing CA

This procedure leaves the cluster trust store unchanged. The new certificate uses the same SQL identity and is signed by the existing client CA.

The example creates separate `.next` files so the working certificate is not overwritten before validation.

### Step 1 - Confirm the existing CA is valid

```bash
openssl x509 \
  -in ~/crdb-cert-lab/certs/ca.crt \
  -noout -subject -issuer -dates -fingerprint -sha256

openssl rsa \
  -in ~/crdb-cert-lab/ca-private/ca.key \
  -check -noout
```

Do not proceed if the CA is expired, the private key is unavailable, or the private key may have been compromised. Use Procedure B instead.

### Step 2 - Generate a new client private key

```bash
openssl genrsa \
  -out ~/crdb-cert-lab/certs/client.ron.next.key \
  2048

chmod 400 ~/crdb-cert-lab/certs/client.ron.next.key

openssl rsa \
  -in ~/crdb-cert-lab/certs/client.ron.next.key \
  -check -noout
```

### Step 3 - Create a new CSR

The Common Name must continue to match the SQL username exactly.

```bash
openssl req \
  -new \
  -key ~/crdb-cert-lab/certs/client.ron.next.key \
  -out ~/crdb-cert-lab/client.ron.next.csr \
  -subj "/O=Cockroach/CN=ron"
```

### Step 4 - Sign the new client certificate

Choose a lifetime that complies with your certificate policy and does not extend beyond the issuing CA's validity. This example uses 365 days.

```bash
cd ~/crdb-cert-lab

openssl ca \
  -config ~/crdb-cert-lab/ca.cnf \
  -keyfile ~/crdb-cert-lab/ca-private/ca.key \
  -cert ~/crdb-cert-lab/certs/ca.crt \
  -policy signing_policy \
  -extensions signing_client_req \
  -days 365 \
  -notext \
  -in ~/crdb-cert-lab/client.ron.next.csr \
  -out ~/crdb-cert-lab/certs/client.ron.next.crt \
  -batch
```

### Step 5 - Validate the new certificate and key

```bash
openssl verify \
  -CAfile ~/crdb-cert-lab/certs/ca.crt \
  ~/crdb-cert-lab/certs/client.ron.next.crt

openssl x509 \
  -in ~/crdb-cert-lab/certs/client.ron.next.crt \
  -noout -subject -issuer -dates -ext extendedKeyUsage
```

Confirm that the certificate contains `CN=ron`, is currently valid, includes `TLS Web Client Authentication`, and verifies as `OK`.

Confirm that the certificate and private key contain the same public key. The two SHA-256 values must match:

```bash
openssl x509 \
  -in ~/crdb-cert-lab/certs/client.ron.next.crt \
  -pubkey -noout |
openssl pkey -pubin -outform DER |
openssl sha256

openssl pkey \
  -in ~/crdb-cert-lab/certs/client.ron.next.key \
  -pubout -outform DER |
openssl sha256
```

### Step 6 - Test a new SQL connection

```bash
cockroach sql --url \
"postgresql://${SQL_USERNAME}@${CLUSTER_HOST}:26257/defaultdb?sslmode=verify-full&sslrootcert=${SERVER_CA}&sslcert=${HOME}/crdb-cert-lab/certs/client.ron.next.crt&sslkey=${HOME}/crdb-cert-lab/certs/client.ron.next.key" \
  --execute="SELECT current_user(), now();"
```

The connection must succeed without requesting a password.

### Step 7 - Deploy the new client credential

Update the application's secret or connection configuration to use the new certificate and key, then restart or reload the application as required by its connection driver.

Test freshly established connections from every application instance. Existing pooled connections do not prove that the new certificate works because authentication occurs when a connection is created.

Retain the old client certificate and key until the rollout is verified. Afterward, remove the old private key according to your organization's secrets-destruction policy.

No Cloud API `POST` or `PATCH` is required for Procedure A.

## Procedure B - Rotate the client CA without an authentication outage

This procedure temporarily configures the cluster to trust both the current and new client CAs.

### Step 1 - Preserve and verify the current CA

Complete the **Check the current client CA state** section and retain:

```text
~/crdb-cert-lab/certs/client-ca-current.pem
```

Verify that the current client certificate still connects before starting the rotation. This provides a known-good rollback credential.

### Step 2 - Generate a new CA without overwriting the old CA

```bash
openssl genrsa \
  -out ~/crdb-cert-lab/ca-private/ca-new.key \
  2048

chmod 400 ~/crdb-cert-lab/ca-private/ca-new.key

openssl req \
  -new \
  -x509 \
  -config ~/crdb-cert-lab/ca.cnf \
  -key ~/crdb-cert-lab/ca-private/ca-new.key \
  -sha256 \
  -days 3660 \
  -out ~/crdb-cert-lab/certs/ca-new.crt
```

Validate the new CA:

```bash
openssl rsa \
  -in ~/crdb-cert-lab/ca-private/ca-new.key \
  -check -noout

openssl verify \
  -CAfile ~/crdb-cert-lab/certs/ca-new.crt \
  ~/crdb-cert-lab/certs/ca-new.crt

openssl x509 \
  -in ~/crdb-cert-lab/certs/ca-new.crt \
  -noout -subject -issuer -dates -fingerprint -sha256
```

Record the old and new CA fingerprints in the rotation change record.

### Step 3 - Issue a new test client credential from the new CA

Create a new key and CSR:

```bash
openssl genrsa \
  -out ~/crdb-cert-lab/certs/client.ron.ca-new.key \
  2048

chmod 400 ~/crdb-cert-lab/certs/client.ron.ca-new.key

openssl req \
  -new \
  -key ~/crdb-cert-lab/certs/client.ron.ca-new.key \
  -out ~/crdb-cert-lab/client.ron.ca-new.csr \
  -subj "/O=Cockroach/CN=ron"
```

Sign it with the new CA:

```bash
cd ~/crdb-cert-lab

openssl ca \
  -config ~/crdb-cert-lab/ca.cnf \
  -keyfile ~/crdb-cert-lab/ca-private/ca-new.key \
  -cert ~/crdb-cert-lab/certs/ca-new.crt \
  -policy signing_policy \
  -extensions signing_client_req \
  -days 365 \
  -notext \
  -in ~/crdb-cert-lab/client.ron.ca-new.csr \
  -out ~/crdb-cert-lab/certs/client.ron.ca-new.crt \
  -batch
```

Verify the new credential before changing the cluster trust bundle:

```bash
openssl verify \
  -CAfile ~/crdb-cert-lab/certs/ca-new.crt \
  ~/crdb-cert-lab/certs/client.ron.ca-new.crt

openssl x509 \
  -in ~/crdb-cert-lab/certs/client.ron.ca-new.crt \
  -noout -subject -issuer -dates -ext extendedKeyUsage
```

### Step 4 - Build an old-plus-new transition bundle

CockroachDB Cloud accepts multiple client CA certificates concatenated in PEM format with a blank line between certificates.

```bash
cp ~/crdb-cert-lab/certs/client-ca-current.pem \
  ~/crdb-cert-lab/certs/client-ca-transition.pem

printf '\n' \
  >> ~/crdb-cert-lab/certs/client-ca-transition.pem

cat ~/crdb-cert-lab/certs/ca-new.crt \
  >> ~/crdb-cert-lab/certs/client-ca-transition.pem
```

Confirm that the bundle contains the expected certificates:

```bash
grep -c -- 'BEGIN CERTIFICATE' \
  ~/crdb-cert-lab/certs/client-ca-transition.pem

openssl crl2pkcs7 \
  -nocrl \
  -certfile ~/crdb-cert-lab/certs/client-ca-transition.pem |
openssl pkcs7 -print_certs -noout
```

If the current trust bundle contained one CA, the new transition bundle should contain two.

### Step 5 - Upload the transition bundle using `PATCH`

```bash
jq -Rs '{x509_pem_cert: .}' \
  ~/crdb-cert-lab/certs/client-ca-transition.pem \
  > ~/crdb-cert-lab/client-ca-transition.json

curl --silent --show-error --fail-with-body \
  --request PATCH \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "content-type: application/json" \
  --data "@${HOME}/crdb-cert-lab/client-ca-transition.json"
```

The operation is asynchronous. Query the resource until its status is `IS_SET`:

```bash
curl --silent --show-error --fail-with-body \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}" |
jq '{status, x509_pem_cert}'
```

Do not proceed while the status is `PENDING`.

### Step 6 - Test both trust paths

First test the existing client certificate signed by the old CA:

```bash
cockroach sql --url \
"postgresql://${SQL_USERNAME}@${CLUSTER_HOST}:26257/defaultdb?sslmode=verify-full&sslrootcert=${SERVER_CA}&sslcert=${HOME}/crdb-cert-lab/certs/client.ron.crt&sslkey=${HOME}/crdb-cert-lab/certs/client.ron.key" \
  --execute="SELECT current_user(), now();"
```

Then test the new client certificate signed by the new CA:

```bash
cockroach sql --url \
"postgresql://${SQL_USERNAME}@${CLUSTER_HOST}:26257/defaultdb?sslmode=verify-full&sslrootcert=${SERVER_CA}&sslcert=${HOME}/crdb-cert-lab/certs/client.ron.ca-new.crt&sslkey=${HOME}/crdb-cert-lab/certs/client.ron.ca-new.key" \
  --execute="SELECT current_user(), now();"
```

Both connections must succeed without requesting a password. If either test fails, stop the rollout and use the rollback guidance below.

### Step 7 - Issue and deploy new credentials for every client

For each SQL user or application:

1. Generate a new private key.
2. Create a CSR whose `CN` exactly matches the SQL username.
3. Sign the certificate with `ca-new.key` and `ca-new.crt`.
4. Verify the certificate chain, identity, validity, client-auth extension, and key match.
5. Deploy the new certificate and key.
6. Establish and test fresh database connections.

Maintain an inventory and do not remove the old CA until every client has been migrated and verified.

### Step 8 - Remove the old CA from the trust bundle

After all clients use certificates signed by the new CA, prepare a payload containing only the new public CA certificate:

```bash
jq -Rs '{x509_pem_cert: .}' \
  ~/crdb-cert-lab/certs/ca-new.crt \
  > ~/crdb-cert-lab/client-ca-new-only.json
```

Replace the transition bundle:

```bash
curl --silent --show-error --fail-with-body \
  --request PATCH \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "content-type: application/json" \
  --data "@${HOME}/crdb-cert-lab/client-ca-new-only.json"
```

Query the resource until its status is `IS_SET`, then test a fresh connection using the new certificate again.

At this point, new connections using certificates signed only by the old CA will no longer authenticate. Existing database sessions may remain active until they disconnect, so verify the result with newly established connections.

### Step 9 - Complete the rotation

- Update the CA and certificate inventory with fingerprints and expiration dates.
- Retain or destroy the old CA private key according to your security and audit policy.
- Remove old client private keys after confirming they are no longer deployed.
- Store the new CA private key in the approved secrets-management system.
- Schedule client certificate renewal before the new certificates expire.

## Rollback

### During the overlap period

If a new certificate fails, leave the old-plus-new transition bundle configured and keep clients on their working old credentials while correcting the new certificate or deployment.

If the transition bundle itself causes a problem, recreate a JSON payload from the saved current CA bundle:

```bash
jq -Rs '{x509_pem_cert: .}' \
  ~/crdb-cert-lab/certs/client-ca-current.pem \
  > ~/crdb-cert-lab/client-ca-rollback.json

curl --silent --show-error --fail-with-body \
  --request PATCH \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "content-type: application/json" \
  --data "@${HOME}/crdb-cert-lab/client-ca-rollback.json"
```

Wait for `IS_SET`, then verify a new connection with a known-good old client certificate.

### After removing the old CA

If a missed client is discovered and the old CA is still valid and trusted by your organization, restore the transition bundle with `PATCH`, wait for `IS_SET`, migrate the missed client, and then remove the old CA again.

Do not restore or continue trusting a CA whose private key is suspected of compromise. Use an alternative authenticated administrative connection and follow your incident-response procedure.

## Operational cautions

- Never overwrite the active CA or client credential before the replacement has been tested.
- Never upload a CA private key or client private key to CockroachDB Cloud.
- Always use `PATCH` when an existing client CA resource has status `IS_SET`.
- Always wait for `IS_SET` after a `PATCH` before testing or continuing.
- Always test new connections; an existing pooled connection does not exercise certificate authentication again.
- Keep the old CA in the transition bundle only as long as needed.
- If the old CA private key is compromised, treat every certificate it could have issued as untrusted and shorten the overlap period accordingly.
- Continue using the CockroachDB Cloud server CA as `sslrootcert`; do not substitute the customer client CA.

## References

- [SQL Client Certificate Authentication for CockroachDB Advanced](https://www.cockroachlabs.com/docs/cockroachcloud/client-certs-advanced)
- [Authentication on CockroachDB Cloud](https://www.cockroachlabs.com/docs/cockroachcloud/authentication)
- [Transport Layer Security and PKI](https://www.cockroachlabs.com/docs/stable/security-reference/transport-layer-security.html)
