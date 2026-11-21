# 🌊 Egypt Flash Flood Early Warning

> 🇬🇧 English | [🇪🇬 العربية](README.md)

An early warning system for flash floods in Egypt. Here's everything in detail, guys. The process consists of 3 parts, in order:

- **Part 1:** What to install on your machine (one time only)
- **Part 2:** First run of the project (one time only)
- **Part 3:** The commands you'll use every day

All commands are typed in PowerShell from inside the project folder, in the VS Code terminal, i.e. **[Ctrl + `]**.

---

## Part 1: What you need to install first

| # | Software | What for? | Check it's installed |
|---|---|---|---|
| 1 | Git | Pull the project | `git --version` |
| 2 | Docker Desktop | Runs Kafka and Airflow | `docker --version` |
| 3 | SQL Server + SSMS | The project database (Warehouse) | Open SSMS and connect |
| 4 | uv | If you'll run Python scripts from your machine | `uv --version` |

### Installing Git
Download it from https://git-scm.com/download/win and keep clicking Next.

### Installing Docker Desktop
1. Download it from https://www.docker.com/products/docker-desktop and install it.
2. Open it and wait until you see **"Engine running"** at the bottom left.
3. Under Settings → Resources: set memory to at least 4 GB.

### Installing SQL Server
Download SQL Server Developer (free) and SSMS from the Microsoft website and install them. After installing, you need to configure 3 things (we'll do this in Part 2, Step 2).

### Installing uv
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close PowerShell and reopen it, then run `uv --version`.

---

## Part 2: First run (one time only)

### Step 1: Clone the project
```powershell
git clone https://github.com/mohamed-12-tarek/egypt-flash-flood-early-warning.git
cd egypt-flash-flood-early-warning
```

### Step 2: Set up SQL Server

**a) Start the service** (open PowerShell with *Run as administrator*):

```powershell
Start-Service MSSQLSERVER
Set-Service MSSQLSERVER -StartupType Automatic
```

**b) Enable TCP/IP on port 1433:**

1. Press Win+R and type `SQLServerManager16.msc` (if it doesn't open, try 15 or 17).
2. SQL Server Network Configuration → Protocols for MSSQLSERVER.
3. Right-click TCP/IP → Enable.
4. Right-click → Properties → IP Addresses tab → scroll all the way down to **IPAll**:
   - TCP Dynamic Ports → clear it (leave it empty)
   - TCP Port → type `1433`
5. SQL Server Services → right-click SQL Server → Restart.

**d) Open the firewall** (PowerShell as Administrator):

```powershell
New-NetFirewallRule -DisplayName "SQL Server 1433" -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
```

**e) Verify:**

```powershell
Test-NetConnection -ComputerName localhost -Port 1433
```

You must see `TcpTestSucceeded : True`.

### Step 3: The .env file
```powershell
copy .env.example .env
```

Open the `.env` file and change only these two lines (leave the rest as is):

```
SQLSERVER_PASSWORD=your_sa_password
GROQ_API_KEY=your_Groq_key
```

Leave `GROQ_API_KEY` empty as it is — I'll give you one at the end of the project, or I'll have you get one yourself, but only at the end.

Make sure the `SQLSERVER_PASSWORD` you enter is really the correct password you set when you first installed SQL Server, so you don't run into any problems. If you forgot it, go to Claude and tell it:

> "I forgot the password for the sa user, I want to change it in SQL Server using queries."

It will give you 2 queries. Run them with the password you want — that's the same password you'll use in `.env`.

> ⚠️ This `.env` file is yours alone. It must never be sent to anyone or committed.
>
> ⚠️ Don't use `"` or `$` in the password.

### Step 4: Start the containers
Make sure Docker Desktop is running, i.e. it shows **[Engine Running]** at the far bottom left.

Then:

```powershell
docker compose up -d --build
```

The first time takes 5 to 15 minutes. Wait, don't close PowerShell, and don't rerun the command.

When it's done, verify:

```powershell
docker ps
```

You must see 3 containers: `kafka`, `airflow-db` (healthy) and `airflow` — you'll find these names on the far right.

### Step 5: Install the Python libraries inside Airflow
```powershell
docker exec -u root airflow chown -R airflow /opt/uv-venv
```

```powershell
docker exec -u airflow -e UV_PROJECT_ENVIRONMENT=/opt/uv-venv -e UV_HTTP_TIMEOUT=600 -e UV_LINK_MODE=copy airflow bash -c "cd /opt/airflow/project && python -m uv sync > /tmp/uv_sync.log 2>&1"
```

Note that these are two separate commands. The second one runs in the foreground in front of you — don't detach it, so you can see what's happening. It takes 10 to 20 minutes. I'm giving you the timings, guys, so you don't think it's broken or stuck.

When you see a line with `Installed ... packages`, it's done — press **[Ctrl + C]**. Verify:

```powershell
docker exec -u airflow airflow /opt/uv-venv/bin/python -c "import kafka, pandas, pyodbc; print('OK')"
```

It must print `OK`.

> ⚠️ During this step, do NOT run `docker compose up` or restart the container.

### Step 6: Open Airflow
Run this command:

```powershell
docker exec airflow curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/health
```

If the output is `000`, wait a minute or so and try again until it returns `200`.

Then open your browser at http://localhost:8080.

- **User:** `admin`
- Get the password with this command, also typed in the VS Code terminal:

```powershell
docker exec airflow cat /opt/airflow/standalone_admin_password.txt
```

### Step 7: Run the data load (one time only)
On the home page, click **▶** next to `historical_bootstrap` → **Trigger DAG**.

Then click the 3 dots on the right, choose **Graph**, and wait until all three tasks turn green (takes 3–4 minutes):
`setup_db` → `fetch_static_data` → `fetch_historical`

If a task turns red: click it → **Logs** → and send the error to the team or to Claude. If the error is `429 Too Many Requests`, wait a minute and click **Clear** on the task.

### Step 8: Check that the data arrived
Open SQL Server, create a **New Query** and run these queries:

```sql
SELECT COUNT(*) FROM FloodProjectDB.gold.dim_location; -- 15
SELECT city, COUNT(*) FROM FloodProjectDB.bronze.historical_weather GROUP BY city; -- 15 cities
```

### Step 9: The hourly DAG
The DAG is called `hourly_ingestion` and it runs on its own every hour — you don't need to do anything. To test it right away: click **▶** → **Trigger DAG**, then check again in SQL Server:

```sql
SELECT TOP 20 * FROM FloodProjectDB.bronze.live_weather ORDER BY ingested_at DESC;
```

✅ The project is now up and running. The next part is what you'll use every day.

---

## Part 3: Every day when you start working

### When you turn on your machine (or after a restart)
Containers stop when the machine shuts down. Start them in this order:

1) Open Docker Desktop and wait until it says **Engine running**.

2) Make sure SQL Server is running:

```powershell
Get-Service MSSQLSERVER
```

If it's `Stopped` (open PowerShell as Administrator):

```powershell
Start-Service MSSQLSERVER
```

3) Go to the project folder and run:

```powershell
cd egypt-flash-flood-early-warning
docker compose up -d
```

4) Wait 2 to 3 minutes and open http://localhost:8080.

That's all. The data and Python libraries are preserved — you don't need to redo any step from Part 2.

### When you're done working
You don't have to do anything; you can shut down your machine normally. If you want to stop the project properly:

```powershell
docker compose stop
```

### Useful commands

| You want to… | Command |
|---|---|
| See whether the containers are running | `docker ps` |
| View the Airflow logs | `docker logs airflow --tail 30` |
| Restart Airflow only | `docker restart airflow` (and wait two minutes) |
| Recover the forgotten password | `docker exec airflow cat /opt/airflow/standalone_admin_password.txt` |
| See DAGs with errors | `docker exec -u airflow airflow airflow dags list-import-errors` |
| Pull the team's latest updates | `git pull` |
| If someone changed the Dockerfile or docker-compose.yml | `docker compose up -d --build` |

### ⛔ Commands you must NOT run

| Command | Why? |
|---|---|
| `docker compose down -v` | Deletes the Python libraries and Airflow data — you'll have to redo Step 5 |
| `docker compose up` while it's still starting up | Interrupts the initialization and breaks it |
| Sharing the `.env` file | It contains the password and the Groq key |

### 🛠️ Troubleshooting

| Problem | Solution |
|---|---|
| http://localhost:8080 is blank | Wait 2–5 minutes, then `docker logs airflow --tail 20` |
| Task fails with `Login timeout expired` | SQL Server is stopped or TCP/IP isn't enabled: review Step 2 |
| Task fails with `Login failed for user 'sa'` | The password in `.env` is wrong, or SQL login isn't enabled |
| Task fails with `ModuleNotFoundError` | Redo Step 5 |
| Task stuck in `queued` and won't start | `docker restart airflow` and wait 3 minutes |
| Task `ensure_topic` failed | `docker compose up -d kafka` and rerun the task |
| `Cannot open service MSSQLSERVER` | Open PowerShell with *Run as administrator* |
| `port is already allocated` | Another program is using the port (8080 or 19092): close it |
| `airflow-db` not healthy | Wait two minutes. If still not: `docker compose down`, then `docker volume rm egypt-flash-flood-early-warning_airflow-pg`, then `docker compose up -d` |

---

## 📁 Files that must exist in the project

Make sure you have a `.gitignore` with the content below, and if there's anything in your work you don't want pushed to the repo, add it here. `.gitignore` should contain:

```
.env
.venv/
__pycache__/
logs/
```

**Dockerfile:**

```dockerfile
FROM apache/airflow:2.10.2-python3.11

USER root
RUN apt-get update && apt-get install -y curl gnupg unixodbc-dev \
&& curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg \
&& curl -fsSL https://packages.microsoft.com/config/debian/12/prod.list > /etc/apt/sources.list.d/mssql-release.list \
&& apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql18 \
&& mkdir /opt/uv-venv && chown airflow /opt/uv-venv

USER airflow
RUN pip install --no-cache-dir -U uv
```

**.dockerignore:**

```
.venv
__pycache__
logs
.git
```
