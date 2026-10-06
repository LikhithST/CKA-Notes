# TLS & Public Key Infrastructure (PKI) Fundamentals - CKA Exam Notes

> **Exam Domain**: Cluster Architecture, Installation & Configuration (25%) / Cluster Security  
> **Weight / Importance**: Essential Core Foundation (Cryptographic foundation for all Kubernetes communication: mutual TLS between control-plane daemons, kubelet authentication, etcd encryption, API server serving certificates, and client authentication in kubeconfigs)  
> **Target Version**: Verified against v1.31 / v1.32 (Current CKA Curriculum)  
> **Allowed Docs Search Keywords**: `TLS`, `PKI certificates and requirements`, `certificates.k8s.io`, `CertificateSigningRequest`, `kubeadm certs`  
> **Source**: Generated from `security/01-TLS-basics-raw.md`

---

## 1. Quick-Reference Summary

- **Symmetric vs. Asymmetric Cryptography**:
  - **Symmetric Encryption**: Uses a single shared secret key for both encryption and decryption (e.g., AES-256, ChaCha20). Computationally fast and efficient; ideal for high-throughput data transfer. Suffers from the key distribution problem.
  - **Asymmetric Encryption**: Uses a mathematically linked keypair—a **Private Key** (kept confidential) and a **Public Key** (distributed freely) (e.g., RSA 2048/4096-bit, ECDSA P-256). Computationally expensive; used for identity authentication, digital signatures, and secure key exchange.
- **TLS Hybrid Encryption Architecture**:
  - TLS combines both mechanisms: asymmetric encryption or ephemeral Diffie-Hellman (ECDHE) securely authenticates the parties and negotiates an ephemeral **symmetric session key**; symmetric encryption is then used to encrypt the actual application payload.
- **X.509 Public Key Infrastructure (PKI) Core Objects**:
  - **Private Key (`.key`)**: Cryptographic secret kept by the host. Never transmitted over the network.
  - **Public Key (`.pub`)**: Mathematically derived from the private key. Embedded into CSRs and certificates.
  - **Certificate Signing Request (`.csr`)**: Data packet containing the applicant's public key, Subject identity (Common Name, Organization), optional Subject Alternative Names (SANs), and a cryptographic signature created with the private key (proof of possession).
  - **Digital Certificate (`.crt` / `.pem`)**: An X.509 structured document binding an identity to a public key, cryptographically signed by a trusted **Certificate Authority (CA)**.
- **Mutual TLS (mTLS) in Kubernetes**:
  - Unlike standard public web servers where only the server presents a certificate, **Kubernetes mandates mutual TLS (mTLS)** across all internal components.
  - Both the server and the client must present valid X.509 certificates issued by a mutually trusted CA before establishing an encrypted gRPC or HTTPS connection.
- **Encoding Formats**:
  - **PEM (`.pem`, `.crt`, `.key`)**: Base64-encoded ASCII with ASCII armoring (`-----BEGIN ...-----` and `-----END ...-----`). Standard format in Kubernetes.
  - **DER (`.der`)**: Binary ASN.1 format; less common in Kubernetes manifests.
- **Subject Alternative Names (SAN)**:
  - Modern TLS (and Kubernetes) rejects certificates that rely solely on the `Common Name` (CN) field. The certificate must define all DNS names and IP addresses used to reach the service inside the `subjectAltName` extension.

---

## 2. Conceptual Overview & Mental Model

### Dual-Layer Architectural Understanding

- **Simple English Explanation (How It Works)**:
  - If two computers on an untrusted network need to communicate secretly, they face a fundamental paradox: how do they agree on an encryption key without an eavesdropper intercepting that key?
  - Asymmetric encryption solves this. A server generates two keys: a private lock and thousands of copies of a public key. Anyone can take the public key to encrypt a secret message, but only the holder of the private key can unlock and read it.
  - However, an attacker could intercept the connection, pretend to be the server, and hand the client their own attacker public key (a Man-in-the-Middle attack).
  - To prevent this spoofing, we introduce a trusted third party called a **Certificate Authority (CA)**. The server submits its public key and domain identity in a request called a **CSR**. The CA verifies the server's identity and uses the CA's own private key to stamp a digital signature onto the certificate.
  - Because clients (browsers, `kubectl`, `kubelet`) have the CA's public certificate pre-installed in their trust store, they can mathematically verify that the server's certificate was signed by that legitimate authority.
  - In Kubernetes, security goes one step further: the server also demands a certificate from the client (**Mutual TLS / mTLS**). A worker node's `kubelet` or a cluster admin's `kubectl` must present their own client certificate signed by the Kubernetes cluster CA to prove their identity and role before the API server accepts any request.

- **Formal Cryptographic Definition**:
  - Transport Layer Security (TLS) is a cryptographic protocol designed to provide end-to-end communications security over computer networks through confidentiality, message integrity, and endpoint authentication. Mutual authentication is achieved using X.509 v3 public key certificates structured in Abstract Syntax Notation One (ASN.1), validated through an unbroken hierarchical chain of trust up to an authoritative root anchor. Digital signatures verify identity via cryptographic hash functions (SHA-256) encrypted with asymmetric private keys (RSA or ECDSA).

### Architectural Workflow Diagrams

#### 1. The PKI Certificate Lifecycle (Key $\to$ CSR $\to$ Cert)

```mermaid
flowchart TD
    subgraph Host["Target Host / Service (e.g., kube-apiserver)"]
        direction TB
        PrivKey["1. Generate Private Key<br/>(openssl genrsa -out server.key 2048)<br/>Confidential / File Permissions: 0600"]
        PubKey["2. Derive Public Key<br/>(Mathematical derivation)"]
        PrivKey --> PubKey
        CSR["3. Generate CSR (.csr)<br/>- Public Key<br/>- Subject: CN, Organization<br/>- Extensions: Subject Alternative Names (SANs)<br/>- Proof-of-Possession Signature (signed with server.key)"]
        PrivKey -->|Signs request| CSR
        PubKey -->|Embedded into| CSR
    end

    subgraph CA_Server["Certificate Authority (CA)"]
        direction TB
        CA_Priv["CA Private Key<br/>(ca.key - Strictly Protected)"]
        CA_Cert["CA Root Certificate<br/>(ca.crt - Public Trust Anchor)"]
        Verify["4. Verify CSR Identity and Proof-of-Possession"]
        Sign["5. Sign Certificate with CA Private Key<br/>(openssl x509 -req -CA ca.crt -CAkey ca.key)"]
        Verify --> Sign
        CA_Priv --> Sign
    end

    CSR -->|Submit CSR| Verify
    Sign -->|Issue Issued Certificate| X509Cert["6. X.509 Certificate (.crt / .pem)<br/>- Server Public Key<br/>- Subject and SANs<br/>- Validity Period<br/>- CA Cryptographic Signature"]
    X509Cert --> HostService["Service Deployment<br/>(Configured with server.crt and server.key)"]
```

#### 2. Mutual TLS (mTLS) Handshake Sequence in Kubernetes

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client (kubectl / kubelet)
    participant Server as Server (kube-apiserver)

    Note over Client,Server: Step 1: Protocol Negotiation
    Client->>Server: ClientHello (Supported TLS versions, cipher suites, random bytes)
    Server->>Client: ServerHello (Selected TLS version, selected cipher suite, server random)

    Note over Client,Server: Step 2: Server Authentication
    Server->>Client: Server Certificate (apiserver.crt containing Public Key and SANs)
    Server->>Client: CertificateRequest (Requests client certificate signed by Cluster CA)
    Server->>Client: ServerHelloDone

    Note over Client: Client verifies apiserver.crt using local ca.crt trust anchor
    Note over Client,Server: Step 3: Client Authentication (mTLS)
    Client->>Server: Client Certificate (client.crt containing user identity: CN, O)
    Client->>Server: ClientKeyExchange (Pre-master secret encrypted with server public key or ECDHE param)
    Client->>Server: CertificateVerify (Digital signature of handshake messages signed with client.key)

    Note over Server: Server verifies client.crt using CA and validates CertificateVerify signature
    Note over Client,Server: Step 4: Ephemeral Symmetric Key Derivation
    Client->>Client: Derive Symmetric Session Key
    Server->>Server: Derive Symmetric Session Key

    Client->>Server: ChangeCipherSpec and Finished (Encrypted)
    Server->>Client: ChangeCipherSpec and Finished (Encrypted)

    Note over Client,Server: Step 5: Secure High-Throughput Application Traffic
    Client->>Server: HTTPS REST API Requests (Encrypted with symmetric AES-GCM session key)
    Server->>Client: HTTPS REST API Responses (Encrypted with symmetric AES-GCM session key)
```

---

## 3. Deep-Dive Technical Breakdown

### 1. Symmetric vs. Asymmetric Encryption

Modern secure communications require two complementary cryptographic systems:

| Characteristic | Symmetric Encryption | Asymmetric Encryption |
| :--- | :--- | :--- |
| **Key Architecture** | Single shared secret key for encryption and decryption | Keypair: Public key (encrypt/verify) + Private key (decrypt/sign) |
| **Computational Overhead** | Extremely low CPU cycles; hardware acceleration available (AES-NI) | High CPU mathematical operations (modular exponentiation / elliptic curves) |
| **Primary Use Case** | Bulk payload encryption (TCP data stream, file storage) | Identity verification, digital signatures, key negotiation (TLS handshake) |
| **Key Distribution Problem** | High risk: Key must be shared in advance across an untrusted network | Zero risk: Public keys can be broadcast openly without compromising security |
| **Common Algorithms** | AES-128, AES-256 (GCM mode), ChaCha20-Poly1305 | RSA (2048/4096-bit), ECDSA (P-256, P-384), Ed25519 |

---

### 2. The Threat Vector: Man-in-the-Middle (MITM) & Spoofing

If a system uses raw asymmetric public keys without an identity verification mechanism, it remains vulnerable to MITM interception:

1. A client initiates a connection to `https://my-bank.com`.
2. An attacker on the local network (via ARP spoofing or rogue DNS) intercepts the packet.
3. The attacker generates their own keypair: `attacker.key` and `attacker.pub`.
4. The attacker returns `attacker.pub` to the client while establishing a separate TLS connection to `my-bank.com` using the bank's real public key.
5. The client encrypts the symmetric session key using `attacker.pub`.
6. The attacker decrypts the session key with `attacker.key`, reads the user's plaintext credentials, re-encrypts the payload with the bank's key, and forwards it to the bank.

#### The Solution: Digital Certificates
To stop this attack, the server must provide an **X.509 Digital Certificate** instead of a raw public key. The certificate binds the server's public key to its validated hostname (`CN=my-bank.com`) through a digital signature created by a trusted Certificate Authority.

---

### 3. Certificate Authorities (CAs) & The Chain of Trust

A **Certificate Authority (CA)** is an organization or internal server responsible for validating identities and issuing digitally signed certificates.

![Certificate Authority Architecture](../Images/certificate-authority.png)

#### How Trust Chains Work:
1. **Root CA**: Generates a self-signed certificate (`ca.crt`) using its own private key (`ca.key`).
2. **Public Trust Stores**: Operating systems and web browsers ship with pre-installed public Root CA certificates (e.g., DigiCert, Let's Encrypt, Sectigo) in `/etc/ssl/certs/` or OS keychains.
3. **Internal / Private CAs**: Organizations run private CAs (such as HashiCorp Vault, CFSSL, or Kubernetes internal CA) to issue certificates for internal microservices. The private CA's `ca.crt` must be explicitly distributed to all internal clients.
4. **Signature Verification**: When a client receives a certificate, it extracts the signature and decrypts it using the CA's public key stored in its trust store. If the decrypted hash matches the calculated hash of the certificate, the certificate is authentic.

---

### 4. Anatomy of an X.509 Digital Certificate

An X.509 v3 certificate is a structured ASN.1 document containing:

```plaintext
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 45:a1:8f:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes-ca, O = Kubernetes
        Validity:
            Not Before: Oct  6 12:00:00 2026 GMT
            Not After : Oct  6 12:00:00 2027 GMT
        Subject: CN = kube-apiserver, O = Kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus: ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Alternative Name: 
                DNS:kubernetes, DNS:kubernetes.default, DNS:controlplane, IP Address:10.96.0.1, IP Address:192.168.1.100
    Signature Algorithm: sha256WithRSAEncryption
         Signature Value: 8c:3d:e2:...
```

#### Key Fields in Kubernetes:
- **`Subject (CN)`**: Identifies the entity. In Kubernetes client certificates, the `CN` represents the **Username** (e.g., `CN=kubernetes-admin` or `CN=system:node:worker01`).
- **`Subject (O)`**: Organization. In Kubernetes client certificates, the `O` represents the **RBAC Group** (e.g., `O=system:masters` or `O=system:nodes`).
- **`Subject Alternative Name (SAN)`**: Lists all valid DNS hostnames and IP addresses for server certificates. If an administrator connects to `https://10.96.0.1:6443` and `10.96.0.1` is not in the SAN list, TLS validation fails.
- **`Proof-of-Possession`**: When generating a CSR, the client signs the CSR using its private key. This prevents an attacker from creating a certificate request using someone else's public key without possessing the corresponding private key.

---

### 5. Naming Conventions & Encoding Standards

![Certificate Naming Convention](../Images/certificate-naming-convention.png)

#### The Rule of Thumb:
- **Private Keys**: Almost always contain the word `key` in the file name or extension:
  - `server.key`, `server-key.pem`, `ca.key`, `id_rsa`
  - **Permissions**: Must be restricted strictly (e.g., `chmod 600` or `chmod 400`, owned by `root`).
- **Public Certificates**: Never contain the word `key`:
  - `server.crt`, `server.pem`, `ca.crt`, `ca.pem`, `id_rsa.pub`
  - Publicly readable (`chmod 644`).

#### Encoding Formats:
- **PEM (Privacy-Enhanced Mail)**:
  - ASCII text file containing base64 data enclosed between header and footer delimiters:
    ```plaintext
    -----BEGIN CERTIFICATE-----
    MIIDITCCAgmgAwIBAgII...
    -----END CERTIFICATE-----
    ```
  - Flexible container: can store private keys (`BEGIN RSA PRIVATE KEY`), certificates (`BEGIN CERTIFICATE`), CSRs (`BEGIN CERTIFICATE REQUEST`), or certificate bundles.
- **CRT / CER**:
  - A convention designating a certificate. Most `.crt` files in Linux and Kubernetes are PEM-encoded ASCII files.

---

## 4. Command Translation & Mapping Tables

### Table 1: Cryptographic Artifacts & File Extensions

| Artifact Name | Common File Extensions | Privacy Level | Purpose / Role in TLS |
| :--- | :--- | :--- | :--- |
| **Private Key** | `.key`, `.pem` | **Strictly Secret** (Never share) | Decrypts session keys; creates digital signatures and CSR proofs |
| **Public Key** | `.pub`, embedded in `.crt` | **Public** | Encrypts session keys; verifies digital signatures |
| **Certificate Signing Request** | `.csr`, `.pem` | Temporary / Public | Transports public key + identity to CA for validation and signature |
| **X.509 Certificate** | `.crt`, `.pem`, `.cer` | **Public** | Proves identity to peers; binds public key to domain/user |
| **Root CA Certificate** | `ca.crt`, `ca.pem` | **Public** (Trust Anchor) | Installed on clients to validate all certificates signed by that CA |
| **Certificate Bundle / Chain** | `.crt`, `.pem` | **Public** | Concatenation of server certificate + intermediate CA certificates |

---

### Table 2: One-Way Server TLS vs. Mutual TLS (mTLS)

| Property | Standard One-Way Server TLS | Kubernetes Mutual TLS (mTLS) |
| :--- | :--- | :--- |
| **Server Identity Validated?** | Yes (Client validates server's `.crt` against CA) | Yes (Client validates server's `.crt` against CA) |
| **Client Identity Validated?** | No (Client is anonymous or uses application passwords) | **Yes (Server validates client's `.crt` against CA)** |
| **Client Certificate Required?** | No | **Yes (e.g., `client.crt` / `client.key`)** |
| **Primary Domain of Use** | Public web browsing (e-commerce, blogs, portals) | Kubernetes control plane, etcd clusters, service meshes |
| **Protection Provided** | Prevents eavesdropping and server spoofing | Prevents eavesdropping, server spoofing, and unauthorized client access |

---

### Table 3: OpenSSL Inspection & Management Commands

| Task | OpenSSL Command Pattern | Purpose / Expected Output |
| :--- | :--- | :--- |
| **Inspect Certificate Details** | `openssl x509 -in cert.crt -text -noout` | Prints Subject, Issuer, Validity dates, and SAN extensions |
| **Inspect Expiration Dates Only** | `openssl x509 -in cert.crt -noout -dates` | Prints `notBefore` and `notAfter` timestamps |
| **Inspect CSR Details** | `openssl req -in request.csr -text -noout -verify` | Prints Subject, requested SANs, and verifies proof signature |
| **Inspect Private Key Validity** | `openssl rsa -in server.key -check` | Validates internal RSA mathematical consistency |
| **Verify Modulus Match (Key $\leftrightarrow$ Cert)** | `openssl x509 -noout -modulus -in cert.crt \| md5sum`<br/>`openssl rsa -noout -modulus -in server.key \| md5sum` | If MD5 hashes match, the private key belongs to the certificate |
| **Verify Cert against CA** | `openssl verify -CAfile ca.crt cert.crt` | Verifies certificate signature against local CA anchor |

---

## 5. High-Yield CLI & Imperative Commands

### 1. Generating a Private Key
Generate an industry-standard 2048-bit RSA private key:

```bash
# Generate private key
openssl genrsa -out server.key 2048

# Restrict file permissions immediately
chmod 600 server.key

# (Optional) Extract public key from private key
openssl rsa -in server.key -pubout -out server.pub
```

---

### 2. Creating a Private Certificate Authority (Root CA)
To act as a Certificate Authority, generate a private key and a self-signed root certificate:

```bash
# Step 1: Generate CA private key
openssl genrsa -out ca.key 2048
chmod 600 ca.key

# Step 2: Generate self-signed CA certificate (valid for 10 years / 3650 days)
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/CN=kubernetes-ca/O=Kubernetes" \
  -out ca.crt
```

---

### 3. Generating a Certificate Signing Request (CSR) with SANs
Modern Kubernetes clusters require Subject Alternative Names (SANs) for all server components:

```bash
# Step 1: Generate server private key
openssl genrsa -out kube-apiserver.key 2048
chmod 600 kube-apiserver.key

# Step 2: Create OpenSSL configuration with SAN extension
cat <<EOF > apiserver.cnf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
CN = kube-apiserver
O = Kubernetes

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = controlplane
IP.1 = 10.96.0.1
IP.2 = 192.168.1.10
IP.3 = 127.0.0.1
EOF

# Step 3: Generate CSR using configuration file
openssl req -new -key kube-apiserver.key -out kube-apiserver.csr -config apiserver.cnf
```

---

### 4. Signing the CSR with the CA (Issuing the Certificate)
The CA inspects and signs the CSR to generate the final X.509 certificate:

```bash
openssl x509 -req -in kube-apiserver.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out kube-apiserver.crt -days 365 -sha256 \
  -extfile apiserver.cnf -extensions v3_req
```

---

### 5. Generating a Client Certificate (for a Cluster User)
Client certificates do not require IP SANs, but their `CN` and `O` fields dictate identity in Kubernetes:

```bash
# Step 1: Generate user private key
openssl genrsa -out dev-user.key 2048

# Step 2: Generate CSR (CN=developer username, O=RBAC group)
openssl req -new -key dev-user.key -out dev-user.csr \
  -subj "/CN=dev-user/O=developers"

# Step 3: Sign with Kubernetes cluster CA
openssl x509 -req -in dev-user.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out dev-user.crt -days 365 -sha256
```

---

### 6. Verifying Certificates & Modulus Matching

```bash
# 1. Verify certificate chain against CA
openssl verify -CAfile ca.crt kube-apiserver.crt

# 2. Verify that kube-apiserver.key matches kube-apiserver.crt
CERT_MOD=$(openssl x509 -noout -modulus -in kube-apiserver.crt | md5sum)
KEY_MOD=$(openssl rsa -noout -modulus -in kube-apiserver.key | md5sum)

if [ "$CERT_MOD" = "$KEY_MOD" ]; then
    echo "SUCCESS: Key matches Certificate!"
else
    echo "ERROR: Modulus mismatch! Key does NOT belong to certificate!"
fi
```

---

## 6. Troubleshooting & Diagnostic Runbook

### Diagnostic Decision Tree

```mermaid
flowchart TD
    Error["TLS Connection or Validation Failure"] --> Q1{"What is the error message?"}

    Q1 -->|"x509: certificate signed by unknown authority"| CheckCA["CA mismatch: client does not trust the issuing CA.<br/>Check if client specifies --certificate-authority=ca.crt<br/>Ensure CA bundle contains the root CA."]
    
    Q1 -->|"x509: certificate is valid for X, not Y"| CheckSAN["SAN Mismatch:<br/>Inspect certificate SANs:<br/>openssl x509 -in cert.crt -text -noout | grep -A 2 'Alternative Name'<br/>Regenerate certificate with missing DNS or IP in SAN."]

    Q1 -->|"certificate has expired or is not yet valid"| CheckDates["Clock / Expiry issue:<br/>Check system clocks: date -u<br/>Check certificate lifetime:<br/>openssl x509 -in cert.crt -noout -dates<br/>Renew certificate via kubeadm certs renew or OpenSSL."]

    Q1 -->|"tls: bad certificate / bad handshake"| CheckClientCert["mTLS failure:<br/>Server rejected client certificate.<br/>Verify client cert is signed by the CA accepted by server.<br/>Verify server --client-ca-file matches."]

    Q1 -->|"tls: private key does not match public key"| CheckModulus["Key pair mismatch:<br/>Compare MD5 modulus hashes of .crt and .key.<br/>Verify correct private key is referenced in manifest."]
```

---

### Step-by-Step Triage Runbook

| Error Symptom | Root Cause | Diagnosis Command | Remediation Procedure |
| :--- | :--- | :--- | :--- |
| `x509: certificate signed by unknown authority` | The client trust store lacks the CA that signed the server's cert | `openssl verify -CAfile ca.crt server.crt` | Supply `--certificate-authority=ca.crt` or add the CA to system trust store. |
| `x509: certificate is valid for X, not Y` | Client reached the server via an IP or DNS name not listed in SAN | `openssl x509 -in server.crt -text -noout \| grep -A 1 "Subject Alternative Name"` | Re-issue the certificate including the target IP/hostname in `subjectAltName`. |
| `certificate has expired` | Certificate validity period (`notAfter`) has passed | `openssl x509 -in server.crt -noout -enddate` | In kubeadm clusters: `sudo kubeadm certs renew all`. In manual setups: re-sign CSR with CA. |
| `tls: private key does not match public key` | The `.key` path in the pod manifest belongs to a different certificate | Modulus check: `openssl x509 -modulus` vs `openssl rsa -modulus` | Update manifest or configuration to point to the matching private key. |
| `curl: (58) could not load PEM client certificate` | Permission denied reading client `.key` or `.crt` | `ls -la client.key` | Verify file read permissions (`chmod 600 dev-user.key` and readable by user). |

---

## 7. CKA Exam Tips, Gotchas & Traps

> [!CAUTION]
> **Trap 1: The Missing SAN Trap**  
> In modern Kubernetes versions (v1.19+), Go strictly enforces SAN validation and ignores the Common Name (`CN`) field for hostname verification. If you generate a certificate for `kube-apiserver` without including `IP:10.96.0.1` and `DNS:kubernetes` in the Subject Alternative Name extension, `kubelet` and internal cluster pods will fail to connect with `x509: certificate is valid for ..., not 10.96.0.1`.

> [!IMPORTANT]
> **Trap 2: Never Transmit Private Keys**  
> In multi-node troubleshooting questions, remember:
> - Private keys (`.key`) **never leave the host that generated them**.
> - Only public keys, CSRs (`.csr`), and issued certificates (`.crt`) are transmitted across nodes or submitted to the Kubernetes API.

> [!WARNING]
> **Trap 3: Identifying Keys vs. Certificates by Content, Not Just Name**  
> In exam environments, files may be named ambiguously (e.g. `auth.pem`). Inspect the header:
> - `-----BEGIN RSA PRIVATE KEY-----` or `-----BEGIN PRIVATE KEY-----` $\to$ **Private Key**
> - `-----BEGIN CERTIFICATE-----` $\to$ **Public Certificate**
> - `-----BEGIN CERTIFICATE REQUEST-----` $\to$ **CSR**

> [!TIP]
> **Exam Tip 4: Fast Certificate Expiration Check**  
> On kubeadm control plane nodes, do not manually run `openssl` on dozens of files to check for expiration. Run the built-in diagnostic command:
> ```bash
> sudo kubeadm certs check-expiration
> ```

> [!NOTE]
> **Trap 5: Client Certificate RBAC Mapping**  
> In Kubernetes, there is no `User` resource in the API. User identity is derived strictly from the client certificate fields:
> - `Subject: CN = jane` $\implies$ Kubernetes User `jane`
> - `Subject: O = system:masters` $\implies$ Kubernetes Group `system:masters` (superadmin privilege)  
> If an exam question asks to create credentials for user `john` in group `devs`, the CSR Subject **must be** `/CN=john/O=devs`.

---

## 8. Self-Test / Active Recall

<details>
<summary><strong>1. Why can you NOT create or request a digital certificate using only a public key?</strong></summary>

Because generating a Certificate Signing Request (CSR) requires a cryptographic signature created by the corresponding private key (proof of possession). Without the private key, anyone could steal a public key and request a valid certificate in someone else's identity.
</details>

<details>
<summary><strong>2. What is the fundamental difference between one-way TLS and mutual TLS (mTLS)?</strong></summary>

In one-way TLS, only the server presents a certificate to authenticate its identity to the client. In mutual TLS (mTLS), **both** the server and the client present certificates signed by trusted CAs, authenticating both endpoints to each other.
</details>

<details>
<summary><strong>3. How does Kubernetes derive a user's name and group membership from a client TLS certificate?</strong></summary>

The API server parses the certificate's X.509 Subject field:
- The Common Name (`CN`) is interpreted as the **Username**.
- The Organization (`O`) fields are interpreted as the **Groups**.
</details>

<details>
<summary><strong>4. What command verifies whether a private key matches a given certificate?</strong></summary>

Compare the MD5 hashes of their moduli:
```bash
openssl x509 -noout -modulus -in server.crt | md5sum
openssl rsa -noout -modulus -in server.key | md5sum
```
If both hashes match, the key and certificate form a valid pair.
</details>

<details>
<summary><strong>5. Why is the Subject Alternative Name (SAN) extension mandatory for modern Kubernetes server certificates?</strong></summary>

Because modern TLS implementations (including Go's `crypto/tls` library used by Kubernetes) ignore the Common Name (`CN`) field for hostname verification and strictly validate requests against the `subjectAltName` extension (DNS names and IP addresses).
</details>

<details>
<summary><strong>6. What is the difference between a <code>.pem</code> file and a <code>.crt</code> file?</strong></summary>

`PEM` is an ASCII encoding standard using base64 with `-----BEGIN ...-----` delimiters that can store certificates, private keys, or CSRs. `.crt` is a file naming convention commonly used specifically for certificates (which are almost always PEM-encoded in Linux/Kubernetes).
</details>

<details>
<summary><strong>7. What role does symmetric encryption play during a TLS session, given that asymmetric keys are used during the handshake?</strong></summary>

Asymmetric encryption is computationally slow and is only used during the initial handshake to authenticate identity and establish an ephemeral symmetric session key. Once established, high-speed symmetric ciphers (e.g., AES-GCM) encrypt all subsequent application payload traffic.
</details>

<details>
<summary><strong>8. How can you inspect the human-readable text of an X.509 certificate without using graphical tools?</strong></summary>

```bash
openssl x509 -in <certificate-file> -text -noout
```
</details>

<details>
<summary><strong>9. In a standard kubeadm cluster, where are the cluster root CA certificate and private key stored?</strong></summary>

At `/etc/kubernetes/pki/ca.crt` (public certificate) and `/etc/kubernetes/pki/ca.key` (private key).
</details>

<details>
<summary><strong>10. If an administrator connects to the API server using <code>https://192.168.1.50:6443</code> and receives <code>x509: certificate is valid for 10.96.0.1, not 192.168.1.50</code>, what is the root cause?</strong></summary>

The IP address `192.168.1.50` was omitted from the `Subject Alternative Name` (SAN) extension when the API server serving certificate was generated and signed.
</details>

---

## 9. Official Documentation Bookmarks

| Documentation Page | Official URL | Search Keywords | High-Yield Section for Exam |
| :--- | :--- | :--- | :--- |
| **PKI certificates and requirements** | `https://kubernetes.io/docs/setup/best-practices/certificates/` | `PKI certificates and requirements` | Full list of all certificates and CA requirements in Kubernetes |
| **Manage TLS Certificates in a Cluster** | `https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/` | `managing tls in a cluster`, `CertificateSigningRequest` | Creating and approving `CertificateSigningRequest` objects |
| **Certificate Management with kubeadm** | `https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/` | `kubeadm certs`, `check-expiration` | Checking and renewing certificates with `kubeadm certs` |
| **Authenticating** | `https://kubernetes.io/docs/reference/access-authn-authz/authentication/` | `X509 Client Certificates`, `authenticating` | X.509 client certificate CN and O mappings to Users and Groups |
