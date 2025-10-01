# Microservices Architecture - Todo Application

A complete full-stack microservices application built with React frontend, Node.js backend services, and PostgreSQL databases. This project demonstrates modern cloud-native architecture, GitOps deployment, and DevOps best practices.

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Browser                             │
│                      http://localhost:3000                          │
└─────────────────┬───────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Frontend Service (React + Nginx)                   │
│                         Port: 3000                                  │
│  • Login/Logout UI        • Todo Dashboard                         │
│  • User Profile           • Responsive Design                       │
│  • React Router           • Tailwind CSS                           │
└──────────┬────────────────────────────┬─────────────────────────────┘
           │                            │
           │                            │
           ▼                            ▼
┌─────────────────────┐      ┌─────────────────────┐
│   Auth Service      │      │   Backend Service   │
│   Port: 3001        │      │   Port: 3002        │
│                     │      │                     │
│ • POST /login       │◄─────┤ • GET /to-do        │
│ • POST /logout      │      │ • POST /to-do       │
│ • GET /is-logged-in │      │ • PUT /to-do/:id    │
│ • GET /health       │      │ • DELETE /to-do/:id │
│                     │      │ • GET /user         │
│ Cookie-based        │      │ • GET /health       │
│ Session Management  │      │                     │
└─────────┬───────────┘      └─────────┬───────────┘
          │                            │
          │                            │
          ▼                            ▼
┌─────────────────────┐      ┌─────────────────────┐
│  Auth Database      │      │  Todo Database      │
│  PostgreSQL         │      │  PostgreSQL         │
│  Port: 5432         │      │  Port: 5433         │
│                     │      │                     │
│  Tables:            │      │  Tables:            │
│  • users            │      │  • todos            │
│  • sessions         │      │  • migrations       │
│  • migrations       │      │                     │
└─────────────────────┘      └─────────────────────┘
```

## 📦 Services

### 🎨 Frontend Service (`code/fe/`)

**Modern React SPA with Tailwind CSS**

- **Technology Stack:**
  - React 18 with Hooks
  - React Router v6 for navigation
  - Axios for API communication
  - Tailwind CSS for styling
  - Heroicons for UI icons
  - Nginx for production serving

- **Key Features:**
  - ✅ Responsive, mobile-first design
  - ✅ Real-time todo updates with optimistic UI
  - ✅ Protected routes with authentication guards
  - ✅ Session management with automatic logout
  - ✅ Modern component architecture with custom hooks
  - ✅ Docker multi-stage builds for optimized production

- **Project Structure:**
  ```
  src/
  ├── components/       # Reusable UI components
  │   ├── TodoForm.js   # Create/edit todo form
  │   ├── TodoItem.js   # Individual todo display
  │   ├── TodoList.js   # Todos collection
  │   └── UserProfile.js
  ├── hooks/           # Custom React hooks
  │   ├── useAuth.js   # Auth state management
  │   └── useTodos.js  # Todo state management
  ├── pages/           # Route pages
  │   ├── Dashboard.js # Main app page
  │   └── Login.js     # Login page
  ├── services/
  │   └── api.js       # Axios configuration
  ├── App.js           # Main app with routing
  └── index.js         # Entry point
  ```

- **Environment Variables:**
  ```env
  REACT_APP_API_BASE_URL=http://localhost:3002    # Backend service
  REACT_APP_AUTH_BASE_URL=http://localhost:3001   # Auth service
  REACT_APP_FRONTEND_URL=http://localhost:3000
  ```

- **Development:**
  ```bash
  cd code/fe
  npm install
  npm start              # Starts on port 3000
  npm run build          # Production build
  npm run docker:build   # Build Docker image
  ```

---

### 🔐 Auth Service (`code/auth/`)

**Authentication & Session Management Service**

- **Technology Stack:**
  - Node.js with Express
  - PostgreSQL with connection pooling
  - bcrypt for password hashing
  - Cookie-based sessions
  - UUID for session IDs

- **Key Features:**
  - ✅ Secure password hashing with bcrypt (10 salt rounds)
  - ✅ Session management with httpOnly cookies
  - ✅ Database migration system with rollback support
  - ✅ Health check endpoint
  - ✅ CORS configuration for frontend integration
  - ✅ Sample users pre-loaded for testing

- **API Endpoints:**
  | Method | Endpoint | Description | Authentication |
  |--------|----------|-------------|----------------|
  | POST | `/login` | User authentication | ❌ Public |
  | POST | `/logout` | End user session | ✅ Required |
  | GET | `/is-logged-in` | Validate session | ✅ Required |
  | GET | `/health` | Service health check | ❌ Public |

- **Request/Response Examples:**

  **Login:**
  ```bash
  curl -X POST http://localhost:3001/login \
    -H "Content-Type: application/json" \
    -d '{
      "username": "admin",
      "password": "password123"
    }' \
    -c cookies.txt
  ```
  
  Response:
  ```json
  {
    "success": true,
    "message": "Login successful",
    "user": {
      "id": 1,
      "username": "admin"
    }
  }
  ```

  **Session Validation:**
  ```bash
  curl http://localhost:3001/is-logged-in -b cookies.txt
  ```
  
  Response:
  ```json
  {
    "success": true,
    "message": "User is logged in",
    "isLoggedIn": true,
    "user": {
      "id": 1,
      "username": "admin"
    }
  }
  ```

- **Database Schema:**
  ```sql
  -- users table
  CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );

  -- sessions table (optional persistent storage)
  CREATE TABLE sessions (
    session_id VARCHAR(255) PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP
  );
  ```

- **Sample Users:**
  | Username | Password | Role |
  |----------|----------|------|
  | admin | password123 | Admin |
  | user1 | password123 | User |
  | testuser | password123 | User |

- **Environment Variables:**
  ```env
  PORT=3001
  DB_HOST=localhost
  DB_PORT=5432
  DB_NAME=auth_db
  DB_USER=postgres
  DB_PASSWORD=password
  NODE_ENV=production
  ```

- **Development:**
  ```bash
  cd code/auth
  npm install
  npm run dev            # Development with nodemon
  npm run migrate        # Run migrations
  npm run migrate:status # Check migration status
  npm run migrate:rollback # Rollback last migration
  ```

---

### 📝 Backend Service (`code/be/`)

**Todo CRUD Operations Service**

- **Technology Stack:**
  - Node.js with Express
  - PostgreSQL with pg connection pool
  - Axios for auth service integration
  - Cookie-parser for session validation
  - Database migration system

- **Key Features:**
  - ✅ Full CRUD operations for todos
  - ✅ Authentication middleware (validates with auth service)
  - ✅ User context from session
  - ✅ Database health monitoring
  - ✅ Migration system with rollback support
  - ✅ Comprehensive error handling
  - ✅ API documentation endpoint

- **API Endpoints:**
  | Method | Endpoint | Description | Auth Required |
  |--------|----------|-------------|---------------|
  | GET | `/` | API documentation | ❌ |
  | GET | `/health` | Health check (DB + Auth) | ❌ |
  | GET | `/to-do` | Get all todos | ✅ |
  | GET | `/to-do/:id` | Get todo by ID | ✅ |
  | POST | `/to-do` | Create new todo | ✅ |
  | PUT | `/to-do/:id` | Update todo | ✅ |
  | DELETE | `/to-do/:id` | Delete todo | ✅ |
  | GET | `/user` | Get current user info | ✅ |

- **Request/Response Examples:**

  **Get All Todos:**
  ```bash
  curl http://localhost:3002/to-do -b cookies.txt
  ```
  
  Response:
  ```json
  {
    "success": true,
    "data": [
      {
        "id": 1,
        "title": "Learn Microservices",
        "description": "Study microservices architecture",
        "completed": false,
        "created_at": "2025-10-01T10:00:00.000Z"
      }
    ]
  }
  ```

  **Create Todo:**
  ```bash
  curl -X POST http://localhost:3002/to-do \
    -H "Content-Type: application/json" \
    -b cookies.txt \
    -d '{
      "title": "Deploy to Kubernetes",
      "description": "Set up GKE cluster and deploy services",
      "completed": false
    }'
  ```

  **Update Todo:**
  ```bash
  curl -X PUT http://localhost:3002/to-do/1 \
    -H "Content-Type: application/json" \
    -b cookies.txt \
    -d '{
      "title": "Learn Microservices - Completed",
      "description": "Finished studying microservices",
      "completed": true
    }'
  ```

  **Delete Todo:**
  ```bash
  curl -X DELETE http://localhost:3002/to-do/1 -b cookies.txt
  ```

  **Health Check:**
  ```bash
  curl http://localhost:3002/health
  ```
  
  Response:
  ```json
  {
    "success": true,
    "message": "Todo service is running",
    "timestamp": "2025-10-01T12:00:00.000Z",
    "uptime": 3600.5,
    "database": "connected",
    "authService": "connected"
  }
  ```

- **Database Schema:**
  ```sql
  CREATE TABLE todos (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    completed BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  ```

- **Environment Variables:**
  ```env
  PORT=3002
  DB_HOST=localhost
  DB_PORT=5433
  DB_NAME=todo_db
  DB_USER=postgres
  DB_PASSWORD=password
  AUTH_SERVICE_URL=http://localhost:3001
  NODE_ENV=production
  ```

- **Development:**
  ```bash
  cd code/be
  npm install
  npm run dev              # Development with nodemon
  npm run migrate          # Run migrations
  npm run migrate:status   # Check migration status
  npm run migrate:rollback # Rollback last migration
  npm run setup-db         # Setup database and run migrations
  ```

---

## 🚀 Quick Start

### Prerequisites

- **Docker & Docker Compose** (recommended)
- **Node.js 18+** (for local development)
- **PostgreSQL 12+** (if not using Docker)

### Option 1: Docker Compose (Recommended)

Start all services with one command:

```bash
# Clone the repository
git clone <repository-url>
cd elm-phase2

# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f

# Access the application
open http://localhost:3000
```

This starts:
- Frontend on `http://localhost:3000`
- Auth service on `http://localhost:3001`
- Backend service on `http://localhost:3002`
- PostgreSQL databases (auth_db, todo_db)

### Option 2: Local Development

**Terminal 1 - Auth Service:**
```bash
cd code/auth
npm install
cp env.example .env
npm run migrate
npm run dev  # Runs on port 3001
```

**Terminal 2 - Backend Service:**
```bash
cd code/be
npm install
cp env.example .env
npm run migrate
npm run dev  # Runs on port 3002
```

**Terminal 3 - Frontend:**
```bash
cd code/fe
npm install
cp env.example .env
npm start    # Runs on port 3000
```

---

## 🎯 Complete User Flow Example

### 1. User Authentication

```bash
# Login
curl -X POST http://localhost:3001/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "password123"}' \
  -c cookies.txt

# Verify session
curl http://localhost:3001/is-logged-in -b cookies.txt
```

### 2. Todo Operations

```bash
# Create a todo
curl -X POST http://localhost:3002/to-do \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "title": "Setup Kubernetes",
    "description": "Deploy microservices to GKE",
    "completed": false
  }'

# Get all todos
curl http://localhost:3002/to-do -b cookies.txt

# Update todo
curl -X PUT http://localhost:3002/to-do/1 \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{"completed": true}'

# Delete todo
curl -X DELETE http://localhost:3002/to-do/1 -b cookies.txt
```

### 3. User Info & Logout

```bash
# Get current user
curl http://localhost:3002/user -b cookies.txt

# Logout
curl -X POST http://localhost:3001/logout -b cookies.txt -c cookies.txt
```

---

## 🔒 Security Features

- **Password Security:**
  - bcrypt hashing with 10 salt rounds
  - No plain text password storage

- **Session Management:**
  - httpOnly cookies (XSS protection)
  - sameSite=strict (CSRF protection)
  - Secure session validation

- **API Security:**
  - CORS configuration
  - Authentication middleware
  - Input validation
  - Error handling without information leakage

- **Database Security:**
  - Connection pooling
  - Parameterized queries (SQL injection prevention)
  - Credentials via environment variables

---

## 🏗️ Infrastructure & Deployment

### Kubernetes Deployment

The application is deployed on Google Kubernetes Engine (GKE) with:

- **GitOps:** ArgoCD for continuous deployment
- **Ingress:** Nginx Ingress Controller
- **Monitoring:** Prometheus & Grafana
- **Autoscaling:** HPA for backend services
- **Storage:** Persistent volumes for PostgreSQL
- **Backups:** VolumeSnapshots with automated retention

### CI/CD Pipeline

**Development Branch (`dev`):**
- Build Docker images
- Security scanning with Trivy
- Push to GCP Artifact Registry

**Production Branch (`main`):**
- Build & push Docker images
- Update GitOps repository
- Automatic deployment via ArgoCD

### Terraform Infrastructure

```bash
cd infra/terraform
terraform init
terraform plan -var-file=elm-phase2.tfvars
terraform apply -var-file=elm-phase2.tfvars
```

Provisions:
- GKE cluster (unmanaged control plane)
- VPC and networking
- Bastion host
- Worker nodes with autoscaling
- Storage classes

---

## 📊 Database Migrations

All services use a custom migration system:

```bash
# Run pending migrations
npm run migrate

# Check migration status
npm run migrate:status

# Rollback last migration
npm run migrate:rollback

# Rollback specific version
npm run migrate:rollback:version <version>
```

**Migration Files:**
- `migrations/000_create_migrations_table.sql`
- `migrations/001_create_<table>_table.sql`
- `migrations/001_create_<table>_table_rollback.sql`

---

## 🧪 Testing

### Health Checks

```bash
# Frontend
curl http://localhost:3000

# Auth Service
curl http://localhost:3001/health

# Backend Service
curl http://localhost:3002/health
```

### Integration Testing

```bash
# Test complete flow
./scripts/integration-test.sh

# Or manually:
# 1. Login
# 2. Create todo
# 3. Get todos
# 4. Update todo
# 5. Delete todo
# 6. Logout
```

---

## 🛠️ Troubleshooting

### Common Issues

**1. Authentication Failures**
```bash
# Check auth service
curl http://localhost:3001/health

# Verify session cookie
curl http://localhost:3001/is-logged-in -b cookies.txt
```

**2. Database Connection Errors**
```bash
# Check database containers
docker-compose ps

# Test database connectivity
docker-compose exec auth-postgres psql -U postgres -d auth_db -c "SELECT 1;"
docker-compose exec todo-postgres psql -U postgres -d todo_db -c "SELECT 1;"
```

**3. CORS Issues**
- Ensure frontend URL is in CORS whitelist
- Check `Access-Control-Allow-Origin` headers
- Verify cookie domain settings

**4. Port Conflicts**
```bash
# Check what's using ports
lsof -i :3000  # Frontend
lsof -i :3001  # Auth
lsof -i :3002  # Backend
lsof -i :5432  # Auth DB
lsof -i :5433  # Todo DB
```

### Debug Commands

```bash
# Service logs
docker-compose logs -f auth-service
docker-compose logs -f todo-service
docker-compose logs -f frontend

# Database logs
docker-compose logs -f auth-postgres
docker-compose logs -f todo-postgres

# Execute commands in containers
docker-compose exec auth-service sh
docker-compose exec todo-service sh

# Check network connectivity
docker-compose exec todo-service curl http://auth-service:3001/health
```

---

## 📚 Documentation

- **Architecture:** `/docs/ARCHITECTURE.md`
- **Docker Setup:** `/code/auth/DOCKER.md`
- **Migrations:** `/code/auth/MIGRATIONS.md`
- **Frontend Integration:** `/code/fe/docs/ARCHITECTURE.md`
- **Infrastructure:** `/infra/Steps Followed.md`

---

## 🎓 Learning Resources

This project demonstrates:

- ✅ Microservices architecture patterns
- ✅ RESTful API design
- ✅ Authentication & authorization
- ✅ Database design & migrations
- ✅ Docker containerization
- ✅ Kubernetes orchestration
- ✅ GitOps deployment (ArgoCD)
- ✅ Infrastructure as Code (Terraform)
- ✅ CI/CD pipelines (GitHub Actions)
- ✅ Modern React patterns (Hooks, Context)
- ✅ Cloud-native best practices

---

## 📝 License

MIT License - see LICENSE file for details

---

## 👥 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

---

## 🔗 Related Repositories

- **GitOps Repository:** `ZaidAfane3/todo-infra`
- **Deployment Manifests:** `/infra/k8s/todo/`

---

## 📞 Support

For issues and questions:
- Create an issue in this repository
- Check existing documentation
- Review troubleshooting guide

---

**Built with ❤️ using modern DevOps practices**
