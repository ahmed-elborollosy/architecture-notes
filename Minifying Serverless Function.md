# Minifying Serverless Function

## Introduction

One of the main challenges with AWS Lambda, especially when using TypeScript or .NET, is the **cold start**. Unlike traditional container-based services that run continuously, Lambda must initialize the runtime, load your code, and resolve dependencies whenever a new execution environment is spun up. The size of your deployment package (zip or container image) directly impacts this initialization cost. Larger packages mean longer unzips, more files to scan, and longer startup times — all of which increase latency and add to your compute bill.
The goal of “minifying” a serverless function is not about frontend minification, but about reducing the Lambda’s **deployment artifact size** and **runtime footprint** by only including what the function actually needs. This leads to smaller images, faster cold starts, and measurable cost savings. The optimization techniques vary depending on whether you use plain `tsc` compilation, pruned node\_modules, or a bundler like `esbuild` that performs tree-shaking.

---

## Comparison of Deployment Strategies

| Deployment Style         |   Typical Size | Cold Start (Init Duration) | Expected Cost Reduction per 1M Calls\* |
| ------------------------ | -------------: | -------------------------: | -------------------------------------: |
| Raw `tsc` build          | \~200 MB image |                800–1200 ms |                  Baseline (no savings) |
| `tsc` + prune            |   \~100–150 MB |                 400–700 ms |         \~30–40% lower cold start cost |
| **esbuild (tree-shake)** |   **30–50 MB** |             **100–200 ms** |        **\~80% lower cold start cost** |

\* Estimation assumes 10% of calls are cold starts, Lambda at 128 MB memory, billed per ms.

---

## How To: Optimizing Dockerfiles for Lambda

Below are three **Dockerfile variants**, each representing one approach. The differences are explained inline with comments.

---

### 1. Raw `tsc` (Current Style, Heavy)

```dockerfile
# Step 1: Build stage
FROM node:18 AS build
WORKDIR /integration-v1-function-assessment

# Copy package manifests
COPY package*.json ./

# Install ALL dependencies (dev + prod)
RUN npm install

# Install TS compiler globally (redundant, but often seen)
RUN npm install -g typescript
RUN npm i --save-dev @types/node

# Copy source and compile
COPY . .
RUN npx tsc

# Step 2: Runtime
FROM public.ecr.aws/lambda/nodejs:18
WORKDIR /var/task

# Copy full dist and node_modules (bloated)
COPY --from=build /integration-v1-function-assessment/dist/ ./dist/
COPY --from=build /integration-v1-function-assessment/node_modules ./node_modules

# Entry point
CMD ["dist/index.handler"]
```

**Notes:**

* Image is very large (200 MB+).
* Cold starts slow (0.8–1.2s).
* Includes dev deps and redundant libraries (e.g. full AWS SDK).

---

### 2. `tsc` + prune (Improved, Leaner)

```dockerfile
# Step 1: Build stage
FROM node:18 AS build
WORKDIR /app

# Copy package manifests
COPY package*.json ./

# Install dependencies first (with lockfile for reproducibility)
RUN npm install

# Compile TypeScript
COPY . .
RUN npx tsc

# Remove dev dependencies to shrink node_modules
RUN npm prune --production

# Step 2: Runtime
FROM public.ecr.aws/lambda/nodejs:18
WORKDIR /var/task

# Copy only compiled JS + production dependencies
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules

# Entry point
CMD ["dist/index.handler"]
```

**Notes:**

* Package size reduced (\~100 MB).
* Cold starts better (0.4–0.7s).
* Still carries full unused libraries (no tree-shaking).

---

### 3. **esbuild (Preferred, Tree-Shaken)**

```dockerfile
# Step 1: Build stage
FROM node:18 AS build
WORKDIR /app

# Copy package manifests
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Use esbuild to bundle + tree-shake
# This pulls in only the code paths actually used
RUN npx esbuild src/index.ts \
    --bundle \
    --platform=node \
    --target=node18 \
    --outfile=dist/index.js

# Step 2: Runtime
FROM public.ecr.aws/lambda/nodejs:18
WORKDIR /var/task

# Copy just the bundled output (no node_modules at all)
COPY --from=build /app/dist ./dist

# Minimal entry point (tiny image, ~30–50MB total)
CMD ["dist/index.handler"]
```

**Notes:**

* Smallest image (\~30–50 MB).
* Cold start near minimal for Node.js (100–200ms).
* Only ships the code your Lambda actually executes.

---

## Summary

* **Raw tsc**: acceptable for prototypes, but too heavy for production.
* **tsc + prune**: a quick win if you don’t want to introduce bundling.
* **esbuild (tree-shake)**: the preferred approach. Produces much smaller artifacts, improves cold start drastically, and reduces cost per 1M calls by up to 80%.

The main idea is that \*“installing dependencies” vs *“bundling only used code”* are not the same. Dockerfiles that rely on `npm install` will always inflate with full libraries. A tree-shaken esbuild approach ensures you ship **only what Lambda needs to run**, nothing else.
