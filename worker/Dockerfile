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

COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package*.json ./
COPY --chown=node:node src ./src

USER node

CMD ["node", "src/worker.js"]
