# Data Map MVP Implementation Plan

## Phase 1: Project Initialization & Structure
**Goals**: Set up project structure, initialize repositories, and create basic configuration files

### 1.1 Create Project Structure
```bash
mkdir -p datamap/{frontend,backend,infra}
cd datamap
```

### 1.2 Initialize Frontend (Next.js)
```bash
cd frontend
npx create-next-app@latest . --typescript --tailwind --eslint
```

### 1.3 Initialize Backend (FastAPI)
```bash
cd ../backend
poetry init
poetry add fastapi uvicorn pydantic sqlalchemy psycopg2-binary pandas geopandas python-dotenv
poetry add --group dev pytest pytest-cov pytest-asyncio pylint safety
```

### 1.4 Create Initial Project Structure
```
datamap/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   └── styles/
│   ├── public/
│   └── tests/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── core/
│   │   ├── models/
│   │   └── services/
│   ├── tests/
│   └── alembic/
└── infra/
    ├── terraform/
    ├── kubernetes/
    └── docker/
```

### 1.5 Initialize Git Repository
```bash
git init
echo "node_modules/\n__pycache__/\n*.pyc\n.env\n.venv/\n.DS_Store" > .gitignore
git add .
git commit -m "Initial project structure"
```

## Phase 2: Development Environment Setup
**Goals**: Configure local development environment with Docker Compose

### 2.1 Create Docker Compose for Local Development
Create `docker-compose.yml`:
```yaml
version: '3.8'
services:
  frontend:
    build: 
      context: ./frontend
      target: development
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - NEXT_PUBLIC_API_URL=http://localhost:8000

  backend:
    build:
      context: ./backend
      target: development
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
    environment:
      - ENVIRONMENT=development
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/datamap
    depends_on:
      - db

  db:
    image: postgis/postgis:15-3.3
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=datamap
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 2.2 Create Frontend Dockerfile
Create `frontend/Dockerfile`:
```dockerfile
# Development
FROM node:18-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

# Production
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
CMD ["npm", "start"]
```

### 2.3 Create Backend Dockerfile
Create `backend/Dockerfile`:
```dockerfile
# Development
FROM python:3.11-slim AS development
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && \
    poetry config virtualenvs.create false && \
    poetry install --no-interaction --no-ansi
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# Production
FROM python:3.11-slim AS production
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && \
    poetry config virtualenvs.create false && \
    poetry install --no-interaction --no-ansi --no-dev
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Phase 3: Basic Application Setup
**Goals**: Set up core application functionality

### 3.1 Frontend Setup
1. Install required dependencies:
```bash
cd frontend
npm install mapbox-gl three @types/three @types/mapbox-gl
```

2. Create basic map component in `frontend/src/components/Map.tsx`:
```typescript
import { useEffect, useRef } from 'react';
import mapboxgl from 'mapbox-gl';
import 'mapbox-gl/dist/mapbox-gl.css';

export const Map = () => {
  const mapContainer = useRef<HTMLDivElement>(null);
  const map = useRef<mapboxgl.Map | null>(null);

  useEffect(() => {
    if (!mapContainer.current) return;
    
    map.current = new mapboxgl.Map({
      container: mapContainer.current,
      style: 'mapbox://styles/mapbox/streets-v11',
      center: [-74.5, 40],
      zoom: 9
    });

    return () => {
      map.current?.remove();
    };
  }, []);

  return <div ref={mapContainer} style={{ width: '100%', height: '100vh' }} />;
};
```

### 3.2 Backend Setup
1. Create FastAPI application structure:
```bash
cd backend
mkdir -p app/{api,core,models,services}
```

2. Create `backend/app/main.py`:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Data Map API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
async def root():
    return {"message": "Data Map API"}
```

## Phase 4: Testing Setup
**Goals**: Configure testing frameworks and create initial tests

### 4.1 Frontend Testing
Create `frontend/jest.config.js`:
```javascript
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  moduleNameMapper: {
    '^@/components/(.*)$': '<rootDir>/src/components/$1',
  },
};
```

### 4.2 Backend Testing
Create `backend/pytest.ini`:
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
```

## Phase 5: CI/CD Pipeline Setup
**Goals**: Set up GitHub Actions for CI/CD

### 5.1 Create GitHub Actions Workflow
Create `.github/workflows/ci.yml`:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: cd frontend && npm ci
      - run: cd frontend && npm test
      - run: cd frontend && npm run lint

  test-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: cd backend && pip install poetry
      - run: cd backend && poetry install
      - run: cd backend && poetry run pytest
      - run: cd backend && poetry run pylint app tests

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'frontend:latest'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
```

## Phase 6: Infrastructure Setup
**Goals**: Set up infrastructure using Terraform

### 6.1 Create Terraform Configuration
Create `infra/terraform/main.tf`:
```hcl
provider "aws" {
  region = "us-west-2"
}

module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "3.14.0"
  
  name = "datamap-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-west-2a", "us-west-2b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = true
}

module "rds" {
  source = "terraform-aws-modules/rds/aws"
  version = "5.0.0"
  
  identifier = "datamap-db"
  
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  
  db_name  = "datamap"
  username = "postgres"
  port     = "5432"
  
  vpc_security_group_ids = [module.vpc.default_security_group_id]
  subnet_ids             = module.vpc.private_subnets
}
```

## Phase 7: Monitoring & Logging Setup
**Goals**: Set up monitoring and logging infrastructure

### 7.1 Create Prometheus Configuration
Create `infra/prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'backend'
    static_configs:
      - targets: ['backend:8000']
  
  - job_name: 'frontend'
    static_configs:
      - targets: ['frontend:3000']
```

### 7.2 Create Grafana Dashboard
Create `infra/grafana/dashboards/main.json`:
```json
{
  "dashboard": {
    "id": null,
    "title": "Data Map Dashboard",
    "tags": [],
    "timezone": "browser",
    "panels": [
      {
        "title": "API Response Time",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "rate(http_request_duration_seconds_sum[5m])"
          }
        ]
      }
    ]
  }
}
```

## Common Gotchas & Best Practices:

1. **Environment Variables**:
   - Never commit `.env` files
   - Use different environment variables for development/staging/production
   - Use AWS Secrets Manager for production secrets

2. **Docker**:
   - Use multi-stage builds to reduce image size
   - Don't run containers as root
   - Use specific version tags for base images

3. **Security**:
   - Regularly update dependencies
   - Run security scans in CI pipeline
   - Use HTTPS in production
   - Implement rate limiting

4. **Database**:
   - Use migrations for database changes
   - Implement connection pooling
   - Regular backups
   - Use read replicas for scaling

5. **Monitoring**:
   - Set up alerts for critical metrics
   - Monitor application logs
   - Track error rates
   - Monitor database performance

## Getting Started

To get started with the project:

```bash
# Clone repository and install dependencies
git clone <repository-url>
cd datamap

# Start development environment
docker-compose up -d

# Access applications
# Frontend: http://localhost:3000
# Backend: http://localhost:8000
# API Docs: http://localhost:8000/docs
```

This plan provides a solid foundation for your MVP. Each phase builds upon the previous one, and you can implement them incrementally. The infrastructure is set up to be scalable and maintainable, with proper monitoring and logging in place. 