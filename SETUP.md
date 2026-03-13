# Local Development Setup

This guide walks through getting WorldRoot running on your local machine from scratch. It is written for a Windows 11 environment using WSL2, VS Code, and Docker Desktop. Mac and Linux users can skip the WSL2 section and start at Step 2.


## Prerequisites Overview

Here is what you need installed and why:

- **WSL2** -- Gives you a real Linux environment inside Windows. Docker and most dev tooling runs better here than in native Windows.
- **Docker Desktop** -- Runs the database and can run the full stack in containers. Hooks into WSL2 automatically.
- **VS Code** -- Your editor. The WSL extension lets you edit files inside Linux seamlessly from the Windows UI.
- **Node.js 20+** -- Runs the React frontend dev server.
- **Python 3.12+** -- Runs the FastAPI backend.
- **Git** -- Version control. Comes with WSL2's Ubuntu but you should configure it.


## Step 1: Set Up WSL2 (Windows 11)

WSL2 is the foundation. Everything else installs inside it.

### Install WSL2

Open PowerShell as Administrator and run:

```powershell
wsl --install
```

This installs WSL2 with Ubuntu by default. Restart your computer when prompted.

After restart, Ubuntu will open and ask you to create a username and password. These are for the Linux environment only, they do not need to match your Windows credentials.

### Verify it works

Open a new terminal (Windows Terminal is recommended, it ships with Windows 11) and run:

```bash
wsl --version
```

You should see WSL version 2.x. If it says version 1, run:

```powershell
wsl --set-default-version 2
```

### Why this matters

From this point forward, you will do almost all of your development work inside the WSL2 Ubuntu terminal, not in PowerShell or CMD. Your project files should live inside the Linux filesystem (under `~/` or `/home/yourusername/`), not on your Windows drives (not `/mnt/c/`). This is important for performance. Accessing Windows files from WSL2 is slow. Accessing WSL2 files from WSL2 is fast.


## Step 2: Set Up VS Code with WSL

### Install the WSL Extension

1. Open VS Code on Windows
2. Go to Extensions (Ctrl+Shift+X)
3. Search for "WSL" and install the one by Microsoft (also called "Remote - WSL")

### Connect VS Code to WSL

1. Open your WSL2 Ubuntu terminal
2. Navigate to where you want your projects: `cd ~`
3. Type `code .`

The first time you do this, VS Code will install a small server inside WSL2. After that, VS Code opens with a green indicator in the bottom-left corner that says "WSL: Ubuntu." This means VS Code is running against the Linux filesystem. Your terminal panel inside VS Code is now a Linux terminal.

### Recommended VS Code Extensions

Install these inside the WSL environment (VS Code will prompt you to install them in WSL rather than locally):

- ESLint
- Prettier
- Python (Microsoft)
- Pylance
- Tailwind CSS IntelliSense
- Thunder Client (lightweight API testing, alternative to Postman)
- Docker


## Step 3: Install Dev Tools in WSL2

Open your WSL2 terminal (either standalone or inside VS Code) and run these:

### Update Ubuntu packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Python 3.12

Ubuntu may ship with an older Python. Install 3.12:

```bash
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install python3.12 python3.12-venv python3.12-dev -y
```

Verify:

```bash
python3.12 --version
```

### Install Node.js 20 via nvm

nvm (Node Version Manager) is the cleanest way to manage Node versions:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

Close and reopen your terminal, then:

```bash
nvm install 20
nvm use 20
node --version
```

### Install Git and configure it

Git comes with Ubuntu but configure your identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
```

### Set up SSH key for GitHub

```bash
ssh-keygen -t ed25519 -C "your.email@example.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

Copy the output and add it to your GitHub account under Settings > SSH and GPG keys.


## Step 4: Install Docker Desktop

1. Download Docker Desktop for Windows from [docker.com](https://www.docker.com/products/docker-desktop/)
2. During installation, make sure "Use WSL 2 instead of Hyper-V" is checked
3. After installation, open Docker Desktop
4. Go to Settings > Resources > WSL Integration
5. Enable integration with your Ubuntu distro
6. Click "Apply & Restart"

Verify it works from your WSL2 terminal:

```bash
docker --version
docker compose version
```

Both commands should return version numbers. If `docker` is not found, close and reopen your WSL2 terminal.


## Step 5: Clone and Configure WorldRoot

From your WSL2 terminal:

```bash
cd ~
git clone https://github.com/yourusername/worldroot.git
cd worldroot

# Create your local environment file
cp .env.example .env
```

Open the project in VS Code:

```bash
code .
```

Open `.env` and change `JWT_SECRET_KEY` to something unique. You can generate one right in the terminal:

```bash
python3.12 -c "import secrets; print(secrets.token_hex(32))"
```


## Step 6: Start with Docker Compose

```bash
docker compose up --build
```

This starts three containers:
- **frontend** on `http://localhost:5173` -- React dev server with hot reload
- **backend** on `http://localhost:8000` -- FastAPI with auto-reload
- **db** on `localhost:5432` -- PostgreSQL database

The first build takes a few minutes while it downloads images and installs dependencies. Subsequent starts are fast.

Open your browser on Windows and go to `http://localhost:5173`. WSL2 automatically forwards ports to Windows, so localhost just works.

To stop everything:

```bash
docker compose down
```

To stop and wipe the database (fresh start):

```bash
docker compose down -v
```


## Step 7: Running Services Directly (For Active Development)

Docker Compose is great for spinning everything up quickly, but during active development you will often want to run the frontend and backend directly for faster feedback loops. The database stays in Docker either way.

### Start just the database

```bash
docker compose up db -d
```

### Backend

```bash
cd backend

# Create virtual environment
python3.12 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run database migrations
alembic upgrade head

# Start the dev server
uvicorn app.main:app --reload --port 8000
```

The API will be at `http://localhost:8000` and interactive docs at `http://localhost:8000/docs`.

### Frontend

Open a second terminal (Ctrl+Shift+` in VS Code):

```bash
cd frontend

# Install dependencies
npm install

# Start the dev server
npm run dev
```

The frontend will be at `http://localhost:5173`.


## Step 8: Seed Data (Optional)

To populate the database with sample content for development and demos:

```bash
# With Docker
docker compose exec backend python -m app.seed

# Without Docker (from the backend directory with venv active)
python -m app.seed
```

This creates:
- A DM account and a few player accounts
- A sample campaign with invite code
- A set of Codex entries across categories (some locked, some unlocked)
- A collection of materials with properties
- A few recipes for testing the crafting system
- Sample inventory items for each player


## Step 9: Verify Everything Works

1. Open `http://localhost:5173` and confirm the frontend loads
2. Open `http://localhost:8000/docs` and confirm the API docs render
3. Try registering a new account through the frontend
4. If using seed data, log in with the DM account and verify the admin panel loads


## Common Issues

**"docker: command not found" in WSL2**
Docker Desktop's WSL integration may not have activated. Open Docker Desktop > Settings > Resources > WSL Integration and make sure your Ubuntu distro is enabled. Then restart your WSL2 terminal.

**Docker says port 5432 is already in use**
You have a local PostgreSQL instance running. Either stop it or change the port mapping in `docker-compose.yml`.

**Frontend cannot reach the backend**
Make sure the `VITE_API_URL` in your `.env` matches where the backend is running. Default is `http://localhost:8000`.

**Database migrations fail**
Make sure the database container is running (`docker compose up db -d`) and the `DATABASE_URL` in `.env` is correct. If starting fresh, run `docker compose down -v` to wipe the volume and try again.

**Python import errors**
Make sure your virtual environment is activated (you should see `(venv)` in your prompt) and you are in the `backend/` directory.

**Slow file access / hot reload is laggy**
Your project files are probably on the Windows filesystem (`/mnt/c/...`). Move them to the Linux filesystem (`~/worldroot`). This is the most common performance issue with WSL2.

**VS Code opens but does not show WSL indicator**
Make sure you opened the project from inside the WSL2 terminal with `code .`, not from Windows File Explorer.


## Development Tips

- **Backend auto-reload**: FastAPI's `--reload` flag watches for file changes and restarts the server automatically. You will see the reload happen in your terminal.

- **Frontend hot reload**: Vite's dev server handles this. Changes to React components reflect instantly in the browser without a full page refresh.

- **Database GUI**: For inspecting the database visually, DBeaver is a solid free option that runs on Windows and can connect to the database running in WSL2/Docker at `localhost:5432`. The VS Code PostgreSQL extension also works.

- **API testing**: The FastAPI auto-generated docs at `/docs` include a "Try it out" button for every endpoint. Use it during development instead of setting up Postman for every call.

- **Type checking**: Run `npx tsc --noEmit` in the frontend directory to check for TypeScript errors without building. Run `mypy app/` in the backend directory for Python type checking.

- **Terminal multiplexing**: You will frequently need 2-3 terminals open (database, backend, frontend). VS Code's split terminal feature (Ctrl+Shift+5) handles this well, or you can install `tmux` inside WSL2 for more control.
