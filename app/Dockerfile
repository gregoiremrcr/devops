FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

FROM deps AS test
COPY src ./src
COPY test ./test
RUN npm test && touch /tmp/tests-passed

FROM alpine:3.20 AS production
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/sh -D appuser && apk add --no-cache nodejs
WORKDIR /app
ENV NODE_ENV=production
COPY package.json ./
COPY src ./src
COPY --from=test /tmp/tests-passed /tmp/tests-passed
LABEL maintainer="devops-team"
LABEL version="1.0"
RUN chown -R appuser:appgroup /app
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 CMD wget --no-verbose --tries=1 --spider http://127.0.0.1:3000/health || exit 1
CMD ["node", "src/index.js"]
