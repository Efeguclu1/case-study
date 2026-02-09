FROM node:18-alpine

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production --silent && \
    npm cache clean --force
COPY --chown=nodejs:nodejs . .

USER nodejs
EXPOSE 5050

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:5050/healthcheck || exit 1

CMD ["node", "server.mjs"]
