# Securing the Student Records API with Authentication and Authorization

## Overview

Previously, you built a Student Records API with Node.js and Express. Then you connected the React/Vite client. Assuming that you already moved the API connection to verified HTTPS at `https://localhost:3443`. This laboratory extends that same cumulative project with:

- password hashing;
- a login endpoint;
- short-lived JSON Web Tokens (JWTs);
- bearer-token authentication;
- role-based authorization;
- object-level authorization for student records;
- protected OpenAPI documentation; and
- security-focused request tests.

Do not create a replacement API or revert it to HTTP. Continue working in the HTTPS-enabled `student-api` and retain the completed client app for the frontend authentication extension.

## Security Model

The secured API will use two roles:

| Role | Purpose |
|---|---|
| `registrar` | Manages the complete student collection |
| `student` | Reads only the student record linked to the authenticated account |

The finished access policy is:

| Method | Endpoint | Access policy |
|---|---|---|
| `POST` | `/auth/login` | Public; valid credentials return a token |
| `GET` | `/auth/me` | Any authenticated account |
| `GET` | `/students` | `registrar` only |
| `POST` | `/students` | `registrar` only |
| `GET` | `/students/:studentId` | `registrar`, or the student who owns the record |
| `PATCH` | `/students/:studentId` | `registrar` only |
| `DELETE` | `/students/:studentId` | `registrar` only |

This design demonstrates three distinct decisions:

1. **Authentication:** Is the bearer token valid?
2. **Function-level authorization:** Is the account's role permitted to use this operation?
3. **Object-level authorization:** Is the requested student record linked to this account?

## Learning Objectives

After completing the laboratory, you should be able to:

1. Hash and verify passwords with a password-hashing library.
2. Issue a short-lived JWT after successful login.
3. Validate a token's signature, algorithm, issuer, audience, and expiration.
4. Attach an authenticated identity to an Express request.
5. Protect endpoints with reusable authentication and authorization middleware.
6. Enforce both role-level and object-level access rules.
7. Distinguish `401 Unauthorized`, `403 Forbidden`, and `404 Not Found`.
8. Describe bearer authentication in OpenAPI.
9. Demonstrate negative security tests using Postman or `curl`.

## Prerequisites

- The completed Student Records API
- The completed React-Vite frontend client web application
- HTTPS configuration
- Node.js and npm
- A terminal and code editor
- Postman, Bruno, Insomnia, or `curl`

Before editing, confirm that the API still runs through HTTPS:

```bash
npm run dev
```

Open <https://localhost:3443/students>. Supply or trust the development certificate as instructed. At this point the endpoint may still be public. This laboratory will protect it.

---

## Step 1 - Install the Security Packages

Stop the server and run:

```bash
npm install bcryptjs jsonwebtoken dotenv
```

| Package | Purpose |
|---|---|
| `bcryptjs` | Hash and verify passwords |
| `jsonwebtoken` | Sign and verify JWT access tokens |
| `dotenv` | Load local development configuration from `.env` |

This activity uses asynchronous `bcrypt.hash()` and `bcrypt.compare()` so password work does not unnecessarily block the request handler.

---

## Step 2 - Protect Local Secrets

Create `.env` in the project root:

```dotenv
PORT=3443
TLS_KEY_PATH=certs/localhost-key.pem
TLS_CERT_PATH=certs/localhost-cert.pem
JWT_SECRET=replace-this-with-a-long-random-development-secret
JWT_ISSUER=student-api
JWT_AUDIENCE=student-api-users
```

Create `.env.example` with safe placeholders:

```dotenv
PORT=3443
TLS_KEY_PATH=certs/localhost-key.pem
TLS_CERT_PATH=certs/localhost-cert.pem
JWT_SECRET=replace-with-a-long-random-secret
JWT_ISSUER=student-api
JWT_AUDIENCE=student-api-users
```

Add these entries to `.gitignore`:

```gitignore
node_modules/
.env
```

### Security rule

Never commit the real `.env` file. A JWT signing secret allows an attacker to create tokens that the API may accept.

### Checkpoint

Run:

```bash
git status --short
```

The real `.env` file should not appear as an untracked file when `.gitignore` is working.

---

## Step 3 - Add the Authentication Project Structure

Extend the project folder structure: Keep the certificate directory and HTTPS server setup:

```text
student-api/
├── certs/
│   ├── localhost-cert.pem
│   └── localhost-key.pem
├── .env
├── .env.example
├── .gitignore
├── package.json
└── src/
    ├── app.js
    ├── openapi.js
    ├── data/
    │   └── users.js
    ├── middleware/
    │   ├── auth.js
    │   ├── error-handlers.js
    │   └── validation.js
    ├── routes/
    │   ├── auth.js
    │   └── students.js
    ├── security/
    │   └── tokens.js
    └── validators/
        ├── auth.js
        └── students.js
```

Responsibilities remain separated:

| Module | Responsibility |
|---|---|
| `data/users.js` | Demonstration accounts and password hashes |
| `security/tokens.js` | Token signing and verification rules |
| `middleware/auth.js` | Authentication and authorization middleware |
| `validators/auth.js` | Login request validation |
| `routes/auth.js` | Login and current-account routes |
| `routes/students.js` | Existing routes plus access checks |

---

## Step 4 - Validate Required Security Configuration

At the beginning of `src/app.js`, load the environment file before modules use its values:

```javascript
import 'dotenv/config';
```

After the imports, fail at startup when required configuration is absent:

```javascript
const requiredEnvironmentVariables = [
  'JWT_SECRET',
  'JWT_ISSUER',
  'JWT_AUDIENCE'
];

for (const variableName of requiredEnvironmentVariables) {
  if (!process.env[variableName]) {
    throw new Error(`Missing required environment variable: ${variableName}`);
  }
}
```

Failing during startup is safer than discovering the missing secret only after a login request arrives. Keep the Topic 8 certificate loading, `https.createServer()`, TLS minimum version, and port `3443` configuration in `src/app.js`; authentication middleware is added to that HTTPS application rather than replacing its server setup.

---

## Step 5 - Create Demonstration Accounts with Hashed Passwords

Create `src/data/users.js`:

```javascript
import bcrypt from 'bcryptjs';

const passwordCost = 12;

export const users = [
  {
    id: 1,
    email: 'registrar@example.edu',
    password_hash: await bcrypt.hash('RegistrarPass123!', passwordCost),
    role: 'registrar',
    student_id: null
  },
  {
    id: 2,
    email: 'ana@example.edu',
    password_hash: await bcrypt.hash('StudentPass123!', passwordCost),
    role: 'student',
    student_id: 1
  },
  {
    id: 3,
    email: 'carlo@example.edu',
    password_hash: await bcrypt.hash('StudentPass123!', passwordCost),
    role: 'student',
    student_id: 2
  }
];

export function findUserByEmail(email) {
  return users.find((user) => user.email === email);
}

export function toPublicUser(user) {
  return {
    id: user.id,
    email: user.email,
    role: user.role,
    student_id: user.student_id
  };
}
```

### Important limitation

These demonstration passwords appear in the laboratory guide so every student can run the same tests. They are not production credentials. A real system would create accounts through an approved workflow, persist only password hashes, enforce password and recovery policies, and never publish working passwords in documentation.

The API must never return `password_hash` in a response or include it in a token.

---

## Step 6 - Create Login Validation Rules

Create `src/validators/auth.js`:

```javascript
import { body } from 'express-validator';

export const loginRules = [
  body('email')
    .trim()
    .isEmail()
    .withMessage('A valid email address is required')
    .normalizeEmail(),

  body('password')
    .isString()
    .isLength({ min: 1, max: 200 })
    .withMessage('Password is required')
];
```

The validation rule prevents missing or unreasonably large values. It does not disclose whether a particular account exists.

---

## Step 7 - Sign and Verify Short-Lived Tokens

Create `src/security/tokens.js`:

```javascript
import jwt from 'jsonwebtoken';

const tokenOptions = {
  algorithm: 'HS256',
  issuer: process.env.JWT_ISSUER,
  audience: process.env.JWT_AUDIENCE
};

export function createAccessToken(user) {
  return jwt.sign(
    {
      role: user.role,
      student_id: user.student_id
    },
    process.env.JWT_SECRET,
    {
      ...tokenOptions,
      subject: String(user.id),
      expiresIn: '15m'
    }
  );
}

export function verifyAccessToken(token) {
  return jwt.verify(token, process.env.JWT_SECRET, {
    algorithms: ['HS256'],
    issuer: process.env.JWT_ISSUER,
    audience: process.env.JWT_AUDIENCE
  });
}
```

### Why each check matters

| Check | Purpose |
|---|---|
| Signature | Detects unauthorized token modification |
| `algorithms: ['HS256']` | Accepts only the algorithm selected by this API |
| `issuer` | Accepts tokens from the expected issuer |
| `audience` | Accepts tokens intended for this API |
| `exp` through `expiresIn` | Limits how long a stolen token remains usable |
| `sub` | Identifies the account represented by the token |

Do not replace `verify()` with `decode()`. Decoding only reads the token; it does not establish that the signature and claims are acceptable.

---

## Step 8 - Implement Authentication Middleware

Create `src/middleware/auth.js`:

```javascript
import { HttpError } from '../errors/http-error.js';
import { verifyAccessToken } from '../security/tokens.js';

export function authenticate(req, res, next) {
  const authorization = req.get('authorization');

  if (!authorization) {
    return next(new HttpError(
      401,
      'Authentication is required',
      '/problems/authentication-required'
    ));
  }

  const [scheme, token, extra] = authorization.trim().split(/\s+/);

  if (scheme !== 'Bearer' || !token || extra) {
    return next(new HttpError(
      401,
      'Bearer token is required',
      '/problems/invalid-authorization-header'
    ));
  }

  try {
    const payload = verifyAccessToken(token);

    req.auth = {
      userId: Number(payload.sub),
      role: payload.role,
      studentId: payload.student_id
    };

    next();
  } catch {
    next(new HttpError(
      401,
      'Token is invalid or expired',
      '/problems/invalid-access-token'
    ));
  }
}

export function requireRole(...allowedRoles) {
  return (req, res, next) => {
    if (!allowedRoles.includes(req.auth.role)) {
      return next(new HttpError(
        403,
        'You do not have permission to perform this operation',
        '/problems/insufficient-permission'
      ));
    }

    next();
  };
}

export function requireStudentRecordAccess(req, res, next) {
  const requestedStudentId = req.validated.studentId;

  if (
    req.auth.role !== 'registrar' &&
    req.auth.studentId !== requestedStudentId
  ) {
    return next(new HttpError(
      403,
      'You do not have permission to access this student record',
      '/problems/student-access-forbidden'
    ));
  }

  next();
}
```

### Middleware order

Object access depends on both `req.auth` and the validated numeric path parameter. Therefore use this order:

```text
authenticate
    ↓
studentIdRules
    ↓
handleValidationErrors
    ↓
requireStudentRecordAccess
    ↓
route handler
```

---

## Step 9 - Implement Login and Current-Account Routes

Create `src/routes/auth.js`:

```javascript
import { Router } from 'express';
import bcrypt from 'bcryptjs';
import { findUserByEmail, toPublicUser } from '../data/users.js';
import { HttpError } from '../errors/http-error.js';
import { authenticate } from '../middleware/auth.js';
import { handleValidationErrors } from '../middleware/validation.js';
import { createAccessToken } from '../security/tokens.js';
import { loginRules } from '../validators/auth.js';

export const authRouter = Router();

const fallbackPasswordHash = await bcrypt.hash(
  'not-a-real-account-password',
  12
);

authRouter.post(
  '/login',
  loginRules,
  handleValidationErrors,
  async (req, res) => {
    const { email, password } = req.validated;
    const user = findUserByEmail(email);

    const passwordMatches = await bcrypt.compare(
      password,
      user?.password_hash ?? fallbackPasswordHash
    );

    if (!user || !passwordMatches) {
      throw new HttpError(
        401,
        'Email or password is incorrect',
        '/problems/invalid-credentials'
      );
    }

    res.json({
      access_token: createAccessToken(user),
      token_type: 'Bearer',
      expires_in: 900,
      user: toPublicUser(user)
    });
  }
);

authRouter.get('/me', authenticate, (req, res) => {
  res.json({
    id: req.auth.userId,
    role: req.auth.role,
    student_id: req.auth.studentId
  });
});
```

The login failure uses the same message for an unknown email and a wrong password. The fallback hash also performs a password comparison when the account is absent, reducing the most obvious response-time difference between the two paths. These measures reduce direct account enumeration but do not replace login rate limiting and monitoring.

---

## Step 10 - Protect the Student Routes

Update imports in `src/routes/students.js`:

```javascript
import {
  authenticate,
  requireRole,
  requireStudentRecordAccess
} from '../middleware/auth.js';
```

Protect the collection list:

```javascript
studentsRouter.get(
  '/',
  authenticate,
  requireRole('registrar'),
  (req, res) => {
    const activeFilter = req.query.active;
    let results = students;

    if (activeFilter === 'true' || activeFilter === 'false') {
      const active = activeFilter === 'true';
      results = students.filter((student) => student.active === active);
    }

    res.json({
      items: results.map(toStudentResponse),
      count: results.length
    });
  }
);
```

Protect retrieval with role and ownership checks:

```javascript
studentsRouter.get(
  '/:studentId',
  authenticate,
  studentIdRules,
  handleValidationErrors,
  requireStudentRecordAccess,
  (req, res) => {
    const student = findStudent(req.validated.studentId);

    if (!student) {
      throw new HttpError(
        404,
        'Student not found',
        '/problems/student-not-found'
      );
    }

    res.json(toStudentResponse(student));
  }
);
```

Add `authenticate` and `requireRole('registrar')` to the existing create, update, and delete route declarations. Keep the API validation rules and handler callbacks, but arrange each declaration in this order:

```text
POST /
  authenticate
  requireRole('registrar')
  createStudentRules
  handleValidationErrors
  existing create callback

PATCH /:studentId
  authenticate
  requireRole('registrar')
  studentIdRules
  updateStudentRules
  handleValidationErrors
  existing update callback

DELETE /:studentId
  authenticate
  requireRole('registrar')
  studentIdRules
  handleValidationErrors
  existing delete callback
```

Do not replace the existing callbacks. You are inserting security middleware before the validation rules and route logic already implemented.

### Why authenticate before authorization

The API cannot evaluate a role or record relationship until it has established the caller's identity from a valid token.

---

## Step 11 - Mount the Authentication Routes

Update `src/app.js`:

```javascript
import { authRouter } from './routes/auth.js';
```

Mount the router before the not-found and error handlers:

```javascript
app.use('/auth', authRouter);
app.use('/students', studentsRouter);

app.use(notFoundHandler);
app.use(errorHandler);
```

If the React client still calls this API, update its CORS configuration so the browser may send the authorization header:

```javascript
app.use(cors({
  origin: 'http://localhost:5173',
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  allowedHeaders: ['Authorization', 'Content-Type']
}));
```

CORS does not authenticate the caller. It only controls which browser origins may read cross-origin responses.

---

## Step 12 - Describe Bearer Authentication in OpenAPI

In `src/openapi.js`, add this scheme under `components`:

```javascript
securitySchemes: {
  BearerAuth: {
    type: 'http',
    scheme: 'bearer',
    bearerFormat: 'JWT'
  }
}
```

Add a public login operation for `/auth/login`. Its `security` value may be an empty array:

```javascript
security: []
```

For protected operations, add:

```javascript
security: [{ BearerAuth: [] }]
```

Document at least these security responses where applicable:

```javascript
'401': {
  description: 'Authentication is missing, invalid, or expired'
},
'403': {
  description: 'The authenticated account lacks permission'
}
```

Open Swagger UI, use `/auth/login`, copy `access_token`, select **Authorize**, and enter only the token if the interface supplies the `Bearer` prefix automatically.

---

## Step 13 - Test the Authentication Flow

### Registrar login

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H 'Content-Type: application/json' \
  -d '{"email":"registrar@example.edu","password":"RegistrarPass123!"}' \
  https://localhost:3443/auth/login
```

Expected result: `200 OK`, an access token, `token_type: "Bearer"`, `expires_in: 900`, and public account information. The password and hash must be absent.

Copy the access token into a temporary shell variable:

```bash
REGISTRAR_TOKEN='paste-token-here'
```

Retrieve the current identity:

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $REGISTRAR_TOKEN" \
  https://localhost:3443/auth/me
```

### Student login

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H 'Content-Type: application/json' \
  -d '{"email":"ana@example.edu","password":"StudentPass123!"}' \
  https://localhost:3443/auth/login
```

Store the returned token:

```bash
ANA_TOKEN='paste-token-here'
```

Never submit actual token values in screenshots or written answers. Redact most of the token while keeping enough context to show that the bearer header was present.

---

## Step 14 - Test the Authorization Policy

### Missing token

```bash
curl -i --cacert certs/localhost-cert.pem https://localhost:3443/students
```

Expected: `401 Unauthorized`.

### Registrar lists all students

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $REGISTRAR_TOKEN" \
  https://localhost:3443/students
```

Expected: `200 OK`.

### Student tries to list all students

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $ANA_TOKEN" \
  https://localhost:3443/students
```

Expected: `403 Forbidden`.

### Student retrieves own record

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $ANA_TOKEN" \
  https://localhost:3443/students/1
```

Expected: `200 OK`.

### Student retrieves another student's record

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $ANA_TOKEN" \
  https://localhost:3443/students/2
```

Expected: `403 Forbidden`.

### Student tries to update a record

```bash
curl -i -X PATCH \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $ANA_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"year_level":3}' \
  https://localhost:3443/students/1
```

Expected: `403 Forbidden`.

### Registrar creates a valid test record

```bash
curl -i -X POST \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $REGISTRAR_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"student_number":"2026-00901","full_name":"Test Student","email":"test.student@example.edu","year_level":2,"active":true}' \
  https://localhost:3443/students
```

Expected: `201 Created`. Record the returned `id` or the final path segment of the `Location` header:

```bash
TEST_STUDENT_ID=3
```

Replace `3` with the actual value returned by your API.

### Registrar submits invalid student data

```bash
curl -i -X POST \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $REGISTRAR_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"student_number":"bad","full_name":"X","email":"not-an-email","year_level":7}' \
  https://localhost:3443/students
```

Expected: `400 Bad Request` with the existing structured validation problem. Authentication and registrar authorization succeed before validation rejects the body.

### Registrar requests a permitted but missing record

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $REGISTRAR_TOKEN" \
  https://localhost:3443/students/9999
```

Expected: `404 Not Found`. The registrar has permission to retrieve student records, but this resource does not exist.

### Student tries to delete a record

```bash
curl -i -X DELETE \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $ANA_TOKEN" \
  https://localhost:3443/students/1
```

Expected: `403 Forbidden`.

### Registrar deletes a record

Delete the test record created above so the two seeded student-account mappings remain valid:

```bash
curl -i -X DELETE \
  --cacert certs/localhost-cert.pem \
  -H "Authorization: Bearer $REGISTRAR_TOKEN" \
  "https://localhost:3443/students/$TEST_STUDENT_ID"
```

Expected: `204 No Content`.

---

## Step 15 - Test Invalid and Expired Tokens

### Malformed token

```bash
curl -i \
  --cacert certs/localhost-cert.pem \
  -H 'Authorization: Bearer not-a-real-token' \
  https://localhost:3443/auth/me
```

Expected: `401 Unauthorized` with no verification details or stack trace.

### Expiration test

Temporarily change `expiresIn` in `createAccessToken()` to `'5s'`. Restart the API, log in, wait more than five seconds, and call `/auth/me` with that token.

Expected: `401 Unauthorized`.

Restore the duration to `'15m'` after the test.

---

## Step 16 - Security Review Checklist

- [ ] `.env` is ignored by Git.
- [ ] No signing secret is hard-coded in source files.
- [ ] Only password hashes are stored in the user data structure.
- [ ] Neither responses nor tokens contain a password or hash.
- [ ] Login returns the same failure message for unknown email and wrong password.
- [ ] Access tokens expire.
- [ ] Verification restricts the accepted algorithm.
- [ ] Verification checks issuer and audience.
- [ ] Every student route requires authentication.
- [ ] Registrar-only operations use role authorization.
- [ ] Student record retrieval checks ownership.
- [ ] Missing or invalid authentication produces `401`.
- [ ] Insufficient permission produces `403`.
- [ ] Missing resources still produce `404` after the caller passes access checks.
- [ ] Swagger/OpenAPI marks protected operations with bearer security.
- [ ] Error responses do not contain stack traces or token contents.
- [ ] The Topic 8 HTTPS server and certificate verification remain active.

---

## Step 17 - Continue with the Topic 7 Frontend

After the backend security tests pass, complete the [React JWT Authentication Guard Laboratory Guide](topic-09-react-jwt-authentication-guard-laboratory-guide.md). It extends the existing Topic 7 client with a login page, `/auth/me` session verification, authentication and role guards, bearer headers, logout, and separate handling for `401` and `403` responses.

Keep `VITE_API_BASE_URL=https://localhost:3443`. Do not return the frontend to the Topic 7 HTTP URL.

---

## Troubleshooting

### The API stops with “Missing required environment variable”

Confirm `.env` is in the project root, `import 'dotenv/config'` runs before the values are read, and the variable names match exactly.

### Every token is rejected after restarting

Confirm `JWT_SECRET`, `JWT_ISSUER`, and `JWT_AUDIENCE` are unchanged between signing and verification.

### The authorization header is rejected

Use exactly:

```text
Authorization: Bearer <token>
```

Do not add quotation marks around the token. Do not include two `Bearer` prefixes.

### The browser client fails but Postman works

Ensure CORS permits the exact frontend origin and includes `Authorization` in `allowedHeaders`. Remember that successful CORS configuration does not prove that authentication or authorization is correct.

### A student receives `403` for their own record

Check that the account's `student_id` matches the numeric `id` in the student array and that validation converts the path parameter to an integer.

---

## References
- `jsonwebtoken`, [official README](https://github.com/auth0/node-jsonwebtoken/blob/master/README.md), especially `jwt.sign`, `jwt.verify`, `expiresIn`, `algorithms`, `issuer`, and `audience`.
- `bcryptjs`, [official README](https://github.com/dcodeIO/bcrypt.js), especially asynchronous `hash()` and `compare()`.
- OWASP API Security Top 10 (2023), [API1 Broken Object Level Authorization](https://api-security.owasp.org/editions/2023/en/0xa1-broken-object-level-authorization/), [API2 Broken Authentication](https://api-security.owasp.org/editions/2023/en/0xa2-broken-authentication/), and [API5 Broken Function Level Authorization](https://api-security.owasp.org/editions/2023/en/0xa5-broken-function-level-authorization/).
- [RFC 7519: JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519.html), especially §§4 and 7.2.
- [RFC 8725: JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html), especially §§2–3.
