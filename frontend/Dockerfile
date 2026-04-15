# syntax=docker/dockerfile:1.6

ARG NODE_VERSION=16
ARG NGINX_VERSION=1.27
ARG ALPINE_VERSION=3.20

FROM node:${NODE_VERSION}-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --no-audit --no-fund

FROM node:${NODE_VERSION}-alpine AS builder

ARG VUE_APP_API_URL=/api
ENV VUE_APP_API_URL=${VUE_APP_API_URL} \
    NODE_ENV=production

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npx vue-cli-service build --no-clean

FROM nginx:${NGINX_VERSION}-alpine AS runtime

RUN apk add --no-cache --upgrade ca-certificates curl tzdata && \
    rm -rf /usr/share/nginx/html/* /etc/nginx/conf.d/default.conf

COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /app/dist /usr/share/nginx/html

RUN addgroup -S app -g 10001 && \
    adduser  -S app -u 10001 -G app -h /var/cache/nginx -s /sbin/nologin && \
    mkdir -p /var/cache/nginx /var/run /var/log/nginx && \
    chown -R app:app /var/cache/nginx /var/run /var/log/nginx /usr/share/nginx/html /etc/nginx

USER app:app

EXPOSE 80

HEALTHCHECK --interval=15s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -fsS http://127.0.0.1:80/healthz || exit 1

ENTRYPOINT ["nginx", "-g", "daemon off;"]
