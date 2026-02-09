---
name: deploy-website
description: Deploy and serve web projects locally for preview. Automatically detects project type (Node.js, PHP, Python/Django/Flask/Unicorn, Go, Ruby/Rails, Java/Spring Boot, Rust, or static HTML) and starts the appropriate development server.
arguments:
  - name: workspace
    description: Absolute path to the workspace directory to deploy
    required: false
---

# Deploy Website

Automatically detect the project type and start the appropriate development server **IN THE BACKGROUND**, then request a Preview URL with MCP Tool `request_preview`

## Detection Logic

1. **Node.js web project**
   - Look for `package.json` in the workspace root
   - Check if it contains a `dev` or `start` script
   - Execute `npm run dev` (or `npm start` if `dev` is not available)

2. **Static website**
   - Look for `.html` files in the workspace (especially `index.html`)
   - Execute `python3 -m http.server 8000` to serve static files

3. **Other kind of websites**

   Detect project type and use appropriate server:

   - **PHP projects** - Look for `.php` files or `composer.json`
     - Execute `php -S localhost:8000` to serve PHP files
     - If `public/index.php` exists, serve from `public` directory
   - **Python projects**
     - **Django** - Look for `manage.py` and `settings.py`
       - Execute `python manage.py runserver 8000`
     - **Flask** - Look for `app.py` or `wsgi.py`
       - Execute `python app.py` or `flask run --port=8000`
     - **Unicorn/uWSGI** - Look for `gunicorn.conf.py` or `uwsgi.ini`
       - Execute `gunicorn -b 0.0.0.0:8000 app:app` (adjust module name)
     - **Generic** - Look for `requirements.txt` or `pyproject.toml`
       - Execute `python -m http.server 8000`
   - **Go projects** - Look for `go.mod`
     - Execute `go run main.go` or `go run .` (usually handles its own server)
   - **Ruby/Rails** - Look for `Gemfile` with `rails`
     - Execute `rails server -p 8000` or `bundle exec rails server -p 8000`
   - **Java/Spring Boot** - Look for `pom.xml` (Maven) or `build.gradle` (Gradle)
     - Execute `mvn spring-boot:run` or `./gradlew bootRun`
   - **Rust/Axum/Actix** - Look for `Cargo.toml`
     - Execute `cargo run`
     
   - **README-based detection** - If standard detection fails:
     - Search README files (`README.md`, `README.rst`, `README.txt`, `doc/README.md`)
     - Look for keywords like `run`, `start`, `serve`, `dev`, `preview`
     - Extract and execute the suggested command

## Workflow

1. **Check for existing server processes** before starting a new one:

Before attempting to start a new server, check if a similar process is already running and listening on a port:

```bash
# Check for existing Node.js dev servers
ps aux | grep -E "(npm run dev|npm start|node.*server)" | grep -v grep

# Check for existing Python/PHP/Go servers
ps aux | grep -E "(python.*manage.py runserver|python.*app.py|flask run|php -S|go run)" | grep -v grep

# Check listening ports with ss/netstat/lsof
ss -tlnp 2>/dev/null || netstat -tlnp 2>/dev/null || lsof -iTCP -sTCP:LISTEN -P -n

# If a matching process is found listening on a port:
# - DO NOT start a new server
# - Use the existing port number
# - Proceed directly to step 3 to call `request_preview` with that port
```

If an existing server process is detected:
- Extract the port number from the process output
- Skip server startup and proceed to step 3 to request preview URL
- Notify the user that an existing server was detected

2. **Detect the project type and start the appropriate server in the background** (only if no existing server found):

> **Note**: The example commands below run in the foreground. When executing, the model must run these commands in the background (e.g., using the `&` suffix) to avoid blocking the session.
>
> **IMPORTANT**: When starting a server in the background, **CAPTURE AND SAVE THE PID** immediately after launch. This PID must be stored for later process management.

```bash
# Step 1: Node.js project
if [ -f "package.json" ]; then
    npm install  # Ensure dependencies are installed
    if grep -q '"dev"' package.json; then
        npm run dev
    elif grep -q '"start"' package.json; then
        npm start
    fi

# Step 2: PHP project
elif [ -f "composer.json" ] || ls *.php 1> /dev/null 2>&1; then
    if [ -f "public/index.php" ]; then
        php -S localhost:8000 -t public
    else
        php -S localhost:8000
    fi

# Step 3: Python projects
elif [ -f "manage.py" ]; then
    # Django
    python manage.py runserver 8000
elif [ -f "app.py" ] || [ -f "wsgi.py" ]; then
    # Flask or WSGI app
    if command -v flask &> /dev/null; then
        flask run --port=8000
    else
        python app.py || python wsgi.py
    fi
elif [ -f "gunicorn.conf.py" ] || [ -f "uwsgi.ini" ]; then
    # Unicorn/uWSGI
    gunicorn -b 0.0.0.0:8000 app:app
elif [ -f "requirements.txt" ] || [ -f "pyproject.toml" ]; then
    python -m http.server 8000

# Step 4: Go project
elif [ -f "go.mod" ]; then
    go run . || go run main.go

# Step 5: Ruby/Rails project
elif [ -f "Gemfile" ] && grep -q "rails" Gemfile; then
    bundle exec rails server -p 8000 || rails server -p 8000

# Step 6: Java/Spring Boot
elif [ -f "pom.xml" ]; then
    ./mvnw spring-boot:run || mvn spring-boot:run
elif [ -f "build.gradle" ]; then
    ./gradlew bootRun || gradle bootRun

# Step 7: Rust project
elif [ -f "Cargo.toml" ]; then
    cargo run

# Step 8: Static HTML files
elif [ -f "index.html" ] || ls *.html 1> /dev/null 2>&1; then
    python3 -m http.server 8000

# Step 9: README-based detection
else
    # Search README for startup commands
    README_FILE=$(find . -maxdepth 2 -iname "README*" | head -1)
    if [ -n "$README_FILE" ]; then
        # Extract command containing keywords
        grep -E "(run|start|serve|dev|preview)" "$README_FILE" | grep -E "^\s*\`?[a-z]+" | head -1 | \
        sed -E 's/.*\`?([^`]+)$/\1/' | sh
    fi
fi
```

3. **Capture the PID and port number** after starting the server:

When starting a server in the background, immediately capture its PID:

```bash
# Example for Node.js
npm run dev &
SERVER_PID=$!
echo "Server started with PID: $SERVER_PID"

# Example for Python
python manage.py runserver 8000 &
SERVER_PID=$!
echo "Server started with PID: $SERVER_PID"

# Example for PHP
php -S localhost:8000 &
SERVER_PID=$!
echo "Server started with PID: $SERVER_PID"
```

**Store the PID** in a temporary file or environment variable for later use:
```bash
echo $SERVER_PID > /tmp/deploy_website_server.pid
```

Also capture the port number from the server output (or use default 8000).

4. Call MCP Tool `request_preview` with the listening port number to get a preview URL.

5. Present the preview URL to the user:
   - Output a clickable hyperlink pointing to the preview address returned by the tool
   - Inform the user of the server PID for reference

6. Handle IP whitelist issues:
   - If the user reports that access is denied due to IP not being whitelisted, ask the user for their IP address(es) which reported by the deny page
   - Call MCP Tool `request_preview` again with the `additional_ip_whitelist` field to add the user's IP(s) to the whitelist
   - The `additional_ip_whitelist` field accepts multiple IPv4 addresses separated by commas (e.g., `"192.168.1.100,10.0.0.50"`)

## Process Management

### Recording PIDs

When the model starts a server process, **ALWAYS** record the PID:
- Store in a temporary file: `/tmp/deploy_website_server.pid`
- Include PID in user notifications
- Keep track of which workspace the PID belongs to

### Stopping Servers

When the user requests to stop a server, **ALWAYS** use PID-based termination:

```bash
# Read the PID from storage
if [ -f "/tmp/deploy_website_server.pid" ]; then
    SERVER_PID=$(cat /tmp/deploy_website_server.pid)

    # Verify the process exists
    if ps -p $SERVER_PID > /dev/null 2>&1; then
        # Graceful termination first
        kill -TERM $SERVER_PID

        # Wait briefly for graceful shutdown
        sleep 2

        # Force kill if still running
        if ps -p $SERVER_PID > /dev/null 2>&1; then
            kill -9 $SERVER_PID
        fi

        echo "Server (PID: $SERVER_PID) has been stopped"
        rm /tmp/deploy_website_server.pid
    else
        echo "Server process (PID: $SERVER_PID) is not running"
        rm /tmp/deploy_website_server.pid
    fi
else
    echo "No server PID found. Server may have been started manually."
fi
```

**CRITICAL**: Never use `pkill` or `killall` with process names (e.g., `pkill node`, `killall python`) as this may terminate:
- User's own manually-started processes
- Other unrelated development servers
- System processes with similar names

Always prefer PID-based termination for safety.

## Notes

- The server **MUST** run in the background or as a subagent task in order **NOT TO** block the session.
- Always check for existing server processes before starting a new one to avoid port conflicts.
- Default port for static server is 8000.
- For Node.js projects, the port depends on the project configuration.
- The port number can be dynamically changed when conflicted with another service.
- Record and track PIDs for all model-started server processes.
- Use PID-based termination to avoid accidentally killing user processes.
- Make sure dependencies are installed before running:
  - Node.js: `npm install`
  - Python: `pip install -r requirements.txt` (if exists)
  - PHP: `composer install` (if composer.json exists)
  - Ruby/Rails: `bundle install` (if Gemfile exists)
  - Go: `go mod download` (automatically handled by go run)
  - Java: Dependencies are handled by Maven/Gradle wrapper
  - Rust: `cargo fetch` (automatically handled by cargo run)
