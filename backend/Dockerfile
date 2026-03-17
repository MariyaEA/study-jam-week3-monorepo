# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
# Compile migrate.ts separately (standalone script, not in nest build module graph)
RUN node_modules/.bin/tsc --module commonjs --target ES2021 --moduleResolution node \
    --esModuleInterop --skipLibCheck --outDir dist --rootDir src src/database/migrate.ts

# Production stage
FROM node:18-alpine AS production
WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist
# Copy drizzle migration SQL files needed at runtime
COPY --from=builder /app/drizzle ./drizzle

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/main"]
