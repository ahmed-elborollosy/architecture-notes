# One Codebase, Many Deployments: A Practical Guide to Lambda ZIP, Docker, and Lambda Container Images
*Lambda ZIP • Lambda Container Image • Local Docker*

## Abstract
When I first started this experiment, the goal was simple: create a Node.js service that can run anywhere — as a classic AWS Lambda ZIP, as a Lambda container image, or just locally in Docker.  

The first attempt worked, but it quickly got messy: mixing Lambda code with container code, dealing with missing modules, and debugging import errors. So I restructured the project into a cleaner, more maintainable layout — separating **shared business logic** from **deployment-specific implementations**.  

This article documents that journey: how I bootstrapped a multi-target deployment project with a structure that scales. Along the way, I’ll share the exact code snippets, Dockerfile setup, and lessons learned.  

---

## 1. Why Multiple Deployment Types?

Cloud projects rarely stay in one shape. Sometimes you want the speed and simplicity of a **Lambda ZIP** package. Other times you need the flexibility of a **Lambda container image**. And when testing locally, it’s easiest to just run it as a plain Node.js container.  

Instead of choosing one, I wanted a project that supports all three:  
- **Lambda ZIP** → classic AWS deployment.  
- **Lambda Container Image** → for complex dependencies.  
- **Local Docker** → fast iteration during development.  

The key: **separate what’s common from what’s deployment-specific**.  

---

## 2. Final Project Structure

Here’s the evolved layout:  

```bash
my-service/
├── events/                        # Sample events for local/SAM testing
│   └── event.json
├── src/                           # Shared business logic
│   └── app.js                     # Pure business logic
├── lambda/                        # Lambda-specific implementation
│   ├── package.json               # Only Lambda deps (aws-sdk, middy, etc.)
│   ├── package-lock.json
│   ├── index.js                   # Lambda handler (entrypoint)
│   └── bootstrap.js               # (Optional) common init for Lambda flavor
├── container/                     # Container-specific implementation
│   ├── package.json               # Only container deps (express, etc.)
│   ├── package-lock.json
│   ├── server.js                  # Express/Koa/Fastify server entry
│   └── bootstrap.js               # (Optional) container bootstrap
├── template.yaml                  # AWS SAM template for Lambda deploy
├── samconfig.toml                 # SAM config (optional)
├── Dockerfile                     # Multi-stage build (Lambda vs Container)
├── .vscode/
│   └── launch.json                # Debug configs
└── README.md
```

**Philosophy:**  
- `src/` → pure business logic, independent of runtime.  
- `lambda/` → wraps `src/` with a Lambda handler.  
- `container/` → wraps `src/` with an Express server.  
- One **Dockerfile** builds either Lambda image or local container, depending on context.  

---

## 3. Lambda ZIP Deployment

The Lambda entrypoint lives in `lambda/index.js`:  

```js
const { handler: appHandler } = require("./src/app");

exports.handler = async (event) => {
  return appHandler(event);
};
```

To package as ZIP:  

```bash
cd lambda
npm install --production
zip -r ../dist/function.zip . ../src
aws lambda update-function-code   --function-name MyLambda   --zip-file fileb://../dist/function.zip
```

✅ Clean separation: ZIP package includes only Lambda deps + shared code.  

---

## 4. Lambda Container Image

For containers, the Dockerfile handles both flavors with multi-stage builds:  

**Dockerfile**  

```dockerfile
# Stage 1: Lambda ZIP-like (lightweight) 
# --------------------------------------
# Not a real Lambda runtime, just Node.js + your handler
# Useful for local logic testing, not for AWS deploy
FROM node:18-alpine AS lambda-light
WORKDIR /app
COPY lambda/package*.json ./ 
RUN npm install --production
COPY src/ ./src
COPY lambda/index.js .
CMD ["node", "index.js"]
EXPOSE 8080
# To run locally: docker run -p 8080:8080 <imageId>

# Stage 2: Express Container
# --------------------------------------
# Full container with HTTP server (runs anywhere)
FROM node:18-alpine AS container
WORKDIR /app
COPY container/package*.json ./
RUN npm install --production
COPY src/ ./src
COPY container/server.js .
CMD ["node", "server.js"]
EXPOSE 8080
# To run locally: docker run -p 8080:8080 <imageId>

# Stage 3: AWS Lambda Runtime (default)
# --------------------------------------
# This is the FINAL stage -> what you get if you just run `docker build`
FROM public.ecr.aws/lambda/nodejs:18 AS lambda-aws
WORKDIR /var/task
COPY src/ ./src
COPY lambda/index.js .
# Lambda expects CMD in the form: [ "fileName.exportedHandler" ]
CMD ["index.handler"]
EXPOSE 8080
# To run locally (emulates Lambda): docker run -p 9000:8080 <imageId>
# Then call: curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'
```

Build Lambda image:  

```bash
docker build --target lambda-aws -t my-lambda .
```

Build local container:  

```bash
docker build --target container -t my-local .
```

---

## 5. Local Docker Development

The container flavor uses Express (or Koa/Fastify).  

**container/server.js**  

```js
const express = require("express");
const { handler } = require("./src/app");

const app = express();

app.get("/health", async (req, res) => {
  const result = await handler();
  res.json(result);
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`🚀 Local server running at http://localhost:${port}`);
});
```

Run it:  

```bash
docker run -p 3000:3000 my-local
```

Now you can test endpoints locally, no redeploy needed.  

---

## 6. AWS SAM for Local Lambda Testing

The `template.yaml` and `events/` folder let you simulate Lambda locally with [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/).  

```bash
sam local invoke MyFunction --event events/event.json
```

This way, you can test Lambda behavior without deploying.  

---

## 7. Lessons Learned

- **Separate deps per flavor** → keep Lambda light (aws-sdk) and container lean (express).  
- **src/ is sacred** → never pollute business logic with runtime concerns.  
- **Multi-stage Dockerfile** → one file, two targets.  
- **SAM + Docker combo** → covers both local Lambda simulation and local container dev.  
- **Cleaner repo = fewer headaches** → no more “cannot find module” surprises.  

---

## 8. Conclusion

This evolved structure turned a messy experiment into a template I can reuse. With a single codebase, I can now:  
- Deploy as **Lambda ZIP**.  
- Package as a **Lambda container image**.  
- Run locally as a **Docker container**.  

It’s clean, cost-friendly, and future-proof.  

🔗 [Check out the full source code here](https://github.com/ahmed-elborollosy/nodejs-serverless-project-template)

---

💡 If you try it out, let me know how it goes!
