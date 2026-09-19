# Building a Student Records Client with React and Tailwind CSS

## Overview

In this activity, you will build a browser-based client for the Student Records API created in Topic 6. The client will use React for component rendering and state management, Tailwind CSS for interface styling, and the browser Fetch API for HTTP communication.

The completed client will:

- retrieve and display student records;
- submit a student-registration form as JSON;
- show distinct loading, success, empty, and failure states;
- interpret HTTP status codes and Problem Details responses;
- prevent duplicate submissions while a request is pending; and
- call an Express API running on a different origin through CORS.

This activity uses two separate applications:

```text
React client                         Express API
http://localhost:5173               http://localhost:3000
        |                                    |
        |------ GET /students -------------->|
        |<----- 200 + JSON collection -------|
        |                                    |
        |------ POST /students + JSON ------>|
        |<----- 201 or problem response -----|
```

The different port numbers make these applications different origins. The API must therefore return appropriate CORS response headers before browser JavaScript can read its responses.

## API Contract Used in This Activity

The Topic 6 backend provides these operations:

| Method | Endpoint | Purpose | Successful status |
|---|---|---|---:|
| `GET` | `/students` | List students | `200` |
| `GET` | `/students/:studentId` | Retrieve one student | `200` |
| `POST` | `/students` | Create a student | `201` |
| `PATCH` | `/students/:studentId` | Update part of a student | `200` |
| `DELETE` | `/students/:studentId` | Delete a student | `204` |

The core activity uses `GET /students` and `POST /students`. Update and delete operations appear in the extension activity.

## Objectives

After completing the activity, you should be able to:

1. Create and run a React application with Vite.
2. Style React components with Tailwind CSS utility classes.
3. Explain how React state affects what the interface renders.
4. Call an Express API asynchronously with `fetch()`.
5. Submit controlled form data as a JSON request body.
6. Interpret successful HTTP responses and structured error responses.
7. represent idle, loading, success, empty, and failure states in the interface.
8. Prevent unintended duplicate requests while an operation is pending.
9. Explain why a browser requires CORS for requests between different origins.
10. Use browser developer tools to inspect requests, responses, and preflight behavior.

## Prerequisites

Before starting, make sure you have:

- the completed Topic 6 `student-api` project;
- a current Node.js release supported by the installed Vite version;
- npm;
- a code editor such as Visual Studio Code;
- a modern browser; and
- two terminal windows.

Verify Node.js and npm:

```bash
node --version
npm --version
```

The instructions assume that the Express API runs at `http://localhost:3000` and the Vite client runs at `http://localhost:5173`.

---

## Step 1 - Prepare the Express API for CORS

Open a terminal in the Topic 6 `student-api` directory. Install the Express CORS middleware:

```bash
npm install cors
```

Open `src/app.js` and import the package:

```javascript
import cors from 'cors';
import express from 'express';
```

Register the middleware after `express.json()` and before the student routes:

```javascript
app.disable('x-powered-by');
app.use(express.json());

app.use(cors({
  origin: 'http://localhost:5173',
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type']
}));
```

Keep the existing route and error middleware after this configuration:

```javascript
app.use('/students', studentsRouter);
app.use(notFoundHandler);
app.use(errorHandler);
```

### Why middleware placement matters

The CORS middleware must run before the route sends its response. Application-level `app.use(cors(...))` also handles the browser's preflight requests for the configured routes.

`origin` must contain the frontend's exact scheme, hostname, and port. These are different origins:

```text
http://localhost:5173
http://localhost:3000
http://127.0.0.1:5173
https://localhost:5173
```

### Checkpoint

Start the API:

```bash
npm run dev
```

Confirm that this address still returns the student collection:

<http://localhost:3000/students>

---

## Step 2 - Create the React Project with Vite

Open a second terminal beside, not inside, the `student-api` directory. Create the frontend:

```bash
npm create vite@latest student-client -- --template react
cd student-client
npm install
```

The Vite React template supplies the React packages, development scripts, and JSX build configuration.

Start the unmodified application once:

```bash
npm run dev
```

Open the local URL printed by Vite. It is normally:

<http://localhost:5173>

Stop the client with `Ctrl+C` before continuing.

### Checkpoint

The workspace should now resemble:

```text
project-folder/
├── student-api/
└── student-client/
    ├── package.json
    ├── vite.config.js
    ├── index.html
    └── src/
```

---

## Step 3 - Install and Configure Tailwind CSS

Inside `student-client`, install Tailwind CSS and its Vite plugin:

```bash
npm install tailwindcss @tailwindcss/vite
```

Replace `vite.config.js` with:

```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()]
});
```

Replace the contents of `src/index.css` with:

```css
@import "tailwindcss";

:root {
  font-family: Inter, ui-sans-serif, system-ui, sans-serif;
  color: #0f172a;
  background: #f8fafc;
}

body {
  min-width: 320px;
  min-height: 100vh;
}

button,
input,
select {
  font: inherit;
}
```

Tailwind CSS v4 automatically detects class names in the project. This basic setup does not require a `tailwind.config.js` file or the older `@tailwind base`, `@tailwind components`, and `@tailwind utilities` directives.

---

## Step 4 - Create the Client Structure

Inside `student-client/src`, create this structure:

```text
src/
├── api/
│   └── students.js
├── components/
│   ├── RequestNotice.jsx
│   ├── StudentForm.jsx
│   └── StudentList.jsx
├── App.jsx
├── index.css
└── main.jsx
```

You may delete the template's unused `App.css` file and logo assets.

### Separation of responsibilities

| File | Responsibility |
|---|---|
| `api/students.js` | HTTP requests and response interpretation |
| `StudentForm.jsx` | Controlled form fields and submission |
| `StudentList.jsx` | Loading, empty, failure, and data rendering |
| `RequestNotice.jsx` | Accessible success or failure feedback |
| `App.jsx` | Shared state and coordination |

---

## Step 5 - Create the Student API Client

Open `src/api/students.js`:

```javascript
const API_BASE_URL = 'http://localhost:3000';

export class ApiError extends Error {
  constructor(message, status = 0, problem = null) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.problem = problem;
  }
}

async function readResponse(response) {
  if (response.status === 204) {
    return null;
  }

  const contentType = response.headers.get('content-type') || '';
  const hasJson = contentType.includes('application/json') ||
    contentType.includes('application/problem+json');
  const body = hasJson ? await response.json() : null;

  if (!response.ok) {
    throw new ApiError(
      body?.detail || body?.title || `Request failed with ${response.status}`,
      response.status,
      body
    );
  }

  return body;
}

export async function getStudents({ signal } = {}) {
  const response = await fetch(`${API_BASE_URL}/students`, { signal });
  return readResponse(response);
}

export async function createStudent(student) {
  const response = await fetch(`${API_BASE_URL}/students`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json'
    },
    body: JSON.stringify(student)
  });

  return readResponse(response);
}

export async function updateStudent(studentId, changes) {
  const response = await fetch(`${API_BASE_URL}/students/${studentId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json'
    },
    body: JSON.stringify(changes)
  });

  return readResponse(response);
}

export async function deleteStudent(studentId) {
  const response = await fetch(`${API_BASE_URL}/students/${studentId}`, {
    method: 'DELETE'
  });

  return readResponse(response);
}
```

### What happens here

- `API_BASE_URL` identifies the Express server.
- `getStudents()` sends `GET /students`.
- `createStudent()` serializes a JavaScript object and sends JSON.
- `response.ok` covers statuses from `200` through `299`.
- `fetch()` does not reject merely because the server returns `400`, `404`, `409`, or `500`; `readResponse()` converts those responses into `ApiError` objects.
- A `204 No Content` response is returned as `null` instead of being parsed as JSON.
- `AbortSignal` allows an obsolete list request to be canceled during component cleanup.

---

## Step 6 - Create the Request Notice Component

Open `src/components/RequestNotice.jsx`:

```jsx
const styles = {
  success: 'border-emerald-200 bg-emerald-50 text-emerald-800',
  error: 'border-rose-200 bg-rose-50 text-rose-800'
};

export default function RequestNotice({ notice, onDismiss }) {
  if (!notice) {
    return null;
  }

  return (
    <div
      className={`flex items-start justify-between gap-4 rounded-xl border p-4 ${styles[notice.type]}`}
      role={notice.type === 'error' ? 'alert' : 'status'}
      aria-live="polite"
    >
      <p>{notice.message}</p>
      <button
        type="button"
        className="font-semibold underline underline-offset-2"
        onClick={onDismiss}
      >
        Dismiss
      </button>
    </div>
  );
}
```

The component renders nothing when there is no message. It uses semantic live-region behavior so assistive technology can announce a request result.

---

## Step 7 - Build a Controlled Student Form

Open `src/components/StudentForm.jsx`:

```jsx
import { useState } from 'react';

const initialForm = {
  student_number: '',
  full_name: '',
  email: '',
  year_level: '1',
  active: true
};

const inputClass =
  'mt-1 w-full rounded-lg border border-slate-300 bg-white px-3 py-2 ' +
  'outline-none transition focus:border-blue-500 focus:ring-2 focus:ring-blue-200';

export default function StudentForm({ onCreate, isSubmitting }) {
  const [form, setForm] = useState(initialForm);

  function handleChange(event) {
    const { name, value, type, checked } = event.target;

    setForm((current) => ({
      ...current,
      [name]: type === 'checkbox' ? checked : value
    }));
  }

  async function handleSubmit(event) {
    event.preventDefault();

    const created = await onCreate({
      ...form,
      student_number: form.student_number.trim(),
      full_name: form.full_name.trim(),
      email: form.email.trim(),
      year_level: Number(form.year_level)
    });

    if (created) {
      setForm(initialForm);
    }
  }

  return (
    <section className="rounded-2xl bg-white p-6 shadow-sm ring-1 ring-slate-200">
      <div className="mb-5">
        <p className="text-sm font-semibold uppercase tracking-wider text-blue-700">
          New record
        </p>
        <h2 className="text-2xl font-bold text-slate-900">Register a student</h2>
      </div>

      <form className="grid gap-4" onSubmit={handleSubmit}>
        <label className="text-sm font-medium text-slate-700">
          Student number
          <input
            className={inputClass}
            name="student_number"
            value={form.student_number}
            onChange={handleChange}
            minLength="8"
            maxLength="20"
            placeholder="2026-00125"
            required
          />
        </label>

        <label className="text-sm font-medium text-slate-700">
          Full name
          <input
            className={inputClass}
            name="full_name"
            value={form.full_name}
            onChange={handleChange}
            minLength="2"
            maxLength="100"
            placeholder="Maria Cruz"
            required
          />
        </label>

        <label className="text-sm font-medium text-slate-700">
          Email address
          <input
            className={inputClass}
            name="email"
            type="email"
            value={form.email}
            onChange={handleChange}
            placeholder="maria@example.edu"
            required
          />
        </label>

        <label className="text-sm font-medium text-slate-700">
          Year level
          <select
            className={inputClass}
            name="year_level"
            value={form.year_level}
            onChange={handleChange}
          >
            {[1, 2, 3, 4, 5].map((year) => (
              <option key={year} value={year}>{year}</option>
            ))}
          </select>
        </label>

        <label className="flex items-center gap-3 text-sm font-medium text-slate-700">
          <input
            className="size-4 rounded border-slate-300 text-blue-600"
            name="active"
            type="checkbox"
            checked={form.active}
            onChange={handleChange}
          />
          Active student
        </label>

        <button
          type="submit"
          disabled={isSubmitting}
          className="mt-2 rounded-lg bg-blue-600 px-4 py-3 font-semibold text-white transition hover:bg-blue-700 disabled:cursor-not-allowed disabled:bg-slate-400"
        >
          {isSubmitting ? 'Saving student...' : 'Save student'}
        </button>
      </form>
    </section>
  );
}
```

### Controlled component behavior

Each form value comes from React state. `onChange` updates that state, and React renders the new value. On submission:

1. `preventDefault()` prevents page navigation.
2. Text values are trimmed.
3. `year_level` is converted from a form string into a number.
4. The parent performs the API request.
5. The form resets only when creation succeeds.
6. The submit button is disabled while the request is pending.

Browser validation improves feedback, but the Express API remains responsible for authoritative validation.

---

## Step 8 - Render Loading, Empty, Failure, and Success States

Open `src/components/StudentList.jsx`:

```jsx
function StudentCard({ student }) {
  return (
    <article className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
      <div className="flex items-start justify-between gap-4">
        <div>
          <h3 className="font-bold text-slate-900">{student.full_name}</h3>
          <p className="text-sm text-slate-500">{student.student_number}</p>
        </div>
        <span className={`rounded-full px-2.5 py-1 text-xs font-semibold ${
          student.active
            ? 'bg-emerald-100 text-emerald-800'
            : 'bg-slate-200 text-slate-700'
        }`}>
          {student.active ? 'Active' : 'Inactive'}
        </span>
      </div>

      <dl className="mt-4 grid gap-2 text-sm">
        <div className="flex justify-between gap-4">
          <dt className="text-slate-500">Email</dt>
          <dd className="text-right text-slate-800">{student.email}</dd>
        </div>
        <div className="flex justify-between gap-4">
          <dt className="text-slate-500">Year level</dt>
          <dd className="font-medium text-slate-800">{student.year_level}</dd>
        </div>
      </dl>
    </article>
  );
}

function LoadingState() {
  return (
    <div className="grid gap-4 sm:grid-cols-2" aria-label="Loading students">
      {[1, 2, 3, 4].map((item) => (
        <div
          key={item}
          className="h-36 animate-pulse rounded-xl bg-slate-200"
        />
      ))}
    </div>
  );
}

export default function StudentList({ status, students, error, onRetry }) {
  if (status === 'loading') {
    return <LoadingState />;
  }

  if (status === 'error') {
    return (
      <div className="rounded-xl border border-rose-200 bg-rose-50 p-5 text-rose-900" role="alert">
        <h3 className="font-bold">Students could not be loaded</h3>
        <p className="mt-1 text-sm">{error}</p>
        <button
          type="button"
          className="mt-4 rounded-lg bg-rose-700 px-4 py-2 font-semibold text-white"
          onClick={onRetry}
        >
          Try again
        </button>
      </div>
    );
  }

  if (status === 'success' && students.length === 0) {
    return (
      <div className="rounded-xl border border-dashed border-slate-300 bg-white p-8 text-center">
        <h3 className="font-bold text-slate-900">No students yet</h3>
        <p className="mt-1 text-sm text-slate-600">
          The request succeeded, but the collection is empty.
        </p>
      </div>
    );
  }

  return (
    <div className="grid gap-4 sm:grid-cols-2">
      {students.map((student) => (
        <StudentCard key={student.id} student={student} />
      ))}
    </div>
  );
}
```

### Why the state branches are separate

- **Loading** means the result is not known yet.
- **Failure** means the application could not obtain a usable result.
- **Empty** means the request succeeded and returned zero records.
- **Success with data** means the collection can be rendered.

An empty collection is not automatically an error.

---

## Step 9 - Coordinate the Application State

Replace `src/App.jsx` with:

```jsx
import { useCallback, useEffect, useState } from 'react';
import { createStudent, getStudents } from './api/students';
import RequestNotice from './components/RequestNotice';
import StudentForm from './components/StudentForm';
import StudentList from './components/StudentList';

export default function App() {
  const [students, setStudents] = useState([]);
  const [listStatus, setListStatus] = useState('loading');
  const [listError, setListError] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [notice, setNotice] = useState(null);

  const loadStudents = useCallback(async (signal) => {
    setListStatus('loading');
    setListError('');

    try {
      const data = await getStudents({ signal });
      setStudents(data.items);
      setListStatus('success');
    } catch (error) {
      if (error.name === 'AbortError') {
        return;
      }

      console.error(error);
      setListError(error.message || 'Check whether the API is running.');
      setListStatus('error');
    }
  }, []);

  useEffect(() => {
    const controller = new AbortController();
    loadStudents(controller.signal);

    return () => controller.abort();
  }, [loadStudents]);

  async function handleCreate(student) {
    setIsSubmitting(true);
    setNotice(null);

    try {
      const created = await createStudent(student);
      setStudents((current) => [...current, created]);
      setListStatus('success');
      setNotice({
        type: 'success',
        message: `${created.full_name} was registered successfully.`
      });
      return true;
    } catch (error) {
      console.error(error);

      const validationMessage = error.problem?.errors
        ?.map((item) => `${item.field}: ${item.message}`)
        .join(' ');

      setNotice({
        type: 'error',
        message: validationMessage || error.message || 'The student could not be saved.'
      });
      return false;
    } finally {
      setIsSubmitting(false);
    }
  }

  return (
    <main className="min-h-screen bg-slate-50">
      <header className="bg-slate-950 text-white">
        <div className="mx-auto max-w-6xl px-6 py-10">
          <p className="text-sm font-semibold uppercase tracking-[0.2em] text-blue-300">
            ITEC 111A · Client-to-Backend Integration
          </p>
          <h1 className="mt-2 text-4xl font-black tracking-tight">
            Student Records
          </h1>
          <p className="mt-3 max-w-2xl text-slate-300">
            A React client consuming the Topic 6 Express API.
          </p>
        </div>
      </header>

      <div className="mx-auto grid max-w-6xl gap-8 px-6 py-8 lg:grid-cols-[22rem_1fr]">
        <StudentForm onCreate={handleCreate} isSubmitting={isSubmitting} />

        <section>
          <div className="mb-5 flex items-end justify-between gap-4">
            <div>
              <p className="text-sm font-semibold uppercase tracking-wider text-blue-700">
                API collection
              </p>
              <h2 className="text-2xl font-bold text-slate-900">Students</h2>
            </div>
            <span className="rounded-full bg-slate-200 px-3 py-1 text-sm font-semibold text-slate-700">
              {students.length} records
            </span>
          </div>

          <div className="mb-5">
            <RequestNotice
              notice={notice}
              onDismiss={() => setNotice(null)}
            />
          </div>

          <StudentList
            status={listStatus}
            students={students}
            error={listError}
            onRetry={() => loadStudents()}
          />
        </section>
      </div>
    </main>
  );
}
```

Keep the Vite-generated `src/main.jsx`, or confirm that it contains:

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import './index.css';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

### State ownership

`App` owns the student collection because both the list and form affect it. The form owns its field values because no other component needs to edit them. This keeps one authoritative value for each piece of state.

### Why use `finally`?

Whether the POST succeeds or fails, `finally` returns the submit button to its enabled state. Without it, a failed request could leave the interface permanently stuck in “Saving student...” mode.

---

## Step 10 - Run the Integrated Applications

Use two terminals.

### Terminal 1 - Express API

```bash
cd student-api
npm run dev
```

Expected address: <http://localhost:3000>

### Terminal 2 - React client

```bash
cd student-client
npm run dev
```

Expected address: <http://localhost:5173>

Open the React client in the browser. Confirm that the two initial students from Topic 6 appear.

### Development note

React Strict Mode may start, clean up, and start an Effect again during development to help reveal missing cleanup logic. The `AbortController` prevents an obsolete list request from updating the component after cleanup.

---

## Step 11 - Test Every Required State

### Test A - Loading

1. Open the browser developer tools.
2. Select the Network panel.
3. Enable network throttling, such as **Slow 3G**.
4. Reload the page.
5. Confirm that the skeleton cards appear before the records.

Expected result: the list displays a visible loading state while `GET /students` is pending.

### Test B - Success

Submit a valid student:

```text
Student number: 2026-00125
Full name: Maria Cruz
Email: maria@example.edu
Year level: 2
Active: checked
```

Expected result:

- the button displays “Saving student...” while pending;
- the button cannot be clicked again while pending;
- the API returns `201 Created`;
- a success notice appears;
- the new record is added to the list; and
- the form resets.

### Test C - Validation failure

Temporarily bypass or alter one browser validation rule, or use developer tools to remove an input constraint. Submit values the API rejects.

Expected result:

- the API returns `400 Bad Request`;
- the response contains structured validation details;
- the form retains the entered values; and
- the interface displays useful field messages.

### Test D - Conflict

Submit the same student number again with otherwise valid data.

Expected result: the API returns `409 Conflict`, and the interface reports that the student number already exists.

### Test E - Network or server failure

1. Stop the Express API.
2. Reload the React client or submit the form.

Expected result: the request rejects, the interface shows a failure message, and the retry control is available for list loading.

Restart the API before continuing.

### Test F - Empty collection

Delete all in-memory records through the API or temporarily start with an empty `students` array.

Expected result: `GET /students` still returns `200`, but the client displays “No students yet” instead of an error.

---

## Step 12 - Inspect CORS and Preflight Behavior

With both applications running:

1. Open the Network panel.
2. Clear existing entries.
3. Submit the student form.
4. Locate the `OPTIONS` request, if shown.
5. Inspect its request and response headers.
6. Inspect the following `POST /students` request.

Look for headers such as:

```http
Origin: http://localhost:5173
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type
```

The API's preflight response should permit the origin, method, and header:

```http
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Methods: GET,POST,PATCH,DELETE
Access-Control-Allow-Headers: Content-Type
```

### Deliberate CORS failure

Temporarily change the Express configuration to a different allowed origin:

```javascript
origin: 'http://localhost:9999'
```

Restart the API and repeat the POST request.

Record what appears in:

- the browser Console;
- the Network panel;
- the Express terminal; and
- the React interface.

Restore `http://localhost:5173` afterward.

### Important interpretation

CORS is enforced by browsers. It controls whether browser JavaScript may read a cross-origin response; it is not authentication or authorization. A request that fails in the browser may still work in `curl`, Postman, Swagger UI, or another server-side client.

---

## Step 13 - Extension Activity: Add Update and Delete

Use the existing `updateStudent()` and `deleteStudent()` functions to complete both features.

### Update requirements

1. Add an **Active/Inactive** toggle to each student card.
2. Send only the changed field:

```javascript
await updateStudent(student.id, {
  active: !student.active
});
```

3. Replace the updated record in React state without reloading the page.
4. Disable the toggle while its PATCH request is pending.
5. Display a success or failure notice.

### Delete requirements

1. Add a **Delete** button to each record.
2. Ask for confirmation before sending the request.
3. Send `DELETE /students/:studentId`.
4. Do not call `response.json()` for the `204 No Content` response.
5. Remove the record from React state only after the API confirms success.
6. Display the empty state after deleting the final record.

### Optional filter

Add controls for **All**, **Active**, and **Inactive**. Either filter the current React array or call the Topic 6 query parameter:

```text
GET /students?active=true
GET /students?active=false
```

Explain which approach you selected and what could make the other approach more appropriate.

---

## Continue to the Laboratory Activity

After completing the implementation, proceed to the separate [Topic 7 laboratory activity](topic-07-react-tailwind-student-client-laboratory-activity.md). It contains the required analysis and reflection questions, completion checklist, submission requirements, assessment rubric, troubleshooting guide, and references.
