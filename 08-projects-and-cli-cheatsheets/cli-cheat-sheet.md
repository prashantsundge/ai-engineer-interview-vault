## PRODUCTION & NOC OPERATION CLI CHEAT SHEET

### Projects: SharePoint Chatbot & NOC AI Autonomous Project

### Tech Stack: FastAPI, Uvicorn, Redis, Celery, Node.js/NPM

## SECTION 1: API GATEWAY (FastAPI & Uvicorn)

### 1. Core Command Breakdown

```bash
uvicorn app.api.gateway:app --host 0.0.0.0 --port 8000 --reload
```

| Component/Flag          | Meaning & Technical Purpose                                                                                                                                                     | Interview Context / Key Takeaway                                                                                                                                                                                                 |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| uvicorn                 | The ASGI (Asynchronous Server Gateway Interface) server implementation used to run asynchronous Python web frameworks like FastAPI.                                             | Essential for handling concurrent async connections (using async/await).                                                                                                                                                          |
| app.api.gateway:app     | The path to the application instance. Looks inside the app directory → api directory → gateway.py file for an instantiated variable named app.                                   | Tells Uvicorn exactly where the FastAPI entry point is located.                                                                                                                                                                   |
| --host 0.0.0.0         | Binds the server to all available network interfaces on the host machine.                                                                                                       | Crucial Interview Question: 127.0.0.1 only allows localhost access. 0.0.0.0 makes the server accessible from outside the host (e.g., inside Docker containers, other machines on the network, or load balancers).                  |
| --port 8000            | Specifies the network port on which the API gateway will listen for incoming HTTP requests.                                                                                     | Default port for many ASGI applications. Can be changed to avoid port collisions.                                                                                                                                                 |
| --reload               | Enables auto-reload. The server automatically restarts whenever code changes are detected in the project directory.                                                              | Production Warning: Only use this in development. Auto-reload degrades performance and introduces instability in production environments.                                                                                           |

### 2. Basic Troubleshooting Commands (Uvicorn/FastAPI)

Check if port 8000 is already in use (Port Collision):

```bash
# Linux/macOS
lsof -i :8000

# Windows (Command Prompt)
netstat -ano | findstr :8000
```

Kill a stuck process on port 8000:

```bash
# Linux/macOS
kill -9 $(lsof -t -i:8000)

# Windows (where PID is the number found from the netstat command)
taskkill /PID <PID> /F
```

Run Uvicorn in production mode (Multi-workers, No Reload):

```bash
uvicorn app.api.gateway:app --host 0.0.0.0 --port 8000 --workers 4
```

Interview Tip: The rule of thumb for workers is (2 x CPU Cores) + 1.

## SECTION 2: ASYNCHRONOUS TASK QUEUE (Celery)

### 1. Core Command Breakdown

```bash
celery -A app.core.celery_app worker --pool=solo --loglevel=info
```

| Component/Flag          | Meaning & Technical Purpose                                                                                                                                                     | Interview Context / Key Takeaway                                                                                                                                                                                                 |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| celery                  | The core CLI tool for managing distributed task queues.                                                                                                                        | Used for long-running backgrounds tasks (e.g., scanning large SharePoint directories, heavy LLM processing).                                                                                                                      |
| -A app.core.celery_app  | Indicates the application instance (-A stands for Application). Points Celery to the file where the Celery app is initialized (app/core/celery_app.py).                         | Tells Celery where to find configuration specs (like broker URLs and task definitions).                                                                                                                                             |
| worker                  | Starts a worker instance that consumes tasks from the message broker (Redis).                                                                                                   | Without this keyword, Celery won't process tasks; it will just start the administrative CLI.                                                                                                                                      |
| --pool=solo            | Forces Celery to run tasks in a single-threaded execution pool rather than spawning multiple processes.                                                                          | Interview Deep Dive: Celery defaults to a prefork pool (multiprocessing). --pool=solo is highly critical for local debugging on Windows (where prefork has known OS compatibility bugs) or when tracking execution step-by-step.      |
| --loglevel=info        | Sets the verbosity of the execution logs. info displays task received, task succeeded, and execution times.                                                                       | Can be changed to debug for deeper inspection or warning/error for production clean logs.                                                                                                                                          |

### 2. Basic Troubleshooting Commands (Celery)

Inspect active Celery workers to verify connection status:

```bash
celery -A app.core.celery_app status
```

Purge all pending tasks in the queue (Clear stuck backlog):

```bash
celery -A app.core.celery_app purge
```

Warning: This deletes all unlogged, unexecuted tasks currently waiting in the broker.

Monitor Celery visually using Flower (Real-time dashboard):

```bash
pip install flower

celery -A app.core.celery_app flower --port=5555
```

## SECTION 3: MESSAGE BROKER & BACKEND (Redis)

### 1. Essential Redis CLI Commands

Since Celery uses Redis as a broker, troubleshooting often requires logging directly into Redis.

| Command       | Action                                                                 | Technical Context                                                                                                           |
|---------------|------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| redis-cli     | Accesses the interactive Redis terminal interface.                     | Entry point for message broker inspection.                                                                                 |
| ping          | Returns PONG if the Redis server is live and reachable.               | Quickest health check if Celery throws connection errors.                                                                  |
| echo "hello"  | Returns the string passed to it.                                       | Confirms basic read/write text pipeline is functional.                                                                     |
| FLUSHALL      | Destructive. Clears every single key out of all databases.            | Clears out corrupted or stuck queues completely during local debugging.                                                   |
| INFO memory   | Shows real-time RAM usage of the Redis server.                        | Critical for NOC operations to diagnose out-of-memory errors.                                                              |
| MONITOR       | Streams every single command processed by the Redis server in real time.| Excellent for verifying if FastAPI or Celery is actively communicating with the broker.                                   |

### 2. Checking Celery Queue Size via Redis CLI

If tasks are not executing, check if they are reaching the queue:

```bash
redis-cli

> LLEN celery
```

Explanation: `celery` is the default list key name Celery uses in Redis. `LLEN` returns the list length. If this number keeps growing and tasks aren't clearing, your Celery workers are down or stuck.

## SECTION 4: FRONTEND APPLICATION (Node.js & NPM)

### 1. Core Command Breakdown

```bash
frontend > npm start
```

| Component/Flag          | Meaning & Technical Purpose                                                                                                                                                     | Interview Context / Key Takeaway                                                                                                                                                                                                 |
|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| cd frontend             | Navigates into the frontend directory containing package.json.                                                                                                                 | Node commands must be run where package.json resides.                                                                                                                                                                             |
| npm start               | Runs a pre-configured script defined inside package.json under the "scripts" object (often fires up a local Webpack, Vite, or Create React App dev server).                      | Initializes the browser environment, hot-reloading, and asset compilation.                                                                                                                                                        |

### 2. Basic Troubleshooting Commands (NPM/Frontend)

Fix corrupted dependencies / Fresh installation:

If dependencies get mangled or packages are missing:

```bash
# Remove existing dependencies and lockfiles
rm -rf node_modules package-lock.json

# Force clean the local npm cache
npm cache clean --force

# Re-install everything cleanly
npm install
```

Check for known security vulnerabilities in frontend packages:

```bash
npm audit

# To automatically fix safe vulnerabilities:
npm audit fix
```

## SECTION 5: HIGH-FREQUENCY INTERVIEW Q&A (CLI & ARCHITECTURE)

**Q1:** What is the primary difference between running Uvicorn with --host 127.0.0.1 vs --host 0.0.0.0?

**Answer:** 127.0.0.1 binds exclusively to the loopback interface, meaning the API is only accessible from the local machine it is running on. 0.0.0.0 tells Uvicorn to listen on all available network interfaces. If the application is deployed inside a Docker container, using 0.0.0.0 is mandatory so that the host machine can forward requests into the container.

**Q2:** If your Celery tasks are status "PENDING" indefinitely and never execute, how do you troubleshoot using the CLI?

**Answer:**
1. Run `celery -A app.core.celery_app status` to see if the worker is up.
2. Check the message broker using `redis-cli ping` to confirm connectivity.
3. Run `redis-cli LLEN celery` to confirm if tasks are actually arriving in the broker queue.
4. Verify that the worker is using the exact same app instance name (-A) as the FastAPI application initialization code.

**Q3:** Why would you use --pool=solo in Celery? Is it safe for production?

**Answer:** --pool=solo runs tasks inline within a single execution process instead of spawning a worker process pool. It is highly useful for local development on Windows machines where the default prefork pool breaks, or when running step-by-step debugging. It is not recommended for heavy production because it cannot handle concurrent, CPU-heavy task execution effectively.

**Q4:** How do you identify and handle a port conflict on a Linux/Unix server if FastAPI refuses to start?

**Answer:** I check what process is holding the port using `lsof -i :8000`. Once I find the Process ID (PID), I can investigate what it is. If it's a rogue or zombie process, I safely terminate it using `kill -9 <PID>` to free up the port for Uvicorn.
