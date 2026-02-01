---
title: "Docker Best Practices untuk Production"
date: 2025-10-15
draft: false
tags: ["Docker", "DevOps", "Containers", "Best Practices"]
categories: ["DevOps"]
author: "wakwaw"
showToc: true
TocOpen: false
description: "Tips dan best practices menggunakan Docker untuk environment production"
cover:
  image: "https://images.unsplash.com/photo-1605745341112-85968b19335b?w=1200&h=630&fit=crop"
  alt: "Docker Containers"
  caption: "Container best practices"
  relative: false
---

## Introduction

Docker sudah menjadi standar industri untuk containerization. Tapi banyak developer yang masih menggunakan Docker dengan cara yang kurang optimal. Artikel ini akan membahas best practices untuk production-ready Docker images.

## 1. Use Multi-Stage Builds

Multi-stage builds mengurangi ukuran image secara signifikan:

```dockerfile
# ❌ Bad - Single stage
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
CMD ["node", "dist/index.js"]
# Result: ~1.2GB image

# ✅ Good - Multi-stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
# Result: ~200MB image
```

## 2. Use Specific Base Image Tags

Hindari menggunakan `latest` tag:

```dockerfile
# ❌ Bad - Unpredictable
FROM python:latest

# ✅ Good - Specific version
FROM python:3.11.6-slim-bookworm
```

## 3. Don't Run as Root

Jalankan aplikasi dengan non-root user:

```dockerfile
# Create non-root user
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser

# Set ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

CMD ["./app"]
```

## 4. Optimize Layer Caching

Urutan instruksi mempengaruhi caching:

```dockerfile
# ❌ Bad - Rebuilds node_modules on any code change
COPY . .
RUN npm ci

# ✅ Good - Caches node_modules if package.json unchanged
COPY package*.json ./
RUN npm ci
COPY . .
```

## 5. Use .dockerignore

Exclude unnecessary files:

```dockerignore
# .dockerignore
node_modules
.git
.gitignore
*.md
Dockerfile
.dockerignore
.env*
coverage
tests
__pycache__
*.pyc
```

## 6. Health Checks

Tambahkan health check untuk monitoring:

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1
```

Atau untuk aplikasi tanpa curl:

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1
```

## 7. Security Scanning

Gunakan tools untuk scan vulnerabilities:

```bash
# Using Docker Scout
docker scout cves myimage:latest

# Using Trivy
trivy image myimage:latest

# Using Snyk
snyk container test myimage:latest
```

## 8. Proper Signal Handling

Gunakan `exec` form untuk CMD agar signals diteruskan dengan benar:

```dockerfile
# ❌ Bad - Shell form, PID 1 is shell
CMD npm start

# ✅ Good - Exec form, app is PID 1
CMD ["node", "dist/index.js"]
```

Atau gunakan `tini` sebagai init system:

```dockerfile
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

## 9. Minimize Installed Packages

Install hanya yang diperlukan:

```dockerfile
# ❌ Bad - Installs recommended packages
RUN apt-get update && apt-get install -y curl

# ✅ Good - No recommended packages, clean up
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

## 10. Use Labels for Metadata

Tambahkan labels sesuai OCI standard:

```dockerfile
LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="My awesome application" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.vendor="wakwaw" \
      org.opencontainers.image.source="https://github.com/wakwaw/myapp"
```

## Complete Example

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM node:20.10-alpine AS builder
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Build application
COPY . .
RUN npm run build

# Production stage
FROM node:20.10-alpine AS runner
WORKDIR /app

# Security: non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 --ingroup nodejs nextjs

# Install tini for proper signal handling
RUN apk add --no-cache tini

# Copy built application
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./

# Labels
LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.version="1.0.0"

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start with tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

## Conclusion

Dengan menerapkan best practices ini, Docker images kamu akan lebih:
- **Secure:** Non-root user, minimal packages
- **Efficient:** Smaller size, better caching
- **Reliable:** Health checks, proper signal handling

Happy containerizing! 🐳

---

*Punya tips Docker lainnya? Share di comment atau reach out ke [@wakwaw](https://twitter.com/wakwaw)*
