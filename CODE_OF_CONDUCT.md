# Finance Audit Tracker — Starter Repo (From Scratch)

This canvas contains a complete starter repository skeleton you can copy into a real repo. It includes backend (Node + Express + TypeScript), frontend (React + Vite + TypeScript + Tailwind), Docker Compose for local dev (Postgres + MinIO), and an `init.sql` to create the core schema.

---

## What’s included (file tree)

```
finance-audit-tracker-starter/
├── README.md
├── .env.example
├── docker-compose.yml
├── backend/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts
│       ├── routes/
│       │   └── audits.ts
│       ├── db/
│       │   └── init.sql
│       └── lib/
│           └── db.ts
├── frontend/
│   ├── package.json
│   ├── index.html
│   └── src/
│       ├── main.tsx
│       └── App.tsx
└── .gitignore
```

---

> **Note:** All files are included below as code blocks. Copy them into your local project or paste into files in your repo.

---

## README.md

````markdown
# Finance Audit Tracker — Starter Repo

This starter repo contains a minimal Finance Audit Tracker with:
- Backend: Node + Express + TypeScript
- Database: PostgreSQL (init.sql included)
- File store: MinIO (S3 compatible) via Docker Compose
- Frontend: React + Vite + TypeScript + Tailwind (minimal)

## Quick start (local)
1. Copy `.env.example` to `.env` and configure values.
2. Start Docker services:
   ```bash
   docker-compose up -d
````

3. Install backend deps and run backend:

   ```bash
   cd backend
   npm install
   npm run dev
   ```
4. Install frontend deps and run frontend:

   ```bash
   cd frontend
   npm install
   npm run dev
   ```

Backend API runs at [http://localhost:4000](http://localhost:4000)
Frontend runs at [http://localhost:5173](http://localhost:5173)

```
```

---

## .env.example

```env
# Postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=audit_tracker
POSTGRES_PORT=5432

# Server
PORT=4000
JWT_SECRET=replace_this_with_a_secure_secret

# MinIO (S3)
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=audit-evidence

# Frontend
VITE_API_URL=http://localhost:4000/api
```

---

## docker-compose.yml

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./backend/src/db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro

  minio:
    image: minio/minio
    command: server /data
    environment:
      MINIO_ROOT_USER: ${S3_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${S3_SECRET_KEY}
    ports:
      - "9000:9000"
    volumes:
      - minio-data:/data

volumes:
  db-data:
  minio-data:
```

---

## backend/package.json

```json
{
  "name": "audit-tracker-backend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "dotenv": "^16.3.1",
    "aws-sdk": "^2.1320.0",
    "jsonwebtoken": "^9.0.0",
    "multer": "^1.4.5-lts.1"
  },
  "devDependencies": {
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.6.0",
    "@types/express": "^4.17.17",
    "@types/node": "^20.5.1"
  }
}
```

---

## backend/tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

---

## backend/src/lib/db.ts

```ts
import { Pool } from 'pg';
import dotenv from 'dotenv';
dotenv.config();

const pool = new Pool({
  host: 'localhost',
  port: Number(process.env.POSTGRES_PORT || 5432),
  user: process.env.POSTGRES_USER,
  password: process.env.POSTGRES_PASSWORD,
  database: process.env.POSTGRES_DB,
});

export default {
  query: (text: string, params?: any[]) => pool.query(text, params),
};
```

---

## backend/src/routes/audits.ts

```ts
import express from 'express';
import db from '../lib/db';
const router = express.Router();

// Create audit
router.post('/', async (req, res) => {
  try {
    const { title, scope, start_date, end_date, owner_id } = req.body;
    const result = await db.query(
      `INSERT INTO audits (title, scope, start_date, end_date, owner_id, status, created_at)
       VALUES ($1,$2,$3,$4,$5,'Draft',NOW()) RETURNING *`,
      [title, scope, start_date, end_date, owner_id]
    );
    res.json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'internal error' });
  }
});

// Get audits list
router.get('/', async (req, res) => {
  const q = await db.query('SELECT * FROM audits ORDER BY created_at DESC LIMIT 100');
  res.json(q.rows);
});

export default router;
```

---

## backend/src/index.ts

```ts
import express from 'express';
import dotenv from 'dotenv';
import bodyParser from 'body-parser';
import auditsRouter from './routes/audits';
import db from './lib/db';

dotenv.config();
const app = express();
app.use(bodyParser.json());

app.get('/health', (req, res) => res.json({ status: 'ok' }));
app.use('/api/audits', auditsRouter);

const port = process.env.PORT || 4000;
app.listen(port, () => console.log(`Server listening on ${port}`));
```

---

## backend/src/db/init.sql

```sql
-- Minimal schema for Finance Audit Tracker

CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  name TEXT,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT,
  role TEXT,
  company_id INT,
  created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE IF NOT EXISTS audits (
  id SERIAL PRIMARY KEY,
  company_id INT,
  title TEXT NOT NULL,
  scope TEXT,
  start_date DATE,
  end_date DATE,
  status TEXT,
  owner_id INT,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP
);

CREATE TABLE IF NOT EXISTS audit_areas (
  id SERIAL PRIMARY KEY,
  audit_id INT REFERENCES audits(id) ON DELETE CASCADE,
  name TEXT,
  description TEXT
);

CREATE TABLE IF NOT EXISTS checklist_items (
  id SERIAL PRIMARY KEY,
  area_id INT REFERENCES audit_areas(id) ON DELETE CASCADE,
  title TEXT,
  description TEXT,
  expected_control TEXT,
  sample_size INT,
  status TEXT
);

CREATE TABLE IF NOT EXISTS data_requests (
  id SERIAL PRIMARY KEY,
  audit_id INT REFERENCES audits(id) ON DELETE CASCADE,
  requester_id INT,
  assignee_id INT,
  due_date DATE,
  status TEXT,
  description TEXT
);

CREATE TABLE IF NOT EXISTS evidence_files (
  id SERIAL PRIMARY KEY,
  request_id INT,
  checklist_item_id INT,
  uploaded_by INT,
  file_path TEXT,
  version INT DEFAULT 1,
  uploaded_at TIMESTAMP DEFAULT now()
);

CREATE TABLE IF NOT EXISTS findings (
  id SERIAL PRIMARY KEY,
  audit_id INT REFERENCES audits(id) ON DELETE CASCADE,
  checklist_item_id INT,
  title TEXT,
  description TEXT,
  severity TEXT,
  owner_id INT,
  due_date DATE,
  status TEXT,
  created_at TIMESTAMP DEFAULT now(),
  closed_at TIMESTAMP
);

CREATE TABLE IF NOT EXISTS activity_logs (
  id SERIAL PRIMARY KEY,
  audit_id INT,
  user_id INT,
  action TEXT,
  meta_json JSONB,
  created_at TIMESTAMP DEFAULT now()
);
```

---

## frontend/package.json

```json
{
  "name": "audit-tracker-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "typescript": "^5.6.0",
    "@types/react": "^18.2.28",
    "@types/react-dom": "^18.2.11"
  }
}
```

---

## frontend/index.html

```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Finance Audit Tracker</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## frontend/src/main.tsx

```ts
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'

createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
)
```

---

## frontend/src/App.tsx

```tsx
import React, { useEffect, useState } from 'react'

type Audit = { id: number; title: string; scope?: string }

export default function App(){
  const [audits, setAudits] = useState<Audit[]>([])

  useEffect(()=>{
    fetch(import.meta.env.VITE_API_URL + '/audits')
      .then(r=>r.json())
      .then(setAudits)
      .catch(console.error)
  },[])

  return (
    <div style={{padding:24,fontFamily:'Arial'}}>
      <h1>Finance Audit Tracker — Starter</h1>
      <section>
        <h2>Audits</h2>
        <ul>
          {audits.map(a => (
            <li key={a.id}><strong>{a.title}</strong> — {a.scope}</li>
          ))}
        </ul>
      </section>
    </div>
  )
}
```

---

## .gitignore

```gitignore
node_modules/
dist/
.env
```

---

## Next steps I can do right now

1. Package this into a downloadable zip and provide a link.
2. Flesh out authentication (JWT + signup/login) with code.
3. Add file upload endpoint (MinIO S3) and sample frontend upload component.
4. Create advanced frontend pages (Audit detail, Requests, Findings) as React components.

Tell me which of the above you want next and I will generate it immediately in this canvas.
