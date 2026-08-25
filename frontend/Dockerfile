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

# Have the image derive the DNS resolver from /etc/resolv.conf, so the same
# image proxies to the backend on Docker, Podman, Kubernetes and Railway with
# no per-platform configuration. 19-resolver.envsh turns that into
# NGINX_RESOLVER (with a Docker fallback) for the config template; setting
# NGINX_RESOLVER explicitly still overrides both.
ENV NGINX_ENTRYPOINT_LOCAL_RESOLVERS=1
COPY --chmod=755 docker-entrypoint.d/19-resolver.envsh /docker-entrypoint.d/

# `worker_processes auto` counts the host's cores, not the container's limit
# (the image's autotune script is opt-in and needs a cgroup CPU quota to help).
# Serving a static SPA plus one proxy location does not need more than this.
RUN sed -i 's/^worker_processes.*/worker_processes 2;/' /etc/nginx/nginx.conf

EXPOSE 8080
