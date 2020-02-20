# Proof of concept: Running Nestjs w/ Fastify on Google Cloud Run

## Description

Following some feedback I received, I wanted to test running [Nest](https://github.com/nestjs/nest) on Cloud Run, but using Fastify as a driver instead, and run some benchmarks.

[![Run on Google Cloud](https://deploy.cloud.run/button.svg)](https://deploy.cloud.run)

## Creation of the project

```bash
$ npm i -g @nestjs/cli
$ nest new cloud-run
```

I had to update the `main.ts` file like so: 

```typescript
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter({ logger: true }),
  );
  app.enableCors();
  await app.listen(parseInt(process.env.PORT) || 3000, '0.0.0.0');
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
This is ApacheBench, Version 2.3 <$Revision: 1843412 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking cloud-run-url (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Completed 600 requests
Completed 700 requests
Completed 800 requests
Completed 900 requests
Completed 1000 requests
Finished 1000 requests


Server Software:        Google
Server Hostname:        cloud-run-url
Server Port:            443
SSL/TLS Protocol:       TLSv1.2,ECDHE-RSA-CHACHA20-POLY1305,2048,256
Server Temp Key:        ECDH X25519 253 bits
TLS Server Name:        cloud-run-url

Document Path:          /
Document Length:        12 bytes

Concurrency Level:      80
Time taken for tests:   9.300 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      437004 bytes
HTML transferred:       12000 bytes
Requests per second:    107.53 [#/sec] (mean)
Time per request:       743.985 [ms] (mean)
Time per request:       9.300 [ms] (mean, across all concurrent requests)
Transfer rate:          45.89 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       79  461 213.7    430    2633
Processing:    37  208 118.9    200     506
Waiting:       22  163 105.3    139     501
Total:        129  669 220.7    626    2739

Percentage of the requests served within a certain time (ms)
  50%    626
  66%    702
  75%    768
  80%    772
  90%    862
  95%   1161
  98%   1371
  99%   1576
 100%   2739 (longest request)
```

## Conclusion

The difference with a regular installation of NestJS (with Express driver) is minimal. Even after running the tests multiple times,
 I still had better performance with NestJS Express driver.
