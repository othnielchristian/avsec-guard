# Setup Guide for AVSEC-Guard

This guide will help you set up your development environment to contribute to AVSEC-Guard.

## Prerequisites

Before you begin, ensure you have the following installed on your system:

- **Git** (v2.30+)
- **Docker** & **Docker Compose** (v2.0+)
- **Python** (v3.11+)
- **pip** (Python package manager)
- **Code Editor** (VS Code, PyCharm, or your preferred IDE)

### Installing Prerequisites

#### macOS
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install required tools
brew install git docker python@3.11

# Install Docker Desktop (includes Docker Compose)
brew install --cask docker
```

#### Linux (Ubuntu/Debian)
```bash
# Update package manager
sudo apt-get update

# Install required tools
sudo apt-get install -y git python3.11 python3-pip

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group (optional, to avoid sudo)
sudo usermod -aG docker $USER
```

#### Windows
- Install **Git for Windows**: https://gitforwindows.org/
- Install **Python 3.11+**: https://www.python.org/downloads/
- Install **Docker Desktop**: https://www.docker.com/products/docker-desktop

## Cloning the Repository

```bash
git clone https://github.com/othnielchristian/avsec-guard.git
cd avsec-guard
```

## Project Structure

```
avsec-guard/
├── docs/                      # Documentation files
│   ├── setup.md              # This file
│   ├── architecture.md        # Architecture documentation
│   └── contributing.md        # Contribution guidelines
├── src/                       # Source code
│   ├── backend/              # FastAPI backend
│   ├── frontend/             # Streamlit frontend
│   ├── detectors/            # Detection rules & ML models
│   ├── collectors/           # Log collection (ETL)
│   ├── simulators/           # Equipment simulators
│   └── utils/                # Utility functions
├── tests/                    # Test suite
│   ├── unit/                 # Unit tests
│   ├── integration/          # Integration tests
│   └── fixtures/             # Test data
├── docker-compose.yml        # Docker Compose configuration
├── Dockerfile                # Docker image definition
├── requirements.txt          # Python dependencies
├── setup.py                  # Python package setup
├── pytest.ini                # Pytest configuration
├── .env.example              # Example environment variables
└── README.md                 # Project overview
```

## Development Environment Setup

### 1. Create a Virtual Environment

It's recommended to use a Python virtual environment to isolate project dependencies.

```bash
# Create virtual environment
python3.11 -m venv venv

# Activate virtual environment
# On macOS/Linux:
source venv/bin/activate
# On Windows:
venv\Scripts\activate
```

### 2. Install Python Dependencies

```bash
# Upgrade pip
pip install --upgrade pip

# Install project dependencies
pip install -r requirements.txt

# Install development dependencies (optional)
pip install -r requirements-dev.txt
```

### 3. Configure Environment Variables

```bash
# Copy example environment file
cp .env.example .env

# Edit .env with your local configuration
# Default values should work for local development
```

### 4. Set Up Docker Containers

```bash
# Build Docker images
docker-compose build

# Start all services
docker-compose up -d

# Verify services are running
docker-compose ps
```

This will start:
- PostgreSQL database
- Zeek network analyzer
- MQTT broker
- Equipment simulators
- Backend API
- Streamlit frontend

### 5. Initialize the Database

```bash
# Apply database migrations
docker-compose exec backend python -m alembic upgrade head

# Create initial data (optional)
docker-compose exec backend python scripts/seed_db.py
```

## Verifying Your Setup

### Check Backend API

```bash
# Access the API documentation
open http://localhost:8000/docs
# or
curl http://localhost:8000/docs
```

You should see the FastAPI interactive Swagger documentation.

### Check Frontend Dashboard

```bash
# Access the Streamlit dashboard
open http://localhost:8501
# or navigate to http://localhost:8501 in your browser
```

### Check Database Connection

```bash
# Connect to PostgreSQL
docker-compose exec db psql -U avsec -d avsec_guard

# List tables (inside psql)
\dt

# Exit psql
\q
```

### Check Zeek Network Analyzer

```bash
# View Zeek logs
docker-compose logs zeek

# Check if logs are being generated
docker-compose exec zeek ls -la /logs/zeek/
```

## Running Tests

### Unit Tests

```bash
# Run all unit tests
pytest tests/unit/ -v

# Run tests for a specific module
pytest tests/unit/detectors/ -v

# Run with coverage report
pytest tests/ --cov=src --cov-report=html
```

### Integration Tests

```bash
# Run integration tests (requires running services)
pytest tests/integration/ -v
```

### Run All Tests

```bash
pytest -v --cov=src
```

## Development Workflow

### 1. Create a Feature Branch

```bash
# Update main branch
git pull origin main

# Create a new feature branch
git checkout -b feature/your-feature-name
```

Branch naming convention:
- `feature/` for new features
- `fix/` for bug fixes
- `docs/` for documentation
- `refactor/` for code refactoring
- `test/` for test additions

### 2. Make Your Changes

- Write or modify code
- Add tests for new functionality
- Update documentation as needed

### 3. Run Tests and Linting

```bash
# Run tests
pytest tests/ -v

# Check code style
flake8 src/

# Format code (automatic)
black src/

# Check for security issues
bandit -r src/

# Check for vulnerabilities with Semgrep
semgrep --config=p/security-audit src/
```

### 4. Commit Your Changes

```bash
# Stage your changes
git add .

# Commit with a clear message
git commit -m "feat: add new detection rule for default credentials"
```

Commit message format:
- `feat:` for new features
- `fix:` for bug fixes
- `docs:` for documentation
- `test:` for test additions
- `refactor:` for code refactoring

### 5. Push and Create a Pull Request

```bash
# Push your branch
git push origin feature/your-feature-name

# Then create a PR on GitHub
```

## Common Development Tasks

### Adding a New Detection Rule

1. Create a new file in `src/detectors/rules/`
2. Implement the rule class inheriting from `BaseRule`
3. Add tests in `tests/unit/detectors/test_rules.py`
4. Update documentation in `docs/detectors.md`

Example:
```python
# src/detectors/rules/default_credentials.py
from src.detectors.base import BaseRule

class DefaultCredentialsRule(BaseRule):
    """Detects default credentials on exposed services."""
    
    def __init__(self):
        super().__init__("DEFAULT_CREDS", "High")
    
    def evaluate(self, event):
        # Implementation
        pass
```

### Adding a New API Endpoint

1. Create a router in `src/backend/routers/`
2. Define models in `src/backend/schemas/`
3. Implement business logic in `src/backend/services/`
4. Add tests in `tests/unit/backend/`

### Adding Equipment Simulator

1. Create simulator class in `src/simulators/`
2. Implement MQTT publish logic
3. Add configuration in Docker Compose
4. Add tests for simulator behavior

### Updating Database Schema

```bash
# Create a new migration
docker-compose exec backend python -m alembic revision --autogenerate -m "add new table"

# Review the migration file in alembic/versions/

# Apply the migration
docker-compose exec backend python -m alembic upgrade head
```

## Troubleshooting

### Docker Issues

```bash
# Check docker service is running
docker ps

# Rebuild containers from scratch
docker-compose down -v
docker-compose build --no-cache
docker-compose up -d

# View logs for a specific service
docker-compose logs -f backend
docker-compose logs -f db
```

### Port Conflicts

If ports are already in use, modify `docker-compose.yml`:

```yaml
services:
  backend:
    ports:
      - "8001:8000"  # Change host port from 8000 to 8001
```

### Database Connection Issues

```bash
# Check database logs
docker-compose logs db

# Reset database
docker-compose down -v
docker-compose up -d db
docker-compose exec db psql -U avsec -d avsec_guard
```

### Python Environment Issues

```bash
# Regenerate virtual environment
rm -rf venv/
python3.11 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

## IDE Setup

### VS Code

1. Install Python extension by Microsoft
2. Install Pylance extension
3. Select interpreter: `Ctrl+Shift+P` → "Python: Select Interpreter" → choose `./venv/bin/python`
4. Install recommended extensions from `.vscode/extensions.json`

### PyCharm

1. Open project settings: `PyCharm` → `Preferences/Settings`
2. Go to `Project: avsec-guard` → `Python Interpreter`
3. Click gear icon → `Add...` → Select `./venv/bin/python`
4. Enable Django support if needed

## Documentation

- **API Documentation**: http://localhost:8000/docs (when running)
- **Architecture**: `docs/architecture.md`
- **Detection Rules**: `docs/detectors.md`
- **Simulators**: `docs/simulators.md`
- **Database Schema**: `docs/database.md`

## Getting Help

- **GitHub Issues**: Report bugs or request features
- **Discussions**: Ask questions and share ideas
- **Documentation**: Read `docs/` for detailed information
- **Team Chat**: (Link to be added for team communication)

## Next Steps

After successfully setting up your development environment:

1. Read `docs/contributing.md` to understand contribution guidelines
2. Check `docs/architecture.md` to understand the system design
3. Explore the codebase starting with `src/` directory
4. Pick an issue from GitHub Issues labeled `good-first-issue`
5. Follow the development workflow to submit your contribution

---

**Questions or Issues?** Please open a GitHub issue or contact the team!

Happy coding! 🚀
