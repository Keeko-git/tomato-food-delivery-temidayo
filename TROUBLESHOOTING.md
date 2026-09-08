# Troubleshooting Log — Backend Containerization

**Project:** Food-Delivery — Backend Container Setup
**Branch:** `feature/temidayo`
**Date:** [add date]

This log documents the issues encountered while containerizing the backend service and connecting it to MongoDB, along with root causes, solutions, and steps taken — per the project's troubleshooting & documentation requirement.

---

## Issue 1: `npm start` failed — no start script defined

**Error:**
```
npm ERR! missing script: start
```

**Root Cause:**
`package.json` only defined a `server` script (`nodemon server.js`) for local development. Nodemon isn't suitable for a production container (it watches files and auto-restarts, which is unnecessary overhead inside Docker), and the Dockerfile's `CMD ["npm", "start"]` expects a `start` script that didn't exist.

**Solution:**
Added a dedicated `start` script for container use, keeping `server` for local dev:
```json
"scripts": {
  "start": "node server.js",
  "server": "nodemon server.js"
}
```

**Steps Taken:**
1. Opened `package.json`.
2. Added `"start": "node server.js"` under `scripts`.
3. Rebuilt the image and confirmed `npm start` ran successfully.

---

## Issue 2: `.env` file not found when running the container

**Error:**
```
docker: --env-file: open .env: The system cannot find the file specified.
```

**Root Cause:**
The `docker run` command was executed without a `.env` file present in the working directory — either the file hadn't been created yet in this repo copy, or the command was run from the wrong path.

**Solution:**
Created a `.env` file in the `backend/` directory containing the required environment variables (`PORT`, `MONGO_URL`, `JWT_SECRET`, `STRIPE_SECRET_KEY`).

**Steps Taken:**
1. Confirmed working directory was `backend/`.
2. Ran `touch .env` and populated it with required variables.
3. Confirmed `.env` is excluded from the Docker image via `.dockerignore` (secrets should never be baked into an image).
4. Re-ran `docker run ... --env-file .env ...` successfully.

---

## Issue 3: Mongoose connection failed — `uri` parameter undefined

**Error:**
```
MongooseError: The `uri` parameter to `openUri()` must be a string, got "undefined".
```

**Root Cause:**
No MongoDB instance existed yet at this point — the `.env` file had no Mongo connection string defined, so `process.env.MONGO_URL` resolved to `undefined` when passed to `mongoose.connect()`.

**Solution:**
Stood up MongoDB as a separate Docker container (rather than Atlas) and added the connection string to `.env`.

**Steps Taken:**
1. Created a persistent volume: `docker volume create mongo-data`.
2. Created a shared Docker network: `docker network create app-network`.
3. Ran the MongoDB container on that network with the volume mounted:
   ```
   docker run -d -p 27017:27017 -v mongo-data:/data/db --network app-network --name mongodb mongo
   ```
4. Verified it was running via `docker logs mongodb`.

---

## Issue 4: Backend container exited immediately (Exited (1))

**Error:**
`docker ps -a` showed `backend-container` with status `Exited (1)`, and `curl http://localhost:4000` returned:
```
curl: (7) Failed to connect to localhost port 4000: Could not connect to server
```

**Root Cause:**
Same underlying issue as Issue 3 — even after the MongoDB container was running, the backend container was still missing a valid, correctly-named connection string in `.env`, so the app crashed on startup during `connectDB()`.

**Solution:**
Diagnosed via `docker logs backend-container`, which surfaced the original Mongoose "uri undefined" error, pointing back to the `.env` configuration.

**Steps Taken:**
1. Ran `docker ps -a` to confirm the container had exited (not just slow to start).
2. Ran `docker logs backend-container` to capture the exact stack trace.
3. Traced the error to `config/db.js` and the environment variable it expected.

---

## Issue 5: Environment variable name mismatch (`MONGO_URI` vs `MONGO_URL`)

**Error:**
Same Mongoose "uri undef

# Troubleshooting Log — Backend Containerization

**Project:** Food-Delivery — Backend Container Setup
**Branch:** `feature/temidayo`
**Date:** [add date]

This log documents the issues encountered while containerizing the backend service and connecting it to MongoDB, along with root causes, solutions, and steps taken — per the project's troubleshooting & documentation requirement.

---

## Issue 1: `npm start` failed — no start script defined

**Error:**
```
npm ERR! missing script: start
```

**Root Cause:**
`package.json` only defined a `server` script (`nodemon server.js`) for local development. Nodemon isn't suitable for a production container (it watches files and auto-restarts, which is unnecessary overhead inside Docker), and the Dockerfile's `CMD ["npm", "start"]` expects a `start` script that didn't exist.

**Solution:**
Added a dedicated `start` script for container use, keeping `server` for local dev:
```json
"scripts": {
  "start": "node server.js",
  "server": "nodemon server.js"
}
```

**Steps Taken:**
1. Opened `package.json`.
2. Added `"start": "node server.js"` under `scripts`.
3. Rebuilt the image and confirmed `npm start` ran successfully.

---

## Issue 2: `.env` file not found when running the container

**Error:**
```
docker: --env-file: open .env: The system cannot find the file specified.
```

**Root Cause:**
The `docker run` command was executed without a `.env` file present in the working directory — either the file hadn't been created yet in this repo copy, or the command was run from the wrong path.

**Solution:**
Created a `.env` file in the `backend/` directory containing the required environment variables (`PORT`, `MONGO_URL`, `JWT_SECRET`, `STRIPE_SECRET_KEY`).

**Steps Taken:**
1. Confirmed working directory was `backend/`.
2. Ran `touch .env` and populated it with required variables.
3. Confirmed `.env` is excluded from the Docker image via `.dockerignore` (secrets should never be baked into an image).
4. Re-ran `docker run ... --env-file .env ...` successfully.

---

## Issue 3: Mongoose connection failed — `uri` parameter undefined

**Error:**
```
MongooseError: The `uri` parameter to `openUri()` must be a string, got "undefined".
```

**Root Cause:**
No MongoDB instance existed yet at this point — the `.env` file had no Mongo connection string defined, so `process.env.MONGO_URL` resolved to `undefined` when passed to `mongoose.connect()`.

**Solution:**
Stood up MongoDB as a separate Docker container (rather than Atlas) and added the connection string to `.env`.

**Steps Taken:**
1. Created a persistent volume: `docker volume create mongo-data`.
2. Created a shared Docker network: `docker network create app-network`.
3. Ran the MongoDB container on that network with the volume mounted:
   ```
   docker run -d -p 27017:27017 -v mongo-data:/data/db --network app-network --name mongodb mongo
   ```
4. Verified it was running via `docker logs mongodb`.

---

## Issue 4: Backend container exited immediately (Exited (1))

**Error:**
`docker ps -a` showed `backend-container` with status `Exited (1)`, and `curl http://localhost:4000` returned:
```
curl: (7) Failed to connect to localhost port 4000: Could not connect to server
```

**Root Cause:**
Same underlying issue as Issue 3 — even after the MongoDB container was running, the backend container was still missing a valid, correctly-named connection string in `.env`, so the app crashed on startup during `connectDB()`.

**Solution:**
Diagnosed via `docker logs backend-container`, which surfaced the original Mongoose "uri undefined" error, pointing back to the `.env` configuration.

**Steps Taken:**
1. Ran `docker ps -a` to confirm the container had exited (not just slow to start).
2. Ran `docker logs backend-container` to capture the exact stack trace.
3. Traced the error to `config/db.js` and the environment variable it expected.

---

## Issue 5: Environment variable name mismatch (`MONGO_URI` vs `MONGO_URL`)

**Error:**
Same Mongoose "uri undef