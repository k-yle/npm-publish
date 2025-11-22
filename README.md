Since 2017 we've been publishing npm packages from the CI using `npm publish || true`.
If the `version` field in package.json had not been updated, then publishing would silently fail.
So we could simply run this command on every push to the mainline branch.
This is pretty crude but it's easy to setup in all the repos.

Due to [security changes in November 2025](https://github.blog/changelog/2025-09-29-strengthening-npm-security-important-changes-to-authentication-and-token-management/),
we need to switch to [Trusted Publishers](https://docs.npmjs.com/trusted-publishers).

# New Process

1. Go to [the GitHub repo Settings -> Environments](https://github.com/***/***/settings/environments), click <kbd>New Environment</kbd>:
   - environment name: `npm`
   - change <kbd>No restriction</kbd> to <kbd>Protected branches only</kbd>
2. Go to [the npm package settings](https://www.npmjs.com/package/***/access):
   - enter the repo name
   - Workflow filename: `ci.yml`
   - Environment name: `npm`
   - click submit
   - change 'Publishing access' to `Require two-factor authentication and disallow tokens (recommended)`
3. Delete `NPM_AUTH_TOKEN` from [the GitHub secrets](https://github.com/***/***/settings/secrets/actions), to avoid confusion.
4. Remove the `trypublish` script from `package.json`.
5. Replace the existing CI scripts in `.github/workflows` with something like this:

```yaml
# .github/workflows/ci.yaml
name: CI

on:
  push:
    branches:
      - '**'
  pull_request:

permissions:
  contents: read
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x, 22.x, 24.x]

    environment:
      name: npm

    steps:
      - name: ⏬ Checkout code
        uses: actions/checkout@v6

      - name: ⏬ Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v6
        with:
          node-version: ${{ matrix.node-version }}
          registry-url: 'https://registry.npmjs.org'

      - name: ⏬ Install
        run: npm ci

      - name: ✨ Lint
        run: npm run lint

      - name: 🔨 Build
        run: npm run build

      - name: 🧪 Test
        run: npm test

      - uses: k-yle/npm-publish@v1
        if: ${{ matrix.node-version == '24.x' }} # prevent race conditions, only one of the matrix jobs should try publish
```

# Future Optimisations

- defining `on.push` and `on.pull_request` makes the CI run twice for first-party PRs ([known issue](https://github.com/orgs/community/discussions/26940))
- setting `environment` on every build job messes up the github UI; it thinks every commit is deploying to npm… No easy solution since `environment` is defined per-job, and [composite actions](https://docs.github.com/en/actions/tutorials/create-actions/create-a-composite-action) can't define multiple conditional jobs
- surely there'll be an official CI action for something as common as publishing to npm??
