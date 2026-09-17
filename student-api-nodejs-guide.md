# Building a Student Records API with Node.js and Express

## Overview

In this activity, you will build a RESTful Student Records API using Node.js and Express 5. The API will validate incoming data, return appropriate HTTP status codes, handle errors consistently, and publish an OpenAPI document through Swagger UI and Scalar API Reference.

The finished API will provide these operations:

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/students` | Create a student |
| `GET` | `/students` | List students |
| `GET` | `/students/:studentId` | Retrieve one student |
| `PATCH` | `/students/:studentId` | Update part of a student |
| `DELETE` | `/students/:studentId` | Delete a student |

## Objectives

After completing the activity, you should be able to:

1. Create and run an Express application.
2. Define REST routes using HTTP methods and resource paths.
3. Read path parameters, query parameters, and JSON request bodies.
4. Validate requests with `express-validator`.
5. Return appropriate HTTP status codes and JSON responses.
6. Handle expected and unexpected errors through middleware.
7. Describe an API using OpenAPI.
8. Serve the same OpenAPI document through Swagger UI and Scalar.

## Prerequisites

Before starting, make sure you have:

- A current Node.js LTS release
- npm, which is included with Node.js
- A code editor such as Visual Studio Code
- A terminal
- A browser
- Postman, Insomnia, Bruno, or `curl` for optional testing

Verify the installation:

```bash
node --version
npm --version
```

Both commands should print version numbers.

---

## Step 1 - Create the Project

Create a project directory and open it in the terminal:

```bash
mkdir student-api
cd student-api
npm init -y
```

Install the application packages:

```bash
npm install express express-validator
npm install swagger-jsdoc swagger-ui-express
npm install @scalar/express-api-reference
```

### What the packages do

| Package | Purpose |
|---|---|
| `express` | Routing, middleware, requests, and responses |
| `express-validator` | Request validation and sanitization |
| `swagger-jsdoc` | Generates an OpenAPI object from configuration and JSDoc comments |
| `swagger-ui-express` | Serves Swagger UI from an Express route |
| `@scalar/express-api-reference` | Serves Scalar API Reference from an Express route |

### Checkpoint

Your project should now contain `package.json`, `package-lock.json`, and `node_modules`.

---

## Step 2 - Configure ES Modules and Scripts

Open `package.json`. Add `"type": "module"` and replace the scripts section with:

```json
{
  "name": "student-api",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/app.js",
    "dev": "node --watch src/app.js"
  }
}
```

Keep the `dependencies` section that npm created. Do not delete it.

The `start` script runs the server normally. The `dev` script restarts the server when a source file changes.

---

## Step 3 - Create the Project Structure

Create these directories and files in your editor:

```text
student-api/
├── package.json
└── src/
    ├── app.js
    ├── openapi.js
    ├── errors/
    │   └── http-error.js
    ├── middleware/
    │   ├── error-handlers.js
    │   └── validation.js
    ├── routes/
    │   └── students.js
    └── validators/
        └── students.js
```

Keeping routes, validation rules, and error handlers in separate modules makes the application easier to read and extend.

---

## Step 4 - Create a Reusable HTTP Error

Open `src/errors/http-error.js`:

```javascript
export class HttpError extends Error {
  constructor(status, message, type = '/problems/request-error') {
    super(message);
    this.name = 'HttpError';
    this.status = status;
    this.type = type;
  }
}
```

This class stores the HTTP status and public problem type together with the error message. Route handlers can pass the error to central middleware instead of constructing error responses repeatedly.

---

## Step 5 - Define Validation Rules

Open `src/validators/students.js`:

```javascript
import { body, param } from 'express-validator';

export const studentIdRules = [
  param('studentId')
    .isInt({ min: 1 })
    .withMessage('Student ID must be a positive integer')
    .toInt()
];

export const createStudentRules = [
  body('student_number')
    .trim()
    .isLength({ min: 8, max: 20 })
    .withMessage('Student number must contain 8 to 20 characters'),

  body('full_name')
    .trim()
    .isLength({ min: 2, max: 100 })
    .withMessage('Full name must contain 2 to 100 characters'),

  body('email')
    .trim()
    .isEmail()
    .withMessage('A valid email address is required')
    .normalizeEmail(),

  body('year_level')
    .isInt({ min: 1, max: 5 })
    .withMessage('Year level must be an integer from 1 to 5')
    .toInt(),

  body('active')
    .optional()
    .isBoolean()
    .withMessage('Active must be true or false')
    .toBoolean()
];

export const updateStudentRules = [
  body('student_number')
    .optional()
    .trim()
    .isLength({ min: 8, max: 20 })
    .withMessage('Student number must contain 8 to 20 characters'),

  body('full_name')
    .optional()
    .trim()
    .isLength({ min: 2, max: 100 })
    .withMessage('Full name must contain 2 to 100 characters'),

  body('email')
    .optional()
    .trim()
    .isEmail()
    .withMessage('A valid email address is required')
    .normalizeEmail(),

  body('year_level')
    .optional()
    .isInt({ min: 1, max: 5 })
    .withMessage('Year level must be an integer from 1 to 5')
    .toInt(),

  body('active')
    .optional()
    .isBoolean()
    .withMessage('Active must be true or false')
    .toBoolean(),

  body()
    .custom((value) => {
      const allowedFields = [
        'student_number',
        'full_name',
        'email',
        'year_level',
        'active'
      ];

      return allowedFields.some((field) => value[field] !== undefined);
    })
    .withMessage('Provide at least one student field to update')
];
```

### What happens here

- `body()` selects a request-body field.
- `param()` selects a path parameter.
- Validators such as `isEmail()` and `isInt()` check values.
- Sanitizers such as `trim()`, `normalizeEmail()`, and `toInt()` clean or convert accepted values.
- `.optional()` permits a field to be absent.
- The final custom PATCH rule rejects an empty update body.

---

## Step 6 - Handle Validation Results

Open `src/middleware/validation.js`:

```javascript
import { matchedData, validationResult } from 'express-validator';

export function handleValidationErrors(req, res, next) {
  const result = validationResult(req);

  if (!result.isEmpty()) {
    return res.status(400).json({
      type: '/problems/validation-error',
      title: 'Request validation failed',
      status: 400,
      detail: 'One or more request values are invalid.',
      instance: req.originalUrl,
      errors: result.array().map((error) => ({
        field: error.path || 'body',
        location: error.location,
        message: error.msg
      }))
    });
  }

  req.validated = matchedData(req, {
    includeOptionals: true,
    onlyValidData: true
  });

  next();
}
```

The validators record failures but do not send responses automatically. This middleware checks the result, returns `400 Bad Request` when necessary, and stores accepted data in `req.validated`.

### Why use `matchedData()`?

It gives the controller only the fields selected by the validators. This helps prevent undeclared input fields from flowing into application logic.

---

## Step 7 - Create Central Error Handlers

Open `src/middleware/error-handlers.js`:

```javascript
import { HttpError } from '../errors/http-error.js';

export function notFoundHandler(req, res, next) {
  next(
    new HttpError(
      404,
      'Endpoint not found',
      '/problems/endpoint-not-found'
    )
  );
}

export function errorHandler(err, req, res, next) {
  if (res.headersSent) {
    return next(err);
  }

  const status = Number.isInteger(err.status) ? err.status : 500;
  const serverError = status >= 500;

  if (serverError) {
    console.error(err);
  }

  res.status(status).json({
    type: err.type ?? '/problems/internal-server-error',
    title: serverError ? 'Internal server error' : err.message,
    status,
    detail: serverError
      ? 'The server could not complete the request.'
      : err.message,
    instance: req.originalUrl
  });
}
```

### Important rules

- Error-handling middleware has four parameters: `(err, req, res, next)`.
- Register the general error handler after all routes.
- Log internal errors on the server.
- Do not return a stack trace to the client.

---

## Step 8 - Implement the Student Routes

Open `src/routes/students.js`. The example uses an in-memory array, so all records reset when the server restarts.

```javascript
import { Router } from 'express';
import { HttpError } from '../errors/http-error.js';
import { handleValidationErrors } from '../middleware/validation.js';
import {
  createStudentRules,
  studentIdRules,
  updateStudentRules
} from '../validators/students.js';

export const studentsRouter = Router();

let nextId = 3;

const students = [
  {
    id: 1,
    student_number: '2026-00001',
    full_name: 'Ana Reyes',
    email: 'ana@example.edu',
    year_level: 2,
    active: true
  },
  {
    id: 2,
    student_number: '2026-00002',
    full_name: 'Carlo Santos',
    email: 'carlo@example.edu',
    year_level: 1,
    active: true
  }
];

function toStudentResponse(student) {
  return {
    id: student.id,
    student_number: student.student_number,
    full_name: student.full_name,
    email: student.email,
    year_level: student.year_level,
    active: student.active
  };
}

function findStudent(studentId) {
  return students.find((student) => student.id === studentId);
}

function studentNumberExists(studentNumber, ignoredId = null) {
  return students.some(
    (student) =>
      student.student_number === studentNumber && student.id !== ignoredId
  );
}

studentsRouter.get('/', (req, res) => {
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
});

studentsRouter.get(
  '/:studentId',
  studentIdRules,
  handleValidationErrors,
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

studentsRouter.post(
  '/',
  createStudentRules,
  handleValidationErrors,
  (req, res) => {
    const data = req.validated;

    if (studentNumberExists(data.student_number)) {
      throw new HttpError(
        409,
        'Student number already exists',
        '/problems/duplicate-student-number'
      );
    }

    const student = {
      id: nextId++,
      student_number: data.student_number,
      full_name: data.full_name,
      email: data.email,
      year_level: data.year_level,
      active: data.active ?? true
    };

    students.push(student);

    res
      .location(`/students/${student.id}`)
      .status(201)
      .json(toStudentResponse(student));
  }
);

studentsRouter.patch(
  '/:studentId',
  studentIdRules,
  updateStudentRules,
  handleValidationErrors,
  (req, res) => {
    const { studentId, ...changes } = req.validated;
    const student = findStudent(studentId);

    if (!student) {
      throw new HttpError(
        404,
        'Student not found',
        '/problems/student-not-found'
      );
    }

    if (
      changes.student_number &&
      studentNumberExists(changes.student_number, studentId)
    ) {
      throw new HttpError(
        409,
        'Student number already exists',
        '/problems/duplicate-student-number'
      );
    }

    Object.assign(student, changes);
    res.json(toStudentResponse(student));
  }
);

studentsRouter.delete(
  '/:studentId',
  studentIdRules,
  handleValidationErrors,
  (req, res) => {
    const index = students.findIndex(
      (student) => student.id === req.validated.studentId
    );

    if (index === -1) {
      throw new HttpError(
        404,
        'Student not found',
        '/problems/student-not-found'
      );
    }

    students.splice(index, 1);
    res.status(204).end();
  }
);
```

### Route behavior

- `GET /students` returns an object containing `items` and `count`.
- `GET /students?active=true` filters by active status.
- `GET /students/:studentId` returns `404` when the ID does not exist.
- `POST /students` returns `201` and a `Location` header.
- `PATCH` changes only supplied fields.
- `DELETE` returns an empty `204` response.

---

## Step 9 - Define the OpenAPI Document

Open `src/openapi.js`:

```javascript
import swaggerJsdoc from 'swagger-jsdoc';

const options = {
  failOnErrors: true,
  definition: {
    openapi: '3.1.0',
    info: {
      title: 'Student Records API',
      version: '1.0.0',
      description: 'A classroom API for managing student records'
    },
    servers: [
      {
        url: 'http://localhost:3000',
        description: 'Local development server'
      }
    ],
    tags: [
      {
        name: 'Students',
        description: 'Student record operations'
      }
    ],
    components: {
      schemas: {
        StudentInput: {
          type: 'object',
          required: [
            'student_number',
            'full_name',
            'email',
            'year_level'
          ],
          properties: {
            student_number: {
              type: 'string',
              minLength: 8,
              maxLength: 20,
              examples: ['2026-00125']
            },
            full_name: {
              type: 'string',
              minLength: 2,
              maxLength: 100,
              examples: ['Ana Reyes']
            },
            email: {
              type: 'string',
              format: 'email',
              examples: ['ana@example.edu']
            },
            year_level: {
              type: 'integer',
              minimum: 1,
              maximum: 5,
              examples: [2]
            },
            active: {
              type: 'boolean',
              default: true
            }
          }
        },
        Student: {
          allOf: [
            { $ref: '#/components/schemas/StudentInput' },
            {
              type: 'object',
              required: ['id'],
              properties: {
                id: {
                  type: 'integer',
                  readOnly: true,
                  examples: [1]
                }
              }
            }
          ]
        },
        Problem: {
          type: 'object',
          required: ['type', 'title', 'status', 'detail', 'instance'],
          properties: {
            type: { type: 'string' },
            title: { type: 'string' },
            status: { type: 'integer' },
            detail: { type: 'string' },
            instance: { type: 'string' }
          }
        }
      }
    },
    paths: {
      '/students': {
        get: {
          tags: ['Students'],
          summary: 'List students',
          parameters: [
            {
              in: 'query',
              name: 'active',
              schema: { type: 'boolean' },
              description: 'Filter students by active status'
            }
          ],
          responses: {
            200: {
              description: 'Student collection',
              content: {
                'application/json': {
                  schema: {
                    type: 'object',
                    properties: {
                      items: {
                        type: 'array',
                        items: { $ref: '#/components/schemas/Student' }
                      },
                      count: { type: 'integer' }
                    }
                  }
                }
              }
            }
          }
        },
        post: {
          tags: ['Students'],
          summary: 'Create a student',
          requestBody: {
            required: true,
            content: {
              'application/json': {
                schema: { $ref: '#/components/schemas/StudentInput' }
              }
            }
          },
          responses: {
            201: {
              description: 'Student created',
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Student' }
                }
              }
            },
            400: {
              description: 'Request validation failed',
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Problem' }
                }
              }
            },
            409: {
              description: 'Student number already exists',
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Problem' }
                }
              }
            }
          }
        }
      },
      '/students/{studentId}': {
        parameters: [
          {
            in: 'path',
            name: 'studentId',
            required: true,
            schema: { type: 'integer', minimum: 1 }
          }
        ],
        get: {
          tags: ['Students'],
          summary: 'Retrieve one student',
          responses: {
            200: {
              description: 'Student found',
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Student' }
                }
              }
            },
            404: {
              description: 'Student not found',
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Problem' }
                }
              }
            }
          }
        },
        patch: {
          tags: ['Students'],
          summary: 'Update part of a student',
          requestBody: {
            required: true,
            content: {
              'application/json': {
                schema: {
                  $ref: '#/components/schemas/StudentInput'
                }
              }
            }
          },
          responses: {
            200: {
              description: 'Student updated',
              content: {
                'application/json': {
                  schema: { $ref: '#/components/schemas/Student' }
                }
              }
            },
            400: { description: 'Request validation failed' },
            404: { description: 'Student not found' },
            409: { description: 'Student number already exists' }
          }
        },
        delete: {
          tags: ['Students'],
          summary: 'Delete a student',
          responses: {
            204: { description: 'Student deleted' },
            404: { description: 'Student not found' }
          }
        }
      }
    }
  },
  apis: []
};

export const openapiSpecification = swaggerJsdoc(options);
```

This version keeps the complete contract in JavaScript. `swagger-jsdoc` can also combine a smaller base configuration with `@openapi` JSDoc comments stored beside routes.

### Important PATCH note

The example reuses `StudentInput` in the PATCH documentation, so Swagger UI displays all fields as required. As an improvement, create a separate `StudentUpdate` schema with no `required` array and use it for PATCH.

---

## Step 10 - Assemble the Application

Open `src/app.js`:

```javascript
import express from 'express';
import swaggerUi from 'swagger-ui-express';
import { apiReference } from '@scalar/express-api-reference';
import { errorHandler, notFoundHandler } from './middleware/error-handlers.js';
import { openapiSpecification } from './openapi.js';
import { studentsRouter } from './routes/students.js';

const app = express();
const port = 3000;

app.disable('x-powered-by');
app.use(express.json());

app.get('/', (req, res) => {
  res.json({
    message: 'Student Records API',
    openapi: '/openapi.json',
    swagger: '/api-docs',
    scalar: '/reference'
  });
});

app.get('/openapi.json', (req, res) => {
  res.json(openapiSpecification);
});

app.use(
  '/api-docs',
  swaggerUi.serve,
  swaggerUi.setup(openapiSpecification)
);

app.use(
  '/reference',
  apiReference({
    url: '/openapi.json',
    theme: 'default'
  })
);

app.use('/students', studentsRouter);

app.use(notFoundHandler);
app.use(errorHandler);

app.listen(port, () => {
  console.log(`API: http://localhost:${port}`);
  console.log(`Swagger UI: http://localhost:${port}/api-docs`);
  console.log(`Scalar: http://localhost:${port}/reference`);
});
```

### Middleware order

The order of `app.use()` calls matters:

1. Parse JSON.
2. Serve the OpenAPI document and documentation interfaces.
3. Mount student routes.
4. Convert unmatched requests into a `404` error.
5. Handle every forwarded error.

---

## Step 11 - Run the API

Start the development server:

```bash
npm run dev
```

Open these URLs:

- API home: <http://localhost:3000/>
- OpenAPI JSON: <http://localhost:3000/openapi.json>
- Swagger UI: <http://localhost:3000/api-docs>
- Scalar API Reference: <http://localhost:3000/reference>

If the terminal reports a syntax or import error, use the file and line number in the error message to locate the problem. Confirm that every local import ends in `.js`.

---

## Step 12 - Test the API

You can run these commands in a second terminal.

### List students

```bash
curl -i http://localhost:3000/students
```

Expected status: `200 OK`.

### Filter active students

```bash
curl -i 'http://localhost:3000/students?active=true'
```

### Retrieve one student

```bash
curl -i http://localhost:3000/students/1
```

Expected status: `200 OK`.

### Request a missing student

```bash
curl -i http://localhost:3000/students/999
```

Expected status: `404 Not Found` with a problem response.

### Request an invalid ID

```bash
curl -i http://localhost:3000/students/abc
```

Expected status: `400 Bad Request` with validation details.

### Create a student

```bash
curl -i \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"student_number":"2026-00125","full_name":"Maria Cruz","email":"maria@example.edu","year_level":2}' \
  http://localhost:3000/students
```

Expected status: `201 Created`. Confirm that the response includes a `Location` header.

### Submit invalid data

```bash
curl -i \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"student_number":"123","full_name":"","email":"invalid","year_level":8}' \
  http://localhost:3000/students
```

Expected status: `400 Bad Request` with an `errors` array.

### Trigger a conflict

Repeat the valid POST request with the same `student_number`.

Expected status: `409 Conflict`.

### Update a student

```bash
curl -i \
  -X PATCH \
  -H 'Content-Type: application/json' \
  -d '{"year_level":3,"active":false}' \
  http://localhost:3000/students/1
```

Expected status: `200 OK` with the updated student.

### Delete a student

```bash
curl -i -X DELETE http://localhost:3000/students/2
```

Expected status: `204 No Content`. A `204` response must not contain a body.

---

## Step 13 - Compare Swagger UI and Scalar

Open Swagger UI and Scalar side by side.

Check whether both interfaces show:

- All five operations
- The `studentId` path parameter
- Request-body schemas
- Success and error responses
- The same title and version
- A way to send a test request

Both interfaces read `/openapi.json`. If an operation is absent from both, correct the OpenAPI document. If it appears in the documentation but the request behaves differently, align the Express route with the contract.

---

## Step 14 - Improve the PATCH Schema

The current OpenAPI contract reuses a creation schema that marks several fields as required. Add this schema under `components.schemas`:

```javascript
StudentUpdate: {
  type: 'object',
  minProperties: 1,
  properties: {
    student_number: { type: 'string', minLength: 8, maxLength: 20 },
    full_name: { type: 'string', minLength: 2, maxLength: 100 },
    email: { type: 'string', format: 'email' },
    year_level: { type: 'integer', minimum: 1, maximum: 5 },
    active: { type: 'boolean' }
  }
}
```

Then change the PATCH request-body reference:

```javascript
schema: {
  $ref: '#/components/schemas/StudentUpdate'
}
```

Restart or allow watch mode to reload, then inspect the PATCH operation in both documentation interfaces.

---

## Step 15 - Completion Checklist

Before submitting, confirm each requirement.

- [ ] `npm start` or `npm run dev` starts without errors.
- [ ] The API accepts and returns JSON.
- [ ] All five student operations work.
- [ ] IDs and request bodies are validated.
- [ ] Invalid data returns `400` with useful field details.
- [ ] Missing students return `404`.
- [ ] Duplicate student numbers return `409`.
- [ ] Creation returns `201` and a `Location` header.
- [ ] Deletion returns an empty `204` response.
- [ ] Unexpected errors do not expose stack traces.
- [ ] `/openapi.json` returns the OpenAPI document.
- [ ] `/api-docs` displays Swagger UI.
- [ ] `/reference` displays Scalar API Reference.
- [ ] The documentation matches the implemented routes.
- [ ] `node_modules` is excluded from submission or version control.

## Submission Requirements

Submit the following:

1. The project source code without `node_modules`.
2. `package.json` and `package-lock.json`.
3. A screenshot of Swagger UI showing the Students operations.
4. A screenshot of Scalar showing the same API.
5. Test evidence for `201`, `204`, `400`, `404`, and `409` responses.
6. A short reflection answering:
   - Why does middleware order matter?
   - Why should the server validate data even when a client form already validates it?
   - Why can Swagger UI and Scalar share one OpenAPI document?

## Troubleshooting

### `Cannot use import statement outside a module`

Add `"type": "module"` to `package.json`.

### `ERR_MODULE_NOT_FOUND`

Check the import path and include the `.js` extension for local modules. Run `npm install` if a package is missing.

### `req.body` is undefined

Register `app.use(express.json())` before the student routes and send `Content-Type: application/json`.

### The request never finishes

A middleware probably did not send a response or call `next()`.

### The error handler does not run

Confirm that it has four parameters and appears after the routes:

```javascript
app.use((err, req, res, next) => {
  // Handle the error.
});
```

### Swagger UI or Scalar displays no operations

Open `/openapi.json` and check whether `paths` contains the operations. Inspect the terminal for errors from `swagger-jsdoc`.

### Port 3000 is already in use

Stop the process using the port or change `port` in `src/app.js` and the OpenAPI server URL.

## References

### Course reference

- Zalando SE, *Zalando RESTful API and Event Guidelines*:
  - **pp. 12-14:** API terminology, API-first principle, and OpenAPI
  - **pp. 22-29:** Data types and formats
  - **pp. 31-38:** Resource URLs and parameters
  - **pp. 38-44:** JSON payload and schema conventions
  - **pp. 49-57:** HTTP method semantics
  - **pp. 64-76:** Status codes, errors, problem JSON, and stack traces
  - **p. 118:** Publication of OpenAPI specifications

### External references

- Express, [Routing](https://expressjs.com/en/guide/routing/)
- Express, [Using middleware](https://expressjs.com/en/guide/using-middleware/)
- Express, [Error Handling](https://expressjs.com/en/5x/guide/error-handling/)
- Express, [Express 5 API](https://expressjs.com/en/5x/api.html)
- express-validator, [Documentation](https://express-validator.github.io/docs/)
- swagger-jsdoc, [npm documentation](https://www.npmjs.com/package/swagger-jsdoc)
- swagger-ui-express, [npm documentation](https://www.npmjs.com/package/swagger-ui-express)
- Scalar, [`@scalar/express-api-reference`](https://www.npmjs.com/package/@scalar/express-api-reference)
- OpenAPI Initiative, [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- IETF, [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
