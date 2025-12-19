# Docker Compose - Hands-On Exercises

## Exercise 1: Your First Docker Compose Application

### Objective
Create a simple web application with Nginx and understand basic Compose workflow.

### Steps

1. **Create project directory:**
```bash
mkdir my-first-compose
cd my-first-compose
```


2. **Create a simple HTML file:**
```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>My First Compose App</title>
    <style>
        body { 
            font-family: Arial; 
            text-align: center; 
            padding: 50px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
    </style>
</head>
<body>
    <h1>🎉 Docker Compose Works!</h1>
    <p>This is served by Nginx in a container</p>
</body>
</html>
EOF
```

3. **Create docker-compose.yml:**
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
    restart: unless-stopped
EOF
```

4. **Start the application:**
```bash
docker compose up -d
```

5. **Check status:**
```bash
docker compose ps
```

6. **Test the application:**
```bash
curl http://localhost:8080
# Or open in browser: http://localhost:8080
```

7. **View logs:**
```bash
docker compose logs
docker compose logs -f web
```

8. **Stop the application:**
```bash
docker compose down
```

### Expected Results
- ✅ Container starts successfully
- ✅ Web page accessible at http://localhost:8080
- ✅ Logs show Nginx running
- ✅ Single command to start/stop

---

## Exercise 2: Multi-Container WordPress Application

### Objective
Deploy WordPress with MySQL database using Docker Compose.

### Steps

1. **Create project directory:**
```bash
mkdir wordpress-app
cd wordpress-app
```

2. **Create docker-compose.yml:**
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    container_name: my-wordpress
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppassword
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp-content:/var/www/html/wp-content
      - wp-db:/var/www/html/wp-db
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: mysql:8.0
    container_name: wordpress-db
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppassword
    volumes:
      - db-data:/var/lib/mysql
    restart: unless-stopped

volumes:
  wp-content:
  wp-db:
  db-data:
EOF
```

3. **Start the stack:**
```bash
docker compose up -d
```

4. **Monitor startup:**
```bash
docker compose logs -f
# Press Ctrl+C when you see "ready for connections"
```

5. **Check running services:**
```bash
docker compose ps
```

6. **Setup WordPress:**
```
Open browser: http://localhost:8080
Follow WordPress setup wizard
Create admin account
```

7. **Test persistence:**
```bash
# Stop everything
docker compose down

# Start again
docker compose up -d

# Visit http://localhost:8080 - WordPress setup should persist!
```

8. **View resource usage:**
```bash
docker compose top
```

9. **Cleanup:**
```bash
# Stop and remove containers (keep volumes)
docker compose down

# Stop and remove everything including data
docker compose down -v
```

### Expected Results
- ✅ WordPress accessible at http://localhost:8080
- ✅ MySQL running in separate container
- ✅ Data persists across restarts
- ✅ Services communicate by name

---

## Exercise 3: Full-Stack Application (Node.js + PostgreSQL + Redis)

### Objective
Build a complete full-stack application with backend, database, and cache.

### Steps

1. **Create project structure:**
```bash
mkdir fullstack-app
cd fullstack-app
mkdir api
```

2. **Create Node.js API (api/package.json):**
```bash
cat > api/package.json << 'EOF'
{
  "name": "api",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "redis": "^4.6.7"
  }
}
EOF
```

3. **Create API server (api/server.js):**
```bash
cat > api/server.js << 'EOF'
const express = require('express');
const { Pool } = require('pg');
const redis = require('redis');

const app = express();
const PORT = 3000;

// PostgreSQL client
const pool = new Pool({
  host: process.env.DB_HOST || 'postgres',
  port: 5432,
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'secret',
  database: process.env.DB_NAME || 'myapp',
});

// Redis client
const redisClient = redis.createClient({
  url: `redis://${process.env.REDIS_HOST || 'redis'}:6379`
});
redisClient.connect();

app.get('/', (req, res) => {
  res.json({ 
    message: 'API is running!',
    endpoints: ['/health', '/db-test', '/cache-test']
  });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.get('/db-test', async (req, res) => {
  try {
    const result = await pool.query('SELECT NOW()');
    res.json({ 
      database: 'connected', 
      time: result.rows[0].now 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/cache-test', async (req, res) => {
  try {
    const visits = await redisClient.get('visits') || 0;
    const newVisits = parseInt(visits) + 1;
    await redisClient.set('visits', newVisits);
    res.json({ 
      cache: 'connected', 
      visits: newVisits 
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`API running on port ${PORT}`);
});
EOF
```

4. **Create Dockerfile for API (api/Dockerfile):**
```bash
cat > api/Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install

COPY server.js .

EXPOSE 3000

CMD ["node", "server.js"]
EOF
```

5. **Create docker-compose.yml:**
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  api:
    build: ./api
    container_name: fullstack-api
    ports:
      - "3000:3000"
    environment:
      DB_HOST: postgres
      DB_USER: postgres
      DB_PASSWORD: secret
      DB_NAME: myapp
      REDIS_HOST: redis
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    networks:
      - backend

  postgres:
    image: postgres:15-alpine
    container_name: fullstack-db
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - backend

  redis:
    image: redis:7-alpine
    container_name: fullstack-cache
    volumes:
      - redis-data:/data
    restart: unless-stopped
    networks:
      - backend

networks:
  backend:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
EOF
```

6. **Start the stack:**
```bash
docker compose up -d --build
```

7. **Watch logs:**
```bash
docker compose logs -f api
```

8. **Test all endpoints:**
```bash
# Main endpoint
curl http://localhost:3000/

# Health check
curl http://localhost:3000/health

# Database test
curl http://localhost:3000/db-test

# Cache test (multiple times to see counter)
curl http://localhost:3000/cache-test
curl http://localhost:3000/cache-test
curl http://localhost:3000/cache-test
```

9. **Test service communication:**
```bash
# Enter API container
docker compose exec api sh

# Inside container:
ping postgres
ping redis
exit
```

10. **Scale the API:**
```bash
docker compose up -d --scale api=3
docker compose ps
```

11. **Cleanup:**
```bash
docker compose down -v
cd ..
```

### Expected Results
- ✅ API connects to PostgreSQL successfully
- ✅ API connects to Redis successfully
- ✅ Visit counter increases with each request
- ✅ All services on same network
- ✅ Can scale API horizontally

---

## Exercise 4: Development Environment with Hot Reload

### Objective
Create a development environment with live code reloading.

### Steps

1. **Create project:**
```bash
mkdir dev-environment
cd dev-environment
mkdir app
```

2. **Create package.json:**
```bash
cat > app/package.json << 'EOF'
{
  "name": "dev-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
EOF
```

3. **Create server:**
```bash
cat > app/server.js << 'EOF'
const express = require('express');
const app = express();
const PORT = 3000;

app.get('/', (req, res) => {
  res.json({ 
    message: 'Development Server v1.0',
    timestamp: new Date().toISOString()
  });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`🚀 Server running on port ${PORT}`);
});
EOF
```

4. **Create Dockerfile:**
```bash
cat > app/Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
EOF
```

5. **Create docker-compose.yml with bind mount:**
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  app:
    build: ./app
    ports:
      - "3000:3000"
    volumes:
      - ./app:/app          # Bind mount for live reload
      - /app/node_modules   # Anonymous volume for node_modules
    environment:
      NODE_ENV: development
    restart: unless-stopped
EOF
```

6. **Start development environment:**
```bash
docker compose up -d --build
docker compose logs -f
```

7. **Test the API:**
```bash
curl http://localhost:3000/
```

8. **Make a change (live reload!):**
```bash
# Edit server.js
cat > app/server.js << 'EOF'
const express = require('express');
const app = express();
const PORT = 3000;

app.get('/', (req, res) => {
  res.json({ 
    message: 'Development Server v2.0 - UPDATED!',
    timestamp: new Date().toISOString(),
    status: 'Hot reload works! 🔥'
  });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`🚀 Server running on port ${PORT}`);
});
EOF

# Watch logs - you'll see nodemon restart!
docker compose logs -f app
```

9. **Test again:**
```bash
curl http://localhost:3000/
# You'll see the updated message without rebuilding!
```

10. **Cleanup:**
```bash
docker compose down
cd ..
```

### Expected Results
- ✅ Code changes appear immediately
- ✅ No need to rebuild container
- ✅ Nodemon restarts on file changes
- ✅ Perfect for development workflow

---

## Exercise 5: Multi-Environment Configuration

### Objective
Learn to use different configurations for development and production.

### Steps

1. **Create project:**
```bash
mkdir multi-env-app
cd multi-env-app
```

2. **Create base docker-compose.yml:**
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  app:
    image: nginx:alpine
    volumes:
      - ./html:/usr/share/nginx/html

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
volumes:
  db-data:
EOF
```

3. **Create development override (docker-compose.override.yml):**
```bash
cat > docker-compose.override.yml << 'EOF'
version: '3.8'

services:
  app:
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html  # Live reload

  db:
    ports:
      - "5432:5432"  # Expose for debugging
    environment:
      POSTGRES_PASSWORD: dev_password
EOF
```

4. **Create production config (docker-compose.prod.yml):**
```bash
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  app:
    ports:
      - "80:80"
    restart: always

  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}  # From environment
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: always
EOF
```

5. **Create .env file:**
```bash
cat > .env << 'EOF'
DB_PASSWORD=super_secret_production_password
EOF
```

6. **Create sample HTML:**
```bash
mkdir html
cat > html/index.html << 'EOF'
<h1>Multi-Environment App</h1>
<p>Check which environment you're in!</p>
EOF
```

7. **Run in development (uses override automatically):**
```bash
docker compose up -d
docker compose ps
# App on port 8080, DB exposed
```

8. **Stop development:**
```bash
docker compose down
```

9. **Run in production:**
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose ps
# App on port 80, DB not exposed
```

10. **Cleanup:**
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml down -v
cd ..
```

### Expected Results
- ✅ Development uses override automatically
- ✅ Production uses explicit config
- ✅ Different ports and settings per environment
- ✅ Secrets from environment variables

---

## Exercise 6: Health Checks and Dependencies

### Objective
Implement health checks and proper service dependencies.

### Steps

1. **Create project:**
```bash
mkdir healthcheck-app
cd healthcheck-app
mkdir api
```

2. **Create API with health endpoint (api/server.js):**
```bash
cat > api/server.js << 'EOF'
const express = require('express');
const app = express();
const PORT = 3000;

let isHealthy = false;

// Simulate startup time
setTimeout(() => {
  isHealthy = true;
  console.log('✅ API is now healthy');
}, 5000);

app.get('/health', (req, res) => {
  if (isHealthy) {
    res.status(200).json({ status: 'healthy' });
  } else {
    res.status(503).json({ status: 'starting up...' });
  }
});

app.get('/', (req, res) => {
  res.json({ message: 'API is running!' });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`API started on port ${PORT}`);
});
EOF
```

3. **Create package.json:**
```bash
cat > api/package.json << 'EOF'
{
  "name": "api",
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
```

4. **Create Dockerfile:**
```bash
cat > api/Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
EOF
```

5. **Create docker-compose.yml with health checks:**
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  api:
    build: ./api
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    restart: unless-stopped
EOF
```

6. **Start and watch startup:**
```bash
docker compose up -d --build
docker compose ps
# Watch status change from "starting" to "healthy"
```

7. **Monitor health checks:**
```bash
# Keep checking status
watch -n 1 docker compose ps

# In another terminal, view logs
docker compose logs -f api
```

8. **Test failure:**
```bash
# Stop database
docker compose stop db

# Watch API health fail
docker compose ps

# Restart database
docker compose start db

# Watch API recover
docker compose ps
```

9. **Cleanup:**
```bash
docker compose down
cd ..
```

### Expected Results
- ✅ API waits for database to be healthy
- ✅ Health checks run periodically
- ✅ Status visible in `docker compose ps`
- ✅ Services restart if unhealthy

---

## 🎯 Challenge Exercise: E-Commerce Microservices

### Task
Build a complete e-commerce platform with:
- Frontend (Nginx)
- API Gateway (Node.js)
- Auth Service (Node.js)
- Product Service (Node.js)
- Order Service (Node.js)
- PostgreSQL Database
- Redis Cache

### Requirements
1. All services on appropriate networks (frontend/backend)
2. Proper health checks
3. Environment variables from .env
4. Volume persistence for database and cache
5. Frontend accessible on port 80
6. API Gateway routes to services
7. Services can't be accessed directly from outside

**Hint:** You'll need at least 2 networks and multiple depends_on!

Good luck! 🚀

---

## Summary

You've learned:
- ✅ Create and run multi-container applications
- ✅ Use docker-compose.yml effectively
- ✅ Manage services with simple commands
- ✅ Implement health checks
- ✅ Configure different environments
- ✅ Use volumes for persistence
- ✅ Network isolation
- ✅ Service dependencies

**Key Commands:**
```bash
docker compose up -d              # Start all services
docker compose down               # Stop and remove
docker compose down -v            # Also remove volumes
docker compose ps                 # Check status
docker compose logs -f            # Follow logs
docker compose build              # Build images
docker compose exec <svc> sh      # Enter container
docker compose up -d --scale api=3  # Scale service
```

**Next:** Try the Group Exercise! 👥
