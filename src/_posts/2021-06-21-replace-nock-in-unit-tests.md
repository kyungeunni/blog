---
title: Mocking HTTP requests when writing tests
description: Possible replacement for nock when writing the tests
tags: unit-test test nock
draft: true
---
Quite often we need to mock remote HTTP requests when the part you are testing is depending on the payload of the response. You can stub the function that sends request entirely but you might want to test whether it actually sends requests or not.

## Using Nock
[Nock](https://github.com/nock/nock) Has been the go to library when it comes to mocking HTTP requests when writing unit tests. It is a great tool but sometimes it was a bit cumbersome to use it. (For example, it fails to mock some endpoints depending on how you append the path to base url, whether URL has `/` at the end or/and `path` starts with `/` etc.). Also when you write integration tests and you need to mock the endpoints repetitively so it is not so convenient to start the mock and clear it in every `beforeEach` and `afterEach`. One of example would be the authorisation middelware that needs to make HTTP requests. Or simply you'd like to get rid of the dependencies that can change your tests result after updating it to latest version!

# Set up dev HTTP server
What we can do is to set up a HTTP server that serves the endpoints you need to mock. For instance, I need to test my new authorisation middleware (as mentioned above!) and it needs to fetch key sets from the other application.

```ts
    const jwks = await this.request.get(process.env.JWKS_URL);
```
As we can replace the value for tests environment, we can define it to point to our local server.
```
JWKS_URL=http://localhost:3003/keys
```

then, we can create a test server and will start it when we run the tests.
```
```