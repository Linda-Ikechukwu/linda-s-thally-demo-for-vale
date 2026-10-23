# Friction log

While building this demo site, I documented every error and rough edge I ran into, along with how to reproduce it, why it matters, and what I'd suggest. They're ordered by impact.

**Environment:** macOS, Node.js 22.23, npm 10.9, Thally starter version 3 (runtime commit `2e09435`), Thally Cloud free plan, October 22 to 23, 2026.

| # | Issue | Area | Impact |
| --- | --- | --- | --- |
| 1 | The new-repository path requires GitHub access to all repositories | Onboarding | High |
| 2 | MDX changes don't reload under `npm run dev` | Authoring | High |
| 3 | A failed deployment from a push isn't reported anywhere | Publishing | High |
| 4 | `npm run build` passes while `thally check` fails | Validation | Medium |
| 5 | `thally check` doesn't check links in `site.ts` | Validation | Medium |
| 6 | A navigation entry with no file still appears in the sidebar | Validation | Medium |
| 7 | The quickstart and `AGENTS.md` list different pre-publish commands | Docs | Medium |
| 8 | The starter uses `mode: home`, which isn't documented | Docs | Low |
| 9 | The header adds "Docs" to the site name | Configuration | Low |
| 10 | An unmodified starter logs 404s for `/api/brand/logo` | Local development | Low |
| 11 | Deleting the API pages still publishes `openapi.yaml` | Starter | Low |

## 1. The new-repository path requires GitHub access to all repositories

**Steps to reproduce**

1. In Thally Cloud, create a site and choose **Create a new docs repository**.
2. When GitHub asks where to install the Thally GitHub App, follow the quickstart and choose **All repositories**.
3. Finish setup, then open **Site settings > Git**.

**What happens:** The Git settings page shows the account with "All repositories," and the repository dropdown lists every repository in the account, including private ones. Earlier in setup, a screen asks you to pick a repository. That only chooses the site's source and doesn't narrow the app's access, but it reads as if it does.

**Why it matters:** A docs tool asking for access to every private repository is a hard stop for most security reviews, and it happens before the user has seen any value.

**Suggestion:** Create the repository using the user's own GitHub authorization, then install the app on that single repository. Alternatively, prompt the user to narrow access to the new repository right after setup, and explain each permission in plain language.

## 2. MDX changes don't reload under `npm run dev`

**Steps to reproduce**

1. Run `npm run dev` and open `http://localhost:3040`.
2. Edit any file in `src/content/` and save it.
3. Refresh the page.

**What happens:** The page still shows the old content. The `predev` script compiles the pages once, through `npm run runtime-sources:build`, so changes only appear after restarting the server or rerunning that script.

**Why it matters:** Edit, save, and refresh is the core loop for a docs author. Without it, every small change costs a restart, and new users assume their edit didn't work.

**Suggestion:** Watch `src/content` during development and recompile changed pages.

## 3. A failed deployment from a push isn't reported anywhere

**Steps to reproduce**

1. Run `npm install` locally. In my case, this rewrote `package-lock.json`.
2. Commit the changed lockfile and push to `main`.

**What happens:** Thally's build runs `npm ci`, which fails because `package.json` and `package-lock.json` are out of sync (`Missing: @emnapi/runtime@1.11.3 from lock file`). The live site correctly stays on the previous version, but nothing tells you. No status appears on the GitHub commit, and the deployment didn't appear in the dashboard until I clicked **Redeploy site**.

**Why it matters:** The author believes their changes are live when they aren't, and has no clue where to look.

**Suggestion:** Post every deployment result to GitHub as a commit status, and show push-triggered deployments, including failures, in the dashboard as soon as they start.

## 4. `npm run build` passes while `thally check` fails

**Steps to reproduce**

1. Add a page path to `docs.json` without creating its `.mdx` file.
2. Run `npm run build`, then `npm run check`.

**What happens:** The build succeeds. The check reports that the page is listed in `docs.json` but has no file, plus any link to it.

**Why it matters:** There are two answers to "is my site OK?" A site that builds locally can still be rejected when it's published.

**Suggestion:** Make `thally check` the single source of truth by running it as part of `prebuild`, so local builds fail the same way hosted builds do.

## 5. `thally check` doesn't check links in `site.ts`

**Steps to reproduce**

1. Delete `src/content/changelog.mdx`.
2. Leave the `{ label: "Changelog", href: "/changelog" }` entry in `src/data/site.ts`.
3. Run `npm run check`.

**What happens:** The check reports no issues, but the footer's Changelog link goes to a "page not found" error.

**Suggestion:** Validate internal links in `site.ts` the same way as links in MDX files.

## 6. A navigation entry with no file still appears in the sidebar

**Steps to reproduce**

1. Add `guides/house-style` to a group in `docs.json` without creating the file.
2. Run `npm run dev` and open the site.

**What happens:** The sidebar shows "House Style," a title generated from the file name, and clicking it leads to a "page not found" error. `thally check` does catch this, but only if you run it.

**Suggestion:** Show a warning while the dev server runs, or leave the entry out of the sidebar until the file exists.

## 7. The quickstart and `AGENTS.md` list different pre-publish commands

**What happens:** The Thally quickstart says to run `npx @thallylabs/cli@latest check` and `npm run build`. The starter's `AGENTS.md` lists `npm ci --prefix .github/thally-tooling`, `thally check --ci .`, `npm test`, and `npm run build`. `AGENTS.md` also opens with Thally's internal architecture and links to a private repository customers can't open. `package.json` has 38 scripts, with no hint about which ones a docs author needs.

**Why it matters:** Coding agents read `AGENTS.md` first, so they inherit the confusion.

**Suggestion:** Split `AGENTS.md` into customer guidance and internal maintainer notes, with one list of pre-publish commands.

## 8. The starter uses `mode: home`, which isn't documented

**What happens:** `src/content/introduction.mdx` starts with `mode: home` in its frontmatter. The [page modes guide](https://docs.thally.io/guides/page-modes) lists only `default`, `wide`, `center`, and `custom`.

**Why it matters:** When I removed the starter's hero banner, I couldn't tell whether to keep the setting or what it did.

**Suggestion:** Document `home`, or switch the starter to a documented mode.

## 9. The header adds "Docs" to the site name

**Steps to reproduce:** Set the site name to "Vale Docs."

**What happens:** The header shows "Vale Docs Docs." The comments in `src/data/site.ts` don't mention the suffix.

**Suggestion:** Document the suffix next to the setting, or skip it when the name already ends in "Docs."

## 10. An unmodified starter logs 404s for `/api/brand/logo`

**Steps to reproduce:** Run `npm run dev` on a fresh starter and open the browser console.

**What happens:** The console shows repeated 404 errors for `/api/brand/logo`. The site falls back to the default logo, so readers see nothing wrong.

**Why it matters:** Red errors on a fresh site look like a bug to a new user.

**Suggestion:** Skip the request locally when no Thally Cloud brand is configured.

## 11. Deleting the API pages still publishes `openapi.yaml`

**Steps to reproduce:** Delete `src/content/api/` and remove the API Reference tab from `docs.json`.

**What happens:** `openapi.yaml` stays at the repository root, and the site still serves it at `/openapi.yaml` and `/openapi.json`. Nothing tells you to remove it.

**Suggestion:** Mention the file in the starter's cleanup guidance, or only serve it when an API tab is configured.
