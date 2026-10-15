# ACA Form Creator — production deployment

Production is Firebase Hosting and Cloud Firestore for project `culinaryartsnp`. The source of truth in this repository is `.github/workflows/firebase-hosting-merge.yml`, `firebase.json`, and `firestore.rules`. There is no Kubernetes manifest, no Kustomize overlay, and no container image. `kubectl kustomize` was not run.

How the app behaves and how to run it locally: [PROJECT_HANDOVER.md](PROJECT_HANDOVER.md).

## Fact table

| | |
| --- | --- |
| Server | Firebase (Google). Not a host path in this repo. No `/root/Triambha/` tree is in git. |
| Manifest root | Repository root: `firebase.json`, `firestore.rules`, `.github/workflows/firebase-hosting-merge.yml` |
| What to apply | A push to `main` runs workflow `Deploy to Firebase Hosting on merge`, job `build_and_deploy`. Hosting target is `channelId: live`, `projectId: culinaryartsnp`. Rules deploy is `npx firebase-tools deploy --only firestore:rules --project culinaryartsnp`. |
| Public URL | `https://culinaryartsnp.web.app/` and `https://culinaryartsnp.firebaseapp.com/`. Both returned HTTP 200 on 5 October 2026 with the same etag and `<title>ACA Form Creator</title>`. `firebase.json` does not name a custom domain. |
| Image | None |
| Container port | None. Hosting answered on HTTPS. The HTML response included `strict-transport-security`. Local dev is separate: `vite.config.ts` sets no port; the README says usually `http://localhost:5173`. |
| Replicas | Not set. Hosting is not a Deployment. |
| Ingress | Firebase Hosting. `firebase.json` has no ingress class and no certificate resolver. The live response included HSTS `max-age=31556926; includeSubDomains; preload`. |
| Live release observed | `last-modified: Tue, 15 Sep 2026 10:29:41 GMT`. Etag `3e3430000617986a7d03014aa91f7c8a8c449a188fed29a8cc81e6b453fef174`. Assets in that HTML: `/assets/index-CGAFtmG8.js`, `/assets/index-CA2dvAis.css`. `cache-control: max-age=3600`. |

## Traffic flow

```
Internet
  → https://culinaryartsnp.web.app  or  https://culinaryartsnp.firebaseapp.com
    → Firebase Hosting site for project culinaryartsnp
      → file from dist/, or index.html when the path is not a file
        → browser runs the bundle
          → Firebase Auth
          → Cloud Firestore (forms, submissions, admins, settings/general)
```

There is no container listen port. `firebase.json` sets `"public": "dist"` and rewrites `**` to `/index.html`, so `/login` and unknown paths return the same HTML shell. The React router then chooses the page. Checked on 5 October 2026: `/`, `/login`, and `/does-not-exist` on `culinaryartsnp.web.app` all returned that HTML with HTTP 200.

The local Vite server is not this path. Production does not use port 5173.

A public submit is the browser calling Firestore `addDoc` on `forms/{formId}/submissions`. A successful submit is that write, then the “Thank you” screen. Details of fields written and fields dropped are in [PROJECT_HANDOVER.md](PROJECT_HANDOVER.md).

## Files that define production

```
.
├── firebase.json
├── firestore.rules
└── .github/workflows/firebase-hosting-merge.yml
```

There is no `base/` or `overlays/prod/`. There is no `.firebaserc`. There is no Dockerfile and no compose file.

Firebase resource names after deploy, taken from those files:

| Kind | Name |
| --- | --- |
| Firebase project | `culinaryartsnp` |
| Hosting channel | `live` |
| Hosting public directory | `dist` |
| Firestore rules file | `firestore.rules` |
| GitHub Actions workflow | `Deploy to Firebase Hosting on merge` |
| GitHub Actions job | `build_and_deploy` |

No Kubernetes namespace is set. Confirm the Firebase project in the console or with a CLI logged in as an account that can see `culinaryartsnp`. The workflow always passes `--project culinaryartsnp` for rules and `projectId: culinaryartsnp` for Hosting.

## What production applies

The workflow, in full:

```yaml
name: Deploy to Firebase Hosting on merge
'on':
  push:
    branches:
      - main
jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: npm install
    - run: npm run build
      env:
        VITE_FIREBASE_API_KEY: ${{ secrets.VITE_FIREBASE_API_KEY }}
        VITE_FIREBASE_AUTH_DOMAIN: ${{ secrets.VITE_FIREBASE_AUTH_DOMAIN }}
        VITE_FIREBASE_PROJECT_ID: ${{ secrets.VITE_FIREBASE_PROJECT_ID }}
        VITE_FIREBASE_STORAGE_BUCKET: ${{ secrets.VITE_FIREBASE_STORAGE_BUCKET }}
        VITE_FIREBASE_MESSAGING_SENDER_ID: ${{ secrets.VITE_FIREBASE_MESSAGING_SENDER_ID }}
        VITE_FIREBASE_APP_ID: ${{ secrets.VITE_FIREBASE_APP_ID }}
    - uses: FirebaseExtended/action-hosting-deploy@v0
      with:
        repoToken: '${{ secrets.GITHUB_TOKEN }}'
        firebaseServiceAccount: '${{ secrets.FIREBASE_SERVICE_ACCOUNT_CULINARYARTSNP }}'
        channelId: live
        projectId: culinaryartsnp
    - name: Deploy Firestore Rules
      env:
        FIREBASE_SERVICE_ACCOUNT_JSON: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_CULINARYARTSNP }}
      run: |
        key_file="$RUNNER_TEMP/firebase-key.json"
        trap 'rm -f "$key_file"' EXIT
        printf '%s' "$FIREBASE_SERVICE_ACCOUNT_JSON" > "$key_file"
        GOOGLE_APPLICATION_CREDENTIALS="$key_file" npx firebase-tools deploy --only firestore:rules --project culinaryartsnp
```

`firebase.json`, in full:

```json
{
    "hosting": {
        "public": "dist",
        "ignore": [
            "firebase.json",
            "**/.*",
            "**/node_modules/**"
        ],
        "rewrites": [
            {
                "source": "**",
                "destination": "/index.html"
            }
        ]
    },
    "firestore": {
        "rules": "firestore.rules"
    }
}
```

`firestore.rules`, in full:

```
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    function isSignedIn() {
      return request.auth != null;
    }

    function isAdmin() {
      return isSignedIn()
        && exists(/databases/$(database)/documents/admins/$(request.auth.uid));
    }

    match /admins/{uid} {
      allow read: if isAdmin();
      allow write: if isAdmin();
    }

    match /settings/{document=**} {
      allow read: if true;
      allow write: if isAdmin();
    }

    match /forms/{formId} {
      allow read: if isAdmin()
        || (resource.data.status == 'published');
      allow create, update, delete: if isAdmin();

      match /submissions/{submissionId} {
        allow create: if get(/databases/$(database)/documents/forms/$(formId)).data.status == 'published'
          && request.resource.data.keys().hasOnly(['answers', 'submittedAt'])
          && request.resource.data.answers is map;
        allow read: if isAdmin();
        allow update, delete: if false;
      }
    }
  }
}
```

There is no pod template, so there are no pod labels. There is no Service or ClusterIP. The public URL is the Hosting site itself.

Not set in these files: container probes, CPU/memory limits, image pull secrets, `imagePullPolicy`, and an image tag. There is no `:latest` image. `kubectl apply` of an unchanged manifest is not the update path. A new release is a new Hosting upload produced by the workflow. Pushing the same commit still reruns the workflow only if GitHub runs that workflow again; the files above do not include `workflow_dispatch`.

The workflow does not call `actions/setup-node`. It does not run tests or `npm run lint`. `npm install` is not `npm ci`. `npx firebase-tools` has no version.

## Build and publish

The checked-in release path is a push to `main`.

```bash
git push origin main
```

The job then:

1. `actions/checkout@v4`
2. `npm install`
3. `npm run build` (`tsc -b && vite build`) with the six `VITE_` secrets in the environment, writing `dist/`
4. `FirebaseExtended/action-hosting-deploy@v0` uploads that build to the `live` channel of project `culinaryartsnp`
5. Deploys `firestore.rules` with `firebase-tools`

A local `docker build` does not apply. There is no image to push, and k3s containerd is not part of this release. Do not copy a compose port into this section; there is no compose file.

No downloaded service-account JSON file is needed in the repository. The workflow reads the service account from the GitHub Actions secret and writes a temporary file under `$RUNNER_TEMP` only for the Firestore rules deploy. The previously committed local environment and service-account files must be removed from repository history. The exposed service-account key must be revoked and replaced in `FIREBASE_SERVICE_ACCOUNT_CULINARYARTSNP` before the next deployment.

## Deploy and update

1. Confirm the six `VITE_` values and `FIREBASE_SERVICE_ACCOUNT_CULINARYARTSNP` exist in GitHub Actions secrets for `Dipeshcdy/culinaryartsForm`. Replace the service-account secret with a newly created key after revoking the exposed key. The `VITE_` values are Firebase browser configuration and are embedded in the public JavaScript bundle; do not put private credentials in them.
2. Merge or push to `main`.
3. Watch the Actions run for `Deploy to Firebase Hosting on merge` until `build_and_deploy` succeeds. The Hosting step and the Firestore rules step are both in that job. A green Hosting step with a failed rules step still leaves the new static files on `live`.
4. Smoke test:

```bash
curl -sI https://culinaryartsnp.web.app/
curl -s https://culinaryartsnp.web.app/ | head
```

Expect HTTP 200, `content-type: text/html`, and `<title>ACA Form Creator</title>`. Compare `last-modified` and the `/assets/index-*.js` name with the previous release. The HTML observed on 5 October 2026 was cached for `max-age=3600`.

5. One real business call: in a browser, open `https://culinaryartsnp.web.app/{formId}` for a form whose Firestore `status` is `published`, submit it, and confirm the row in `/admin/forms/{formId}/submissions`. This repository does not record a sample form id.

DNS for the two hostnames above is Firebase’s default Hosting DNS. This repo does not name a cert resolver. TLS was present on both hosts when they were fetched.

There is no `kubectl rollout restart`. The Hosting action uploads a new bundle. If the Git files did not change, the workflow does not run on its own, because the trigger is `push` to `main` only.

## Manifest source as stored

The workflow, `firebase.json`, and `firestore.rules` quoted above are the files in git at `a7f1931`. No server copy was provided. No Kustomize `nameSuffix` rewrites these names.

## Configuration in the cluster

There is no cluster env block. Values are GitHub Actions secrets, inlined by Vite during `npm run build`.

| Variable | Value | Effect |
| --- | --- | --- |
| `VITE_FIREBASE_API_KEY` | GitHub secret `VITE_FIREBASE_API_KEY` | `firebaseConfig.apiKey` in the bundle |
| `VITE_FIREBASE_AUTH_DOMAIN` | GitHub secret `VITE_FIREBASE_AUTH_DOMAIN` | `firebaseConfig.authDomain` |
| `VITE_FIREBASE_PROJECT_ID` | GitHub secret `VITE_FIREBASE_PROJECT_ID` | `firebaseConfig.projectId` |
| `VITE_FIREBASE_STORAGE_BUCKET` | GitHub secret `VITE_FIREBASE_STORAGE_BUCKET` | `firebaseConfig.storageBucket`. The app does not call Storage. |
| `VITE_FIREBASE_MESSAGING_SENDER_ID` | GitHub secret `VITE_FIREBASE_MESSAGING_SENDER_ID` | `firebaseConfig.messagingSenderId`. The app does not call Messaging. |
| `VITE_FIREBASE_APP_ID` | GitHub secret `VITE_FIREBASE_APP_ID` | `firebaseConfig.appId` |
| `FIREBASE_SERVICE_ACCOUNT_CULINARYARTSNP` | GitHub secret of that name | Hosting deploy credentials, and a temporary `$RUNNER_TEMP/firebase-key.json` for the rules deploy |
| `GITHUB_TOKEN` | Provided by Actions as `secrets.GITHUB_TOKEN` | `repoToken` on `action-hosting-deploy` |

`firebase.json` does not set `ASPNETCORE_URLS`, `ASPNETCORE_HTTP_PORTS`, or any process environment. Nothing in this app listens on port 8080. If a `VITE_` secret is empty at build time, `initializeApp` receives that empty string. The workflow does not supply a fallback.

`settings/general.websiteUrl` is data in Firestore, not an environment variable. If it is missing, the public page uses `https://culinaryarts.com.np/`.

## Rollback

This release has no Deployment, no image tag, and no `imagePullPolicy`. `kubectl rollout history` and `kubectl rollout undo` do not apply. `:latest` does not make `rollout undo` useless here, because there is no image and no rollout.

The reproducible unit is a git commit. The workflow builds whatever is on `main` at that push and uploads it to the Hosting `live` channel. Redeploying an older commit means checking that commit out onto `main` (or otherwise pushing that tree) so the same workflow runs again. This repository does not define a Hosting rollback command.

The release checked on 5 October 2026 has `last-modified: Tue, 15 Sep 2026 10:29:41 GMT`, the date of commit `a7f1931`.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Actions never starts | The trigger is `push` to `main` only. Confirm the branch and the workflow file name `Deploy to Firebase Hosting on merge`. |
| `npm run build` fails in CI | The build step is the only step that receives the six `VITE_` secrets. `tsc -b` runs first. The workflow does not select a Node version. |
| Site is blank or Firebase says the config is invalid | The bundle was built without the `VITE_` secrets, or with empty ones. They are compile-time. Rebuild on `main` after the secrets exist. |
| Hosting updated, rules did not | The rules step is after `action-hosting-deploy`. Inspect that step’s log. It runs `npx firebase-tools deploy --only firestore:rules --project culinaryartsnp` with `GOOGLE_APPLICATION_CREDENTIALS` pointing at a temporary key file created from `FIREBASE_SERVICE_ACCOUNT_CULINARYARTSNP`. |
| `/login` or a form URL returns the home shell and then “Form not found” | Hosting is correct: every path is `index.html`. The form id is missing, unpublished, or `admin` / `login`. |
| Public submit fails | Rules allow create only when the form is `published` and the document has only `answers` and `submittedAt`. |
| Admin can sign in at Firebase and is immediately returned to `/login` | `admins/{uid}` is missing. The uid must be the document id. |
| Deleted admin can still authenticate | `/admin/users` deletes the Firestore doc only. The Auth user remains. |
| Deleted form still has answers in Firestore | `deleteForm` deletes `forms/{formId}` only. |
| New deploy not visible | HTML was served with `cache-control: max-age=3600`. Compare `last-modified` and the hashed asset name. |
| Local `firebase deploy` targets the wrong project | There is no `.firebaserc`. Pass `--project culinaryartsnp`. |

## Release checklist

1. Build by pushing `main`, which runs `npm install` and `npm run build` with the six `VITE_` secrets, then uploads that `dist/` to Hosting channel `live` for project `culinaryartsnp`. There is no image name to push.
2. Let the workflow finish, including `Deploy Firestore Rules`.
3. There is no tag-stability restart step. A new upload happens when the workflow runs. If the tree did not change, push alone will not retrigger; the trigger is a push.
4. `curl -sI https://culinaryartsnp.web.app/` returns 200 and `text/html`.
5. Open one published `https://culinaryartsnp.web.app/{formId}` and submit one response. Confirm it as an admin under submissions.

The public host does not ask for authentication. The HTML and published forms are public. `/admin` requires a Firebase Auth email/password user whose uid is an `admins/{uid}` document.
