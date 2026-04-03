---
title: "Self-Custody Signing"
linkTitle: "Self-Custody"
weight: 3
description: >
  Sign transactions with your own private key using the self-custody model.
---

This guide explains the self-custody signing model and walks through generating a keypair, signing transactions, and verifying receipts in Go, Python, and JavaScript.

## Overview

In the self-custody model, the account holder controls the signing key. Instead of delegating transaction authorization to Sankofa Labs, each transaction is signed with the account's own ECDSA P-256 private key before submission. The engine verifies the signature against the public key registered during account setup.

This means:

- **No custodial dependency** -- Sankofa Labs never holds or has access to your private key. The engine cannot authorize transactions on your behalf.
- **Cryptographic authorization** -- every transaction carries a mathematically verifiable proof that the account holder approved it.
- **Non-repudiation** -- the account holder cannot deny having authorized a signed transaction.

{{% alert title="Note" color="info" %}}
Even in the self-custody model, a valid JWT is still required for API access. The transaction signature provides an additional layer of authorization proving the account holder approved the specific transaction. The JWT authenticates the API caller; the signature authorizes the transaction.
{{% /alert %}}

## Step 1: Generate a Keypair

Generate an ECDSA P-256 keypair. Store the private key securely -- it should never leave your infrastructure.

{{< tabpane text=true >}}
{{% tab header="Go" lang="go" %}}
```go
package main

import (
	"crypto/ecdsa"
	"crypto/elliptic"
	"crypto/rand"
	"crypto/x509"
	"encoding/pem"
	"fmt"
	"os"
)

func main() {
	// Generate P-256 keypair
	privateKey, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
	if err != nil {
		panic(err)
	}

	// Encode private key to PEM
	privBytes, err := x509.MarshalECPrivateKey(privateKey)
	if err != nil {
		panic(err)
	}
	privPEM := pem.EncodeToMemory(&pem.Block{
		Type:  "EC PRIVATE KEY",
		Bytes: privBytes,
	})
	os.WriteFile("private_key.pem", privPEM, 0600)

	// Encode public key to PEM
	pubBytes, err := x509.MarshalPKIXPublicKey(&privateKey.PublicKey)
	if err != nil {
		panic(err)
	}
	pubPEM := pem.EncodeToMemory(&pem.Block{
		Type:  "PUBLIC KEY",
		Bytes: pubBytes,
	})
	os.WriteFile("public_key.pem", pubPEM, 0644)

	fmt.Println("Keypair generated: private_key.pem, public_key.pem")
}
```
{{% /tab %}}
{{% tab header="Python" lang="python" %}}
```python
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import serialization

# Generate P-256 keypair
private_key = ec.generate_private_key(ec.SECP256R1())

# Save private key
with open("private_key.pem", "wb") as f:
    f.write(private_key.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption(),
    ))

# Save public key
public_key = private_key.public_key()
with open("public_key.pem", "wb") as f:
    f.write(public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo,
    ))

print("Keypair generated: private_key.pem, public_key.pem")
```
{{% /tab %}}
{{% tab header="JavaScript" lang="javascript" %}}
```javascript
import { generateKeyPairSync } from "node:crypto";
import { writeFileSync } from "node:fs";

// Generate P-256 keypair
const { publicKey, privateKey } = generateKeyPairSync("ec", {
  namedCurve: "P-256",
  publicKeyEncoding: {
    type: "spki",
    format: "pem",
  },
  privateKeyEncoding: {
    type: "pkcs8",
    format: "pem",
  },
});

writeFileSync("private_key.pem", privateKey, { mode: 0o600 });
writeFileSync("public_key.pem", publicKey, { mode: 0o644 });

console.log("Keypair generated: private_key.pem, public_key.pem");
```
{{% /tab %}}
{{< /tabpane >}}

{{% alert title="Security" color="warning" %}}
Never commit private keys to version control, embed them in application code, or transmit them over unencrypted channels. Use a secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager) in production.
{{% /alert %}}

## Step 2: Register Your Public Key

During account setup, register your public key with the Sankofa Engine. This is typically done once by your Sankofa Labs onboarding contact, or via an administrative API call.

Provide the PEM-encoded public key from `public_key.pem`:

```text
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...base64-encoded...
-----END PUBLIC KEY-----
```

Once registered, the engine will verify all transaction signatures from this account against this public key. To rotate keys, register a new public key and begin signing with the new private key.

## Step 3: Construct the Transaction Payload

Build the JSON payload that will be signed. The payload includes all transaction fields **except** the `signature` field itself.

```json
{
  "account_id": "alice@example.com",
  "amount": "100.00",
  "type": "debit",
  "idempotency_key": "txn-20260403-001"
}
```

**Canonicalization rules:**

- Keys must be sorted alphabetically.
- No extra whitespace (use compact JSON separators).
- No trailing newlines.

The canonical form of the above payload is:

```text
{"account_id":"alice@example.com","amount":"100.00","idempotency_key":"txn-20260403-001","type":"debit"}
```

## Step 4: Sign the Payload

Canonicalize the JSON, compute the SHA-256 digest, and sign it with your ECDSA P-256 private key.

{{< tabpane text=true >}}
{{% tab header="Go" lang="go" %}}
```go
package main

import (
	"crypto/ecdsa"
	"crypto/rand"
	"crypto/sha256"
	"crypto/x509"
	"encoding/base64"
	"encoding/json"
	"encoding/pem"
	"fmt"
	"os"
)

func main() {
	// Build payload
	payload := map[string]string{
		"account_id":     "alice@example.com",
		"amount":         "100.00",
		"type":           "debit",
		"idempotency_key": "txn-20260403-001",
	}

	// Canonicalize (json.Marshal sorts keys by default in Go)
	canonical, _ := json.Marshal(payload)

	// SHA-256 digest
	digest := sha256.Sum256(canonical)

	// Load private key
	keyPEM, _ := os.ReadFile("private_key.pem")
	block, _ := pem.Decode(keyPEM)
	privateKey, _ := x509.ParseECPrivateKey(block.Bytes)

	// Sign
	sig, err := ecdsa.SignASN1(rand.Reader, privateKey, digest[:])
	if err != nil {
		panic(err)
	}

	// Base64url encode
	signature := base64.RawURLEncoding.EncodeToString(sig)
	fmt.Println("Signature:", signature)
}
```
{{% /tab %}}
{{% tab header="Python" lang="python" %}}
```python
import json
import base64
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import hashes, serialization

# Build payload
payload = {
    "account_id": "alice@example.com",
    "amount": "100.00",
    "type": "debit",
    "idempotency_key": "txn-20260403-001",
}

# Canonicalize
canonical = json.dumps(payload, sort_keys=True, separators=(",", ":"))

# Load private key
with open("private_key.pem", "rb") as f:
    private_key = serialization.load_pem_private_key(f.read(), password=None)

# Sign (ECDSA with SHA-256)
signature_bytes = private_key.sign(
    canonical.encode(),
    ec.ECDSA(hashes.SHA256()),
)

# Base64url encode
signature = base64.urlsafe_b64encode(signature_bytes).decode().rstrip("=")
print("Signature:", signature)
```
{{% /tab %}}
{{% tab header="JavaScript" lang="javascript" %}}
```javascript
import { createSign, createPrivateKey } from "node:crypto";
import { readFileSync } from "node:fs";

// Build payload
const payload = {
  account_id: "alice@example.com",
  amount: "100.00",
  idempotency_key: "txn-20260403-001",
  type: "debit",
};

// Canonicalize (keys sorted alphabetically)
const canonical = JSON.stringify(payload, Object.keys(payload).sort());

// Load private key
const privateKey = createPrivateKey(readFileSync("private_key.pem"));

// Sign (ECDSA with SHA-256)
const signer = createSign("SHA256");
signer.update(canonical);
const signatureDer = signer.sign(privateKey);

// Base64url encode
const signature = signatureDer
  .toString("base64url");

console.log("Signature:", signature);
```
{{% /tab %}}
{{< /tabpane >}}

## Step 5: Submit the Signed Transaction

Include the signature in the `signature` field of the transaction request:

```bash
curl -X POST https://api.example.com/v1/transactions \
  -H "Authorization: Bearer eyJhbGciOiJFUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "alice@example.com",
    "amount": "100.00",
    "type": "debit",
    "idempotency_key": "txn-20260403-001",
    "signature": "MEUCIQDx...base64url-encoded..."
  }'
```

**Expected response (202 Accepted):**

```json
{
  "txn_id": "txn_01HYX3KPVW9QDMZ6R8F5GT7N4E",
  "account_id": "alice@example.com",
  "status": "pending",
  "shard_id": "shard-07",
  "enqueued_at": "2026-04-03T14:22:01.456Z"
}
```

The API Gateway verifies the signature against the registered public key before enqueueing the transaction. If the signature is invalid, you will receive a `403 Forbidden` response:

```json
{
  "type": "https://api.example.com/errors/forbidden",
  "title": "Forbidden",
  "status": 403,
  "detail": "Transaction signature verification failed. Ensure the payload is canonicalized and signed with the registered key.",
  "instance": "/v1/transactions"
}
```

## Step 6: Verify the Receipt

After the transaction is finalized, you can verify the engine's receipt signature to confirm that the Sankofa Engine processed the transaction and the receipt has not been tampered with.

First, obtain the engine's public key (published at a well-known endpoint or provided during onboarding).

Then retrieve the finalized transaction and verify the receipt signature:

{{< tabpane text=true >}}
{{% tab header="Go" lang="go" %}}
```go
package main

import (
	"crypto/ecdsa"
	"crypto/sha256"
	"crypto/x509"
	"encoding/base64"
	"encoding/json"
	"encoding/pem"
	"fmt"
	"os"
)

func main() {
	// The receipt payload fields (from the finalized transaction response)
	receipt := map[string]string{
		"group_id":   "grp_01HYX3MQNW8RDAZ5T7F4HS6K3D",
		"account_id": "alice@example.com",
		"amount":     "100.00",
		"type":       "debit",
		"audit_hash": "sha256:a3f8c2d1e9b04567...",
		"shard_id":   "shard-07",
		"timestamp":  "2026-04-03T14:22:02.112Z",
	}

	// Canonicalize
	canonical, _ := json.Marshal(receipt)
	digest := sha256.Sum256(canonical)

	// Load engine's public key
	keyPEM, _ := os.ReadFile("engine_public_key.pem")
	block, _ := pem.Decode(keyPEM)
	pubKeyInterface, _ := x509.ParsePKIXPublicKey(block.Bytes)
	publicKey := pubKeyInterface.(*ecdsa.PublicKey)

	// Decode the receipt signature
	sigB64 := "MEYCIQDy...base64url-encoded..."
	sigBytes, _ := base64.RawURLEncoding.DecodeString(sigB64)

	// Verify
	valid := ecdsa.VerifyASN1(publicKey, digest[:], sigBytes)
	fmt.Printf("Receipt signature valid: %t\n", valid)
}
```
{{% /tab %}}
{{% tab header="Python" lang="python" %}}
```python
import json
import base64
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import hashes, serialization

# Receipt payload fields (from the finalized transaction response)
receipt = {
    "group_id": "grp_01HYX3MQNW8RDAZ5T7F4HS6K3D",
    "account_id": "alice@example.com",
    "amount": "100.00",
    "type": "debit",
    "audit_hash": "sha256:a3f8c2d1e9b04567...",
    "shard_id": "shard-07",
    "timestamp": "2026-04-03T14:22:02.112Z",
}

# Canonicalize
canonical = json.dumps(receipt, sort_keys=True, separators=(",", ":"))

# Load engine's public key
with open("engine_public_key.pem", "rb") as f:
    public_key = serialization.load_pem_public_key(f.read())

# Decode receipt signature
sig_b64 = "MEYCIQDy...base64url-encoded..."
sig_bytes = base64.urlsafe_b64decode(sig_b64 + "==")

# Verify
try:
    public_key.verify(sig_bytes, canonical.encode(), ec.ECDSA(hashes.SHA256()))
    print("Receipt signature valid: True")
except Exception:
    print("Receipt signature valid: False")
```
{{% /tab %}}
{{% tab header="JavaScript" lang="javascript" %}}
```javascript
import { createVerify, createPublicKey } from "node:crypto";
import { readFileSync } from "node:fs";

// Receipt payload fields (from the finalized transaction response)
const receipt = {
  account_id: "alice@example.com",
  amount: "100.00",
  audit_hash: "sha256:a3f8c2d1e9b04567...",
  group_id: "grp_01HYX3MQNW8RDAZ5T7F4HS6K3D",
  shard_id: "shard-07",
  timestamp: "2026-04-03T14:22:02.112Z",
  type: "debit",
};

// Canonicalize (keys sorted alphabetically)
const canonical = JSON.stringify(receipt, Object.keys(receipt).sort());

// Load engine's public key
const publicKey = createPublicKey(readFileSync("engine_public_key.pem"));

// Decode receipt signature
const sigB64 = "MEYCIQDy...base64url-encoded...";
const sigBuffer = Buffer.from(sigB64, "base64url");

// Verify
const verifier = createVerify("SHA256");
verifier.update(canonical);
const valid = verifier.verify(publicKey, sigBuffer);

console.log("Receipt signature valid:", valid);
```
{{% /tab %}}
{{< /tabpane >}}

## Summary

| Step | Action | What It Proves |
|------|--------|----------------|
| Generate keypair | Create ECDSA P-256 key pair | You control the signing key |
| Register public key | Share public key with engine | Engine can verify your signatures |
| Sign payload | Canonicalize JSON, SHA-256, ECDSA sign | You authorized this specific transaction |
| Submit transaction | Include signature in request | Gateway verifies before enqueueing |
| Verify receipt | Verify engine's receipt signature | Engine processed the transaction; receipt is authentic |
