# Troubleshooting Log — Backend Containerization

*Project:* Food-Delivery — Backend Container Setup
*Branch:* feature/temidayo
*Date:* [add date]

This log documents the issues encountered while containerizing the backend service and connecting it to MongoDB, along with root causes, solutions, and steps taken — per the project's troubleshooting & documentation requirement.

---

## Issue 1: npm start failed — no start script defined

*Error:*

npm ERR! missing script: start


*Root Cause:*
package.json only defined a server script (nodemon server.js) for local development. Nodemon isn't suitable for a production container (it watches files and auto-restarts, which is unnecessary overhead inside Docker), and the Dockerfile's CMD ["npm", "start"] expects a start script that didn't exist.

*Solution:*
Added a dedicated start script for container use, keeping server for local dev:
json
"scripts": {
  "start": "node server.js",
  "server": "nodemon server.js"
}


*Steps Taken:*
1. Opened package.json.
2. Added "start": "node server.js" under scripts.
3. Rebuilt the image and confirmed npm start ran successfully.

---

## Issue 2: .env file not found when running the container

*Error:*

docker: --env-file: open .env: The system cannot find the file specified.


*Root Cause:*
The docker run command was executed without a .env file present in the working directory — either the file hadn't been created yet in this repo copy, or the command was run from the wrong path.

*Solution:*
Created a .env file in the backend/ directory containing the required environment variables (PORT, MONGO_URL, JWT_SECRET, STRIPE_SECRET_KEY).

*Steps Taken:*
1. Confirmed working directory was backend/.
2. Ran touch .env and populated it with required variables.
3. Confirmed .env is excluded from the Docker image via .dockerignore (secrets should never be baked into an image).
4. Re-ran docker run ... --env-file .env ... successfully.

---

## Issue 3: Mongoose connection failed — uri parameter undefined

*Error:*

MongooseError: The `uri` parameter to `openUri()` must be a string, got "undefined".


*Root Cause:*
No MongoDB instance existed yet at this point — the .env file had no Mongo connection string defined, so process.env.MONGO_URL resolved to undefined when passed to mongoose.connect().

*Solution:*
Stood up MongoDB as a separate Docker container (rather than Atlas) and added the connection string to .env.

*Steps Taken:*
1. Created a persistent volume: docker volume create mongo-data.
2. Created a shared Docker network: docker network create app-network.
3. Ran the MongoDB container on that network with the volume mounted:
   
   docker run -d -p 27017:27017 -v mongo-data:/data/db --network app-network --name mongodb mongo
   
4. Verified it was running via docker logs mongodb.

---

## Issue 4: Backend container exited immediately (Exited (1))

*Error:*
docker ps -a showed backend-container with status Exited (1), and curl http://localhost:4000 returned:

curl: (7) Failed to connect to localhost port 4000: Could not connect to server


*Root Cause:*
Same underlying issue as Issue 3 — even after the MongoDB container was running, the backend container was still missing a valid, correctly-named connection string in .env, so the app crashed on startup during connectDB().

*Solution:*
Diagnosed via docker logs backend-container, which surfaced the original Mongoose "uri undefined" error, pointing back to the .env configuration.

*Steps Taken:*
1. Ran docker ps -a to confirm the container had exited (not just slow to start).
2. Ran docker logs backend-container to capture the exact stack trace.
3. Traced the error to config/db.js and the environment variable it expected.

---

## Issue 5: Environment variable name mismatch (MONGO_URI vs MONGO_URL)

*Error:*
Same Mongoose "uri undefined" error persisted even after adding a Mongo connection string to .env.

*Root Cause:*
Naming mismatch between the .env file and the code:
- .env defined: MONGO_URI=mongodb://mongodb:27017/food-delivery
- config/db.js read: process.env.MONGO_URL (note: URL, not URI)

Since the variable names didn't match, process.env.MONGO_URL was undefined even though a connection string existed under a different key.

*Solution:*
Renamed the variable in .env to match what the code expects:

MONGO_URL=mongodb://mongodb:27017/food-delivery


*Steps Taken:*
1. Compared cat .env output against cat config/db.js line by line.
2. Identified the URI vs URL naming mismatch.
3. Updated .env to use MONGO_URL.
4. Removed the old container: docker rm -f backend-container.
5. Re-ran the backend container on app-network with the corrected .env.
6. Confirmed via docker logs backend-container that the server started and connected to MongoDB successfully.
7. Verified with curl http://localhost:4000 → API Working.

---

## Issue 6: Frontend hardcoded to production backend URL

*Error:*
No error thrown — but a serious silent misconfiguration. src/context/StoreContext.jsx had:
js
const url = "https://food-delivery-backend-5b6g.onrender.com";

This meant the local frontend container would visually load and appear to work, while every API call silently hit the live production backend instead of the local Dockerized one — completely bypassing the local stack we were trying to verify.

*Root Cause:*
The API base URL was hardcoded at the source level rather than driven by an environment variable, so there was no way to point it at a different backend per environment (local vs. production) without editing code.

*Solution:*
Made the URL configurable via a Vite environment variable, with the production URL kept as a safe fallback:
js
const url = import.meta.env.VITE_API_URL || "https://food-delivery-backend-5b6g.onrender.com";

Added frontend/.env:

VITE_API_URL=http://localhost:4000

Since Vite injects env vars into the browser bundle *at build time*, not runtime, the Dockerfile was updated to accept a build argument:
dockerfile
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

And the image was built with:
bash
docker build --build-arg VITE_API_URL=http://localhost:4000 -t frontend-app .


*Steps Taken:*
1. Grepped src/ for hardcoded API references; initial searches for axios/API_URL/localhost:4000 returned nothing.
2. Broadened the search to grep -n "http" src/context/StoreContext.jsx, which surfaced the hardcoded Render URL.
3. Updated the code to read from import.meta.env.VITE_API_URL with a fallback.
4. Created frontend/.env with the local URL, excluded from the image via .dockerignore.
5. Updated the Dockerfile with ARG/ENV to receive the variable at build time (runtime --env-file alone doesn't work for Vite, since the value must be baked into the static build).
6. Rebuilt with --build-arg VITE_API_URL=http://localhost:4000.
7. Verified via browser DevTools → Network tab: the list XHR request showed Request URL: http://localhost:4000/api/food/list, confirming the fix.

---

## Full-Stack Verification (Docker Volume, Network & Container Setup)

With all three containers (frontend-container, backend-container, mongodb) running on the shared app-network, the following flows were tested and confirmed:

| Flow | Method | Result |
|---|---|---|
| *Frontend → Backend* | Opened localhost:3001, inspected Network tab in DevTools | Confirmed request to http://localhost:4000/api/food/list, status 304 |
| *Backend → Database* | Registered a test user via the frontend Login form, then queried MongoDB directly | User document found in food-delivery.users with correctly bcrypt-hashed password |
| *Database → Volume* | Stopped and fully removed the mongodb container, recreated it with the same named volume (-v mongo-data:/data/db), then re-queried | Test user document was still present, confirming the named volume persists data independently of container lifecycle |

*Verification commands used:*
bash
# Backend -> Database check
docker exec -it mongodb mongosh
use food-delivery
show collections
db.users.find().pretty()

# Database -> Volume persistence check
docker stop mongodb
docker rm mongodb
docker run -d -p 27017:27017 -v mongo-data:/data/db --network app-network --name mongodb mongo
docker exec -it mongodb mongosh
use food-delivery
db.users.find().pretty()   # user still present after full container recreation


*Note:* MongoDB database names are case-sensitive. An early check used use Food-Delivery (capitalized, matching the repo folder name) and returned an empty result — this was a different, empty database from the one the app actually connects to (food-delivery, lowercase, as defined in MONGO_URL). Always match the exact case used in the connection string.

---

## Issue 7: Multi-stage frontend build — sh: vite: not found

*Error:*

sh: vite: not found

Occurred when running the multi-stage frontend image, on the CMD ["npm", "run", "preview", ...] step.

*Root Cause:*
The multi-stage runtime stage installed dependencies with npm install --omit=dev, which deliberately skips everything listed under devDependencies — and vite (required for vite preview, the command serving the built app) lives in devDependencies, not dependencies. Since the app's own package.json was copied into the runtime stage, vite was never installed there.

Attempting to explicitly install it alongside the omit flag —
dockerfile
RUN npm install --omit=dev vite
RUN npm install --omit=dev vite@5.3.4

— silently failed to install vite at all, even with a pinned version. npm ls vite confirmed it stayed empty. This appears to be npm treating the explicit package argument as overridden by the already-declared devDependencies entry combined with --omit=dev, rather than as an override.

A stray attempt to add the install as a new line placed after CMD also failed — CMD must be the final instruction in a Dockerfile; commands placed after it do not run during the build.

*Solution:*
Split the install into two separate RUN steps, installing vite with --no-save so npm treats it as a standalone binary install rather than trying to reconcile it against the existing devDependencies declaration:
dockerfile
RUN npm install --omit=dev
RUN npm install vite@5.3.4 --no-save


*Steps Taken:*
1. Confirmed the error only appeared after switching to a multi-stage Dockerfile with --omit=dev.
2. Verified inside the image (docker run -it --rm --entrypoint sh frontend-app) that node_modules/.bin had no vite entry, and npm ls vite returned empty — ruling out a false alarm.
3. Tried installing vite explicitly alongside --omit=dev (both unpinned and pinned to vite@5.3.4) — still failed.
4. Also hit a Docker build-cache red herring: an earlier misplaced install line (after CMD) got cached under the same command text, causing a later, correctly-placed line to appear to "succeed" in the build log while not actually installing anything. Resolved by rebuilding with --no-cache to force fresh execution and confirm behavior.
5. Split the install into two steps with --no-save, rebuilt with --no-cache, and confirmed via the same in-container check that vite now appears in node_modules/.bin.

---

## Issue 8: GHCR push rejected — uppercase in image path

*Error:*
Docker registries (including GHCR) reject image repository paths containing uppercase letters.

*Root Cause:*
The initial tag command used the GitHub username as-is (Keeko-git, capitalized) inside the image path, e.g. ghcr.io/Keeko-git/food-delivery-backend. Registry paths must be fully lowercase, even if the actual GitHub username has capital letters.

*Solution:*
Used the lowercase form of the username in the tag/push commands, while keeping the actual docker login username as originally cased (login itself isn't affected by this restriction):
bash
docker tag backend-app ghcr.io/keeko-git/food-delivery-backend:latest
docker push ghcr.io/keeko-git/food-delivery-backend:latest


*Steps Taken:*
1. Attempted the push with the capitalized username in the image path.
2. Corrected the tag command to use all-lowercase throughout the path.
3. Verified the package appeared under the GitHub account's Packages tab after pushing.

---

## Issue 9: Backend container repeatedly exits — MongoDB container not running

*Error:*
Backend container would start and then stop shortly after, particularly after pulling and re-running images from Docker Hub.

*Root Cause:*
The mongodb container is not set to auto-restart, and had been stopped in a prior session (e.g. during the earlier volume-persistence test) without being explicitly restarted. Since it wasn't running, the backend's connection attempt failed at boot, and the container exited — same underlying symptom as Issue 4, but here the trigger was simply forgetting to bring mongodb back up before starting the backend.

*Solution:*
Started the mongodb container before starting the backend:
bash
docker start mongodb


*Steps Taken:*
1. Ran docker ps -a and confirmed backend-container had exited.
2. Checked docker ps and found mongodb was not in the running containers list.
3. Started mongodb, then restarted the backend container — it connected successfully.

*Note for future runs:* consider adding --restart unless-stopped to long-running containers (mongodb, backend-container, frontend-container) so they automatically come back up after a system reboot or unexpected stop, reducing this class of issue.

---

## Registry Push Summary

Images were built as multi-stage, non-root, and pushed to two of the three required registries:

| Registry | Status | Notes |
|---|---|---|
| Docker Hub | ✅ Pushed | docker login, tag, push — straightforward |
| GHCR | ✅ Pushed | Required a GitHub PAT with write:packages scope; image path must be lowercase |
| AWS ECR | ⬜ Skipped | Not required for this pass; process documented in plan but not executed |

Verified via a full pull-and-run test: local images were deleted (docker rmi), then re-pulled from Docker Hub and re-run on app-network, confirming the pushed images work identically to the locally built versions.

---

## Key Learnings

- *Container hostname resolution:* Containers on the same Docker network resolve each other by container name, not localhost. The backend must reference mongodb://mongodb:27017/..., not mongodb://localhost:27017/....
- *Env var naming must match exactly* between .env and the code reading it (process.env.X) — a single-character mismatch (URI vs URL) caused a full outage that looked like a missing database.
- *Always check docker ps -a, not just docker ps*, when a container isn't responding — a container that exited won't show in the default docker ps list.
- *docker logs <container> is the fastest diagnostic step* for any container that starts and then becomes unreachable.
- Secrets (.env) must be excluded from the image via .dockerignore and injected at runtime with --env-file, never baked into the image with COPY.
- *Frontend env vars (Vite) behave differently from backend env vars (Node/Express).* Node reads process.env at runtime, so --env-file on docker run works fine. Vite bakes import.meta.env.* values into the static bundle at **build time**, so those variables must be passed via --build-arg during docker build, not docker run.
- *Hardcoded URLs are a silent failure mode.* The frontend loaded perfectly and looked fully functional while quietly talking to a production backend instead of the local one — visual success does not equal correct wiring. Always verify actual network requests via DevTools, not just that a page renders.
- *Docker named volumes persist independently of the container.* Deleting and recreating a container with the same -v volume:/path mount retains all prior data — this is the mechanism that makes databases in containers safe to restart/recreate.
- *MongoDB database names are case-sensitive* — a typo in case (Food-Delivery vs food-delivery) silently connects to a different, empty database rather than throwing an error.
- *npm install --omit=dev <package> does not reliably override an existing devDependencies entry.* If a package the runtime needs (like vite, for vite preview) is only declared under devDependencies, explicitly naming it alongside --omit=dev can silently fail to install it. Installing it as a separate step with --no-save avoids the conflict.
- *Docker layer caching can mask a fix.* If an earlier, broken instruction shares identical command text with a later, corrected one, Docker may reuse the cached (broken) result instead of re-running it — even though the file/position changed elsewhere. docker build --no-cache is the reliable way to rule this out when a fix "should" have worked but didn't.
- *Registry image paths must be lowercase*, even if the account username itself has capital letters (applies to GHCR specifically).
- *Long-running containers without --restart policies stop and stay stopped* across sessions/reboots — easy to forget a dependency container (like mongodb) isn't running when debugging an unrelated-looking crash in another container.

---

## Current Status

✅ Backend container builds and runs successfully (multi-stage, non-root user)
✅ Frontend container builds and runs successfully (multi-stage, non-root user)
✅ MongoDB container running with persistent volume
✅ Backend successfully connects to MongoDB over app-network
✅ Frontend correctly configured to call local backend via VITE_API_URL build arg
✅ Full-stack flow verified: Frontend → Backend → Database → Volume persistence
✅ Images pushed to Docker Hub and GHCR; verified via pull-and-run test
✅ curl http://localhost:4000 returns API Working

*Next steps:* Git workflow (commit, push, PR to dev), then LinkedIn post summarizing key learnings.
Task 6: Docker Compose Orchestration

Migrated from manually running individual docker run commands to a single docker-compose.yml orchestrating all three services (mongodb, backend, frontend) together.

What was configured:

All three services defined under one Compose file, reusing the existing named volume (mongo-data) and network (app-network) so prior data and networking behavior carried over unchanged
restart: unless-stopped added to all services — directly resolves the recurring "MongoDB container not running" issue (Issue 9) from earlier tasks, since containers now automatically come back up after a crash or reboot
depends_on used to sequence startup order (mongodb → backend → frontend)
Health checks added for backend and mongodb, verified via:
bash
  docker compose ps
  docker inspect --format='{{json .State.Health}}' backend-container

Confirmed (healthy) status appearing after containers stabilized.

Restart-policy behavior confirmed: after a Docker Desktop restart, docker compose ps showed all containers automatically back to Up/(healthy) without manual intervention.
docker compose down confirmed to stop and remove containers/network while preserving the named volume, per the checklist's requirement not to lose persisted data on cleanup.

Known limitation identified: the backend healthcheck (curl -f http://localhost:4000) only confirms the Node/Express process is responding — it does not verify the MongoDB connection is actually healthy. A backend that's up but silently disconnected from the database would still report (healthy). A more complete check would add a dedicated /health route in server.js that verifies mongoose.connection.readyState and returns a non-200 status if the DB connection is down, so Docker's healthcheck reflects true application health rather than just process liveness. Not implemented in this pass, but noted as a follow-up improvement.

Key Learnings
Container hostname resolution: Containers on the same Docker network resolve each other by container name, not localhost. The backend must reference mongodb://mongodb:27017/..., not mongodb://localhost:27017/....
Env var naming must match exactly between .env and the code reading it (process.env.X) — a single-character mismatch (URI vs URL) caused a full outage that looked like a missing database.
Always check docker ps -a, not just docker ps, when a container isn't responding — a container that exited won't show in the default docker ps list.
docker logs <container> is the fastest diagnostic step for any container that starts and then becomes unreachable.
Secrets (.env) must be excluded from the image via .dockerignore and injected at runtime with --env-file, never baked into the image with COPY.
Frontend env vars (Vite) behave differently from backend env vars (Node/Express). Node reads process.env at runtime, so --env-file on docker run works fine. Vite bakes import.meta.env.* values into the static bundle at build time, so those variables must be passed via --build-arg during docker build, not docker run.
Hardcoded URLs are a silent failure mode. The frontend loaded perfectly and looked fully functional while quietly talking to a production backend instead of the local one — visual success does not equal correct wiring. Always verify actual network requests via DevTools, not just that a page renders.
Docker named volumes persist independently of the container. Deleting and recreating a container with the same -v volume:/path mount retains all prior data — this is the mechanism that makes databases in containers safe to restart/recreate.
MongoDB database names are case-sensitive — a typo in case (Food-Delivery vs food-delivery) silently connects to a different, empty database rather than throwing an error.
npm install --omit=dev <package> does not reliably override an existing devDependencies entry. If a package the runtime needs (like vite, for vite preview) is only declared under devDependencies, explicitly naming it alongside --omit=dev can silently fail to install it. Installing it as a separate step with --no-save avoids the conflict.
Docker layer caching can mask a fix. If an earlier, broken instruction shares identical command text with a later, corrected one, Docker may reuse the cached (broken) result instead of re-running it — even though the file/position changed elsewhere. docker build --no-cache is the reliable way to rule this out when a fix "should" have worked but didn't.
Registry image paths must be lowercase, even if the account username itself has capital letters (applies to GHCR specifically).
Long-running containers without --restart policies stop and stay stopped across sessions/reboots — easy to forget a dependency container (like mongodb) isn't running when debugging an unrelated-looking crash in another container.
Current Status

✅ Backend container builds and runs successfully (multi-stage, non-root user) ✅ Frontend container builds and runs successfully (multi-stage, non-root user) ✅ MongoDB container running with persistent volume ✅ Backend successfully connects to MongoDB over app-network ✅ Frontend correctly configured to call local backend via VITE_API_URL build arg ✅ Full-stack flow verified: Frontend → Backend → Database → Volume persistence ✅ Images pushed to Docker Hub and GHCR; verified via pull-and-run test ✅ curl http://localhost:4000 returns API Working ✅ Full stack orchestrated via docker-compose.yml — single docker compose up -d replaces manual multi-command startup ✅ Health checks configured and verified for backend/mongodb; restart policies confirmed to survive Docker Desktop restart ✅ docker compose down confirmed to preserve the named volume (data persists across cleanup)

Next steps: Git workflow (commit, push, PR to dev), then LinkedIn post summarizing key learnings.