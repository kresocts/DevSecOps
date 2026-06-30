# syntax=docker/dockerfile:1.7

FROM node:20-alpine AS dev
ENV NODE_ENV=development
WORKDIR /app

COPY package*.json ./
RUN npm ci --ignore-scripts \
    && npm cache clean --force
COPY --chown=node:node src ./src

USER node
CMD ["npm", "run", "dev"]

FROM node:20-alpine AS deps
WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev --ignore-scripts \
    && npm cache clean --force

FROM node:20-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app

# Keep the final runtime image smaller and reduce Trivy findings from packages
# that are only needed during build/install, not when running the service.
RUN apk upgrade --no-cache \
    && rm -rf \
      /usr/local/lib/node_modules/npm \
      /usr/local/lib/node_modules/corepack \
      /usr/local/bin/npm \
      /usr/local/bin/npx \
      /usr/local/bin/corepack \
      /opt/yarn* \
      /usr/local/bin/yarn \
      /usr/local/bin/yarnpkg \
      /tmp/*

COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node src ./src

USER node

CMD ["node", "src/worker.js"]
