# Docker NodeJs

1. Use deterministic docker base image tags

* Avoid FROM node
* Avoid FROM node:lts
* Avoid FROM node:14-alpine


Instead of generic image aliases, use SHA256 hashes or specific image version tags for deterministic builds. For example:

* FROM node:lts-alpine@sha256:5c4c0dd64aa
* FROM node:14.2.0-alpine3.11

2. Install only production dependencies
Avoid pulling devDependencies and non-deterministic package install like the ones below:

* Avoid RUN npm install
* Avoid RUN yarn install
* Avoid RUN npm ci
Instead, ensure you are installing only production dependencies in a reproducible way:
* RUN npm ci --only=production

3. Optimize Node.js apps for production
Some Node.js libraries and frameworks will only enable production-related optimization if they detect that the NODE_ENV env var set to production:
* ENV NODE_ENV production

4. Don’t run Node.js apps as root
Docker defaults to running the process in the container as the root user, which is a precarious security practice. Use a low privileged user and proper filesystem permissions:
* USER node
* COPY --chown=node:node . /usr/src/app

5. Properly handle events to safely terminate a Node.js application
Docker creates processes as PID 1, and they must inherently handle process signals to function properly. This is why you should avoid any of these variations:
* CMD “npm” “start”
* CMD [“yarn”, “start”]
* CMD “node” “server.js” 
* CMD “start-app.sh”
Instead, use a lightweight init system, such as dumb-init, to properly spawn the Node.js runtime process with signals support:
* CMD [“dumb-init”, “node”, “server.js”] 

6. Gracefully tear down Node.js apps
Avoid an abrupt termination of a running Node.js application that halts live connections. Instead, use a process signal event handler:

```JS
async function closeGracefully(signal) {
   await fastify.close()
}  process.exit()
process.on('SIGINT', closeGracefully)
```

7. Find and fix security vulnerabilities in your Node.js Docker image
Docker base images may include security vulnerabilities in the software toolchain they bundle, including the Node.js runtime itself. Scan and fix security vulnerabilities with the free Snyk Container tool which also provides base image recommendations:

* npm install -g snyk
* snyk auth
* snyk container test node14.2.0-alpine

8. Use multi-stage builds
Avoid having one big build stage when attempting to clean up sensitive data from it or dangling dependencies. Instead, use multi-stage Docker image builds and separate concerns between the build flow and the creation of a production base image.

9. Use .dockerignore
Use .dockerignore to ensure:
* local artifacts of node_modules/ aren’t copied into
the container image.
* sensitive files, such as .npmrc, .env or others, aren’t leaked into the container image.
* a small Docker base image without redundant and unnecessary files.

10. Mount secrets into the Docker image
Secrets are a tricky thing to manage. Avoid the following security pitfalls:
* passing secrets via build arguments in non multi-stage builds
* putting secrets inside the Dockerfile
Instead, use the built-in secrets mounting. To mount a .npmrc file for package install:
* In the Dockerfile: RUN --mount=type=secret,id=npmrc, target=/usr/src/app/.npmrc npm ci --only=production
* Then build the image with: docker build . --build-arg NPM_TOKEN=1234 --secret id=npmrc,src=.npmrc
   --file=Dockerfile
