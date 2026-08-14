# Client Certificate Authentication for CockroachDB Advanced

## Overview

This guide is a practical, OpenSSL-based recipe for creating client certificates that can be used to connect to a CockroachDB Advanced cluster without supplying a SQL password. It is intended as a reusable demonstration and starting point for customers who want to evaluate certificate-based SQL authentication.

The procedure creates a customer-managed client Certificate Authority (CA), uploads the public CA certificate to the CockroachDB Advanced cluster, and uses that CA to sign a certificate for a SQL user. The CA private key and the client's private key remain under the customer's control and must never be uploaded to CockroachDB Cloud.

> **Important:** This walkthrough manages private key material manually and is most appropriate for a lab, demonstration, or proof of concept. For production, use your organization's approved PKI, secrets-management, certificate-lifetime, rotation, and key-protection practices.

## How it works

1. Create a customer-managed **client CA** consisting of a private signing key and a public CA certificate.
2. Upload only the public client CA certificate to the CockroachDB Advanced cluster. This adds the CA to the cluster's trust store for SQL client authentication.
3. Create a private key and Certificate Signing Request (CSR) for the SQL user. The certificate's Common Name (`CN`) identifies the SQL user.
4. Sign the CSR with the client CA to create the user's client certificate.
5. Connect with the SQL username, client certificate, client private key, and the CockroachDB Cloud server CA certificate.
6. During connection, the cluster verifies that the client certificate was signed by the uploaded client CA and maps its `CN` to the SQL user. The client independently verifies the cluster's identity using the CockroachDB Cloud server CA.

There are two different CA certificates in this workflow:

| Certificate | Managed by | Purpose |
| --- | --- | --- |
| Customer client CA (`certs/ca.crt`) | Customer | Uploaded to the cluster so it can trust client certificates issued by the customer. |
| CockroachDB Cloud server CA | Cockroach Labs | Downloaded from the cluster's **Connect** dialog and supplied as `sslrootcert` so the client can verify the cluster. |

## Prerequisites

- A CockroachDB **Advanced** cluster. Client certificate authentication using a customer-managed CA is specific to Advanced clusters.
- `Cluster Admin` or `Organization Admin` access to configure the cluster's client CA.
- A CockroachDB Cloud service account and API key for the Cloud API calls. Treat the API key as a secret and do not commit it to source control.
- The cluster ID and Cloud API server configured in the shell:

  ```bash
  export COCKROACH_SERVER="https://cockroachlabs.cloud"
  export CLUSTER_ID="<cluster-id>"
  export API_KEY="<service-account-api-key>"
  ```

- `openssl`, `jq`, `curl`, and the `cockroach` SQL CLI installed and available on the command path.
- Network connectivity to the cluster through an approved IP allowlist entry or private connection.
- The CockroachDB Cloud server CA certificate downloaded from the cluster's **Connect** dialog. Its local path is used as `sslrootcert` in the final connection string.
- An existing CockroachDB SQL user for the certificate identity. The certificate `CN`, username in the connection URL, and SQL username must match exactly. For this example, all three are `ron`:

  ```sql
  -- Run as a SQL administrator if the user does not already exist.
  CREATE USER ron;
  ```

  The CSR later uses `-subj "/O=Cockroach/CN=ron"`, and the connection URL uses `postgresql://ron@...`. A certificate with a different `CN` will not authenticate as `ron`.

## Step 1 — Create a clean workspace

```bash
rm -rf ~/crdb-cert-lab
mkdir -p ~/crdb-cert-lab/certs
mkdir -p ~/crdb-cert-lab/ca-private

chmod 700 ~/crdb-cert-lab/ca-private

cd ~/crdb-cert-lab
```

## Step 2 — Create `ca.cnf`

An `.cnf` file is an OpenSSL configuration file. It is a plain text file that tells OpenSSL how to behave when creating keys, CSRs, certificates, and signing requests.

```bash
vi ~/crdb-cert-lab/ca.cnf
```

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
default_days = 3660
database = index.txt
new_certs_dir = newcerts
serial = serial.txt
default_md = sha256
copy_extensions = copy
unique_subject = no

[ req ]
prompt = no
distinguished_name = distinguished_name
x509_extensions = extensions

[ distinguished_name ]
organizationName = Cockroach
commonName = Cockroach CA

[ extensions ]
keyUsage = critical,digitalSignature,nonRepudiation,keyEncipherment,keyCertSign
basicConstraints = critical,CA:true,pathlen:1

[ signing_policy ]
organizationName = supplied
commonName = supplied

[ signing_client_req ]
keyUsage = critical,digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth
```

## Step 3 — Create the CA private key

```bash
openssl genrsa \
  -out ~/crdb-cert-lab/ca-private/ca.key \
  2048
```

```bash
chmod 400 ~/crdb-cert-lab/ca-private/ca.key
```

Verify

```bash
openssl rsa \
  -in ~/crdb-cert-lab/ca-private/ca.key \
  -check \
  -noout
```

Output should be

```text
RSA key ok
```

## Step 4 — Create the self-signed CA certificate

```bash
openssl req \
  -new \
  -x509 \
  -config ~/crdb-cert-lab/ca.cnf \
  -key ~/crdb-cert-lab/ca-private/ca.key \
  -sha256 \
  -days 3660 \
  -out ~/crdb-cert-lab/certs/ca.crt
```

```bash
openssl x509 \
  -in ~/crdb-cert-lab/certs/ca.crt \
  -text \
  -noout
```

Verify

```bash
openssl verify \
  -CAfile ~/crdb-cert-lab/certs/ca.crt \
  ~/crdb-cert-lab/certs/ca.crt
```

Output should be

```text
/home/ec2-user/crdb-cert-lab/certs/ca.crt: OK
```

## Step 5 — Prepare the JSON payload for CockroachDB Cloud

```bash
jq -Rs '{x509_pem_cert: .}' \
  certs/ca.crt \
  > cockroach_client_ca_cert.json
```

## Step 6 — Upload the CA certificate to the Advanced cluster

```bash
curl --request POST \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "content-type: application/json" \
  --data "@cockroach_client_ca_cert.json"
```

The upload is asynchronous. A successful `POST` means that CockroachDB Cloud accepted the request, not necessarily that the client CA is active yet. Check its status:

```bash
curl --request GET \
  --url "${COCKROACH_SERVER}/api/v1/clusters/${CLUSTER_ID}/client-ca-cert" \
  --header "Authorization: Bearer ${API_KEY}"
```

The first `GET` may legitimately return `PENDING` while the cluster is being updated:

```json
{
  "status": "PENDING",
  "x509_pem_cert": ""
}
```

Repeat the same `GET` request until the status is `IS_SET`. Do not continue to client authentication until the CA is active:

```text
{
  "status": "IS_SET",
  ...
}
```

## Step 7 — Create Ron’s private key

```bash
openssl genrsa \
  -out ~/crdb-cert-lab/certs/client.ron.key \
  2048

chmod 400 ~/crdb-cert-lab/certs/client.ron.key

openssl rsa \
  -in ~/crdb-cert-lab/certs/client.ron.key \
  -check \
  -noout
```

## Step 8 — Create a certificate signing request for Ron

```bash
openssl req \
  -new \
  -key ~/crdb-cert-lab/certs/client.ron.key \
  -out ~/crdb-cert-lab/client.ron.csr \
  -subj "/O=Cockroach/CN=ron"
```

## Step 9 — Initialize the OpenSSL CA bookkeeping files

```bash
cd ~/crdb-cert-lab
mkdir -p ~/crdb-cert-lab/newcerts

touch index.txt
echo 1000 > serial.txt
```

## Step 10 — Sign the CSR

```bash
openssl ca \
  -config ~/crdb-cert-lab/ca.cnf \
  -keyfile ~/crdb-cert-lab/ca-private/ca.key \
  -cert ~/crdb-cert-lab/certs/ca.crt \
  -policy signing_policy \
  -extensions signing_client_req \
  -in ~/crdb-cert-lab/client.ron.csr \
  -out ~/crdb-cert-lab/certs/client.ron.crt \
  -batch

openssl x509 \
  -in ~/crdb-cert-lab/certs/client.ron.crt \
  -text \
  -noout

```

Verify the chain

```bash
openssl verify \
  -CAfile ~/crdb-cert-lab/certs/ca.crt \
  ~/crdb-cert-lab/certs/client.ron.crt
```

## Step 11 — Connect to the database
Example Only!
```bash
cockroach sql --url \
"postgresql://ron@nollen-cert-test-zgh.aws-us-west-2.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full&sslrootcert=$HOME/Library/CockroachCloud/certs/1d4d68ed-a173-461e-a522-4fbca2b062e1/nollen-cert-test-ca.crt&sslcert=$HOME/crdb-cert-lab/certs/client.ron.crt&sslkey=$HOME/crdb-cert-lab/certs/client.ron.key"
```

## CSR proof versus TLS handshake proof

The CockroachDB cluster never receives or examines the CSR. The CSR is used only during certificate issuance between Ron and the client CA.

When the client CA receives the CSR, it uses Ron's public key contained inside the CSR to verify the CSR signature. This lets the CA establish:

- The CSR contains public key X.
- The requester possessed the private key matching X when the CSR was created.

The CA must then decide whether it is willing to certify that public key as belonging to the identity `ron`. It signs `client.ron.crt` with `ca.key` to record that approval and identity-to-public-key binding.

Later, during the TLS connection, the cluster receives `client.ron.crt`, not the CSR. The cluster uses the uploaded public CA certificate to verify the CA signature on `client.ron.crt`. The client then signs TLS handshake data with `client.ron.key`, and the cluster verifies that signature using the public key in `client.ron.crt`. This is a new, connection-time proof that the connecting client currently possesses Ron's private key.

The important distinction is:

- The CSR is signed by `client.ron.key`: proof to the CA of key possession during certificate issuance.
- `client.ron.crt` is signed by `ca.key`: proof of the CA's approval of the identity-to-public-key binding.
- The TLS handshake is signed by `client.ron.key`: proof to the cluster of key possession during the live connection.

## Another explanation

When `client.ron.crt` is issued:

1. The certificate contents are assembled: `CN=ron`, Ron’s public key, expiration, extensions, issuer, and other metadata.
2. Those contents are hashed.
3. The CA signs that hash using the private `ca.key`.
4. The resulting signature is stored in `client.ron.crt`.

The cluster then uses the CA public key from the uploaded `ca.crt` to verify that signature. If the certificate were altered afterward—even changing `CN=ron` to another username—the signature verification would fail.

Uploading `ca.crt` also makes that CA a trusted issuer. More precisely, the cluster accepts client certificates signed by that CA only when they also:

- Are currently valid and otherwise structurally valid.
- Are suitable for client authentication.
- Identify an allowed SQL user, such as `CN=ron`.
- Match the username used for the connection.
- Are accompanied by proof that the connecting client possesses the corresponding private key.

So it means “trust valid client certificates issued by this CA,” not blindly accept every arbitrary object bearing its signature.

This is why `ca.key` is extremely sensitive: anyone possessing it could issue a valid-looking client certificate for `CN=ron` or another SQL user trusted by the cluster.
