FROM node:22-slim AS build

WORKDIR /app

ARG VITE_APP_VERSION
ENV VITE_APP_VERSION=$VITE_APP_VERSION

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# Serve with nginx
FROM nginxinc/nginx-unprivileged:alpine

COPY --from=build /app/dist /usr/share/nginx/html
COPY default.conf.template /etc/nginx/templates/default.conf.template

# DNS resolver nginx uses to look up BACKEND_URL at request time. Docker's
# embedded DNS by default; override where the platform's internal resolver
# differs (Railway: "[fd12::10] ipv6=on").
ENV NGINX_RESOLVER=127.0.0.11

EXPOSE 8080
