# Storybook Test Runner

Storybook test runner turns all of your stories into executable tests.

## Table of Contents

- [Storybook Test Runner](#storybook-test-runner)
  - [Table of Contents](#table-of-contents)
  - [Features](#features)
  - [Getting started](#getting-started)
  - [Configuration](#configuration)
  - [Running against a deployed Storybook](#running-against-a-deployed-storybook)
    - [Stories.json mode](#storiesjson-mode)
  - [Running in CI](#running-in-ci)
    - [1. Running against deployed Storybooks on Github Actions deployment](#1-running-against-deployed-storybooks-on-github-actions-deployment)
    - [2. Running against locally built Storybooks in CI](#2-running-against-locally-built-storybooks-in-ci)
  - [Experimental test hook API](#experimental-test-hook-api)
  - [Troubleshooting](#troubleshooting)
    - [The test runner seems flaky and keeps timing out](#the-test-runner-seems-flaky-and-keeps-timing-out)
    - [Adding the test runner to other CI environments](#adding-the-test-runner-to-other-ci-environments)
  - [Future work](#future-work)

## Features

- ⚡️ Zero config setup
- 💨 Smoke test all stories
- ▶️ Test stories with play functions
- 🏃 Test your stories in parallel in a headless browser
- 👷 Get feedback from error with a link directly to the story
- 🐛 Debug them visually and interactively in a live browser with [addon-interactions](https://storybook.js.org/docs/react/essentials/interactions)
- 🎭 Powered by [Jest](https://jestjs.io/) and [Playwright](https://playwright.dev/)
- 👀 Watch mode, filters, and the conveniences you'd expect

## Getting started

1. Install the test runner and the interactions addon in Storybook:

```jsx
yarn add @storybook/test-runner -D
```

Jest is a peer dependency. If you don't have it, also install it

```jsx
yarn add jest -D
```

<details>
  <summary>1.1 Optional instructions to install the Interactions addon for visual debugging of play functions</summary>

```jsx
yarn add @storybook/addon-interactions @storybook/jest @storybook/testing-library -D
```

Then add it to your `.storybook/main.js` config and enable debugging:

```jsx
module.exports = {
  stories: ['@storybook/addon-interactions'],
  features: {
    interactionsDebugger: true,
  },
};
```

</details>

2. Add a `test-storybook` script to your package.json

```json
{
  "scripts": {
    "test-storybook": "test-storybook"
  }
}
```

3. Run Storybook (the test runner runs against a running Storybook instance):

```jsx
yarn storybook
```

4. Run the test runner:

```jsx
yarn test-storybook
```

> **NOTE:** The runner assumes that your Storybook is running on port `6006`. If you're running Storybook in another port, set the TARGET_URL before running your command like:
>
> ```jsx
> TARGET_URL=http://localhost:9009 yarn test-storybook
> ```

## Configuration

The test runner is based on [Jest](https://jestjs.io/) and will accept the [CLI options](https://jestjs.io/docs/cli) that Jest does, like `--watch`, `--maxWorkers`, etc.

The test runner works out of the box, but you can override its Jest configuration by adding a `test-runner-jest.config.js` that sets up your environment in the root folder of your project.

```js
// test-runner-jest.config.js
module.exports = {
  cacheDirectory: 'node_modules/.cache/storybook/test-runner',
  testMatch: ['**/*.stories.[jt]s(x)?'],
  transform: {
    '^.+\\.stories\\.[jt]sx?$': '@storybook/test-runner/playwright/transform',
    '^.+\\.[jt]sx?$': 'babel-jest',
  },
  preset: 'jest-playwright-preset',
  testEnvironment: '@storybook/test-runner/playwright/custom-environment.js',
  testEnvironmentOptions: {
    'jest-playwright': {
      browsers: ['chromium', 'firefox', 'webkit'], // run tests against all browsers
    },
  },
};
```

The runner uses [jest-playwright](https://github.com/playwright-community/jest-playwright) and you can pass [testEnvironmentOptions](https://github.com/playwright-community/jest-playwright#configuration) to further configure it, such as how it's done above to run tests against all browsers instead of just chromium.

## Running against a deployed Storybook

By default, the test runner assumes that you're running it against a locally served Storybook.
If you want to define a target url so it runs against deployed Storybooks, you can do so by passing the `TARGET_URL` environment variable:

```bash
TARGET_URL=https://the-storybook-url-here.com yarn test-storybook
```

### Stories.json mode

By default, the test runner transforms your story files into tests. It also supports a secondary "stories.json mode" which runs directly against your Storybook's `stories.json`, a static index of all the stories.

This is particularly useful for running against a deployed storybook because `stories.json` is guaranteed to be in sync with the Storybook you are testing. In the default, story file-based mode, your local story files may be out of sync--or you might not even have access to the source code.

To run in stories.json mode, first make sure your Storybook has a v3 `stories.json` file. You can navigate to:

```
https://the-storybook-url-here.com/stories.json
```

It should be a JSON file and the first key should be `"v": 3` followed by a key called `"stories"` containing a map of story IDs to JSON objects.

If your Storybook does not have a `stories.json` file, you can generate one provided:

- You are running SB6.4 or above
- You are not using `storiesOf` stories

To enable `stories.json` in your Storybook, set the `buildStoriesJson` feature flag in `.storybook/main.js`:

```js
module.exports = {
  features: { buildStoriesJson: true },
};
```

Once you have a valid `stories.json` file, you can run the test runner against it with the `--stories-json` flag:

```bash
TARGET_URL=https://the-storybook-url-here.com yarn test-storybook --stories-json
```

> **NOTE:** stories.json mode is not compatible with watch mode.

## Running in CI

If you want to add the test-runner to CI, there are a couple of ways to do so:

### 1. Running against deployed Storybooks on Github Actions deployment

On Github actions, once services like Vercel, Netlify and others do deployment runs, they follow a pattern of emitting a `deployment_status` event containing the newly generated URL under `deployment_status.target_url`. You can use that URL and set it as `TARGET_URL` for the test-runner.

Here's an example of an action to run tests based on that:

```yml
name: Storybook Tests
on: deployment_status
jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    if: github.event.deployment_status.state == 'success'
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '14.x'
      - name: Install dependencies
        run: yarn
      - name: Run Storybook tests
        run: yarn test-storybook
        env:
          TARGET_URL: '${{ github.event.deployment_status.target_url }}'
```

> **_NOTE:_** If you're running the test-runner against a `TARGET_URL` of a remotely deployed Storybook (e.g. Chromatic), make sure that the URL loads a publicly available Storybook. Does it load correctly when opened in incognito mode on your browser? If your deployed Storybook is private and has authentication layers, the test-runner will hit them and thus not be able to access your stories. If that is the case, use the next option instead.

### 2. Running against locally built Storybooks in CI

In order to build and run tests against your Storybook in CI, you might need to use a combination of commands involving the [concurrently](https://www.npmjs.com/package/concurrently), [http-server](https://www.npmjs.com/package/http-server) and [wait-on](https://www.npmjs.com/package/wait-on) libraries. Here's a recipe that does the following: Storybook is built and served locally, and once it is ready, the test runner will run against it.

```json
{
  "test-storybook:ci": "concurrently -k -s first -n \"SB,TEST\" -c \"magenta,blue\" \"yarn build-storybook --quiet && npx http-server storybook-static --port 6006 --silent\" \"wait-on tcp:6006 && yarn test-storybook\""
}
```

And then you can essentially run `test-storybook:ci` in your CI:

```yml
name: Storybook Tests
on: push
jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '14.x'
      - name: Install dependencies
        run: yarn
      - name: Run Storybook tests
        run: yarn test-storybook:ci
```

> **_NOTE:_** Building Storybook locally makes it simple to test Storybooks that could be available remotely, but are under authentication layers. If you also deploy your Storybooks somewhere (e.g. Chromatic, Vercel, etc.), the Storybook URL can still be useful with the test-runner. You can pass it to the `REFERENCE_URL` environment variable when running the test-storybook command, and if a story fails, the test-runner will provide a helpful message with the link to the story in your published Storybook instead.

## Experimental test hook API

The test runner renders a story and executes its [play function](https://storybook.js.org/docs/react/writing-stories/play-function) if one exists. However, there are certain behaviors that are not possible to achieve via the play function, which executes in the browser. For example, if you want the test runner to take visual snapshots for you, this is something that is possible via Playwright/Jest, but must be executed in Node.

To enable use cases like visual or DOM snapshots, the test runner exports test hooks that can be overridden globally. These hooks give you access to the test lifecycle before and after the story is rendered.

Consider the following pseudocode:

```js
it('component--widget', async () => {
  const page = newPage();
  const context = { id: 'component--widget', title: 'Component', name: 'Widget' };
  await page.goto(STORYBOOK_URL);

  // pre-render hook
  if (preRender) await preRender(page, context);

  // render the story and run its paly function (if applicable)
  await page.execute('render', context);

  // post-render hook
  if (postRender) await postRender(page, context);
});
```

The hooks here, `preRender` and `postRender` are functions that take a [Playwright Page](https://playwright.dev/docs/pages) and a context object with the current story `id`, `title`, and `name`. They are globally settable by `@storybook/test-runner`'s `setPreRender` and `setPostRender` APIs.

Thus, to make the test runner perform image snapshotting, you might set up the following in your `jest-setup.js`:

```js
const { toMatchImageSnapshot } = require('jest-image-snapshot');
const { setPostRender } = require('@storybook/test-runner');

expect.extend({ toMatchImageSnapshot });

// use custom directory/id to align CSF and stories.json mode outputs
const customSnapshotsDir = `${process.cwd()}/__snapshots__`;

setPostRender(async (page, context) => {
  const image = await page.screenshot();
  expect(image).toMatchImageSnapshot({
    customSnapshotsDir,
    customSnapshotIdentifier: context.id,
  });
});
```

> **NOTE:** These test hooks are experimental and may be subject to breaking changes. We encourage you to test as much as possible within the story's play function when that's possible.

## Troubleshooting

#### The test runner seems flaky and keeps timing out

If your tests are timing out with `Timeout - Async callback was not invoked within the 15000 ms timeout specified by jest.setTimeout`, it might be that playwright couldn't handle to test the amount of stories you have in your project. Maybe you have a large amount of stories or your CI has a really low RAM configuration.

In either way, to fix it you should limit the amount of workers that run in parallel by passing the [--maxWorkers](https://jestjs.io/docs/cli#--maxworkersnumstring) option to your command:

```json
{
  "test-storybook:ci": "concurrently -k -s first -n \"SB,TEST\" -c \"magenta,blue\" \"yarn build-storybook --quiet && npx http-server storybook-static --port 6006 --silent\" \"wait-on tcp:6006 && yarn test-storybook --maxWorkers=2\""
}
```

#### Adding the test runner to other CI environments

As the test runner is based on playwright, depending on your CI setup you might need to use specific docker images or other configuration. In that case, you can refer to the [Playwright CI docs](https://playwright.dev/docs/ci) for more information.

## Future work

Future plans involve adding support for the following features:

- 🧪 Custom test functions
- 📄 Run addon reports
