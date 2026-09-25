# Securing the Student Records API with HTTPS

**Course:** ITEC 111A - Integrated Programming and Technologies 2  
**Topic:** Secure Connections  
**Base project:** previous activity Student Records API using Node.js and Express  
**Target outcome:** Run the existing API through HTTPS using a locally generated TLS certificate

## Overview

In previous activity, you created a Student Records API with these operations:

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/students` | Create a student |
| `GET` | `/students` | List students |
| `GET` | `/students/:studentId` | Retrieve one student |
| `PATCH` | `/students/:studentId` | Partially update a student |
| `DELETE` | `/students/:studentId` | Delete a student |

In previous activity, you connected a React/Vite client to that API. This guide secures the existing connection without replacing either project. You will generate a self-signed development certificate, configure Node.js to create an HTTPS server, protect the private key from accidental version-control exposure, update the OpenAPI server URL and React API base URL, and test the secured integration.

> [!IMPORTANT]
> This activity protects data **in transit**. It does not add user authentication or authorization to the Student Records API. API authentication is addressed separately under API Security.

## Learning Objectives

After completing this guide, you should be able to:

1. Explain what HTTPS adds to an HTTP API.
2. Generate and inspect a self-signed development certificate.
3. Distinguish a public certificate from its private key.
4. Configure an Express application to run through Node's HTTPS server.
5. test the certificate, TLS connection, and existing Student API operations.
6. Explain why a self-signed certificate produces a trust warning.
7. Keep private keys and local secrets out of version control.
8. Reconnect the completed React/Vite client app to the HTTPS API.

## Prerequisites

Before beginning, confirm that you have:

- Completed the Previous Activity: Student Records API
- Node.js and npm
- OpenSSL
- A terminal and code editor
- A browser and `curl`, Postman, Bruno, or a similar API client

Verify the required programs:

```bash
node --version
npm --version
openssl version
```

Your previous activity project should contain at least:

```text
project-folder/
├── student-api/
│   ├── package.json
│   └── src/
│       ├── app.js
│       ├── openapi.js
│       ├── errors/
│       ├── middleware/
│       ├── routes/
│       └── validators/
└── student-client/
    ├── package.json
    └── src/
        ├── api/
        ├── components/
        └── App.jsx
```

## Security Model for This Activity

The resulting connection will use:

- **HTTPS** to carry HTTP messages through TLS.
- A **self-signed certificate** representing `localhost`.
- A **private key** retained by the local API server.
- A **public certificate** presented to the client.
- TLS session keys negotiated by the client and server to protect application data.

Because the certificate is self-signed, it will not automatically chain to a public certificate authority trusted by the browser or operating system. The warning is expected in this laboratory. Do not treat bypassing that warning as an acceptable production practice.

---

## Step 1 - Make a Backup or Commit the Working Integration

Before modifying the project, confirm that the  API still works:

```bash
npm run dev
```

In a second terminal:

```bash
curl -i http://localhost:3000/students
```

Stop the server after confirming a `200 OK` response.

Before stopping it, start the frontend app client in another terminal:

```bash
cd student-client
npm run dev
```

Open <http://localhost:5173> and confirm that the client loads the student list from `http://localhost:3000`. Record this as the integration baseline, then stop both applications.

If the project uses Git, create a commit before continuing. Do not include `node_modules`.

### Checkpoint

- [ ] The original HTTP API starts without errors.
- [ ] `GET /students` returns `200 OK`.
- [ ] You can return to the working  version if necessary.

---

## Step 2 - Create a Directory for Local TLS Material

From the root of `student-api`, create a local certificate directory:

```bash
mkdir certs
```

The finished project will include two local files:

```text
student-api/
├── certs/
│   ├── localhost-cert.pem
│   └── localhost-key.pem
├── package.json
└── src/
    └── ...
```

| File | Purpose | May be shared publicly? |
|---|---|:---:|
| `localhost-cert.pem` | Public certificate presented by the server | Yes, although this lab copy need not be submitted |
| `localhost-key.pem` | Private key proving control of the certificate identity | **No** |

---

## Step 3 - Prevent Accidental Key Exposure

Create or update `.gitignore` in the project root:

```gitignore
node_modules/
certs/*.pem
.env
*.key
```

This prevents new matching files from being staged automatically. It does not remove a private key that was already committed.

Check the repository state:

```bash
git status
```

If `localhost-key.pem` was already committed at any point, assume it was exposed. Remove it from active use and generate a new key and certificate. Simply deleting the file or adding it to `.gitignore` is not sufficient remediation.

### Checkpoint

- [ ] `.gitignore` excludes PEM files in `certs`.
- [ ] No private key appears as a file ready to commit.

---

## Step 4 - Generate a Self-Signed Certificate

Run this command from the project root:

```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout certs/localhost-key.pem \
  -out certs/localhost-cert.pem \
  -sha256 -days 30 -nodes \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

### Meaning of the options

| Option | Meaning |
|---|---|
| `req` | Uses OpenSSL's certificate-request command |
| `-x509` | Outputs a self-signed certificate instead of only a certificate signing request |
| `-newkey rsa:2048` | Generates a new 2048-bit RSA key pair |
| `-keyout` | Selects the private-key output file |
| `-out` | Selects the certificate output file |
| `-sha256` | Uses SHA-256 when signing the certificate |
| `-days 30` | Makes the disposable certificate valid for 30 days |
| `-nodes` | Leaves the private key unencrypted so the development server can start without a passphrase prompt |
| `-subj` | Supplies the certificate subject non-interactively |
| `-addext` | Adds identities used for hostname verification |

> [!CAUTION]
> An unencrypted private key is convenient for a local laboratory server, but anyone who can read the file can use it. Limit access to the file and never use this laboratory key in production.

If your OpenSSL build does not support `-addext`, consult the installed OpenSSL documentation and use a configuration file that defines the same Subject Alternative Name values.

---

## Step 5 - Inspect the Certificate

Display the important fields:

```bash
openssl x509 \
  -in certs/localhost-cert.pem \
  -noout \
  -subject \
  -issuer \
  -dates \
  -serial \
  -fingerprint \
  -sha256 \
  -ext subjectAltName
```

Expected observations:

- The subject contains `CN = localhost`.
- The issuer is the same as the subject because the certificate is self-signed.
- The validity period is approximately 30 days.
- Subject Alternative Name contains `DNS:localhost` and `IP Address:127.0.0.1`.
- A SHA-256 fingerprint is displayed.

Optional full inspection:

```bash
openssl x509 -in certs/localhost-cert.pem -noout -text
```

### Checkpoint

- [ ] Subject and issuer are visible.
- [ ] The certificate has not expired.
- [ ] `localhost` appears in Subject Alternative Name.
- [ ] You recorded the SHA-256 fingerprint for later comparison.

---

## Step 6 - Confirm That the Certificate Matches the Private Key

The following commands calculate a digest of the public-key material from each file:

```bash
openssl x509 -in certs/localhost-cert.pem -noout -pubkey \
  | openssl pkey -pubin -outform der \
  | openssl dgst -sha256
```

```bash
openssl pkey -in certs/localhost-key.pem -pubout -outform der \
  | openssl dgst -sha256
```

The two digest values should match. Matching values show that the certificate contains the public key corresponding to the generated private key.

---

## Step 7 - Configure the Express Application for HTTPS

Open `src/app.js`.

Add these imports near the top:

```javascript
import { readFileSync } from 'node:fs';
import { createServer } from 'node:https';
import { resolve } from 'node:path';
```

Keep the existing Express-related imports. After `const app = express();`, replace the fixed port declaration with:

```javascript
const port = Number(process.env.PORT ?? 3443);

const tlsKeyPath = resolve(
  process.env.TLS_KEY_PATH ?? 'certs/localhost-key.pem'
);

const tlsCertPath = resolve(
  process.env.TLS_CERT_PATH ?? 'certs/localhost-cert.pem'
);

const tlsOptions = {
  key: readFileSync(tlsKeyPath),
  cert: readFileSync(tlsCertPath),
  minVersion: 'TLSv1.2'
};
```

At the bottom of the file, replace the existing `app.listen(...)` block with:

```javascript
const httpsServer = createServer(tlsOptions, app);

httpsServer.listen(port, () => {
  const baseUrl = `https://localhost:${port}`;

  console.log(`Secure API: ${baseUrl}`);
  console.log(`OpenAPI JSON: ${baseUrl}/openapi.json`);
  console.log(`Swagger UI: ${baseUrl}/api-docs`);
  console.log(`Scalar: ${baseUrl}/reference`);
});
```

### Complete HTTPS-related structure of `src/app.js`

The important structure should now resemble:

```javascript
import { readFileSync } from 'node:fs';
import { createServer } from 'node:https';
import { resolve } from 'node:path';
import express from 'express';
import swaggerUi from 'swagger-ui-express';
import { apiReference } from '@scalar/express-api-reference';
import { errorHandler, notFoundHandler } from './middleware/error-handlers.js';
import { openapiSpecification } from './openapi.js';
import { studentsRouter } from './routes/students.js';

const app = express();
const port = Number(process.env.PORT ?? 3443);

const tlsKeyPath = resolve(
  process.env.TLS_KEY_PATH ?? 'certs/localhost-key.pem'
);

const tlsCertPath = resolve(
  process.env.TLS_CERT_PATH ?? 'certs/localhost-cert.pem'
);

const tlsOptions = {
  key: readFileSync(tlsKeyPath),
  cert: readFileSync(tlsCertPath),
  minVersion: 'TLSv1.2'
};

app.disable('x-powered-by');
app.use(express.json());

// Keep the existing home route, documentation routes,
// student routes, not-found handler, and error handler here.

const httpsServer = createServer(tlsOptions, app);

httpsServer.listen(port, () => {
  const baseUrl = `https://localhost:${port}`;

  console.log(`Secure API: ${baseUrl}`);
  console.log(`OpenAPI JSON: ${baseUrl}/openapi.json`);
  console.log(`Swagger UI: ${baseUrl}/api-docs`);
  console.log(`Scalar: ${baseUrl}/reference`);
});
```

### Why use environment variables for paths?

The source code contains file locations, not certificate or key contents. A deployment can supply different paths without editing the application. The development defaults continue to work from the project root.

---

## Step 8 - Update the OpenAPI Server URL

Open `src/openapi.js`. Locate the `servers` array and change the local server entry to:

```javascript
servers: [
  {
    url: 'https://localhost:3443',
    description: 'Local HTTPS development server'
  }
],
```

This change tells Swagger UI, Scalar, generated clients, and readers that the documented API is served through HTTPS.

> [!NOTE]
> If you choose a different HTTPS port, update both the runtime configuration and the OpenAPI URL.

---

## Step 9 - Start the HTTPS API

From the project root:

```bash
npm run dev
```

Expected terminal output includes:

```text
Secure API: https://localhost:3443
OpenAPI JSON: https://localhost:3443/openapi.json
Swagger UI: https://localhost:3443/api-docs
Scalar: https://localhost:3443/reference
```

If Node reports `ENOENT`, confirm that you started the command from the project root and that both PEM files exist in `certs`.

---

## Step 10 - Observe Strict Certificate Verification

In another terminal, run:

```bash
curl -i https://localhost:3443/students
```

The request should fail certificate verification because the self-signed certificate is not in the client's default trust store. This is an expected and useful result.

Read the error message. It should indicate an untrusted or self-signed certificate rather than a failed TCP connection.

### Security lesson

The server is offering encryption, but the client does not yet have a trusted basis for authenticating the server. Silently accepting any certificate would allow an attacker to substitute another certificate.

---

## Step 11 - Test by Explicitly Trusting the Lab Certificate

Tell `curl` to use this specific certificate as a trusted CA file for the request:

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  https://localhost:3443/students
```

Expected status: `200 OK`.

This is preferable to disabling verification because `curl` still performs certificate and hostname checks using the certificate you deliberately supplied.

### Diagnostic comparison only

The following command disables certificate verification:

```bash
curl -k -i https://localhost:3443/students
```

Use `-k` or `--insecure` only to demonstrate the difference during this controlled activity. It must not be the submitted solution or a production configuration.

---

## Step 12 - Retest the Student CRUD Operations over HTTPS

### List students

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  https://localhost:3443/students
```

Expected status: `200 OK`.

### Retrieve one student

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  https://localhost:3443/students/1
```

Expected status: `200 OK`.

### Create a student

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"student_number":"2026-00801","full_name":"Sam Rivera","email":"sam@example.edu","year_level":2,"active":true}' \
  https://localhost:3443/students
```

Expected status: `201 Created` with a `Location` header.

Record the new student's `id` from the response body or the final path segment of the `Location` header. The remaining update and delete examples refer to this value as `NEW_STUDENT_ID`. In the shell, assign the actual numeric value returned by your API:

```bash
NEW_STUDENT_ID=3
```

Replace `3` when the API returns a different ID. Do not use the seeded IDs `1` or `2` because next activity links demonstration accounts to those records.

### Update a student

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -X PATCH \
  -H 'Content-Type: application/json' \
  -d '{"year_level":3}' \
  "https://localhost:3443/students/$NEW_STUDENT_ID"
```

Expected status: `200 OK`.

### Delete a student

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -X DELETE \
  "https://localhost:3443/students/$NEW_STUDENT_ID"
```

Expected status: `204 No Content`.

Confirm that the seeded records remain available:

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  https://localhost:3443/students/1

curl -i \
  --cacert certs/localhost-cert.pem \
  https://localhost:3443/students/2
```

Both requests should return `200 OK`.

The HTTP methods, validation rules, status codes, and JSON representations should behave as they did in previous API activity. TLS changes how messages are transported, not the intended Student API contract.

---

## Step 13 - Inspect the Live TLS Connection

Use OpenSSL's TLS client:

```bash
openssl s_client \
  -connect localhost:3443 \
  -servername localhost \
  -CAfile certs/localhost-cert.pem
```

Look for:

- The certificate subject and issuer
- The negotiated TLS protocol version
- The negotiated cipher suite
- `Verify return code: 0 (ok)` when the supplied certificate is accepted as the trust anchor

Press `Ctrl+C` to exit.

To print only the live certificate for comparison:

```bash
openssl s_client \
  -connect localhost:3443 \
  -servername localhost \
  -showcerts </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -fingerprint -sha256
```

Confirm that the live fingerprint matches the fingerprint recorded in Step 5.

---

## Step 14 - Inspect the API in a Browser

Open:

- <https://localhost:3443/>
- <https://localhost:3443/openapi.json>
- <https://localhost:3443/api-docs>
- <https://localhost:3443/reference>

The browser will normally warn that the certificate is not trusted. Inspect the certificate details and verify:

- Subject or identity includes `localhost`.
- Subject Alternative Name includes `localhost`.
- The validity dates match the certificate generated for the activity.
- The fingerprint matches the value recorded in Step 5.

If the instructor permits continuing past the warning for this laboratory, do so only after checking the certificate details. Do not install the disposable certificate as a system-wide trusted root unless explicitly instructed and unless you understand how to remove that trust afterward.

---

## Step 15 - Reconnect the React Client

In `student-client`, create or update `.env.local`:

```dotenv
VITE_API_BASE_URL=https://localhost:3443
```

Update `src/api/students.js` so the base URL comes from Vite configuration rather than a fixed HTTP address:

```javascript
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;

if (!API_BASE_URL) {
  throw new Error('VITE_API_BASE_URL is not configured');
}
```

Keep the API CORS allowlist configured for the origin where the React page itself runs:

```javascript
app.use(cors({
  origin: 'http://localhost:5173'
}));
```

The client origin remains HTTP unless Vite is configured separately for HTTPS. Its requests may still target the HTTPS API. Before testing `fetch()`, open <https://localhost:3443> directly and follow the instructor-approved process for trusting or continuing past the disposable development certificate warning.

Restart Vite after changing `.env.local`:

```bash
cd student-client
npm run dev
```

Open <http://localhost:5173> and confirm that the existing list and create operations now use `https://localhost:3443`. Inspect the browser Network panel and verify that requests no longer target `http://localhost:3000`.

Do not solve certificate failures by disabling browser security. Establish deliberate development trust or use the instructor-approved local certificate procedure.

---

## Step 16 - Verify Version-Control Safety

Run:

```bash
git status --short
```

The `.pem` files should not appear as untracked or staged files.

If the project has Git history, also check whether PEM or key files are tracked:

```bash
git ls-files
```

Review the output. Neither `certs/localhost-key.pem` nor another private-key file should appear.

Do not submit the private key. Your evidence may show certificate metadata and fingerprints, but it must not show the private-key contents.

---

## Completion Checklist

- [ ] The original  Student API behavior remains functional.
- [ ] A self-signed certificate exists for `localhost`.
- [ ] Subject Alternative Name includes `DNS:localhost`.
- [ ] The certificate and private key match.
- [ ] PEM and key files are ignored by Git.
- [ ] The API starts at `https://localhost:3443`.
- [ ] OpenAPI declares the HTTPS server URL.
- [ ] Strict `curl` verification initially rejects the untrusted certificate.
- [ ] `curl --cacert` verifies the certificate and returns `200 OK`.
- [ ] Student CRUD requests work through HTTPS.
- [ ] The React client uses `VITE_API_BASE_URL=https://localhost:3443`.
- [ ] The React client loads and creates records through the HTTPS API.
- [ ] The live certificate fingerprint matches the local certificate.
- [ ] The private key is not included in the submission.

## Troubleshooting

### `openssl: command not found`

Install OpenSSL using the method approved for the laboratory computer, or use the instructor-provided environment.

### `unknown option -addext`

Your OpenSSL version may be old. Use a configuration file with a Subject Alternative Name section or update OpenSSL using the approved installation method.

### Node reports `ENOENT`

The application cannot find a certificate or private-key file. Start Node from the project root, verify both paths, or set `TLS_KEY_PATH` and `TLS_CERT_PATH` to valid locations.

### Node reports a PEM or key error

Regenerate the files. Confirm that the certificate was not placed in the key path and that the key was not placed in the certificate path.

### `curl` reports a self-signed certificate error

This is expected before trust is configured. Use the activity's `--cacert certs/localhost-cert.pem` option and keep hostname verification enabled.

### `curl` reports a hostname mismatch

Connect using `https://localhost:3443`, and confirm that Subject Alternative Name contains `DNS:localhost`. If connecting through `127.0.0.1`, confirm that `IP:127.0.0.1` is also present.

### Swagger UI sends requests to HTTP or port 3000

Update the `servers` entry in `src/openapi.js` to `https://localhost:3443`, then restart the development server and reload the documentation.

### Port 3443 is already in use

Stop the process using the port or choose another port. Keep the application port and OpenAPI server URL consistent.

## References

### Course references

- Node.js, [`https.createServer()`](https://nodejs.org/api/https.html#httpscreateserveroptions-requestlistener)
- Node.js, [TLS documentation](https://nodejs.org/api/tls.html)
- OpenSSL, [`openssl-req`](https://docs.openssl.org/3.5/man1/openssl-req/)
- OpenSSL, [`openssl-x509`](https://docs.openssl.org/3.5/man1/openssl-x509/)
- OpenSSL, [`openssl-s_client`](https://docs.openssl.org/3.5/man1/openssl-s_client/)
- RFC 8446, [The Transport Layer Security Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446.html), especially §§2 and 4
- OWASP, [Transport Layer Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html)
- OWASP, [Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
