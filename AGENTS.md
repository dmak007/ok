# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Overview

**ok.py** is a Flask-based web application that serves as a submission and grading platform for educational courses. Originally developed for CS 61A at UC Berkeley, it collects student submissions from the ok-client and provides analysis of student progress through a web interface.

### Core Purpose
- Maintain student code backups submitted via the ok-client
- Enable composition grading with staff comments
- Support autograding of student submissions
- Provide course management and analytics

### Technology Stack
- **Backend Framework**: Flask 1.0.4 with Python 3.5+
- **Database**: SQLAlchemy 1.3.0 with MySQL (via PyMySQL)
- **Caching & Queue**: Redis 2.10.5 with RQ (Redis Queue) for background jobs
- **Authentication**: OAuth 2.0 (Google, Microsoft) via Flask-OAuthlib
- **Storage**: Pluggable storage backend (Local, Azure Blob, or Apache Libcloud)
- **Frontend Assets**: Flask-Assets with webassets for CSS/JS bundling
- **Error Tracking**: Sentry (Raven) and Azure Application Insights
- **Email**: SendGrid for notifications
- **WSGI Server**: Gunicorn 19.9.0

### Architecture Patterns

**Application Factory Pattern**: The app is created via `create_app()` in `server/__init__.py`, which:
- Loads configuration from environment variable `OK_SERVER_CONFIG` or defaults to `server.settings.<env>`
- Initializes extensions (database, cache, login manager, storage, etc.)
- Registers blueprints for different functional areas
- Sets up error handlers and context processors

**Blueprint Organization**:
- `auth` - Authentication and login flows
- `oauth` - OAuth provider endpoints
- `student` - Student-facing views
- `admin` - Administrative interface (mounted at `/admin`)
- `about` - Public information pages (mounted at `/about`)
- `queue` - RQ dashboard for monitoring background jobs (mounted at `/rq`)
- `api_endpoints` - RESTful API (mounted at `/api`)
- `files` - File upload/download handling

**Background Jobs**: Uses RQ (Redis Queue) for asynchronous tasks like:
- Autograding submissions
- Exporting grades to Canvas/bCourses
- Running MOSS plagiarism detection
- Sending email notifications
- Generating analytics

**Storage Abstraction**: Supports multiple storage backends via `server.storage`:
- Local filesystem
- Azure Blob Storage
- Any provider supported by Apache Libcloud

## Building and Running

### Initial Setup
```bash
# Create and activate virtual environment
virtualenv -p python3 env
source env/bin/activate

# Install dependencies
pip install -r requirements.txt

# Create database and seed with sample data
./manage.py createdb
./manage.py seed

# Start development server
./manage.py server
```

The server will be available at `http://localhost:5000`.

### Running Background Workers
```bash
# Requires Redis to be running
./manage.py worker
```

### Key Management Commands

**Database Management**:
- `./manage.py createdb` - Create database tables and setup defaults
- `./manage.py dropdb` - Drop all tables (non-production only)
- `./manage.py resetdb` - Drop, recreate, and seed database (non-production only)
- `./manage.py seed` - Populate database with sample data
- `./manage.py db migrate` - Generate migration script
- `./manage.py db upgrade` - Apply migrations

**Development**:
- `./manage.py server` - Run development server on localhost:5000
- `./manage.py shell` - Open Python REPL with app context
- `./manage.py test` - Run test suite
- `./manage.py worker` - Run RQ background worker

**Utilities**:
- `./manage.py cacheflush` - Clear Redis cache
- `./manage.py generate_session_key` - Generate secret key for sessions
- `./manage.py assets build` - Build frontend assets

### Environment Configuration

The application reads configuration from:
1. `OK_SERVER_CONFIG` environment variable (path to config module)
2. Falls back to `server.settings.<env>` where `<env>` is from `OK_ENV` (defaults to 'dev')

**Available environments**:
- `dev` - Local development with SQLite
- `test` - Testing configuration
- `staging` - Staging environment
- `prod` - Production environment

**Critical environment variables**:
- `SECRET_KEY` - Flask session secret (required)
- `DATABASE_URL` - Database connection string
- `REDIS_URL` - Redis connection string
- `STORAGE_PROVIDER` - Storage backend (LOCAL, AZURE_BLOBS, etc.)
- `GOOGLE_ID` / `GOOGLE_SECRET` - Google OAuth credentials
- `SENTRY_DSN` - Sentry error tracking DSN

## Development Conventions

### Code Style
- Follow [The Elements of Python Style](https://github.com/amontalenti/elements-of-python-style)
- Run `make lint` to check for style issues
- Use `flake8` and `pylint` for linting

### Branch Naming Convention
Format: `<category>/<GithubUsername>/<descriptive-name>`

**Categories**:
- `enhancement` - New features
- `bug` - Bug fixes
- `cleanup` - Technical debt reduction (tests, refactoring, etc.)

**Example**: `enhancement/sumukh/add-moss-integration`

### Testing
- Tests are located in the `tests/` directory
- Run tests with `./manage.py test` or `pytest`
- New features should include corresponding tests
- Test coverage is tracked via Coveralls

### Database Migrations
- Migrations are managed with Flask-Migrate (Alembic)
- Located in `migrations/versions/`
- Always test migrations in both directions (upgrade and downgrade)
- Use descriptive migration messages

### Security Considerations
- All routes are CSRF-protected by default (via Flask-WTF)
- OAuth and API endpoints are explicitly exempted from CSRF
- Use `@transaction` decorator for database operations that need atomicity
- Never commit secrets to version control
- Use environment variables for sensitive configuration

### Model Patterns
- All models inherit from `Model` base class which provides:
  - Automatic `created` and `modified` timestamps
  - Serialization methods
- Use custom SQLAlchemy types for complex data:
  - `Json` - JSON stored as TEXT
  - `JsonBlob` - JSON stored as MEDIUMBLOB
  - `Timezone` - Timezone-aware datetime handling
  - `StringList` - Space-separated list storage
- Use `@transaction` decorator for methods that modify database state

### Logging
- Use Python's standard `logging` module
- Logger is configured in `server/logging/`
- Gunicorn logging is customized in `server/logging/gunicorn.py`
- Errors are automatically sent to Sentry in production

### Asset Management
- Frontend assets are defined in `server/assets.py`
- Use Flask-Assets for bundling and minification
- Run `./manage.py assets build` to compile assets
- Assets are served from `server/static/`

## Deployment

### Docker
- Dockerfile is provided for containerized deployments
- Docker Compose configuration in `docker/docker-compose.yml`
- Supports deployment to Kubernetes (see `kubernetes/`)

### Cloud Platforms
- **Google Container Engine**: Kubernetes deployment (see `kubernetes/kubernetes.md`)
- **Azure**: One-click setup available (see `azure/paas/README.md`)
- **Heroku**: Supported via Procfile
- **Dokku**: Documentation in `documentation/dokku.md`

### Production Checklist
- Set `OK_ENV=prod`
- Configure `SECRET_KEY` (use `./manage.py generate_session_key`)
- Set up external database (MySQL recommended)
- Configure Redis for caching and job queue
- Set up cloud storage (Azure Blob or similar)
- Configure OAuth credentials
- Set up Sentry for error tracking
- Configure SendGrid for email
- Use Gunicorn with appropriate worker count
- Set up SSL/TLS certificates
- Configure `NUM_PROXIES` if behind load balancer

## Key Files and Directories

- `manage.py` - CLI entry point for all management commands
- `server/__init__.py` - Application factory and initialization
- `server/models.py` - Database models (large file, consider splitting)
- `server/controllers/` - Blueprint controllers for different routes
- `server/jobs/` - Background job definitions
- `server/settings/` - Configuration files per environment
- `server/static/` - Frontend assets (CSS, JS, images)
- `server/templates/` - Jinja2 templates
- `tests/` - Test suite
- `migrations/` - Database migration scripts
- `documentation/` - Additional documentation

## Testing with ok-client

To test the server with the ok-client:
1. Follow ok-client setup instructions
2. Build ok-client binary with `ok-publish`
3. Start local ok-server
4. Run ok-client with flags: `--insecure --server localhost:5000`
5. Use demo assignments from `ok-client/demo/`

## Additional Resources

- [API Documentation](documentation/api.md)
- [OAuth Setup](documentation/oauth.md)
- [Migration Guide](documentation/migrations.md)
- [Full Setup Guide](documentation/SETUP.md)
- [V3 Design Document](documentation/v3-design-doc.md)
