# ACA Form Creator — project handover

| | |
| --- | --- |
| Repository | https://github.com/Dipeshcdy/culinaryartsForm.git |
| Branch | `main` (`origin/main`, up to date at time of writing) |
| Last commit | `a7f1931` (2026-09-15) `feat: add general settings and website URL button to public forms` |
| Document date | 5 October 2026 |
| Working tree | Four tracked source files are modified and not committed. One untracked service-account JSON file is on disk. See [Worktree drift](#worktree-drift). |

This is a React single-page app for the Academy of Culinary Arts. Signed-in admins create forms in Cloud Firestore, and anyone with a published form’s link fills it in the browser. Answers are stored in Firestore under that form.

It does not run its own API server, send mail, or forward submissions to another system. Production hosting is Firebase. The release steps are in [DEPLOYMENT.md](DEPLOYMENT.md).

## What this service does

Admins call it in a browser at `/login`, then use `/admin`. The public calls a published form at `/{formId}`. The app owns form definitions, the admin allow-list (`admins/{uid}`), the document `settings/general`, and each form’s `submissions` subcollection. The only outbound system is Firebase (Authentication and Cloud Firestore) through the Firebase JS SDK in the browser.

```
Browser
  → Firebase Hosting (static dist/, every path returns index.html)
    → React router
      → Firebase Auth (email/password) for /admin
      → Cloud Firestore
           forms/{formId}
           forms/{formId}/submissions/{submissionId}
           admins/{uid}
           settings/general
```

A successful public submit means `createSubmission` resolved. The page then shows “Thank you” and the form title. The Firestore document written on that call contains only `answers` and `submittedAt`.

## Technology

| | |
| --- | --- |
| Language | TypeScript, compiled by `tsc -b` then bundled by Vite |
| Runtime | Browser. Local tooling on the machine used for this document: Node `v24.13.0`, npm `11.6.2`. `package.json` has no `engines` field. There is no `.nvmrc` or `.node-version`. The GitHub workflow does not pin Node. |
| SDK pin | `firebase` resolves to `12.19.0` (`package.json` says `^12.19.0`). Lockfile: `package-lock.json`, `lockfileVersion` 3. |
| Web stack | React 19, React Router 7, Tailwind CSS 4 via `@tailwindcss/vite` |
| Data store | Cloud Firestore in Firebase project `culinaryartsnp` (project id is set in the GitHub workflow). Firebase Authentication, email/password. |
| Container | None. There is no Dockerfile and no compose file. |
| Production image name | None. Firebase Hosting serves the `dist/` directory. |
| Git host | GitHub: `https://github.com/Dipeshcdy/culinaryartsForm.git` |

## Repository layout

Source and config only. `node_modules/`, `dist/`, and `.git/` are omitted. `dist/` exists on disk and is gitignored; it is the Hosting `public` directory after `npm run build`.

```
.
├── .env.example
├── .github/workflows/firebase-hosting-merge.yml
├── .gitignore
├── .oxlintrc.json
├── README.md
├── firebase.json
├── firestore.rules
├── index.html
├── package.json
├── package-lock.json
├── public/
│   ├── aca-logo.svg          referenced by index.html and BrandLogo
│   ├── favicon.svg           present; not referenced by src/ or index.html
│   └── icons.svg             present; not referenced by src/ or index.html
├── src/
│   ├── App.tsx
│   ├── main.tsx
│   ├── index.css
│   ├── assets/hero.png       present; not referenced by src/
│   ├── components/
│   │   ├── AdminLayout.tsx
│   │   ├── BrandLogo.tsx
│   │   ├── FormFieldRenderer.tsx
│   │   ├── FormPreview.tsx
│   │   └── ProtectedRoute.tsx
│   ├── contexts/AuthContext.tsx
│   ├── lib/
│   │   ├── firebase.ts
│   │   ├── forms.ts
│   │   └── submissions.ts
│   ├── pages/
│   │   ├── AdminDashboard.tsx
│   │   ├── FormBuilderPage.tsx
│   │   ├── FormsListPage.tsx
│   │   ├── LoginPage.tsx
│   │   ├── PublicFormPage.tsx
│   │   ├── SettingsPage.tsx
│   │   ├── SubmissionsPage.tsx
│   │   └── UsersPage.tsx
│   └── types/form.ts
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
└── vite.config.ts
```

There is no `.firebaserc`. There is no test file.

## How a request is handled

Startup is client-only. `src/main.tsx` mounts `App` on `#root`. `AuthProvider` subscribes to `onAuthStateChanged`. If a user is signed in, it reads `admins/{uid}`. If that document is missing, or the read throws, it signs the user out. `BrowserRouter` then matches the routes in `src/App.tsx`.

There is no server process in this repository. Firestore security rules in `firestore.rules` are the server-side gate. The client checks are not sufficient on their own.

### `GET /` 

Redirects to `/login` (`<Navigate to="/login" replace />`).

### `GET /login`

Email and password are both required by the form. `login()` calls `signInWithEmailAndPassword`, then reads `admins/{uid}`. If the document does not exist, the user is signed out and the UI shows `You are not authorized as an admin.` Other Firebase errors are shown as `err.message`. A signed-in admin is sent to `/admin`.

### `GET /admin` and child routes

`ProtectedRoute` waits until auth loading finishes. If `user` or `isAdmin` is missing, it redirects to `/login`. Child routes:

| Path | Behavior |
| --- | --- |
| `/admin` | `listForms()`, then `listSubmissions()` for every form, to show counts. |
| `/admin/forms` | `listForms()`. Delete calls `deleteForm` after `window.confirm`. Copy link writes `{origin}/{formId}` to the clipboard. |
| `/admin/forms/new` and `/admin/forms/:formId` | Builder. Save draft writes `status: "draft"`. Publish writes `status: "published"`. Create stores `createdBy` as the admin uid. Update does not change `createdBy` or `createdAt`. |
| `/admin/forms/:formId/submissions` | `getForm` plus `listSubmissions`. |
| `/admin/users` | Lists `admins`. Add creates a Firebase Auth user on a second app named `Secondary` (`createUserWithEmailAndPassword`) and writes `admins/{uid}` with `email`, `createdAt` (ISO string), and `createdBy`. Delete removes only the `admins/{uid}` document. |
| `/admin/settings` | Reads and writes `settings/general` field `websiteUrl`. Email change calls `verifyBeforeUpdateEmail`. Password change reauthenticates with the current password, then calls `updatePassword`. The new-password inputs set `minLength={6}`. |

Builder validation before a write: title required, at least one field, every field has a label, and `select` / `radio` / `checkbox` need at least one non-empty option.

### `GET /:formId` (public)

`RESERVED_PATHS` is `admin` and `login`. Those path segments set the page to not found. React Router already matches `/login` and `/admin` on earlier routes.

`getPublishedForm` loads `forms/{formId}` and returns null unless `status === "published"`. A missing or unpublished form shows “Form not found”. The header reads `settings/general`. If that document is missing, `websiteUrl` is empty, or the read throws, the “Visit Website” link stays `https://culinaryarts.com.np/`.

Submit runs client validation, then `createSubmission(form.id, answers)`, which adds a document at `forms/{formId}/submissions` with `answers` and `submittedAt: serverTimestamp()`.

Client validation on the working tree (these checks are in the uncommitted edit; commit `a7f1931` only checks required fields):

| Condition | Message |
| --- | --- |
| Required field empty | `This field is required.` |
| `number` not all digits | `Please enter a valid number.` |
| `number` shorter than `validation.minLength` | `Must be at least {n} digits.` |
| `number` longer than `validation.maxLength` | `Must be no more than {n} digits.` |
| `email` failing `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` | `Please enter a valid email address.` |

Empty means: checkbox with options has length 0; checkbox without options is not `true`; every other type is not a non-blank string. Optional empty fields skip format checks.

A thrown submit shows `err.message`, or `Could not submit. Please try again.`

### Accepted form-document fields

`mapForm` reads only these. Any other Firestore field on the form document is dropped.

| Field | Required in the client | Default when missing |
| --- | --- | --- |
| `title` | On save, yes | `''` when mapping a read |
| `description` | No | `''` |
| `status` | On save, yes (`draft` or `published`) | `'draft'` |
| `fields` | On save, at least one | `[]` if not an array |
| `createdAt` | Written by `serverTimestamp()` on create | `null` if not a Timestamp or Date |
| `updatedAt` | Written by `serverTimestamp()` on create and update | `null` if not a Timestamp or Date |
| `createdBy` | Set to the admin uid on create | `''` |

Field objects stored inside `fields`:

| Field | Required | Default / notes |
| --- | --- | --- |
| `id` | Yes | `crypto.randomUUID()` |
| `label` | Yes | trimmed on save |
| `type` | Yes | new fields start as `text`. Working tree also allows `email`. Commit `a7f1931` does not. |
| `required` | Yes | new fields start `true` |
| `placeholder` | No | stored as `''` when blank |
| `options` | Required for `select`, `radio`, `checkbox` | `[]` for other types. Each option is `{ id, label }`. |
| `order` | Yes | rewritten to the array index on save |
| `validation.minLength` | No | written only when truthy, and only from the uncommitted builder |
| `validation.maxLength` | No | written only when truthy, and only from the uncommitted builder |
| `validation.length` | Present on the type in the working tree | never written. Save copies only `minLength` and `maxLength`. |

Submission documents. `mapSubmission` keeps `id`, `answers`, and `submittedAt`. Anything else on the document is dropped. Rules require the create payload’s keys to be exactly `answers` and `submittedAt`, and `answers` must be a map.

| Answer shape | When |
| --- | --- |
| `string` | `text`, `textbox`, `number`, `email`, `select`, `radio` |
| `string[]` | `checkbox` that has options. Values are option **labels**, not option ids. |
| `boolean` | `checkbox` with no options |

The client does not add an auth header of its own. The Firebase SDK attaches the signed-in user’s token on admin calls. Public `getPublishedForm` and `createSubmission` run with no signed-in user. The client does not retry failed writes. It does not persist drafts in `localStorage`. It does not record the respondent’s uid, email, IP, or user agent. `updateForm` does not send `createdBy`. `deleteForm` deletes the form document only. `deleteDoc` on `admins/{uid}` does not delete the Firebase Auth user.

## HTTP API

This repository does not expose an HTTP API. Routes are client-side. Firebase Hosting returns `index.html` for every path (`firebase.json` rewrite `**` → `/index.html`). Checked on 5 October 2026: `https://culinaryartsnp.web.app/`, `/login`, and `/does-not-exist` each returned HTTP 200, `content-type: text/html`, the same etag, and `<title>ACA Form Creator</title>`.

Local dev base URL: `vite.config.ts` does not set `server.port`. The README says the app is usually `http://localhost:5173`.

```bash
npm install
npm run dev
```

```bash
curl -sI http://localhost:5173/
curl -sI http://localhost:5173/login
```

Both should return the Vite HTML shell while `npm run dev` is running. Signing in and submitting a form are browser actions against Firebase, not `curl` JSON calls.

Production:

```bash
curl -sI https://culinaryartsnp.web.app/
curl -sI https://culinaryartsnp.firebaseapp.com/
```

On 5 October 2026 both returned HTTP 200 with etag `3e3430000617986a7d03014aa91f7c8a8c449a188fed29a8cc81e6b453fef174` and `last-modified: Tue, 15 Sep 2026 10:29:41 GMT`. `cache-control` on that HTML response was `max-age=3600`. `firebase.json` does not set cache headers.

Validation errors are strings rendered in the page, listed in the previous section. They are not HTTP status codes.

There is no second client in this repository, and no call site for a route that was removed.

## Configuration

Vite inlines `import.meta.env.VITE_*` at build time. In GitHub Actions those values are step environment variables on `npm run build`, taken from repository secrets. Locally, `.env.local` is present and gitignored by the `*.local` rule. `.env.example` lists the same keys with empty values. This working tree has no `.env`, `.env.development`, or `.env.production` file. There is no launchSettings or .NET launch profile.

| `import.meta.env` key | `firebaseConfig` field | Where production sets it |
| --- | --- | --- |
| `VITE_FIREBASE_API_KEY` | `apiKey` | GitHub secret `VITE_FIREBASE_API_KEY` |
| `VITE_FIREBASE_AUTH_DOMAIN` | `authDomain` | GitHub secret `VITE_FIREBASE_AUTH_DOMAIN` |
| `VITE_FIREBASE_PROJECT_ID` | `projectId` | GitHub secret `VITE_FIREBASE_PROJECT_ID` |
| `VITE_FIREBASE_STORAGE_BUCKET` | `storageBucket` | GitHub secret `VITE_FIREBASE_STORAGE_BUCKET` |
| `VITE_FIREBASE_MESSAGING_SENDER_ID` | `messagingSenderId` | GitHub secret `VITE_FIREBASE_MESSAGING_SENDER_ID` |
| `VITE_FIREBASE_APP_ID` | `appId` | GitHub secret `VITE_FIREBASE_APP_ID` |

The app calls `getAuth` and `getFirestore` only. It does not call Storage or Cloud Messaging. The rules deploy uses GitHub secret `FIREBASE_SERVICE_ACCOUNT_CULINARYARTSNP` and writes it to `firebase-key.json` inside the runner as `GOOGLE_APPLICATION_CREDENTIALS`. Do not print that secret.

`.env.local` on this machine has all six `VITE_` keys set. Values are not copied here.

`.env.example` in full:

```
VITE_FIREBASE_API_KEY=
VITE_FIREBASE_AUTH_DOMAIN=
VITE_FIREBASE_PROJECT_ID=
VITE_FIREBASE_STORAGE_BUCKET=
VITE_FIREBASE_MESSAGING_SENDER_ID=
VITE_FIREBASE_APP_ID=
```

### Worktree drift

`git status` on 5 October 2026:

- Modified, not committed: `src/components/FormFieldRenderer.tsx`, `src/pages/FormBuilderPage.tsx`, `src/pages/PublicFormPage.tsx`, `src/types/form.ts`. The diff adds field type `email`, number `minLength` / `maxLength` in the builder, and public validation for numbers and email. Commit `a7f1931` does not contain that behavior. The Hosting release observed the same day has `last-modified` 15 September 2026, the date of `a7f1931`.
- Untracked: `culinaryartsnp-firebase-adminsdk-fbsvc-e9ba0d600b.json`. This is a service-account key. It is not referenced by the workflow. The workflow uses the GitHub secret named above. Do not commit this file and do not paste its contents.
- No tracked file is deleted.

To print a committed file:

```bash
git show HEAD:src/types/form.ts
git show HEAD:src/pages/PublicFormPage.tsx
git show HEAD:src/pages/FormBuilderPage.tsx
git show HEAD:src/components/FormFieldRenderer.tsx
```

To drop the uncommitted edits and match `a7f1931`:

```bash
git restore src/types/form.ts src/pages/PublicFormPage.tsx src/pages/FormBuilderPage.tsx src/components/FormFieldRenderer.tsx
```

## Runbook

Prerequisites: Node and npm, a Firebase project with Email/Password auth, a Firestore database, and an `admins/{uid}` document whose id is that user’s Auth uid and which has an `email` field. Copy env keys from `.env.example` into `.env.local` and fill them from Firebase console → Project settings. Do not commit `.env.local`.

On the machine:

```bash
npm install
npm run dev
```

Open the URL Vite prints (the README says usually `http://localhost:5173`). Sign in at `/login`.

`npm run build` runs `tsc -b && vite build` and writes `dist/`. `npm run preview` serves that build. `npm run lint` runs `oxlint`.

There is no Dockerfile and no compose file, so there is no local container run. Production release is [DEPLOYMENT.md](DEPLOYMENT.md).

There is no image build. The production artifact is the Vite bundle in `dist/`. Vite 8.3.0 uses Rolldown (`rolldown@1.2.8` in the installed tree). There is no .NET RID, no OS package install, and no container entrypoint. `index.html` loads `/src/main.tsx` in dev and the hashed `/assets/index-*.js` file after build. The live HTML on 5 October 2026 referenced `/assets/index-CGAFtmG8.js` and `/assets/index-CA2dvAis.css`.

## Dependency files

Resolved on 5 October 2026 with `npm ls --depth=1` against `package-lock.json` (`lockfileVersion` 3, 197 package entries, 106,474 bytes). The lockfile body is the file in the repository. Manifest ranges are in parentheses when they differ from the installed version.

### Direct

| Package | Resolved | Manifest |
| --- | --- | --- |
| `firebase` | 12.19.0 | `^12.19.0` |
| `react` | 19.3.0 | `^19.2.8` |
| `react-dom` | 19.3.0 | `^19.2.8` |
| `react-router-dom` | 7.18.3 | `^7.18.3` |
| `@tailwindcss/vite` | 4.3.3 | `^4.3.3` |
| `@types/node` | 24.13.4 | `^24.13.3` |
| `@types/react` | 19.3.0 | `^19.2.18` |
| `@types/react-dom` | 19.3.0 | `^19.2.7` |
| `@vitejs/plugin-react` | 6.1.1 | `^6.1.1` |
| `oxlint` | 1.82.0 | `^1.81.0` |
| `tailwindcss` | 4.3.3 | `^4.3.3` |
| `typescript` | 6.0.3 | `~6.0.2` |
| `vite` | 8.3.0 | `^8.3.0` |

### Transitive, depth 1, installed

| Package | Resolved | Under |
| --- | --- | --- |
| `@tailwindcss/node` | 4.3.3 | `@tailwindcss/vite` |
| `@tailwindcss/oxide` | 4.3.3 | `@tailwindcss/vite` |
| `undici-types` | 7.18.2 | `@types/node` |
| `csstype` | 3.2.3 | `@types/react` |
| `@rolldown/pluginutils` | 1.0.1 | `@vitejs/plugin-react` |
| `@firebase/ai` | 2.16.0 | `firebase` |
| `@firebase/analytics` | 0.10.25 | `firebase` |
| `@firebase/analytics-compat` | 0.2.31 | `firebase` |
| `@firebase/app` | 0.16.2 | `firebase` |
| `@firebase/app-check` | 0.13.1 | `firebase` |
| `@firebase/app-check-compat` | 0.4.7 | `firebase` |
| `@firebase/app-compat` | 0.5.18 | `firebase` |
| `@firebase/app-types` | 0.9.6 | `firebase` |
| `@firebase/auth` | 1.13.6 | `firebase` |
| `@firebase/auth-compat` | 0.6.11 | `firebase` |
| `@firebase/data-connect` | 0.7.4 | `firebase` |
| `@firebase/database` | 1.1.5 | `firebase` |
| `@firebase/database-compat` | 2.1.7 | `firebase` |
| `@firebase/firestore` | 4.17.2 | `firebase` |
| `@firebase/firestore-compat` | 0.4.14 | `firebase` |
| `@firebase/functions` | 0.14.0 | `firebase` |
| `@firebase/functions-compat` | 0.5.0 | `firebase` |
| `@firebase/installations` | 0.6.24 | `firebase` |
| `@firebase/installations-compat` | 0.2.24 | `firebase` |
| `@firebase/messaging` | 0.13.3 | `firebase` |
| `@firebase/messaging-compat` | 0.2.30 | `firebase` |
| `@firebase/performance` | 0.7.14 | `firebase` |
| `@firebase/performance-compat` | 0.2.27 | `firebase` |
| `@firebase/remote-config` | 0.9.2 | `firebase` |
| `@firebase/remote-config-compat` | 0.2.29 | `firebase` |
| `@firebase/storage` | 0.14.5 | `firebase` |
| `@firebase/storage-compat` | 0.4.5 | `firebase` |
| `@firebase/util` | 1.15.3 | `firebase` |
| `@oxlint/binding-darwin-arm64` | 1.82.0 | `oxlint` (this machine is darwin arm64; other `@oxlint/binding-*` packages were unmet optional) |
| `scheduler` | 0.28.0 | `react-dom` |
| `react-router` | 7.18.3 | `react-router-dom` |
| `jiti` | 2.7.0 | `vite` |
| `lightningcss` | 1.33.0 | `vite` |
| `picomatch` | 4.0.7 | `vite` |
| `postcss` | 8.5.28 | `vite` |
| `rolldown` | 1.2.8 | `vite` |
| `tinyglobby` | 0.2.17 | `vite` |

`npm ls` also reported unmet optional dependencies (`@rolldown/plugin-babel`, `babel-plugin-react-compiler`, `oxc-transform-react`, `esbuild`, `fsevents`, `less`, `sass`, `sass-embedded`, `stylus`, `sugarss`, `terser`, `tsx`, `yaml`, `vite-plus`, `oxlint-tsgolint`, `@vitejs/devtools`, and the non-darwin-arm64 oxlint bindings). They are not installed.

`firebase-tools` is not a dependency. The workflow runs `npx firebase-tools` with no version pin.

There is no container base image and no OS library install in this repository.

### `package.json`

```json
{
  "name": "form-creator",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "oxlint",
    "preview": "vite preview"
  },
  "dependencies": {
    "firebase": "^12.19.0",
    "react": "^19.2.8",
    "react-dom": "^19.2.8",
    "react-router-dom": "^7.18.3"
  },
  "devDependencies": {
    "@tailwindcss/vite": "^4.3.3",
    "@types/node": "^24.13.3",
    "@types/react": "^19.2.18",
    "@types/react-dom": "^19.2.7",
    "@vitejs/plugin-react": "^6.1.1",
    "oxlint": "^1.81.0",
    "tailwindcss": "^4.3.3",
    "typescript": "~6.0.2",
    "vite": "^8.3.0"
  }
}
```

### `tsconfig.json`

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

### `tsconfig.app.json`

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo",
    "target": "es2023",
    "lib": ["ES2023", "DOM"],
    "module": "esnext",
    "types": ["vite/client"],
    "allowArbitraryExtensions": true,
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"]
}
```

### `tsconfig.node.json`

```json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.node.tsbuildinfo",
    "target": "es2023",
    "lib": ["ES2023"],
    "types": ["node"],
    "skipLibCheck": true,
    "module": "nodenext",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["vite.config.ts"]
}
```

### `vite.config.ts`

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
})
```

### `.oxlintrc.json`

```json
{
  "$schema": "./node_modules/oxlint/configuration_schema.json",
  "plugins": ["react", "typescript", "oxc"],
  "rules": {
    "react/rules-of-hooks": "error",
    "react/only-export-components": ["warn", { "allowConstantExport": true }]
  }
}
```

`firebase.json`, `firestore.rules`, and the GitHub workflow are quoted in full in [DEPLOYMENT.md](DEPLOYMENT.md).

## Code map

| File | Responsibility |
| --- | --- |
| `src/main.tsx` | React root |
| `src/App.tsx` | Routes |
| `src/lib/firebase.ts` | `initializeApp`, `auth`, `db` |
| `src/lib/forms.ts` | CRUD for `forms` |
| `src/lib/submissions.ts` | Create and list `forms/{id}/submissions` |
| `src/types/form.ts` | Field, form, and submission types |
| `src/contexts/AuthContext.tsx` | Session and `admins/{uid}` check |
| `src/components/ProtectedRoute.tsx` | Redirect anonymous or non-admin users |
| `src/components/AdminLayout.tsx` | Admin nav and sign-out |
| `src/components/FormFieldRenderer.tsx` | Inputs for each field type |
| `src/components/FormPreview.tsx` | Disabled preview in the builder. It does not submit. |
| `src/components/BrandLogo.tsx` | `/aca-logo.svg` |
| `src/pages/LoginPage.tsx` | Email/password sign-in |
| `src/pages/AdminDashboard.tsx` | Counts and the five most recently updated forms |
| `src/pages/FormsListPage.tsx` | List, copy link, delete |
| `src/pages/FormBuilderPage.tsx` | Create and update forms |
| `src/pages/PublicFormPage.tsx` | Published form and submit |
| `src/pages/SubmissionsPage.tsx` | Answer table |
| `src/pages/UsersPage.tsx` | Create Auth users and `admins` docs; delete `admins` docs |
| `src/pages/SettingsPage.tsx` | `websiteUrl`, email verification, password change |
| `firestore.rules` | Who may read and write each collection |
| `.github/workflows/firebase-hosting-merge.yml` | Build, Hosting live channel, Firestore rules |

## Operational limits and handover risks

1. The live Hosting release checked on 5 October 2026 was last modified 15 September 2026. Uncommitted email and number-length validation are not in commit `a7f1931`.
2. `culinaryartsnp-firebase-adminsdk-fbsvc-e9ba0d600b.json` is an untracked private key on disk. The workflow does not read it. Do not commit it.
3. The six `VITE_` values are compiled into the public JavaScript bundle. They are client config. The service-account secret is not client config.
4. Hosting does not require a login. `/admin` is enforced in the browser and by Firestore rules. Published forms and `settings/{document}` are world-readable. Anyone can create a submission on a published form if the payload is only `answers` and `submittedAt`.
5. Rules set `allow update, delete: if false` on submissions. The client has no edit or delete for answers.
6. `deleteForm` deletes the form document only. Firestore does not remove the `submissions` subcollection with that call.
7. Deleting a user on `/admin/users` deletes `admins/{uid}` only. The Firebase Auth user remains and can still authenticate; `AuthContext` then signs them out because the admin document is gone.
8. Creating a user runs in the admin’s browser via a second Firebase app. The new password is sent to Firebase Auth from that browser.
9. `admins` rules allow any admin to read and write every admin document.
10. The dashboard loads every submission of every form in the browser to count them. There is no pagination.
11. Select, radio, and checkbox answers store option labels. Renaming an option does not rewrite old answers.
12. The workflow does not pin Node and does not pin `firebase-tools`. `npm install` in CI is not `npm ci`.
13. There is no test suite, no lint step in CI, and no App Check in the client.
14. HTML on the live hosts was served with `cache-control: max-age=3600`. A new `index.html` can stay cached for up to an hour. Built JS and CSS file names are hashed.
15. There is no `.firebaserc`. A manual CLI deploy has to pass `--project culinaryartsnp`, as the workflow does for rules.

## Extending it safely

1. Add a field type on `FieldType` and `FIELD_TYPE_LABELS` in `src/types/form.ts`.
2. Render it in `FormFieldRenderer` and accept it in the builder.
3. Validate it in `PublicFormPage` before `createSubmission`.
4. Keep the Firestore write shape inside `firestore.rules` (`answers` + `submittedAt` only, unless you change the rules in the same change).
5. Deploy rules with the client. The workflow deploys both on push to `main`. See [DEPLOYMENT.md](DEPLOYMENT.md).

## History worth knowing

| Date | Commit | What changed |
| --- | --- | --- |
| 2026-09-12 | `c383609` | First commit |
| 2026-09-12 | `3ace47b` | Workflow injects the `VITE_` environment variables at build |
| 2026-09-12 | `177d5bf` | Add `firebase.json` |
| 2026-09-15 | `56acc4d` | Number field type |
| 2026-09-15 | `0157e54` | Users page, settings page, Firestore rules deploy in the workflow |
| 2026-09-15 | `94c7afc` | Password reauthentication, confirmation, email verification; service-account credentials used by the workflow |
| 2026-09-15 | `a7f1931` | `settings/general` `websiteUrl` and the public “Visit Website” button |

## Checklist for the next owner

1. Clone `https://github.com/Dipeshcdy/culinaryartsForm.git` and check out `main` at `a7f1931`, or keep the local edits on purpose.
2. Install dependencies with npm. This document’s resolved tree used Node `v24.13.0`. The repo does not pin a Node version.
3. If `.env.local` is missing, copy `.env.example` to `.env.local` and fill the six keys. If you need the committed source of a modified file, use `git show HEAD:<path>`.
4. Run `npm run dev` and open `/login`.
5. Sign in as a user that has an `admins/{uid}` document and open one published form at `/{formId}`.
6. Release with [DEPLOYMENT.md](DEPLOYMENT.md).
