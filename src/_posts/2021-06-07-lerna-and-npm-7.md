---
title: How to maintain a monorepo using Lerna and NPM 7
description: Upgrading to NPM 7 while using Lerna@4 for your monorepo
tags: monorepo lerna npm node.js workspace
draft: true
---
I've been putting off upgrading to NPM 7 for my monorepo using [Lerna@4.0.0](https://github.com/lerna/lerna) due to several issues have been occured. However I've reached a point where I couldn't juggle around anymore as it was a significant productivity killer.

I'm going to write down what issues I've faced and how I've solved them (partially, also temporarily.)

### Indentation updated when running `lerna bootstrap`
First issue I've encountered is that whenever running `lerna bootstrap`, it updated `package-lock.json` files to use tabs instead of spaces. The issue is well described in this [github issue](https://github.com/lerna/lerna/issues/2845). It didn't happen when using NPM 6 (lockVersion 1). It was a bit annoying as it created unnecessary lines of code changes when creating PRs, not really sure if you are acutally updating any dependencies. Also more likely to get conflics if one of your team mates uses a different version of NPM(not specifically lerna related issue though).

### Use NPM workspaces instead
As `lerna bootstrap` was misbehaving and we needed it to symlink the local dependencies within the project, we decided to give it a try npm's [workspaces](https://docs.npmjs.com/cli/v7/using-npm/workspaces) feature. It would automatically symlink those dependencies on `npm install`. All I need change was adding `workspaces: ["packages/*"]` to package.json(copy value of _packages_ properties you defined in `lerna.json`). This worked like a charm, so we could ditch the `lerna bootstrap` command so thus we could avoid unncessary updates in package-lock.json.

### Setting up npm 7 in Github actions
> It's not relavant if you don't use Github actions for CI/CD.

As I mentioned earlier, npm's workspaces feature was introduced with npm version 7. In order to use this feature, we need to make sure you use the right version otherwise it will fail to install the dependencies.

I use github actions for CI/CD. we run unit tests build a docker image. I've been using the Active LTS, node 14, for my application, However NPM 7 is bundled with node 15.x or higher so in order to use that in github actions, you need to either use v15.x node or use node 14 and install npm@7 manually in your CI job. Unfortunately, there is a permission issue when running `npm i -g npm@7` (Find more about it in this [github issue](https://github.com/actions/setup-node/issues/213)). So my option was just to bump the node version, possibly next LTS version rather than experimental version. In order to avoid any discrepancy, I prefered to use the same node version for my tests and application, so I bumped the node version for the both process.

Luckily, all tests passed, deployed it to staging, it worked great. In the end it was deployed to production 🚀

I've moved on to other projects too and repeated the same procedure.


### Conclusion
It was a bit frustrating process, keeping up-to-date with several dependencies is a bit challenging. Probably some of the fix would be released in a few weeks from amazing folks, and all of these effort won't be necessary. However I'm relieved that I could find the work around so that I don't get stressed whenever I switch to a project requires "special treatment" and cognitive effort to remember things(e.g. I need to switch my npm versions for this project otherwise I will update some random package-lock.json file which I'm not sure what I'm updating).
