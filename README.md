# Proof of concept: Running Nestjs w/ Fastify on Google Cloud Run

## Description

Following some feedback I received, I wanted to test running [Nest](https://github.com/nestjs/nest) on Cloud Run, but using Fastify as a driver instead, and run some benchmarks.

## Creation of the project

```bash
$ npm i -g @nestjs/cli
$ nest new cloud-run
```

I had to update the `main.ts` file like so: 

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors();
  await app.listen(process.env.PORT || 3000);
}
bootstrap();
```

## Creating the Dockerfile

For better performance, I decided to build the app before hand and run the `start:prod` command.

```dockerfile
# Use the official lightweight Node.js 12 image.
# https://hub.docker.com/_/node
FROM node:12-alpine

# Create and change to the app directory.
WORKDIR /usr/src/app

# Copy application dependency manifests to the container image.
# A wildcard is used to ensure both package.json AND package-lock.json are copied.
# Copying this separately prevents re-running npm install on every code change.
COPY package*.json ./

# Install production dependencies.
RUN npm install

# Copy local code to the container image.
COPY . ./

RUN npm run build

# Run the web service on container startup.
CMD [ "npm", "run", "start:prod" ]
```

## Deployment

I used Cloud Build to build the docker image

```bash
$ gcloud builds submit --tag gcr.io/${PROJECT_ID}/helloworld
```

Then deployed it:

```bash
$ gcloud run deploy --image gcr.io/${PROJECT_ID}/helloworld --platform managed
```

## Benchmark

Ran a small benchmark (to avoid crazy costs) with Apache Benchmark:

```bash
$ ab -n 1000 -c 80 https://cloud-run-url/
```

Here are the results: 

```

```

## Conclusion

// TODO
