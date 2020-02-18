# Proof of concept: Running Nestjs on Google Cloud Run

## Description

I wanted to test running [Nest](https://github.com/nestjs/nest) on Cloud Run and run some benchmarks.

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
Time taken for tests:   8.624 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      486004 bytes
HTML transferred:       12000 bytes
Requests per second:    115.95 [#/sec] (mean)
Time per request:       689.939 [ms] (mean)
Time per request:       8.624 [ms] (mean, across all concurrent requests)
Transfer rate:          55.03 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:       61  402 219.1    375    2652
Processing:    29  207 117.5    192    1328
Waiting:       24  168 114.6    146    1279
Total:        163  609 236.4    567    2819

Percentage of the requests served within a certain time (ms)
  50%    567
  66%    622
  75%    681
  80%    714
  90%    804
  95%    920
  98%   1221
  99%   1754
 100%   2819 (longest request)
```

## Conclusion

Pretty straightforward to build and deploy a container to Cloud Run. The response time can sometimes be pretty slow, 
but overall if the container is small and quick to start it should be fine.
