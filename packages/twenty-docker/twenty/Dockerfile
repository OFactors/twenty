# =============================================================================
# Twenty CRM - Optimized for low-memory servers (8GB RAM)
# Single sequential build stage to avoid BuildKit parallel memory contention
# =============================================================================

# Stage 1: Install dependencies
FROM node:24-alpine AS deps

WORKDIR /app

COPY ./package.json ./yarn.lock ./.yarnrc.yml ./tsconfig.base.json ./nx.json /app/
COPY ./.yarn/releases /app/.yarn/releases
COPY ./.yarn/patches /app/.yarn/patches

COPY ./packages/twenty-emails/package.json /app/packages/twenty-emails/
COPY ./packages/twenty-server/package.json /app/packages/twenty-server/
COPY ./packages/twenty-server/patches /app/packages/twenty-server/patches
COPY ./packages/twenty-ui/package.json /app/packages/twenty-ui/
COPY ./packages/twenty-shared/package.json /app/packages/twenty-shared/
COPY ./packages/twenty-front/package.json /app/packages/twenty-front/
COPY ./packages/twenty-sdk/package.json /app/packages/twenty-sdk/

ENV NODE_OPTIONS="--max-old-space-size=4096"

RUN yarn && yarn cache clean && npx nx reset


# Stage 2: Build everything SEQUENTIALLY (server then front, not parallel)
FROM deps AS builder

# Copy all source at once
COPY ./packages/twenty-emails /app/packages/twenty-emails
COPY ./packages/twenty-shared /app/packages/twenty-shared
COPY ./packages/twenty-server /app/packages/twenty-server
COPY ./packages/twenty-front /app/packages/twenty-front
COPY ./packages/twenty-ui /app/packages/twenty-ui
COPY ./packages/twenty-sdk /app/packages/twenty-sdk

# 1. Build server first
RUN npx nx run twenty-server:build

# 2. Build front after server is done (gets ALL available memory)
ARG REACT_APP_SERVER_BASE_URL
RUN npx nx build twenty-front

# 3. Move front build output to where server serves it
RUN cp -r /app/packages/twenty-front/build /app/packages/twenty-server/dist/front

# 4. Prune to production dependencies (front build already saved above)
RUN yarn workspaces focus --production twenty-emails twenty-shared twenty-server


# Stage 3: Minimal runtime image
FROM node:24-alpine AS twenty

RUN apk add --no-cache curl jq postgresql-client
RUN npm install -g tsx

COPY ./packages/twenty-docker/twenty/entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh
WORKDIR /app/packages/twenty-server

ARG REACT_APP_SERVER_BASE_URL
ENV REACT_APP_SERVER_BASE_URL=$REACT_APP_SERVER_BASE_URL

ARG APP_VERSION
ENV APP_VERSION=$APP_VERSION

COPY --chown=1000 --from=builder /app /app

LABEL org.opencontainers.image.source=https://github.com/twentyhq/twenty
LABEL org.opencontainers.image.description="This image provides a consistent and reproducible environment for the backend and frontend, ensuring it deploys faster and runs the same way regardless of the deployment environment."

RUN mkdir -p /app/.local-storage /app/packages/twenty-server/.local-storage && \
    chown -R 1000:1000 /app

USER 1000

CMD ["node", "dist/main"]
ENTRYPOINT ["/app/entrypoint.sh"]
