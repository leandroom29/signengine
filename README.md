# SignEngine

SignEngine is a secure PDF signing and document-exchange project built around a Java desktop client, a transport/security protocol layer, and a Spring Boot document server. The system is designed to let a user choose a certificate, sign a PDF locally, upload the signed result to a remote service, and verify the document on the server side before accepting it.

At a high level, the project combines:

- a JavaFX client for end-user interaction,
- a custom encrypted TCP protocol for trusted communication,
- a server-side PDF verifier and cryptographic layer,
- key handling and secure storage for RSA/AES operations,
- document management for listing, downloading, and storing signed files.

---

## Project overview

This repository contains the following main components:

- `client/`: desktop JavaFX client that manages certificate selection, PDF signing, and remote document exchange
- `protocol/`: shared protocol and encryption layer used by the client and server
- `server/`: Spring Boot application that exposes a TCP document service and verifies signed PDFs
- `keys/`: RSA key material used by the server for secure communication
- `documents/`: repository of document files managed by the server

The application is versioned as `1.0.0` and is organized as a multi-module Maven project centered around the parent artifact `com.leandroom29:signengine`.

---

## Architecture

### 1) Client module

The client module builds a JavaFX desktop application (`ClientApplication`) and exposes the UI (`ClientView`).

Key responsibilities:

- open a connection to the server,
- choose a local certificate and password,
- inspect certificate metadata,
- sign PDF documents with a visible or embedded signature,
- download document listings from the server,
- upload signed files back to the server.

Relevant classes from the compiled artifact:

- `com.leandroom29.signengine.client.ClientApplication`
- `com.leandroom29.signengine.client.ClientView`
- `com.leandroom29.signengine.client.PdfSigner`
- `com.leandroom29.signengine.client.TcpClientService`
- `com.leandroom29.signengine.client.CryptoUtils`

The client uses JavaFX controls for UI layout and PDFBox for local PDF processing. It also depends on the protocol module and security libraries such as Bouncy Castle.

### 2) Protocol module

The protocol module implements the secure channel used for client/server communication.

Core class:

- `com.leandroom29.signengine.protocol.SecureChannel`

This class provides:

- session establishment with RSA key exchange,
- symmetric AES encryption for message payloads,
- per-message nonce handling,
- sequence tracking for integrity and replay protection,
- secure read/write methods for encrypted payloads.

From the compiled code, it uses:

- RSA-OAEP with SHA-256 for the key transport step,
- AES-GCM for payload encryption,
- secure random generation,
- message framing through `DataInputStream` / `DataOutputStream` wrappers.

This gives the application a lightweight encrypted transport layer without requiring a full HTTPS/WebSocket stack.

### 3) Server module

The server is a Spring Boot application (`ServerApplication`) configured with `spring.main.web-application-type=none`, so it runs as a non-web console service while still using the Spring Boot lifecycle.

Core server classes:

- `com.leandroom29.signengine.server.ServerApplication`
- `com.leandroom29.signengine.server.TcpServerRunner`
- `com.leandroom29.signengine.server.TcpDocumentServer`
- `com.leandroom29.signengine.server.DocumentService`
- `com.leandroom29.signengine.server.SecurityCryptoService`
- `com.leandroom29.signengine.server.PdfSignatureVerifier`
- `com.leandroom29.signengine.server.StartupDocumentPrompt`

Key responsibilities:

- bind a TCP server on a configurable address and port,
- manage document storage in `documents/` plus a secure storage directory,
- generate or load RSA key pairs from the `keys/` directory,
- list and download authorized documents,
- receive signed PDF payloads,
- verify PDF signatures before accepting them.

---

## Runtime flow

### 1) Server startup

When the server boots, it loads the configured properties:

- `signengine.documents-dir=documents`
- `signengine.secure-dir=secure-storage`
- `signengine.rsa-key-dir=keys`
- `signengine.tcp.bind-address=127.0.0.1`
- `signengine.tcp.port=9090`

The server starts the TCP listener and initializes document and cryptographic services.

### 2) Secure channel setup

The client and server use the `SecureChannel` abstraction to establish an encrypted communication channel. The protocol layer:

- creates a session key,
- exchanges it using the remote public key,
- encrypts outgoing messages with AES-GCM,
- verifies incoming messages using the matching sequence state.

This protects document transfer between the JavaFX client and the server.

### 3) Document listing and download

The `TcpDocumentServer` can handle operations such as:

- listing available documents,
- downloading a document by name,
- accepting signed files from the client.

The `DocumentService` uses safe path validation before reading or writing files. It also computes SHA-256 hashes to track document integrity.

### 4) PDF signing

The client module signs documents using `PdfSigner`, which relies on PDFBox and cryptographic support from Bouncy Castle.

The signing flow includes:

- loading the signer certificate and private key,
- preparing the PDF for signing,
- adding a visual signature layer,
- generating a PDF/A-like signature structure,
- preserving document validity and metadata.

### 5) Server-side verification

The `PdfSignatureVerifier` checks uploaded content for validity. This is a critical trust boundary: the server does not blindly accept signed PDFs; it validates the signature before storing or accepting them as final artifacts.

---

## Security model

The project places the strongest emphasis on security and integrity in the document flow.

### Encryption used

- RSA key generation and exchange using OAEP with SHA-256
- AES symmetric encryption for session traffic
- GCM-style authenticated encryption for payload confidentiality and integrity
- per-message sequence numbers to reduce replay risks

### Key material

The repository includes:

- `keys/server-private.der`
- `keys/server-public.der`

These are used by the server-side `SecurityCryptoService` to create or read the private/public keypair needed for secure operations.

### Document protection

The server stores documents in a controlled folder and uses sanitized file paths to avoid path traversal attacks or arbitrary file exposure.

---

## Technical stack

### Java ecosystem

- Java 21+ (the project build artifacts reference the JDK toolchain)
- JavaFX desktop UI
- Spring Boot 3.x runtime

### Libraries

- Apache PDFBox for PDF parsing, manipulation, and signing
- Bouncy Castle (`bcprov`, `bcpkix`) for cryptographic support
- JUnit 5 for protocol and server-side test coverage

### Maven modules

The generated metadata indicates a parent artifact:

- `com.leandroom29:signengine` version `1.0.0`

Child modules:

- `signengine-protocol`
- `signengine-client`
- `signengine-server`

---

## Configuration

The server configuration is defined in `application.properties` inside the Spring Boot jar:

```properties
spring.main.web-application-type=none
signengine.documents-dir=documents
signengine.secure-dir=secure-storage
signengine.rsa-key-dir=keys
signengine.tcp.bind-address=${SIGNENGINE_TCP_BIND:127.0.0.1}
signengine.tcp.port=${SIGNENGINE_TCP_PORT:9090}
```

This makes the server easy to run locally and also allows environment-variable overrides when deployed.

---

## Build and run

The project is organized as a Maven multi-module build, so the general workflow is:

```bash
mvn clean package
```

Then run the server and client as separate Java applications.

### Start the server

The server is a Spring Boot executable jar and is intended to run as a service process:

```bash
java -jar server/target/signengine-server-1.0.0.jar
```

### Run the client

The client is a JavaFX desktop application:

```bash
java --module-path <path-to-javafx-libs> --add-modules javafx.controls,javafx.graphics -jar client/target/signengine-client-1.0.0.jar
```

The exact JavaFX module path depends on your local environment and installed JavaFX distribution.

---

## Testing

The unit and integration test reports confirm the project is functioning in the expected core areas:

- `SecureChannelTest`: 1 test, 0 failures, 0 errors
- `SecurityCryptoServiceTest`: 2 tests, 0 failures, 0 errors
- `PdfSignatureVerifierTest`: 2 tests, 0 failures, 0 errors

These tests cover the secure channel and cryptographic validation behavior, which are central to the trust model of the solution.

---

## Typical usage scenario

1. Start the server.
2. Open the client UI.
3. Choose the certificate to use for signing.
4. Select a PDF from the local filesystem.
5. Enter signing metadata such as signer name, location, and reason.
6. Sign the document locally with the PDF signer.
7. Upload the signed PDF to the server.
8. The server verifies the file and stores it in the managed document area.

---

## Security notes and recommendations

This project is well-suited for controlled internal or demo environments, but a production-grade deployment should also include:

- certificate pinning or trust validation for the client/server relationship,
- stronger key lifecycle management and rotation,
- TLS termination or secure network segmentation for remote deployments,
- hardened authentication and authorization before document operations,
- audit logging for every document upload, verification, and access event,
- secure storage for private keys outside the repository or under a protected keystore.

---

## License

This project is distributed under the MIT license as described in the repository’s `LICENSE` file.

---

## Summary

SignEngine is a compact but sophisticated PDF signing ecosystem: a JavaFX client signs documents, a custom encrypted protocol secures communication, and a Spring Boot server verifies and stores authorized signed PDFs. It demonstrates practical use of Java cryptography, document processing, and secure file transfer in a single, cohesive application.
