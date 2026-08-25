# Lab 3: Encryption and Key Management

| Item | Details |
| --- | --- |
| Course | IKB42603 – Cloud Security |
| Lab | Lab 3: Encryption and Key Management |
| Student | WAN MUHAMMAD IRFAN BIN MOHD ISA |
| Student ID | 52215225234 |
| Platform | Kali Linux, OpenSSL, Docker/Nginx and AWS CLI with LocalStack KMS |

## 1. Introduction

This lab demonstrates how confidentiality, integrity, authentication, and key lifecycle controls work together in a cloud-security setting. The practical work covers encryption of data at rest, asymmetric cryptography and signing, TLS for data in transit, AWS Key Management Service (KMS), envelope encryption, per-tenant key handling/cryptographic erasure, and tamper evidence.

> **Sensitive-data note:** Passwords, private keys, plaintext data keys, and KMS ciphertext blobs should not be committed to a repository or copied into a production report. Commands below use the lab values only to explain the demonstrated workflow.

## 2. Learning Objectives

After completing this lab, the student should be able to:

1. Encrypt and decrypt stored data with AES-256-CBC and PBKDF2.
2. Explain how public/private key pairs and digital signatures provide confidentiality, authentication, and integrity.
3. Create a self-signed X.509 certificate and serve protected content over TLS.
4. Create and use a customer-managed KMS key.
5. Apply envelope encryption so KMS protects a data key rather than encrypting a large file directly.
6. Explain per-tenant keys and cryptographic erasure.
7. Detect unauthorised modification with SHA-256 checksums and a hash chain.

## 3. Task 1 – Symmetric Encryption (Data at Rest)

### Purpose

Symmetric encryption uses the same secret to encrypt and decrypt data. AES-256-CBC provides confidentiality for the record while it is stored. `-pbkdf2` derives a stronger encryption key from the password, and `-salt` ensures identical plaintexts do not produce identical ciphertexts.

### Commands and observations

```bash
echo 'Patient: Ahmad, Diagnosis: confidential' > record.txt
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc
cat record.enc
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt
diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```

| Command | What it does |
| --- | --- |
| `echo ... > record.txt` | Creates a sample confidential medical record. |
| `openssl enc -aes-256-cbc ...` | Encrypts `record.txt` with AES-256 in CBC mode; OpenSSL prompts for a password. |
| `cat record.enc` | Displays the ciphertext, which is unreadable binary/salted output rather than the original text. |
| `openssl enc -d ...` | Decrypts the ciphertext using the correct password and parameters. |
| `diff ... && echo ...` | Compares the original and decrypted files; `MATCH` confirms successful recovery. |

### Result

The encrypted file was unreadable when viewed directly, and decryption reproduced the original file exactly. This verifies confidentiality for data at rest, subject to protecting the password.

### Evidence

![Task 1 evidence: AES-256-CBC encryption, unreadable ciphertext, decryption, and matching output](<evidence/Task1-SymmetricEncryption(Data at Rest).png>)

### What is the key-distribution problem with symmetric encryption, and why does it matter for the cloud?

Symmetric encryption requires securely sharing the same secret key with authorised users/services. In the cloud, sharing one key across apps, users, backups, and servers is risky; if it leaks, all protected data may be exposed.

## 4. Task 2 – Asymmetric Encryption and Digital Signatures

### Purpose

Asymmetric cryptography uses a public key and a private key. A recipient's public key can encrypt data that only their private key can decrypt. A private key can create a digital signature that anyone with the matching public key can verify. This supports authentication, integrity, and non-repudiation (subject to proper identity/key management).

### Commands and explanations

```bash
# Generate a 2048-bit RSA private key and its public key
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem

# Encrypt for the public-key holder, then decrypt with the private key
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa.enc
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa.enc -out record.rsa.dec.txt

# Sign a file and verify the signature
openssl dgst -sha256 -sign private.pem -out record.sig record.txt
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

| Command | What it does |
| --- | --- |
| `openssl genpkey ...` | Generates the RSA private key; it must remain secret. |
| `openssl pkey ... -pubout` | Exports the shareable public key from the private key. |
| `openssl pkeyutl -encrypt ...` | Encrypts a small input using the public key. In practice, RSA normally protects a symmetric data key rather than a large file. |
| `openssl pkeyutl -decrypt ...` | Uses the private key to recover the encrypted input. |
| `openssl dgst -sha256 -sign ...` | Hashes the file with SHA-256 and signs that digest with the private key. |
| `openssl dgst ... -verify ...` | Verifies that the signature matches the file and public key. |

### Result

The asymmetric/signature workflow demonstrates separation between an encryption/verification key that can be shared and a private key that must be protected. A signature verification should fail after the signed file is changed.

### Evidence

The supplied Task 2 screenshot records an OpenSSL **bad password read** error rather than a completed RSA/signature command. It is included as the available evidence; the commands above state the required successful workflow.

![Task 2 evidence: supplied OpenSSL output](<evidence/Task2-AsymmetricEncryption&DigitalSignatures.png>)

## 5. Task 3 – TLS Encryption (Data in Transit)

### 5.1 Generate a self-signed certificate

```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 7 -nodes -subj '/CN=localhost'
```

This command creates a new RSA private key (`key.pem`) and a self-signed X.509 certificate (`cert.pem`) valid for seven days. `-nodes` leaves the key unencrypted so the web server can start non-interactively; this is acceptable only for the controlled lab and is not recommended for production.

![Task 3.1 evidence: self-signed certificate creation](<evidence/Task3.1-GenerateSelf-signedCertificate.png>)

### 5.2 Serve HTTPS on port 8443

```bash
docker run -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx
```

This starts an Nginx container, maps host port `8443` to HTTPS port `443`, and mounts the certificate, private key, and test file into the container. Docker downloads the Nginx image if it is not already available locally.

![Task 3.2 evidence: Nginx image download/container setup](<evidence/Task3.2-Serve HTTPSOnPort8443.png>)

### 5.3 Connect over TLS

```bash
curl -k https://localhost:8443/record.txt
```

`curl` requests the record through HTTPS. `-k` skips certificate-chain verification because the lab certificate is self-signed. The returned record confirms the HTTPS endpoint is reachable; in production, use a certificate issued by a trusted CA and omit `-k`.

![Task 3.3 evidence: successful HTTPS request](<evidence/Task3.3-ConnectOverTLS.png>)

### Result

TLS protects data in transit by encrypting the HTTP session and authenticating the server certificate. The self-signed certificate provides encryption in the lab but does not provide third-party trust validation.

## 6. Task 4 – Create and Use a KMS Master Key

### Purpose

AWS KMS centralises key storage, authorisation, auditing, rotation, and lifecycle controls. The lab uses a LocalStack endpoint (`$EP`) to emulate AWS KMS.

### Commands and explanations

```bash
aws $EP kms create-key --description 'CCSE tenant-A master key'
KEY_A='9aa092ab-d146-4818-8be8-c5cd0b27258a'
aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" --query CiphertextBlob --output text
```

| Command | What it does |
| --- | --- |
| `aws $EP kms create-key ...` | Creates a customer-managed symmetric KMS key and returns its metadata, ARN, and key ID. |
| `KEY_A=...` | Stores the generated key ID in a shell variable for later commands. The actual ID will differ per lab run. |
| `aws $EP kms encrypt ...` | Sends base64-encoded plaintext to KMS and returns a KMS ciphertext blob. KMS performs the encryption without exposing its key material. |

### Result

The evidence shows an enabled customer-managed symmetric KMS key with `ENCRYPT_DECRYPT` usage and the encryption request returning a ciphertext blob.

![Task 4.1 evidence: KMS key creation and metadata](<evidence/Task4.1-Create and Use a KMS Master Key.png>)

![Task 4.2 evidence: encrypting data with the KMS key](<evidence/Tak4.2-Create and Use a KMS Master Key.png>)

## 7. Task 5 – Envelope Encryption

### Purpose

Envelope encryption uses a locally generated plaintext data key to encrypt the data and retains only a KMS-encrypted (wrapped) copy of that data key. This scales better than sending large files directly to KMS and allows the KMS key to control access to the data key.

### 5.1 Request a data key from KMS

```bash
aws $EP kms generate-data-key --key-id $KEY_A --key-spec AES_256 \
  --query '[Plaintext,CiphertextBlob]' --output text
echo '<plaintext-data-key-base64>' > datakey.b64
echo '<kms-wrapped-data-key-base64>' > datakey.enc
```

`generate-data-key` returns two versions of the same AES-256 data key: a base64 plaintext version for immediate local encryption and a ciphertext version wrapped by KMS for safe storage.

![Task 5.1 evidence: plaintext and KMS-wrapped data-key outputs](<evidence/Task5.1-Ask KMS for a data key (returns plaintext + encrypted versions).png>)

### 5.2 Encrypt the file locally

```bash
base64 -d datakey.b64 > datakey.bin
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc -pass file:./datakey.bin
```

`base64 -d` converts the KMS plaintext data-key output to raw key bytes. The OpenSSL command then encrypts `record.txt` locally with that data key, producing `record.env.enc`.

![Task 5.2 evidence: decoding the data key and local AES encryption](<evidence/Task5.2-Encrypt the big file locally with the PLAINTEXT data key.png>)

### 5.3 Remove plaintext key material

```bash
rm datakey.bin datakey.b64
echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```

`rm` removes the local plaintext forms of the data key. Only the wrapped copy is retained; it must be decrypted by KMS before the encrypted record can be recovered. On production storage, use an approved secure deletion/lifecycle approach because ordinary file deletion may not erase physical media immediately.

![Task 5.3 evidence: plaintext data-key files removed](<evidence/Task5.3-Destroy the plaintext data key from disk — keep only the wrapped copy.png>)

### Result

The evidence shows the envelope-encryption pattern: a plaintext AES data key is used only briefly, while `datakey.enc` is kept as the KMS-protected key-encryption-key output.

## 8. Task 6 – Per-Tenant Keys and Cryptographic Erasure

### Purpose

Using a separate KMS key per tenant reduces the blast radius of a compromise and enables tenant-specific access control, auditing, rotation, and deletion. Cryptographic erasure makes protected data unrecoverable by permanently deleting the required key material.

### 6.1 Schedule, cancel, and disable a tenant key

```bash
aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7
aws $EP kms describe-key --key-id $KEY_A --query 'KeyMetadata.KeyState' --output text
aws $EP kms cancel-key-deletion --key-id $KEY_A
aws $EP kms disable-key --key-id $KEY_A
aws $EP kms describe-key --key-id $KEY_A --query 'KeyMetadata.KeyState' --output text
```

| Command | What it does |
| --- | --- |
| `schedule-key-deletion` | Starts the KMS deletion waiting period. The evidence shows a seven-day window and `PendingDeletion` state. |
| `describe-key ... KeyState` | Displays only the key lifecycle state. |
| `cancel-key-deletion` | Cancels pending deletion before the waiting period expires. |
| `disable-key` | Immediately blocks KMS cryptographic operations with the key without deleting it. |

The evidence shows `PendingDeletion`, successful cancellation, and the final `Disabled` state. The attempted `disable-key` while deletion was pending correctly returned `KMSInvalidStateException`; this is expected state enforcement.

![Task 6.1 evidence: key deletion lifecycle and disable state](<evidence/Task6.1-Per-Tenant Keys & Cryptographic Erasure.png>)

### 6.2 Attempt recovery after key/key-blob handling change

```bash
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```

This asks KMS to unwrap the persisted ciphertext data key. The supplied evidence returns `NotFoundException` with an invalid key ID embedded in the blob; that demonstrates recovery did not succeed. The screenshot also shows that `datakey.enc` was written using `echo` of base64 text. For a correct recovery workflow, decode the stored Base64 ciphertext blob first or use a CLI format that preserves binary data, for example:

```bash
base64 -d datakey.enc > datakey.wrapped.bin
aws $EP kms decrypt --ciphertext-blob fileb://datakey.wrapped.bin --query Plaintext --output text
```

Even with a correctly stored wrapped blob, decryption must fail after permanent deletion of the tenant's KMS key. Disabling a key is reversible; scheduled deletion can be cancelled before its deadline, so neither alone is permanent cryptographic erasure.

![Task 6.2 evidence: unsuccessful KMS decrypt attempt](<evidence/Task6.2-Per-Tenant Keys & Cryptographic Erasure.png>)

## 9. Task 7 – Integrity and Tamper Evidence

### Purpose

Encryption does not automatically prove a file has not changed. SHA-256 provides a fixed-length digest that changes when the file changes. A hash chain also binds each event to the preceding event, making later modification detectable.

### Commands and explanations

```bash
sha256sum record.txt
cp record.txt tampered.txt; echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt

PREV=0
for line in 'login ok' 'file read' 'export data'; do
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d ' ' -f1)
  echo "$line | $PREV"
done
```

| Command | What it does |
| --- | --- |
| `sha256sum record.txt` | Calculates the baseline SHA-256 digest of the original record. |
| `cp ...; echo 'x' >> ...` | Copies the file and deliberately appends data to simulate tampering. |
| `sha256sum record.txt tampered.txt` | Shows different digests, proving the copy has changed. |
| `PREV=0` | Sets the initial value for the hash chain. |
| `for line ...` loop | Calculates `SHA-256(previous_hash || event_text)` for every event and prints an audit-chain entry. |

### Result

The original and tampered files have different SHA-256 hashes. Each hash-chain output depends on the prior output, so altering, removing, or reordering an earlier event changes all following expected hashes when the chain is verified.

![Task 7 evidence: checksum comparison and chained event hashes](<evidence/Task7-Integrity & Tamper-Evidence.png>)

## 10. Security Analysis and Conclusion

The lab demonstrates defence in depth:

| Security need | Lab control | Key takeaway |
| --- | --- | --- |
| Data at rest | AES-256-CBC with PBKDF2 and salt | Ciphertext is protected only as well as the password/data key. |
| Data in transit | HTTPS/TLS | Self-signed certificates encrypt traffic but require explicit trust handling. |
| Authentication and integrity | RSA digital signatures | Verification detects document modification and identifies the holder of the private key. |
| Key governance | AWS KMS | Centralised policies and lifecycle state control the use of master keys. |
| Scalable file encryption | Envelope encryption | Encrypt data locally with a data key and keep only its KMS-wrapped version. |
| Tenant isolation/erasure | Per-tenant KMS keys | Permanent key deletion can make all data encrypted under that key unrecoverable. |
| Tamper detection | SHA-256 and hash chains | Hashes make modifications detectable; they do not prevent modification. |

In conclusion, encryption is effective only when paired with disciplined key management. Private keys and plaintext data keys must be tightly controlled, certificates must be validated, KMS permissions must follow least privilege, and deletion/retention processes must be designed carefully because key destruction can make data permanently unrecoverable.

## 11. Short-Answer Questions

### Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

Symmetric encryption is fast and uses one shared key. Asymmetric encryption is slower and uses public/private keys. Symmetric is for files/data; asymmetric is for key sharing and signatures.

### Q2. Why is key management described as the weakest link, not the algorithm?

Encryption algorithms are strong, but stolen or badly managed keys let attackers decrypt everything.

### Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.

Envelope encryption uses a data key to encrypt files and a master key to encrypt the data key. Only the master key needs strong hardware protection because it protects many data keys.

### Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot in the cloud?

Cryptographic erasure deletes the encryption key. Even if cloud backups remain, the encrypted data cannot be read without the key.

### Q5. How does a hash chain make a log tamper-evident? (Week 6 tamper-proof logs)

A hash chain links each log entry to the previous hash. Changing one log entry changes all later hashes, showing tampering.

