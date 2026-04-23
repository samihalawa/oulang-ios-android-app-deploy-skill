---
name: oulang-ios-android-app-deploy-skill
description: "Deploy Oulang web plus iOS/Android app releases using the repo's real GitHub Actions and Fastlane flow, with commit/push, store upload, and review-submission rules grounded in the current codebase."
---

# Oulang iOS Android App Deploy Skill

Use this skill when the user wants to ship Oulang code and then redeploy:

- web
- Android to Google Play
- iOS to App Store Connect
- both stores together
- review submission when the repo already supports it

This skill exists to reuse the deploy path that already works in the Oulang repo instead of inventing a second release system.

The user's requested release scope is:

`$ARGUMENTS`

## Source Of Truth

Base this workflow on the current Oulang repo evidence first:

- root `README.md`
- repo `AGENTS.md`
- root `package.json`
- root `fastlane/Fastfile`
- `ios/App/fastlane/Fastfile`
- `.github/workflows/deploy.yml`
- `scripts/materialize-mobile-release-assets.mjs`
- current `.env`

If a previous session or thread is named explicitly, read that too before shipping.

## Core Rule

Do not invent new signing, release, or packaging logic.

Prefer the repo's existing flow in this order:

1. commit and push the code
2. let `git push origin main` trigger the web deploy pipeline
3. trigger `.github/workflows/deploy.yml` for mobile
4. fall back to the existing root Fastlane lanes only when GitHub Actions is unavailable or the user explicitly wants local release execution

Use the iOS-local `ios/App/fastlane/Fastfile` only when the root Fastlane path is insufficient for the requested task.

## Exact Working Oulang Flow

### Web

The repo's web deploy path is:

1. commit changes
2. push `main`
3. verify Cloud Build / Cloud Run rolled forward
4. verify the live site

Treat web as deployed only after live checks pass.

### Mobile default path

Preferred mobile release path:

```bash
gh workflow run deploy.yml --ref main --field deploy_target=both
```

Or:

```bash
gh workflow run deploy.yml --ref main --field deploy_target=android
gh workflow run deploy.yml --ref main --field deploy_target=ios
```

Why this is preferred:

- it already materializes release assets from the root `.env`
- it already runs the repo Fastlane production lanes
- it already handles Capacitor sync
- it already uploads the Android AAB artifact
- it already installs the Apple certificate and provisioning profile in CI

### Local fallback path

If local execution is required, use the root repo lanes:

```bash
pnpm run fastlane:android:production
pnpm run fastlane:ios:production
```

Those lanes already:

- validate payments
- materialize mobile release assets
- build web if needed
- run Capacitor sync
- increment Android versionCode from Play
- increment iOS build number from TestFlight
- upload to Play / App Store Connect

For local iOS fallback, keep the real repo preconditions explicit:

- run from a Mac with Xcode available
- ensure release signing material can be materialized from the root `.env`
- make sure pods are installed in `ios/App` before treating a local iOS release as blocked by Fastlane itself

## iOS Review Submission Rule

Be exact about stage truth.

`bundle exec fastlane ios production` uploads to App Store Connect.

Review submission only happens when `APPLE_SUBMIT_FOR_REVIEW=true`.

So:

- `uploaded to App Store Connect` is not the same as `submitted for review`
- `build created` is not the same as `available in TestFlight`
- `processing` is not the same as `ready for sale`

If the user asks for review submission and the workflow does not currently pass `APPLE_SUBMIT_FOR_REVIEW=true`, either:

1. run the iOS production lane locally with that env var set, or
2. explicitly update the release execution path so review submission is actually enabled

Do not falsely collapse these stages.

## Required Workflow

### 1. Read Current State

Before deploying:

- read the current thread
- read the current repo deploy files
- check `git status --short --ignore-submodules=all`
- check `git log --oneline -5`
- confirm branch and remote
- check whether the worktree is clean enough to ship

If the task depends on prior release attempts, recover the relevant local session history first.

### 2. Validate Minimally

Run the smallest credible proof for the touched change before shipping.

Typical default:

```bash
pnpm -s exec tsc --noEmit --pretty false
pnpm build
```

If the repo already has a narrower proof path that better matches the change, use that instead.

### 3. Commit And Push

If there are tracked changes for the requested work:

1. stage only the intended files
2. create an atomic commit
3. push to the correct remote branch
4. if the intention is a real Oulang web release, land it on `main`

Do not stop at a local commit when push access exists.

### 4. Trigger Mobile Release

Default:

```bash
gh workflow run deploy.yml --ref main --field deploy_target=both
```

Then inspect the run:

```bash
gh run list --workflow deploy.yml --limit 5
gh run watch <run-id>
```

If the user wants only one store, trigger only that target.

### 5. Local Fallback If Needed

If GitHub Actions is blocked or unavailable, run the root Fastlane lanes from the repo.

Android:

```bash
pnpm run fastlane:android:production
```

iOS upload:

```bash
pnpm run fastlane:ios:production
```

iOS upload plus review submission:

```bash
APPLE_SUBMIT_FOR_REVIEW=true pnpm run fastlane:ios:production
```

Only use this fallback when it is actually needed.

If local iOS execution is used, ensure repo-native prerequisites are satisfied first:

```bash
cd ios/App && pod install
```

Do not misdiagnose a missing pod install or missing local signing material as a broken Fastlane lane.

### 6. Verify Exact Stage

Always report the true stage:

- committed
- pushed
- web deployed
- Android build uploaded
- Android processing
- iOS build uploaded
- iOS processing
- submitted for review
- approved
- released

Do not compress these into "deployed".

For web, verify with the repo's live checks:

```bash
gcloud builds list --project=megacursos --limit=3 --format='table(id,status,startTime,source.repoSource.commitSha)'
gcloud run services describe oulang-web --project=megacursos --region=us-central1 --format='value(status.latestReadyRevisionName,status.latestCreatedRevisionName)'
gcloud run services describe oulang-web --project=megacursos --region=us-central1 --format='yaml(status.traffic)'
curl -sI 'https://oulang.ai' | grep -iE 'x-cloud-trace|last-modified|etag'
```

For mobile, verify through GitHub Actions run state first, then any available store-upload evidence.

Remember the current workflow default:

- Android CI path uploads production to Play
- iOS CI path uploads to App Store Connect
- iOS CI path does not automatically submit for review unless the execution path is explicitly changed to do that

## Release Asset Rules

Respect the repo's current asset materialization path.

Do not hand-roll files that the repo already materializes from `.env`.

Current flow already handles:

- Android `google-services.json`
- Google Play service account JSON
- Android release keystore
- iOS `GoogleService-Info.plist`
- App Store Connect API key `.p8`
- Apple distribution certificate `.p12`
- provisioning profile

## Notifications And Failure Signals

If previous sessions mention App Store / Fastlane / notification release issues:

- treat them as release-history evidence
- prefer the current repo workflow over stale memory
- check the exact failing stage before changing anything

Examples:

- signing failure
- missing provisioning profile
- missing API key materialization
- build uploaded but not review-submitted
- store-processing delay mistaken for failure

Do not rewrite the whole release flow because of one stale failure log.

## Hard Rules

- do not add wrapper scripts for release tasks
- do not invent a second deploy pipeline
- do not bypass the repo's `.env`-driven asset materialization
- do not claim App Store review submission unless it actually happened
- do not claim release completion from a green build alone
- do not mutate `.env` or secrets unless the user explicitly asks
- do not leave the repo dirty after release work

## Default Outcome

When this skill is used well, the result should be:

1. requested code shipped with a clean commit and push
2. web redeployed through the existing main-branch path
3. Android and/or iOS release triggered through the existing deploy workflow
4. exact release stage verified and reported truthfully
