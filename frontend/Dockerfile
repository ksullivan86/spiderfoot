# =============================================================================
# SpiderFoot React SPA — Self-Contained Frontend
# Multi-stage build: Node (build) → Nginx (serve)
#
# The SPA talks to the API server via /api/* which Nginx reverse-proxies
# to the backend API container (configurable via SF_API_URL at runtime).
#
# Build:
#   docker build -t spiderfoot-frontend:latest ./frontend
#
# Run standalone:
#   docker run -p 3000:80 -e SF_API_URL=http://host.docker.internal:8001 spiderfoot-frontend:latest
# =============================================================================

# ---------- Stage 1: Build ----------
FROM node:20-alpine AS builder

WORKDIR /app

# Install deps first (layer caching)
COPY package.json package-lock.json* yarn.lock* pnpm-lock.yaml* ./
RUN npm ci --prefer-offline --no-audit --no-fund 2>/dev/null || npm install --prefer-offline --no-audit --no-fund

# Copy source & build
COPY . .

# Override the outDir so Vite builds into /app/dist (not ../spiderfoot/static/react)
RUN npx vite build --outDir dist

# ---------- Stage 2: Serve ----------
FROM nginx:1.27-alpine

# Install envsubst for runtime config injection
RUN apk add --no-cache gettext

# Remove default site
RUN rm -rf /usr/share/nginx/html/* /etc/nginx/conf.d/default.conf

# Copy built assets
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy Nginx config template
COPY nginx.conf /etc/nginx/templates/default.conf.template

# Runtime env defaults
ENV SF_API_URL=http://api:8001
ENV SF_WS_URL=http://api:8001
ENV SF_AGENTS_URL=http://sf-agents:8100

# ── Security hardening & Permissions for Talos ──────────────────────────────
RUN apk --no-cache del apk-tools 2>/dev/null || true

# 1. Create necessary directories
RUN mkdir -p /var/cache/nginx /var/run /var/log/nginx /etc/nginx/conf.d

# 2. Fix Nginx main config: remove root user and move PID to a writable location
RUN sed -i '/user  nginx;/d' /etc/nginx/nginx.conf && \
    sed -i 's|/var/run/nginx.pid|/tmp/nginx.pid|g' /etc/nginx/nginx.conf

# 3. Comprehensive ownership fix for UID 1000
RUN chown -R 1000:1000 /var/cache/nginx /var/run /var/log/nginx /etc/nginx/conf.d /etc/nginx/templates /usr/share/nginx/html /tmp

# 4. Set permissions for config writing
RUN chmod -R 775 /etc/nginx/conf.d

USER 1000:1000