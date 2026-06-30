# syntax=docker/dockerfile:1.7

FROM node:20-alpine AS dev
ENV NODE_ENV=development
WORKDIR /app

COPY package*.json ./
RUN npm ci --ignore-scripts \
    && npm cache clean --force
COPY --chown=node:node src ./src

USER node
EXPOSE 3000
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
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "const http=require('http'); const port=process.env.FRONTEND_PORT||3000; const req=http.get({host:'127.0.0.1', port, path:'/healthz', timeout:2000}, res=>process.exit(res.statusCode===200?0:1)); req.on('error',()=>process.exit(1)); req.on('timeout',()=>{req.destroy(); process.exit(1);});"

CMD ["node", "src/server.js"]
