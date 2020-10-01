<p align="center">
  <img alt="node-recorder logo" src="./logo.gif" width="50%">
</p>

- Automatically record new HTTP(s) requests.
- Replay recordings when testing.
- Customize responses.
- Works well with [supertest](https://github.com/visionmedia/supertest).
- Predictable, deterministic filepaths.
- Normalize the `request` & `response`.
- Associate session-based cookies & OAuth tokens to users.
- Ignore requests you don't want to record.

---

<p align="center">
  <img alt="node-recorder demo" src="./demo.gif">
</p>

---

## Installation

```shell
$ yarn add node-recorder --dev
# or
$ npm install node-recorder --save-dev
```

## Getting Started

- By simply including `node-recorder`, **all HTTP(s) requests are intercepted**.
- By default, `RECORD` mode records new recordings, and replays existing fixures.
- When in `NODE_ENV=test` or `CI=true`, `REPLAY` mode replays existing recordings, and throws an error when one doesn't exist.
  _(So that local tests don't suddenly fail in CI)_

### Recorder Modes

- `bypass` - All network requests bypass the recorder and respond as usual.
- `record` - Record only new network requests (i.e. those without recordings), while replaying existing recordings.
- `replay` - Replay all network requests using recordings. **If a recording is missing, an error is thrown**.
- `rerecord` - Re-record all network requests.

### Using `node --require`

```shell
$ node -r node-recorder path/to/server.js
```

_(This also works with `mocha`!)_

### Setting the `mode` via `RECORDER=...`

```shell
$ RECORDER=ignore node -r node-recorder path/to/server.js
```

### Using Jest

Included is a `jest-preset` that will automatically include `node-recorder` and a custom plugin to make toggling modes easier.

```js
// jest.config.js
module.exports = {
  preset: "node-recorder/jest-preset"
};
```

Now, running `jest --watch` will add a new `r` option:

```
Watch Usage
 › Press a to run all tests.
 › Press f to run only failed tests.
 › Press p to filter by a filename regex pattern.
 › Press t to filter by a test name regex pattern.
 › Press q to quit watch mode.
 › Press r to change recording mode from "REPLAY".
 › Press Enter to trigger a test run.
```

Pressing `r` will toggle between the various modes:

```
  ╭─────────────────────────────╮
  │                             │
  │   node-recorder:  RECORD    │
  │                             │
  ╰─────────────────────────────╯
```

### Configuring `recorder.config.js`

Within your project, you can create a `recorder.config.js` that exports:

```js
// recorder.conig.js
module.exports = {
  identify(request, response) {...},
  ignore(request) {...},
  normalize(request, response) {...}
}
```

- `request` is the same as the recording (e.g. `body`, `headers`, `href`, `method`), but
  with an additional `url` property from https://github.com/unshiftio/url-parse to simplify conditional logic.
- `response` contains `body`, `headers`, & `statusCode`.

#### `identify` a `request` or `response

This is useful when network requests are stateful, in that they rely on an authorization call first, then they pass along a token/cookie to subsequent calls:

1. Suppose you login by calling `/login?user=foo&password=bar`.
2. The response contains `{ "token": "abc123" }3. Now, to get data, you call`/api?token=abc123`.

When recording recordings, the token `abc123` isn't clearly associated with the user `foo`.

To address this, you can `identify` the `request` and `response`, so that the recordings are aliased accordingly:

```js
identify(request, response) {
  const { user, token } = request.query

  if (request.href.endsWith("/login")) {
    // We know the user, but not the token yet
    if (!response) {
      return user
    }

    // Upon login, associate this `user` with the `token`
    return [user, response.body.token]
  }

  // API calls supply a `token`, which has been associated with a `user`
  if (request.href.endsWith("/api")) {
    return token
  }
}
```

Now, when recorded recordings will look like:

- `127.0.0.1/login/${hash}.${user}.json`
- `127.0.0.1/api/${hash}.${user}.json`

This way, similar-looking network requests (e.g. login & GraphQL) can be differentiated and easily searched for.

#### `ignore` a `request`

Typically, you don't want to record recordings for things like analytics or reporting.

```js
// recorder.conig.js
module.exports = {
  ignore(request) {
    if (request.href.includes("www.google-analytics.com")) {
      return true;
    }

    return false;
  }
};
```

#### `normalize` a `request` or `response`

Recordings are meant to make development & testing _easier_, so modification is necessary.

- **Changing `request` changes the filename `hash` of the recording**. You may need to `record` again.
- `normalize` is called **before** the network request and **after**. This means that `response` may be `undefined`!
- You can **change `response` by hand, or via `normalize` without affecting the filename `hash` of the recording**.

```js
module.exports = {
  normalize(request, response) {
    // Suppose you never care about `user-agent`
    delete request.headers["user-agent"];

    // We may not have a response (yet)
    if (response) {
      // ...or the `date`
      delete response;
    }
  }
};
```

## MIT License

## Author

- Eric Clemmons
