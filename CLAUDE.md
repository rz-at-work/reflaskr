# CLAUDE.md - AI Assistant Guide for ReFlaskr

> Last Updated: 2025-11-15
> Repository: ReFlaskr - A modernized Flask blog application

## Table of Contents

1. [Project Overview](#project-overview)
2. [Repository Structure](#repository-structure)
3. [Technology Stack](#technology-stack)
4. [Development Workflows](#development-workflows)
5. [Key Files and Their Purposes](#key-files-and-their-purposes)
6. [Code Conventions](#code-conventions)
7. [Security Considerations](#security-considerations)
8. [Testing Guidelines](#testing-guidelines)
9. [Common Tasks](#common-tasks)
10. [CI/CD Pipeline](#cicd-pipeline)
11. [AI Assistant Guidelines](#ai-assistant-guidelines)

---

## Project Overview

**ReFlaskr** is an updated version of the Flaskr blog application from the Flask tutorial. It's a minimal blog engine demonstrating Flask best practices with modern Docker deployment.

**Key Characteristics:**
- Minimal blog engine with CRUD operations
- SQLite database backend
- Session-based authentication
- Docker containerization with non-root execution
- Automated CI/CD with GitHub Actions
- Production-ready infrastructure with nginx + uWSGI

**Project Status:** Educational/demonstration project with production-ready infrastructure

**Default Credentials:**
- Username: `admin`
- Password: `default`

---

## Repository Structure

```
reflaskr/
├── .github/
│   ├── dependabot.yml              # Dependency update automation (incomplete)
│   └── workflows/
│       ├── build-image.yaml        # Reusable Docker build workflow
│       ├── claude.yml              # AI assistant integration
│       ├── claude-code-review.yml  # Automated PR code review
│       ├── trigger-main-tag.yaml   # Release build trigger
│       └── trigger-pr.yaml         # PR build trigger
│
├── configs/
│   ├── nginx.conf                  # Nginx web server configuration
│   ├── requirements.txt            # Python dependencies (pinned versions)
│   └── uwsgi.ini                   # uWSGI application server config
│
├── src/
│   ├── app.cfg                     # Flask application configuration
│   ├── app.py                      # Main application file (all routes)
│   ├── schema.sql                  # Database schema definition
│   ├── static/
│   │   ├── js/
│   │   │   ├── jquery.min.js       # jQuery 1.12.4 (CDN fallback)
│   │   │   └── script.js           # Custom JavaScript (AJAX, confirmations)
│   │   └── style.css               # Application styles
│   └── templates/
│       ├── layout.html             # Base template
│       ├── login.html              # Login page
│       ├── page_not_found.html     # 404 error page
│       └── show_entries.html       # Main blog page
│
├── .gitignore                      # Git ignore patterns
├── buildImage.sh                   # Docker image build script
├── Dockerfile                      # Multi-stage Docker build
├── README.md                       # User documentation
└── start.sh                        # Container entrypoint script
```

---

## Technology Stack

### Backend
- **Python:** 3.11 (slim-bookworm base image)
- **Web Framework:** Flask 3.1.0
- **Template Engine:** Jinja2 3.1.6
- **Database:** SQLite3 (file-based)
- **Application Server:** uWSGI 2.0.28

### Frontend
- **Templates:** Jinja2 HTML templates
- **Styling:** Custom CSS
- **JavaScript:** jQuery 1.12.4 + custom scripts

### Infrastructure
- **Web Server:** Nginx (reverse proxy)
- **Container Runtime:** Docker
- **CI/CD:** GitHub Actions
- **Container Registry:** GitHub Container Registry (ghcr.io)

### Dependencies (from requirements.txt)
```
click==8.1.7           # CLI utilities
Flask==3.1.0           # Web framework
itsdangerous==2.2.0    # Cryptographic signing
Jinja2==3.1.6          # Template engine
MarkupSafe==3.0.2      # String escaping
uWSGI==2.0.28          # Application server
Werkzeug==3.1.3        # WSGI utilities
```

---

## Development Workflows

### Local Development (Manual)

```bash
# Clone repository
git clone https://github.com/zxpower/reflaskr.git
cd reflaskr/src

# Initialize database
sqlite3 /tmp/flaskr.db < schema.sql

# Run development server
python app.py  # or python3 app.py

# Access at http://localhost:5000/
```

### Docker Development

```bash
# Build image
./buildImage.sh

# Run container
docker run --name reflaskr -d -p 8080:8080 reflaskr:latest

# Access at http://localhost:8080/
```

### Database Management

```bash
# Initialize database
sqlite3 /tmp/flaskr.db < src/schema.sql

# Query database
sqlite3 /tmp/flaskr.db "SELECT * FROM entries;"

# Reset database
rm /tmp/flaskr.db
sqlite3 /tmp/flaskr.db < src/schema.sql
```

---

## Key Files and Their Purposes

### Core Application Files

#### `src/app.py` (Main Application)
**Lines of Code:** ~140
**Purpose:** Contains all application logic, routes, and database operations

**Key Functions:**
- `connect_db()` - Database connection handler
- `init_db()` - Database initialization from schema
- `query_db(query, args, one)` - Database query helper
- `before_request()` - Opens DB connection before each request
- `teardown_request(exception)` - Closes DB connection after request

**Routes:**
- `GET /` → `show_entries()` - Display all blog entries (line 45)
- `POST /add` → `add_entry()` - Create new entry [auth required] (line 55)
- `GET|POST /edit/<id>` → `edit_entry(articleid)` - Edit entry [auth required] (line 68)
- `GET|POST /delete/<id>` → `delete_entry(articleid)` - Delete entry [auth required] (line 78)
- `GET|POST /login` → `login()` - Authentication endpoint (line 88)
- `GET /logout` → `logout()` - Session termination (line 104)
- Error handler: `page_not_found(error)` - Custom 404 page (line 112)

**CRITICAL BUG:** Line 86 in `login()` - Login logic is inverted (sets logged_in=True in else clause)

#### `src/app.cfg` (Configuration)
**Purpose:** Flask application configuration (NOT for production use as-is)

```python
DEBUG = False                    # Debug mode (disabled)
DATABASE = '/tmp/flaskr.db'     # SQLite database path
SECRET_KEY = '7\x14\x8e4...'    # Session encryption key (INSECURE - hardcoded)
USERNAME = 'admin'               # Admin username (INSECURE - hardcoded)
PASSWORD = 'default'             # Admin password (INSECURE - hardcoded)
```

**SECURITY ISSUES:**
- Hardcoded credentials in source control
- Hardcoded secret key in source control
- Should use environment variables

#### `src/schema.sql` (Database Schema)
**Purpose:** Defines database structure

```sql
CREATE TABLE entries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title STRING NOT NULL,
  text STRING NOT NULL
);
```

**Schema Details:**
- Single table design (no user management)
- No timestamps, authors, or metadata
- No indexes beyond primary key
- No foreign key constraints

### Configuration Files

#### `configs/requirements.txt`
**Purpose:** Python dependency specification with pinned versions

**Update Process:**
1. Test compatibility with updated versions
2. Update pinned versions
3. Rebuild Docker image
4. Test thoroughly before deployment

#### `configs/uwsgi.ini`
**Purpose:** uWSGI application server configuration

```ini
module = src.app:app        # Flask application module
uid = 1000                   # Non-root user ID
gid = 1000                   # Non-root group ID
master = true                # Master process
processes = 5                # Worker processes
socket = /tmp/uwsgi.socket  # Unix socket path
chmod-sock = 664             # Socket permissions
vacuum = true                # Clean up on exit
die-on-term = true           # Graceful shutdown
```

#### `configs/nginx.conf`
**Purpose:** Nginx reverse proxy configuration

**Key Settings:**
- Listens on port 8080 (non-privileged)
- Proxies to uWSGI via Unix socket
- Logs to stdout/stderr (container-friendly)
- Non-root compatible cache directories

### Frontend Files

#### `src/static/js/script.js`
**Functions:**
- `postToURL(url, values)` - AJAX POST handler for delete operations
- `confirmation(question, location, post, values)` - Confirmation dialog for destructive actions

#### `src/templates/layout.html`
**Base template** providing:
- Site header and navigation
- Flash message display area
- Content block for child templates
- Footer with credits
- jQuery CDN loading with local fallback

### Infrastructure Files

#### `Dockerfile`
**Multi-stage build process:**
1. Install system dependencies (gcc, make, nginx, sqlite3)
2. Install Python dependencies
3. Create non-root user (uid 1000)
4. Copy application code
5. Configure nginx and uwsgi
6. Set proper permissions

**Security features:**
- Non-root execution
- Minimal image footprint
- Cleanup of build artifacts
- Proper permission management

#### `start.sh` (Container Entrypoint)
**Startup sequence:**
1. Initialize SQLite database from schema.sql
2. Start nginx service
3. Start uwsgi application server

---

## Code Conventions

### Python Style
- **Style Guide:** Follow PEP 8 conventions
- **Indentation:** 4 spaces
- **Line Length:** Not strictly enforced (educational project)
- **Naming:**
  - Functions: `snake_case`
  - Variables: `snake_case`
  - Constants: `UPPER_CASE` (in app.cfg)

### Flask Conventions
- **Route Decorators:** Use `@app.route()` with explicit methods
- **Request Handling:** Access `request.form` for POST data
- **Session Management:** Use Flask's `session` object
- **Database:** Connection per request (before_request/teardown_request)
- **Templates:** Use Jinja2 template inheritance
- **Flash Messages:** Use `flash()` for user feedback

### Database Conventions
- **Queries:** Use parameterized queries (protection against SQL injection)
- **Helper Function:** Prefer `query_db()` over raw database operations
- **Connection Management:** Automatic via request lifecycle hooks

### JavaScript Conventions
- **Style:** jQuery-based
- **AJAX:** Use custom `postToURL()` function
- **User Confirmation:** Use custom `confirmation()` function

### Template Conventions
- **Inheritance:** All pages extend `layout.html`
- **Blocks:** Override `{% block body %}` for page content
- **URLs:** Use `url_for()` for route references
- **Static Files:** Use `url_for('static', filename='...')`

---

## Security Considerations

### CRITICAL Security Issues

#### 1. Hardcoded Credentials (HIGH SEVERITY)
**Location:** `src/app.cfg`
**Issue:** Username and password stored in source control
**Impact:** Anyone with repository access has admin credentials
**Remediation:**
```python
# Use environment variables instead
import os
USERNAME = os.environ.get('ADMIN_USERNAME', 'admin')
PASSWORD = os.environ.get('ADMIN_PASSWORD')  # No default!
```

#### 2. Hardcoded Secret Key (HIGH SEVERITY)
**Location:** `src/app.cfg:3`
**Issue:** SECRET_KEY committed to repository
**Impact:** Session hijacking, cookie forgery
**Remediation:**
```python
SECRET_KEY = os.environ.get('SECRET_KEY')
if not SECRET_KEY:
    raise ValueError("SECRET_KEY environment variable must be set")
```

#### 3. Login Logic Bug (CRITICAL SEVERITY)
**Location:** `src/app.py:86`
**Issue:** Login logic is inverted - sets `logged_in=True` in the else clause
**Impact:** Authentication bypass vulnerability
**Current Code:**
```python
if request.form['username'] != app.config['USERNAME']:
    error = 'Invalid username'
elif request.form['password'] != app.config['PASSWORD']:
    error = 'Invalid password'
else:
    session['logged_in'] = True  # This is CORRECT
```

**Verify this is actually the current state before fixing!**

#### 4. Missing CSRF Protection (MEDIUM SEVERITY)
**Issue:** No CSRF tokens on forms
**Impact:** Cross-site request forgery attacks
**Remediation:** Use Flask-WTF or implement CSRF tokens manually

#### 5. No Password Hashing (HIGH SEVERITY)
**Location:** `src/app.py:88-104`
**Issue:** Plaintext password comparison
**Impact:** Passwords stored in plaintext
**Remediation:**
```python
from werkzeug.security import generate_password_hash, check_password_hash
# Hash passwords and use check_password_hash() for comparison
```

#### 6. No Input Validation/Sanitization
**Location:** Throughout `app.py`
**Issue:** User input not validated or sanitized
**Impact:** Potential XSS, SQL injection (mitigated by parameterized queries)
**Remediation:** Add input validation and use auto-escaping (Jinja2 does this by default)

### Security Best Practices to Follow

**When Making Changes:**
1. Never commit secrets, API keys, or credentials
2. Use environment variables for sensitive configuration
3. Always use parameterized database queries
4. Validate and sanitize all user input
5. Implement CSRF protection on state-changing operations
6. Use HTTPS in production
7. Keep dependencies updated (use Dependabot)
8. Add security headers (CSP, X-Frame-Options, etc.)

---

## Testing Guidelines

### Current State
**Status:** No test infrastructure exists

**Missing Components:**
- No test files
- No pytest configuration
- No test coverage tools
- No CI/CD test execution

### Recommended Testing Approach

#### 1. Unit Tests
**Framework:** pytest
**Test File:** `tests/test_app.py`

**Test Coverage Should Include:**
- Database operations (connect_db, query_db, init_db)
- Route handlers (show_entries, add_entry, etc.)
- Authentication logic (login, logout)
- Session management
- Error handlers

**Example Test Structure:**
```python
import pytest
from src.app import app, init_db

@pytest.fixture
def client():
    app.config['TESTING'] = True
    app.config['DATABASE'] = ':memory:'
    with app.test_client() as client:
        with app.app_context():
            init_db()
        yield client

def test_show_entries(client):
    rv = client.get('/')
    assert b'No entries yet' in rv.data

def test_login_logout(client):
    # Test login
    rv = client.post('/login', data=dict(
        username='admin',
        password='default'
    ), follow_redirects=True)
    assert b'You were logged in' in rv.data

    # Test logout
    rv = client.get('/logout', follow_redirects=True)
    assert b'You were logged out' in rv.data

def test_add_entry_requires_login(client):
    rv = client.post('/add', data=dict(
        title='Test',
        text='Test content'
    ), follow_redirects=True)
    # Should redirect to login or show error
```

#### 2. Integration Tests
**Test Areas:**
- Database initialization
- Full request/response cycle
- Template rendering
- Static file serving

#### 3. Security Tests
**Test Areas:**
- Authentication bypass attempts
- SQL injection attempts (should be blocked by parameterized queries)
- XSS injection attempts
- CSRF attacks

### Adding Tests to CI/CD

**Update `.github/workflows/trigger-pr.yaml`:**
```yaml
- name: Run tests
  run: |
    pip install pytest pytest-cov
    pytest tests/ --cov=src --cov-report=term-missing
```

---

## Common Tasks

### Adding a New Route

1. **Define route in `src/app.py`:**
```python
@app.route('/your-route', methods=['GET', 'POST'])
def your_function():
    # Your logic here
    return render_template('your_template.html')
```

2. **Create template in `src/templates/your_template.html`:**
```html
{% extends "layout.html" %}
{% block body %}
  <!-- Your content here -->
{% endblock %}
```

3. **Update navigation in `src/templates/layout.html`** (if needed)

4. **Test the route locally**

### Modifying the Database Schema

1. **Update `src/schema.sql`:**
```sql
-- Add new columns or tables
ALTER TABLE entries ADD COLUMN created_at TIMESTAMP;
```

2. **Create migration strategy** (SQLite doesn't support all ALTER operations)

3. **Update `app.py`** to handle new fields

4. **Rebuild database:**
```bash
rm /tmp/flaskr.db
sqlite3 /tmp/flaskr.db < src/schema.sql
```

### Adding a New Dependency

1. **Install and test locally:**
```bash
pip install new-package==x.y.z
```

2. **Update `configs/requirements.txt`:**
```
new-package==x.y.z
```

3. **Rebuild Docker image:**
```bash
./buildImage.sh
```

4. **Test in container:**
```bash
docker run --name reflaskr -d -p 8080:8080 reflaskr:latest
```

### Updating Frontend Assets

#### CSS Changes
- Edit `src/static/style.css`
- Test in browser
- Hard refresh (Ctrl+Shift+R) to clear cache

#### JavaScript Changes
- Edit `src/static/js/script.js`
- Test functionality
- Consider browser compatibility

#### Template Changes
- Edit files in `src/templates/`
- Changes take effect immediately (if DEBUG=True)
- Restart server (if DEBUG=False)

### Building and Deploying

#### Local Docker Build
```bash
./buildImage.sh
docker run --name reflaskr -d -p 8080:8080 reflaskr:latest
```

#### CI/CD Build (Automatic)
- **On PR:** Build triggered automatically
- **On Merge to Master:** Build and push to ghcr.io
- **On Tag:** Build and push with tag version

---

## CI/CD Pipeline

### Workflow Overview

```
Pull Request
    ↓
trigger-pr.yaml → build-image.yaml → Docker Build
                                            ↓
                                     ghcr.io (PR tag)

Merge to Master / Tag
    ↓
trigger-main-tag.yaml → build-image.yaml → Docker Build
                                                  ↓
                                           ghcr.io (latest + version tags)
```

### GitHub Actions Workflows

#### 1. `build-image.yaml` (Reusable Workflow)
**Purpose:** Build and push Docker images to GHCR

**Steps:**
1. Checkout repository
2. Generate Docker metadata (tags, labels)
3. Set up QEMU (multi-platform support)
4. Set up Docker Buildx
5. Login to GitHub Container Registry
6. Build and push image
7. Output image digest

**Permissions Required:**
- `contents: read`
- `packages: write`

#### 2. `trigger-pr.yaml`
**Trigger:** Pull request events
**Action:** Calls `build-image.yaml`
**Concurrency:** Cancels in-progress builds for same PR

#### 3. `trigger-main-tag.yaml`
**Trigger:** Push to master branch or any tag
**Action:** Calls `build-image.yaml`
**Result:** Publishes to ghcr.io with `latest` tag

#### 4. `claude.yml` (AI Assistant)
**Trigger:**
- Issue comments
- PR comments
- PR reviews
- Issues opened

**Action:** AI-powered code assistance when @claude is mentioned
**Features:**
- Responds to questions
- Reads CI results on PRs
- Provides code suggestions

**Required Secret:** `CLAUDE_CODE_OAUTH_TOKEN`

#### 5. `claude-code-review.yml` (AI Code Review)
**Trigger:** PR opened or synchronized
**Action:** Automated code review
**Review Areas:**
- Code quality and best practices
- Potential bugs or issues
- Performance considerations
- Security concerns
- Test coverage

**Required Secret:** `CLAUDE_CODE_OAUTH_TOKEN`

### Dependabot Configuration

**File:** `.github/dependabot.yml`

**ISSUE:** Configuration is incomplete (line 8 has empty package-ecosystem)

**Should be:**
```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/configs"
    schedule:
      interval: "weekly"
```

### Container Registry

**Registry:** GitHub Container Registry (ghcr.io)
**Image Path:** `ghcr.io/<owner>/reflaskr`

**Available Tags:**
- `latest` - Latest master build
- `sha-<git-sha>` - Specific commit
- `<tag-name>` - Git tag releases

**Pulling Images:**
```bash
docker pull ghcr.io/<owner>/reflaskr:latest
```

---

## AI Assistant Guidelines

### When Working on This Repository

#### 1. Always Check Security
Before implementing any feature:
- Review the security considerations section
- Never commit hardcoded credentials or secrets
- Use environment variables for sensitive data
- Validate and sanitize user input
- Use parameterized database queries

#### 2. Understand the Architecture
This is a **monolithic Flask application** with:
- Single file for all routes (`src/app.py`)
- No ORM (raw SQLite queries)
- Template-based rendering (Jinja2)
- Session-based authentication
- File-based SQLite database

#### 3. Database Operations
**Always use the `query_db()` helper:**
```python
# Correct
entries = query_db('SELECT * FROM entries ORDER BY id DESC')

# Also correct (for single result)
entry = query_db('SELECT * FROM entries WHERE id = ?', [article_id], one=True)

# Incorrect (bypasses helper)
cur = g.db.execute('SELECT * FROM entries')
```

**Database connection lifecycle:**
- Connection opened in `before_request()`
- Connection closed in `teardown_request()`
- Never manually manage connections

#### 4. Authentication Checks
**For protected routes:**
```python
@app.route('/protected', methods=['POST'])
def protected_route():
    if not session.get('logged_in'):
        abort(401)
    # Your logic here
```

#### 5. Flash Messages
**User feedback pattern:**
```python
flash('Your message here')
return redirect(url_for('route_name'))
```

**Message categories:**
- Default (no category) - Info messages
- Consider adding categories for error/success/warning

#### 6. Template Rendering
**Always use `render_template()`:**
```python
return render_template('template_name.html',
                      variable_name=value,
                      another_var=another_value)
```

**Template inheritance:**
```html
{% extends "layout.html" %}
{% block body %}
  <!-- Your content -->
{% endblock %}
```

#### 7. URL Generation
**Always use `url_for()`:**
```python
# In Python code
redirect(url_for('function_name'))

# In templates
<a href="{{ url_for('function_name') }}">Link</a>
```

#### 8. Static Files
**Reference pattern:**
```html
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<script src="{{ url_for('static', filename='js/script.js') }}"></script>
```

#### 9. Docker Development
**When modifying Docker setup:**
1. Test build locally: `./buildImage.sh`
2. Test container execution: `docker run --name test-reflaskr -p 8080:8080 reflaskr:latest`
3. Check logs: `docker logs test-reflaskr`
4. Verify non-root execution: `docker exec test-reflaskr whoami` (should be "app", not "root")
5. Clean up: `docker rm -f test-reflaskr`

#### 10. Code Review Checklist
Before submitting changes:
- [ ] Security: No hardcoded secrets or credentials
- [ ] Security: Input validation and sanitization
- [ ] Security: Parameterized database queries
- [ ] Code Quality: Follows PEP 8 conventions
- [ ] Functionality: Tested locally (manual or Docker)
- [ ] Templates: Proper Jinja2 syntax and escaping
- [ ] Database: Uses `query_db()` helper
- [ ] Authentication: Protected routes check `session['logged_in']`
- [ ] User Feedback: Appropriate flash messages
- [ ] Documentation: Updated CLAUDE.md if architecture changes

#### 11. Common Pitfalls to Avoid

**Don't:**
- Commit secrets or credentials
- Bypass the `query_db()` helper
- Forget authentication checks on protected routes
- Use string formatting in SQL queries (SQL injection risk)
- Hardcode file paths (use `app.config['DATABASE']`)
- Forget to handle errors (use try/except where appropriate)
- Break the Docker non-root execution model

**Do:**
- Use environment variables for configuration
- Use parameterized queries
- Add authentication checks to protected routes
- Use `url_for()` for URL generation
- Follow the existing code patterns
- Test changes in Docker container
- Update documentation when making architectural changes

#### 12. Debugging Tips

**Local Development:**
```python
# Enable debug mode (src/app.cfg)
DEBUG = True  # Shows detailed error pages

# Use print debugging
print(f"Debug: variable value = {value}", file=sys.stderr)

# Check database state
sqlite3 /tmp/flaskr.db "SELECT * FROM entries;"
```

**Docker Debugging:**
```bash
# View container logs
docker logs reflaskr

# Execute commands in running container
docker exec -it reflaskr /bin/bash

# Check database in container
docker exec -it reflaskr sqlite3 /tmp/flaskr.db "SELECT * FROM entries;"

# Check nginx logs
docker exec reflaskr cat /var/log/nginx/error.log
```

#### 13. Making Pull Requests

**PR Description Should Include:**
1. What changed and why
2. Testing performed (local/Docker)
3. Security considerations
4. Breaking changes (if any)
5. Screenshots (for UI changes)

**PR Checklist:**
- [ ] Code follows project conventions
- [ ] No security issues introduced
- [ ] Tested locally and in Docker
- [ ] Documentation updated (if needed)
- [ ] CLAUDE.md updated (if architecture changed)
- [ ] No hardcoded secrets

#### 14. Understanding the Login Bug

**CRITICAL:** There is a known login logic bug at `src/app.py:86`

**What to check:**
1. Read the actual code to verify current state
2. If bug exists: Login succeeds when credentials are WRONG
3. If fixed: Login succeeds when credentials are CORRECT

**When reviewing login-related code:**
- Verify the logic flow carefully
- Test with both correct and incorrect credentials
- Consider adding unit tests for authentication

#### 15. Environment Variables (Production)

**Required Environment Variables:**
```bash
# Flask Configuration
SECRET_KEY=<random-secure-key>          # Generate with: python -c 'import secrets; print(secrets.token_hex(32))'
FLASK_ENV=production

# Authentication
ADMIN_USERNAME=<secure-username>
ADMIN_PASSWORD=<secure-password-hash>   # Use hashed password, not plaintext

# Database
DATABASE_PATH=/data/flaskr.db           # Persistent volume in production

# Optional
DEBUG=False
```

**Docker Run with Environment Variables:**
```bash
docker run --name reflaskr \
  -e SECRET_KEY="your-secret-key" \
  -e ADMIN_USERNAME="admin" \
  -e ADMIN_PASSWORD="hashed-password" \
  -v /data/flaskr:/data \
  -p 8080:8080 \
  reflaskr:latest
```

---

## Quick Reference

### File Locations
- **Main Application:** `src/app.py`
- **Configuration:** `src/app.cfg`
- **Database Schema:** `src/schema.sql`
- **Dependencies:** `configs/requirements.txt`
- **Templates:** `src/templates/`
- **Static Files:** `src/static/`
- **Docker Build:** `Dockerfile`
- **Workflows:** `.github/workflows/`

### Important Line References
- **Route Definitions:** `src/app.py:45-112`
- **Database Helper:** `src/app.py:20-27`
- **Request Lifecycle:** `src/app.py:30-40`
- **Login Logic:** `src/app.py:88-104` (CHECK FOR BUG)
- **Configuration:** `src/app.cfg:1-5`

### Port Numbers
- **Local Development:** 5000 (Flask default)
- **Docker Container:** 8080 (nginx)
- **uWSGI Socket:** Unix socket at /tmp/uwsgi.socket

### Default Credentials
- **Username:** admin
- **Password:** default
- **Location:** `src/app.cfg:4-5`

### Key Commands
```bash
# Local development
python src/app.py

# Docker build
./buildImage.sh

# Docker run
docker run --name reflaskr -d -p 8080:8080 reflaskr:latest

# Database init
sqlite3 /tmp/flaskr.db < src/schema.sql

# View logs
docker logs reflaskr
```

---

## Document Maintenance

**Update Frequency:** When architectural changes are made

**Update Triggers:**
- New routes added
- Database schema changes
- Configuration changes
- Security fixes
- Major dependency updates
- CI/CD pipeline changes

**Last Updated:** 2025-11-15
**Next Review:** When significant changes are made to codebase

---

## Additional Resources

- **Flask Documentation:** https://flask.palletsprojects.com/
- **Jinja2 Documentation:** https://jinja.palletsprojects.com/
- **uWSGI Documentation:** https://uwsgi-docs.readthedocs.io/
- **Docker Documentation:** https://docs.docker.com/
- **Original Flask Tutorial:** http://flask.pocoo.org/docs/tutorial/

---

*This document is maintained for AI assistants working on the ReFlaskr repository. Keep it updated as the codebase evolves.*
