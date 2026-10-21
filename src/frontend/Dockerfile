# Stage 1: Install dependencies
FROM oven/bun:1 AS deps
WORKDIR /app

# Copy dependency files first to cache layer
# (all deps, not --production: bun-plugin-tailwind needs tailwindcss at runtime
#  because Bun.serve bundles the HTML imports when the server starts)
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

# Stage 2: Runtime - runs the Bun server (src/index.ts) directly
FROM oven/bun:1-slim AS runner
WORKDIR /app
ENV NODE_ENV=production

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Run as non-root user for security
USER bun

EXPOSE 3000
CMD ["bun", "src/index.ts"]

LABEL org.opencontainers.image.description "React Frontend for managing the Patient Repo reference data and admin tasks"