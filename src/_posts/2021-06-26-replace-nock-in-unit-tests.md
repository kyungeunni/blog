---
title: Mocking HTTP requests when writing tests (Node.js)
description: Some alternatives to Nock when mocking HTTP requests in your tests.
tags: unit-test test nock mocha node.js
draft: false
---
Quite often we need to mock remote HTTP requests when testing, either for integration tests or unit tests that your logic needs to react differently depending on the variation of the response of the request. There are several options to achieve that, but I will write down my favorite ones.

## Using Nock
[Nock](https://github.com/nock/nock) has been the go-to library when it comes to mocking HTTP requests for tests. It usually looks like this:

```ts
describe('nock usage', () => {
    let scope;

    beforeEach(() => {
        scope = nock('http://www.example.com')
            .get('/hello')
            .reply('boo!');
    });

    afterEach(() => {
        scope.cleanAll();
    });
});
```

It is a great tool, quite easy to use and you can see that HTTP requests are invoked for this portion of tests at glance. But sometimes it can be a bit cumbersome to use it. For example, it fails to mock some endpoints depending on how you append the path to base url, whether URL has `/` at the end or/and `path` starts with `/` etc. Or you forgot to `scope.cleanAll` or you accidentally use `.persist()`, all the following tests can be affected by the interceptors. But the biggest reason I've been looking for other solutions is: I just prefer to have fewer dependencies, I just don't enjoy updating all the code when the newer version of that library requires some updates on my test.

## Setting up your own dev HTTP server
If it's simple enough to do so, you can set up your HTTP server that serves the endpoints you need to mock. For instance, I need to test my new authorisation middleware, which will forward the authorization header to auth server to authorise them. my code would look like this:

```ts
async function authMiddleware(ctx: Koa.Context, next: Koa.Next) {
    const headers = {
        authorization: ctx.req.headers.authorization
    };
    const actor = await fetch('https://internal-auth/authorize,', {
        method: 'get',
        headers,
    });
    if (!actor) {
        throw new AuthorizationError(ctx.method, ctx.url);
    }

    return await next();
};
```
What we need to do in our dev mock server is:
    1. We should be able to set the url to point to dev server
    2. Ability to control the response for each test (e.g. authorise the user or not, what data it returns)

As most of the time we define config values per environment, we can define one for the test environment and we can make it points to the local one:
```
AUTH_URL=http://localhost:3003/authorize
```
and use `process.env.AUTH_URL` as url for the requests.

then, we can create a test server that start when we run the tests:
```ts
// auth-server.ts
import Koa from 'koa';
import Router from 'koa-router2';
import { actorPool } from './auth-helper';

export const server = new Koa();
export const router = new Router();

router.get('/authorize', async ctx => {
    const auth = ctx.headers.authorization;
    if (!auth) {
        ctx.status = 401;
        throw new Error('not authorized');
    }
    const actor = actorPool.get(auth);
    if (!actor) {
        return ctx.status = 401;
    }
    ctx.body = actor;
});

server.use(router.routes());
server.listen(3003).on('error', err => {
    console.warn(err.message);
});

// auth-server.ts
export const actorPool = new Map<string, Actor>();
```

So the endpoint is now available for your tests and as you can see there's one logic in the `/authorize` endpoints. It uses `actorPool` hash map to determine how it should respond with a certain payload. In my tests, it will create a `User`, and this User will need to be registered in `actorPool`. Then when the request is made, the server will check the same hashmap and determine whether to respond with `401` or not.

## Inject your module using Inversify
This is probably my favourite among all, using [Inversify.js](https://github.com/inversify/InversifyJS)! But it requires a bit of commitment, you might end up restructuring your application from scratch. Inversify is an inversion of control (IoC) container for TypeScript and JavaScript. They have a great set of documentation so I highly recommend you to read them through if you are not familiar with IoC. Also if you'd like to check how we use Inversify at [Ubio](https://ub.io), check out our [`node-framework`](https://github.com/ubio/node-framework) repository, It is pretty cool :)

tldr, you can replace your Service with MockService that is designed for your tests. for example, If you made that middleware above as a Service, you can just replace that service to act in a certain way while respecting the same interface and contract. Imagine you have an AuthorizationService:
```ts
export abstract class AuthService {
    abstract authorize(ctx: Koa.Context, next: Koa.Next): Promise<Actor>;
}

export class KyungeunAuthService extends AuthService {
    async authorize(ctx: Koa.Context, next: Koa.Next) {
        const headers = {
            authorization: ctx.req.headers.authorization
        };
        const actor = await fetch(process.env.AUTH_URL, {
            method: 'get',
            headers,
        });
        if (!actor) {
            throw new AuthorizationError(ctx.method, ctx.url);
        }

        return await next();
    }
}
```
We defined the abstract class so we define the contract and interface for `authorize()` method. In the application, the `KyungeunAuthService` will be bound to the container and it will fire the request to the auth server.

```ts
export class Application {
    container: Container;

    constructor() {
        const container = new Container({ skipBaseClassChecks: true });
        this.container = container;
        ...
        this.container.bind(AuthService).to(KyungeunAuthService);
    },
    ...
}
```

and when we run the test, we can rebind this AuthService, even to a constant value.
```ts
// authorize.test.ts
    beforeEach(() => {
        const app = new Application();
        app.container.rebind(AuthService).toConstantValue({
            async authorize() {
                return new Actor({
                    id: 'user-number-one',
                    name: 'KK',
                });
            }
        });
    });

    it('authorises user', () => {
        ...
    });
```

You noticed that is not technically mocking the requests, but just stubbing the behaviour of the service and method. To be fair, why do we need to make the actual requests if we can simply mimic the behaviour while safely respecting the contract? The main goal (to me)is to manipulate the component in a way that you wish so your tests can be run in a controlled context that fit into your scenario. You can argue that it won't be a suitable option if your goal is simply checking whether your tests fire the request to the designated url or not.
