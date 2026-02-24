Since 2017 we've been publishing npm packages from the CI using `npm publish || true`.
If the `version` field in package.json had not been updated, then publishing would silently fail.
So we could simply run this command on every push to the mainline branch.
This is pretty crude but it's easy to setup in all the repos.

Due to [security changes in November 2025](https://github.blog/changelog/2025-09-29-strengthening-npm-security-important-changes-to-authentication-and-token-management/),
we need to switch to [Trusted Publishers](https://docs.npmjs.com/trusted-publishers).

# New Process

1. Ensure that [branch protection rules](https://github.com/***/***/settings/branch_protection_rules) are enabled for the mainline branch
2. Go to [the npm package settings](https://www.npmjs.com/package/***/access):
   - enter the repo name
   - Workflow filename: `ci.yml`
   - Environment name: _blank_
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
        node-version:
          - { v: 18 }
          - { v: 20 }
          - { v: 22 }
          - { v: 24, isLatest: yes }

    steps:
      - name: ⏬ Checkout code
        uses: actions/checkout@v6

      - name: ⏬ Use Node.js ${{ matrix.node-version.v }}
        uses: actions/setup-node@v6
        with:
          node-version: ${{ matrix.node-version.v }}
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
        if: ${{ matrix.node-version.isLatest }} # prevent race conditions, only one of the matrix jobs should try publish
```

# Future Optimisations

- defining `on.push` and `on.pull_request` makes the CI run twice for first-party PRs ([known issue](https://github.com/orgs/community/discussions/26940))
- surely there'll be an official CI action for something as common as publishing to npm??
