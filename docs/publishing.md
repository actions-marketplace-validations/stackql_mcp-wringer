# Publishing releases

Both publication steps are performed by a maintainer. npm publication is manual and uses interactive 2FA. Publishing a GitHub release does not publish the npm package: the [release workflow](../.github/workflows/release.yml) only validates the tagged artifacts. StackQL CI integration is a separate follow-up.

## Prepare the release

Use Node 24 and a current npm CLI. Start from a reviewed checkout with dependencies installed using `npm ci`. Run each command below separately, and stop if any command fails:

```sh
npm run check
npm run test:matrix
npm run build
npm run check:action
npm run test:action
npm run check:plugin-api
npm run check:package
npm run docs:config:check
```

`check:action` rebuilds the committed Action bundle and checks for drift. If the bundle needs updating, review and commit the generated changes before rerunning the checks. `check:package` previews the npm tarball and checks its file allowlist and required entry points. Review that listing for credentials or unwanted files as well.

The initial package version is `0.1.0`, with release tag `v0.1.0`. For later releases, update the package and lockfile together with `npm version patch --no-git-tag-version` (or `minor` / `major` as appropriate), then repeat the checks. Commit and merge the release changes, and confirm a clean working tree. npm and the Action must be released from the same commit. Published npm versions and `vX.Y.Z` tags must never be overwritten.

## Publish to npmjs

The maintainer's npm account must have 2FA enabled and permission to publish under the `@stackql` organization. Check that the intended version has not already been published. From the validated release checkout, run these commands separately and stop on any error:

```sh
npm login --registry=https://registry.npmjs.org/
npm whoami --registry=https://registry.npmjs.org/
npm publish --access public --registry=https://registry.npmjs.org/
```

Complete the browser login and any 2FA prompts. `--access public` is required for this scoped public package; the package also sets public access in `publishConfig`. Do not put an OTP or npm token into source, scripts, or GitHub secrets. Local publication does not provide GitHub Actions OIDC provenance; do not add `--provenance` to this manual procedure.

Verify the published version (substitute the release version for later releases). Run the `npx` command from a directory outside the repository checkout: inside it, `npx` matches the checkout's own package name and fails with `'mcp-wringer' is not recognized` or `command not found`, because the package's bin is not linked into its own `node_modules/.bin`.

```sh
npm view @stackql/mcp-wringer@0.1.0 version dist.integrity --registry=https://registry.npmjs.org/
cd "$(mktemp -d)" && npx --yes --registry=https://registry.npmjs.org/ @stackql/mcp-wringer@0.1.0 --help
```

Confirm the package is public on [npmjs](https://www.npmjs.com/package/@stackql/mcp-wringer). The [npm scoped-package guide](https://docs.npmjs.com/creating-and-publishing-scoped-public-packages) describes account and publication requirements.

## Publish the GitHub Marketplace Action

The repository is public and has one root [action.yml](../action.yml), with display name `MCP Wringer` and entry point `dist-action/index.cjs`. Marketplace publication uses a GitHub release, not `npm publish`, and must be completed in the GitHub UI by a maintainer with release permissions.

1. Confirm npm publication succeeded and the release commit passed CI, including the committed Action bundle check.
2. Open [action.yml on GitHub](https://github.com/stackql/mcp-wringer/blob/main/action.yml) and select **Draft a release** from the Marketplace banner, or open the repository's [new release form](https://github.com/stackql/mcp-wringer/releases/new).
3. Select **Publish this Action to the GitHub Marketplace**. If disabled, a StackQL organization owner must accept the GitHub Marketplace Developer Agreement using the link in the release form.
4. Check GitHub's metadata validation, including the uniqueness of `MCP Wringer`. Resolve any error before proceeding; name availability is not guaranteed until GitHub validates it.
5. Choose **Testing** as the primary category, and optionally **Security** as the secondary category.
6. Create tag `v0.1.0` at the exact validated release commit (use the matching `vX.Y.Z` for later releases), title the release `v0.1.0`, and include release notes. Do not select a prerelease for the stable release.
7. Click **Publish release** and complete GitHub's authentication prompts. Confirm the Marketplace listing is visible and the **Validate release** workflow succeeds. This workflow does not publish to npm.
8. After validation, create or update the moving `v0` major tag to the same commit. Keep `v0.1.0` immutable. The workflow examples use `@v0` to receive updates within that major version.
9. Run the [Published Action smoke](../.github/workflows/published-action-smoke.yml) workflow from the Actions tab. It pulls the Action from `@v0` and the package from the npm registry, runs both against the fixture server at that tag on Linux (x64 and arm64), macOS and Windows, and checks that the npm `latest` version matches the tag. It also runs weekly as a canary.

Test the published Action against an isolated authorized server before integrating it into another project's CI. The StackQL repository runs it as a pull request gate in its `mcp-wringer` workflow, which builds the server from the pull request and fuzzes stdio and Streamable HTTP in both protocol revisions on each platform it ships. Record the Marketplace URL and release commit SHA. See [GitHub's Marketplace publication guide](https://docs.github.com/en/actions/how-tos/create-and-publish-actions/publish-in-github-marketplace) and the [maintainer release checklist](./dev/release-checklist.md).
