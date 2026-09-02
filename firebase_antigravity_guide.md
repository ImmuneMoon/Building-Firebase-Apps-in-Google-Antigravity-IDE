# Building Firebase Apps in Google Antigravity IDE

A working guide to developing Firebase apps locally with the Google Antigravity IDE — whether you're migrating an existing Firebase Studio project or starting from scratch. Covers Cloud Firestore, the Local Emulator Suite, security rules unit tests, and automated deployments.

> **If you're coming from Firebase Studio:** new workspace creation was disabled on June 22, 2026 and the editor sunsets on March 22, 2027. Apps already deployed to Firebase keep running, but the editor goes away — so everything below is about making your local setup self-sufficient. Migration steps are in Section 3; new projects start at Section 4.

## Contents

1. [Local Runtime Environment Setup](#1-local-runtime-environment-setup)
2. [Install Antigravity and Connect It to Firebase](#2-install-antigravity-and-connect-it-to-firebase)
3. [Path 1 — Migrate an Existing Firebase Studio Project](#3-path-1--migrate-an-existing-firebase-studio-project)
4. [Path 2 — Start a New Project in Antigravity](#4-path-2--start-a-new-project-in-antigravity)
5. [Working With the Agent: Prompts That Do Real Work](#5-working-with-the-agent-prompts-that-do-real-work)
6. [Choose Your Deployment Path](#6-choose-your-deployment-path)
7. [Production Firestore Security Rules](#7-production-firestore-security-rules)
8. [Local Emulator Seeding](#8-local-emulator-seeding)
9. [Firestore Rules Unit Tests](#9-firestore-rules-unit-tests)
10. [Continuous Deployment](#10-continuous-deployment)
11. [Next Steps](#11-next-steps)

---

## 1. Local Runtime Environment Setup

Antigravity and the Firebase CLI both require **Node.js 20 or higher**, and the migration workflow requires **Firebase CLI 15.10.0 or higher**.

### Windows

1. Download the Node.js LTS installer from [nodejs.org](https://nodejs.org).
2. Run the installer and accept the defaults. Leaving **"Automatically install the necessary tools"** checked pulls in Chocolatey and native build dependencies, which some npm packages need.
3. Open PowerShell and verify:

   ```powershell
   node -v
   npm -v
   ```

#### If `node -v` works but `npm -v` doesn't

This happens often enough after the MSI installer that it's worth knowing the two causes. Read the error message — it tells you which one you have.

**Error mentions `npm.ps1` or "running scripts is disabled"** — PowerShell's execution policy is blocking the npm launcher script. Node and npm are both installed and on PATH; PowerShell just refuses to run `.ps1` files. Fix it for your user account:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Then rerun `npm -v`. If you'd rather not change the policy, `npm.cmd -v` works immediately and always.

**Error says "npm is not recognized"** — the npm folder isn't on PATH. First try a fresh terminal: the installer updates PATH, but windows opened before the install don't see it. If a new window still fails, add the paths for the current session to confirm that's the problem:

```powershell
$env:Path += ";$env:ProgramFiles\nodejs;$env:AppData\npm"
npm -v
```

If that works, make it permanent:

1. Press `Win + R`, type `sysdm.cpl`, press Enter.
2. **Advanced** tab → **Environment Variables…**
3. Under **User variables**, select **Path** → **Edit…** → **New** → add `%AppData%\npm`.
4. Under **System variables**, select **Path** → **Edit…** and confirm `C:\Program Files\nodejs\` is present. Add it if not.
5. **OK** through every dialog, then open a **new** PowerShell window.

Verify with `where.exe npm` — it should print a path under `C:\Program Files\nodejs\`.

**Nothing works** — run the Node.js MSI again and choose **Repair**, then reboot. As a last resort, [nvm-windows](https://github.com/coreybutler/nvm-windows) manages Node and npm together and sidesteps PATH issues entirely, but uninstall the MSI version first or the two will fight.

### macOS

1. Install Homebrew if you don't have it:

   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

2. Install and link Node.js 20:

   ```bash
   brew install node@20
   brew link node@20
   ```

3. Verify:

   ```bash
   node -v
   npm -v
   ```

### Linux (Ubuntu / Debian)

1. Register the NodeSource repository:

   ```bash
   sudo apt-get update && sudo apt-get install -y curl gnupg
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
     | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
   echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" \
     | sudo tee /etc/apt/sources.list.d/nodesource.list
   ```

2. Install Node.js and build tools:

   ```bash
   sudo apt-get update
   sudo apt-get install -y nodejs build-essential
   ```

3. Verify:

   ```bash
   node -v
   npm -v
   ```

### Firebase CLI

Install globally, then confirm the version meets the 15.10.0 minimum:

```bash
npm install -g firebase-tools
firebase --version
```

If you hit permission errors on macOS or Linux, prefix with `sudo` (or fix your npm prefix — the cleaner fix long-term).

Authenticate:

```bash
firebase login
```

### Java (for the emulators)

The Firestore emulator runs on the JVM. Install a current JDK if `java -version` fails — the emulator will tell you which minimum version it needs when it starts. On macOS, `brew install openjdk` works; on Ubuntu, `sudo apt-get install -y default-jdk`.

---

## 2. Install Antigravity and Connect It to Firebase

Do this once, regardless of whether you're migrating or starting fresh.

### Install the IDE

1. Download the installer for your OS from [antigravity.google/download](https://antigravity.google/download).
2. Run it and open Antigravity. Sign in with your Google account when prompted.
3. During onboarding you'll be offered **Build with Google** integrations — tick **Firebase** if you see it. If you skip it, you can enable it later (next step).

### Enable the Firebase Bundle (agent skills)

**Settings → Customizations → Build with Google Plugins** → enable **Firebase**.

This installs a set of agent *skills*: version-aware instructions that teach the agent how to initialize Firebase services, write security rules, and deploy to App Hosting or Hosting using current best practices. Without this, the agent falls back on whatever it remembers from training, which can be stale.

### Install the Firebase MCP server (agent tools)

In the **Agent** pane, open the three-dot (`more_horiz`) menu → **MCP Servers** → find **Firebase** → **Install**.

Skills tell the agent *how*; the MCP server gives it the *hands*. With it installed, the agent can list your Firebase projects, read your web app config, query and write Firestore data, manage Auth users, and run deploys — all through the Firebase CLI credentials you set up with `firebase login` in Section 1.

You can confirm it's working by asking the agent: *"List my Firebase projects."*

### How the agent pane works

A few things worth knowing before you start prompting:

- **Model picker** — bottom of the Agent pane. Gemini Flash is cheaper and faster; use it for mechanical work like file conversion. Use the larger models for design decisions and debugging.
- **Plans and walkthroughs** — for non-trivial tasks the agent produces an implementation plan you review before it touches files. Read them. This is where you catch it doing something you didn't intend.
- **Permissions** — the agent asks before running commands like `firebase deploy`. Say no if a plan looks wrong; it will revise.
- **`@workflows <name>`** — runs a saved workflow file. `@fbs-to-agy-export` (used in Section 3) is one of these.

---

## 3. Path 1 — Migrate an Existing Firebase Studio Project

Skip to Section 4 if you're starting a new codebase.

### Export from Firebase Studio

1. Open your workspace in Firebase Studio and click the **Move now** button at the top.
2. Follow the export method that appears:
   - If you see a **Zip and Download** button, click it.
   - Otherwise open the command palette (`Cmd+Shift+P` on Mac, `Ctrl+Shift+P` elsewhere) and run **Firebase Studio: Zip & Download**.
3. Extract the archive to a local folder.

> If no download window appears, check your browser's address bar for a blocked pop-up.

### Convert the project — Option A, agent-driven

1. Open the extracted folder in Antigravity (**File → Open Folder…**).
2. In the Agent pane, select the **Gemini Flash** model.
3. Enter:

   ```text
   @fbs-to-agy-export
   ```

   The agent reads the Studio workspace config (including the Nix environment definition), rewrites it into a standard local project layout, and installs anything missing. It will ask for help along the way — follow its guidance. If it errors, tell it to try again.

### Convert the project — Option B, manual (no AI tokens)

```bash
npx firebase-tools@latest studio:export <path-to-folder-or-zip>
```

Currently optimized for Next.js, Flutter, and Angular workspaces. Other stacks may need hand-fixing afterward.

### Reconnect to your existing Firebase project

Your production data and deployed app are untouched by the export — but the local checkout needs to know which project it belongs to.

1. Check whether `.firebaserc` exists in the project root and names the right project. If not:

   ```bash
   firebase use --add
   ```

   and pick your project from the list.

2. Ask the agent to verify the wiring:

   > *"Link this app to my existing Firebase project and verify the Firestore configuration. Don't modify any data."*

   It will check `.firebaserc`, `firebase.json`, and your web SDK config against what the MCP server sees in the cloud, and flag mismatches.

### Preview locally

Open **Run and Debug** in the left sidebar and click play, or run `npm run dev` in the terminal. Studio-generated Next.js apps usually serve on `http://localhost:9002`. Confirm the app loads and can read from your live Firestore before going further.

### Republish

Once the local preview works:

> *"Publish my app"*

When the agent asks to run `firebase deploy`, choose **Yes**. It detects whether you previously deployed to App Hosting or Hosting and updates your existing live URL. If this is the first App Hosting deploy from this machine, it walks you through creating the backend.

Continue at Section 5.

---

## 4. Path 2 — Start a New Project in Antigravity

### Create the Firebase project

Do this in the [Firebase console](https://console.firebase.google.com) first: **Add project**, name it, decide on Analytics. If you plan to use App Hosting (the default for Next.js), upgrade to the **Blaze** plan now — App Hosting requires it.

### Scaffold the app

Pick a framework. These are the three the Firebase tooling supports best:

```bash
# Next.js (App Router, TypeScript) — best fit for App Hosting
npx create-next-app@latest my-app --typescript --app --eslint
# Vite + React — best fit for static Hosting
npm create vite@latest my-app -- --template react-ts
# Angular
npx @angular/cli@latest new my-app
```

Open the new folder in Antigravity.

### Initialize Firebase — Option A, agent-driven

With the Firebase Bundle and MCP server installed (Section 2), prompt:

> *"Initialize Firebase in this project using my `<project-id>` project. Set up Firestore, Authentication with Google sign-in, and the local emulators."*

The agent will register a web app in your Firebase project (or reuse one), fetch its config, write the SDK initialization code, create `firebase.json` / `.firebaserc` / `firestore.rules`, and present a plan before executing. Review it, then approve.

### Initialize Firebase — Option B, manual

```bash
firebase init
```

Select the features you need. For a typical app: **Firestore**, **Emulators**, and either **App Hosting** or **Hosting**. Accept the defaults for rules and index file names.

Then register a web app and grab its config from **Project settings → Your apps → Add app (Web)** in the console, and initialize the SDK:

```bash
npm install firebase
```

```javascript
// src/lib/firebase.ts (or .js)
import { initializeApp } from 'firebase/app';
import { getFirestore, connectFirestoreEmulator } from 'firebase/firestore';
import { getAuth, connectAuthEmulator } from 'firebase/auth';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID
};

const app = initializeApp(firebaseConfig);
export const db = getFirestore(app);
export const auth = getAuth(app);

// Talk to the local emulators in development
if (process.env.NODE_ENV === 'development') {
  connectFirestoreEmulator(db, '127.0.0.1', 8080);
  connectAuthEmulator(auth, 'http://127.0.0.1:9099');
}
```

(For Vite, use `import.meta.env.VITE_*` instead of `process.env.NEXT_PUBLIC_*`.) Put the values in `.env.local` and keep that file out of git. The web API key is not a secret — it's shipped to every browser — but keeping config out of source is still good hygiene.

### Build features with the agent

From here, development is prompt-driven. Some starting points:

> *"Create a `users` collection with a signup form that stores email and username in Firestore. Validate the inputs client-side and write matching security rules."*

> *"Add Google sign-in with a header showing the signed-in user's avatar and a sign-out button."*

The agent writes code, rules, and tests together when you ask it to. Always ask for the rules alongside the feature — it's the step most often skipped.

Continue at Section 5.

---

## 5. Working With the Agent: Prompts That Do Real Work

These prompts apply to both paths. Phrase them your own way; the agent understands intent, not incantations.

**Audit security rules**

> *"Scan `firestore.rules`. Flag any rule that allows unauthenticated writes, any collection with no rule at all, and any rule that lets a user modify a document they don't own. Propose fixes and show me a diff."*

The agent locates the rules file from `firebase.json`, walks each `match` block, and prints current-vs-proposed rules in the chat. The full rules blueprint in Section 7 is a good baseline to hand it.

**Verify rules against the emulator**

> *"Start the Firestore emulator and write a rules unit test that proves an anonymous user cannot read `/users/{id}`."*

Or do it yourself with `firebase emulators:start --only firestore` and use the Emulator UI at `http://localhost:4000`, which has a rules playground. Section 9 has a full test harness.

**Pre-deploy check**

> *"Prepare this app for deployment. Run the production build, check that `firestore.rules` is registered in `firebase.json`, confirm no secrets are in source, and list any environment variables I still need to set."*

**Deploy**

> *"Publish my app"* — full deploy (build + hosting + rules).

> *"Deploy only my Firestore rules"* — or run `firebase deploy --only firestore:rules` yourself. This replaces the live rules immediately; there is no staging step for rules.

**When the agent gets it wrong**

Tell it what's wrong and ask it to revise the plan rather than fixing files by hand and then continuing. It loses track of manual edits it didn't make.

---

## 6. Choose Your Deployment Path

This is the decision that shapes everything downstream, so make it deliberately.

| | **App Hosting** | **Firebase Hosting (static)** |
| --- | --- | --- |
| What it is | Server-rendered hosting on Cloud Run with built-in CDN and GitHub rollouts | CDN for pre-built static files |
| Fits | Next.js / Angular apps with server components, API routes, server actions, Genkit flows, or middleware | Pure client-side SPAs: Vite/React, Angular without SSR, Next.js with `output: 'export'` |
| Firebase Studio default | ✅ Apps built with the App Prototyping agent deploy here | ❌ |
| Billing | Requires the Blaze (pay-as-you-go) plan | Free tier available |
| CI/CD | Built in — push to the live branch triggers a rollout | You wire it up yourself (see Section 10) |

**Rule of thumb:** if your app came from Studio's App Prototyping agent, it's a Next.js app already deployed to App Hosting. Stay there unless you have a specific reason to go static. Forcing `output: 'export'` onto an app that uses any server-side feature will break it at build time.

### Path A — App Hosting

App Hosting is configured with an `apphosting.yaml` in your project root, not `firebase.json`. If the export didn't leave one behind, scaffold it:

```bash
firebase init apphosting
```

A minimal `apphosting.yaml`:

```yaml
runConfig:
  minInstances: 0
  maxInstances: 4

env:
  - variable: NODE_ENV
    value: production
```

Secrets should reference Cloud Secret Manager rather than being pasted in:

```yaml
env:
  - variable: GEMINI_API_KEY
    secret: gemini-api-key
```

Create the secret with `firebase apphosting:secrets:set gemini-api-key`.

Then either let the agent handle it (`Publish my app`) or connect the backend to GitHub yourself — see Section 10, Path A.

### Path B — Firebase Hosting (static)

Your `firebase.json` maps the build output directory to Hosting. Pick the block that matches your framework.

**Vite / React** — build output is `dist`:

```json
{
  "hosting": {
    "public": "dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      { "source": "**", "destination": "/index.html" }
    ]
  }
}
```

**Next.js static export** — set `output: 'export'` in `next.config.js`; build output is `out`. Don't add a catch-all rewrite: Hosting serves `out/404.html` automatically for missing paths, and rewriting `**` to it would send every route to the 404 page.

```json
{
  "hosting": {
    "public": "out",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "cleanUrls": true,
    "trailingSlash": false
  }
}
```

**Angular** — output is `dist/<project-name>/browser`:

```json
{
  "hosting": {
    "public": "dist/your-project-name/browser",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      { "source": "**", "destination": "/index.html" }
    ]
  }
}
```

---

## 7. Production Firestore Security Rules

Create `firestore.rules` in the project root. This schema isolates per-user documents, validates input shape at the database boundary, and prevents ownership transfer on posts.

```javascript
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {

    // ---- Helpers ----
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    function isValidEmail(email) {
      return email.matches("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
    }

    function isValidUsername(username) {
      return username.size() >= 3 && username.size() <= 20;
    }

    // ---- Users ----
    match /users/{userId} {
      // NOTE: any signed-in user can read any profile, including its email.
      // Tighten to isOwner(userId) if profiles should be private.
      allow read: if isAuthenticated();

      allow create: if isOwner(userId)
                    && isValidEmail(request.resource.data.email)
                    && isValidUsername(request.resource.data.username);

      // Email is frozen after creation
      allow update: if isOwner(userId)
                    && request.resource.data.email == resource.data.email;

      // No client-side deletes; handle account deletion server-side
      allow delete: if false;
    }

    // ---- Posts ----
    match /posts/{postId} {
      allow read: if true;

      allow create: if isAuthenticated()
                    && request.resource.data.authorId == request.auth.uid
                    && request.resource.data.title is string
                    && request.resource.data.title.size() > 0;

      allow update: if isAuthenticated()
                    && resource.data.authorId == request.auth.uid
                    && request.resource.data.authorId == resource.data.authorId;

      allow delete: if isAuthenticated() && resource.data.authorId == request.auth.uid;
    }
  }
}
```

Register the rules file in `firebase.json` so both the emulator and `firebase deploy` pick it up. This block coexists with any `hosting` or `emulators` blocks you already have:

```json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  }
}
```

If you don't have a `firestore.indexes.json` yet, `firebase init firestore` creates both files.

---

## 8. Local Emulator Seeding

The Local Emulator Suite lets you develop and test against Firestore without touching production data or incurring reads.

### Configure the emulators

Add an `emulators` block to `firebase.json` (merge with existing keys; don't overwrite the file):

```json
{
  "emulators": {
    "firestore": { "port": 8080 },
    "auth": { "port": 9099 },
    "ui": { "enabled": true, "port": 4000 }
  }
}
```

Or run `firebase init emulators` and pick Firestore, Authentication, and the UI.

### Write the seed script

Install the Admin SDK as a dev dependency — it's only used for seeding, never in the browser bundle:

```bash
npm install --save-dev firebase-admin
```

Create `scripts/seed.js`:

```javascript
const { initializeApp } = require('firebase-admin/app');
const { getFirestore } = require('firebase-admin/firestore');

// Point the Admin SDK at the local emulator instead of production
process.env.FIRESTORE_EMULATOR_HOST = '127.0.0.1:8080';

initializeApp({ projectId: 'demo-antigravity-migration' });
const db = getFirestore();

async function runSeeding() {
  console.log('Seeding local Firestore...');

  const sampleUsers = [
    {
      id: 'user_alice',
      data: {
        username: 'alice_dev',
        email: 'alice@example.com',
        createdAt: new Date().toISOString()
      }
    },
    {
      id: 'user_bob',
      data: {
        username: 'bob_tester',
        email: 'bob@example.com',
        createdAt: new Date().toISOString()
      }
    }
  ];

  for (const user of sampleUsers) {
    await db.collection('users').doc(user.id).set(user.data);
  }

  console.log('Done.');
}

runSeeding().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

> **Why `demo-` as the project ID prefix?** The Firebase CLI treats any project ID starting with `demo-` as emulator-only and refuses to make production calls with it. That's a safety net worth keeping.

### Capture a reusable snapshot

1. Start the emulators:

   ```bash
   firebase emulators:start --project demo-antigravity-migration
   ```

2. In a second terminal, run the seed script:

   ```bash
   node scripts/seed.js
   ```

3. In a third terminal, export the emulator state to disk:

   ```bash
   firebase emulators:export ./firebase-seed-data --project demo-antigravity-migration
   ```

4. From now on, start with the snapshot loaded — and persist any changes you make during the session on exit:

   ```bash
   firebase emulators:start \
     --project demo-antigravity-migration \
     --import=./firebase-seed-data \
     --export-on-exit
   ```

Add `firebase-seed-data/` to `.gitignore` unless you want the fixture committed.

---

## 9. Firestore Rules Unit Tests

The `@firebase/rules-unit-testing` library runs your real `firestore.rules` against the emulator, so you can assert what's allowed and denied before anything ships.

### Install

```bash
npm install --save-dev jest @firebase/rules-unit-testing
```

Add a script to `package.json`. `emulators:exec` boots the Firestore emulator, runs the command, then shuts it down:

```json
{
  "scripts": {
    "test:rules": "firebase emulators:exec --only firestore --project demo-antigravity-migration 'jest --runInBand'"
  }
}
```

### Write the tests

Create `firestore.test.js` in the project root:

```javascript
const {
  initializeTestEnvironment,
  assertFails,
  assertSucceeds
} = require('@firebase/rules-unit-testing');
const fs = require('fs');

let testEnv;

beforeAll(async () => {
  testEnv = await initializeTestEnvironment({
    projectId: 'demo-antigravity-migration',
    firestore: {
      rules: fs.readFileSync('firestore.rules', 'utf8')
      // host/port are picked up from FIRESTORE_EMULATOR_HOST,
      // which emulators:exec sets automatically
    }
  });
});

beforeEach(async () => {
  await testEnv.clearFirestore();
});

afterAll(async () => {
  await testEnv.cleanup();
});

describe('users collection', () => {
  test('denies anonymous reads', async () => {
    const db = testEnv.unauthenticatedContext().firestore();
    await assertFails(db.collection('users').doc('user_alice').get());
  });

  test('allows authenticated reads', async () => {
    const db = testEnv.authenticatedContext('user_bob').firestore();
    await assertSucceeds(db.collection('users').doc('user_alice').get());
  });

  test('denies writing to another user\'s document', async () => {
    const db = testEnv.authenticatedContext('user_attacker').firestore();
    await assertFails(
      db.collection('users').doc('user_alice').set({
        username: 'hacked_alice',
        email: 'attacker@evil.com'
      })
    );
  });

  test('denies creation with a malformed email', async () => {
    const db = testEnv.authenticatedContext('user_alice').firestore();
    await assertFails(
      db.collection('users').doc('user_alice').set({
        username: 'alice_dev',
        email: 'not-an-email'
      })
    );
  });

  test('allows creation when the schema is valid', async () => {
    const db = testEnv.authenticatedContext('user_alice').firestore();
    await assertSucceeds(
      db.collection('users').doc('user_alice').set({
        username: 'alice_dev',
        email: 'alice@example.com'
      })
    );
  });

  test('denies changing email after creation', async () => {
    const db = testEnv.authenticatedContext('user_alice').firestore();
    const doc = db.collection('users').doc('user_alice');
    await assertSucceeds(doc.set({ username: 'alice_dev', email: 'alice@example.com' }));
    await assertFails(doc.update({ email: 'new@example.com' }));
  });
});
```

Run:

```bash
npm run test:rules
```

---

## 10. Continuous Deployment

### Path A — App Hosting (built in)

App Hosting has GitHub integration built into the backend itself. There's nothing to write in `.github/workflows/`.

1. Push your project to a GitHub repository.
2. In the Firebase console go to **Hosting & Serverless → App Hosting → Get started** (or run `firebase apphosting:backends:create`).
3. **Connect to GitHub**, authorize the Firebase GitHub app, and pick the repo.
4. Set the **root directory** (where `package.json` lives, usually `/`) and the **live branch** (usually `main`).
5. Leave **automatic rollouts** enabled.

From then on, every push to the live branch triggers a Cloud Build and a rollout. You can disable auto-rollouts in the backend's **Deployment settings** tab and trigger manually instead with:

```bash
firebase apphosting:rollouts:create BACKEND_ID --project PROJECT_ID
```

**Firestore rules aren't part of an App Hosting rollout.** Deploy them separately when they change:

```bash
firebase deploy --only firestore:rules
```

If you want rules tests and rules deployment gated in CI too, add the small workflow at the end of Path B — just drop the hosting step.

### Path B — Firebase Hosting via GitHub Actions

#### Service account

The simplest route is to let the CLI set everything up:

```bash
firebase init hosting:github
```

It creates a scoped service account, stores the key as a repository secret named `FIREBASE_SERVICE_ACCOUNT_<PROJECT_ID>`, and writes a starter workflow.

If you'd rather do it by hand, create a service account in the [Google Cloud Console](https://console.cloud.google.com/iam-admin/serviceaccounts) with **only** these roles (this is the set the deploy action's own documentation specifies):

| Role | ID | Why |
| --- | --- | --- |
| Firebase Hosting Admin | `roles/firebasehosting.admin` | Deploy to Hosting channels |
| Firebase Authentication Admin | `roles/firebaseauth.admin` | Add preview URLs to Auth's authorized domains |
| Cloud Run Viewer | `roles/run.viewer` | Only if Hosting rewrites to Cloud Run / Functions |
| API Keys Viewer | `roles/serviceusage.apiKeysViewer` | Required by the CLI |
| Firebase Rules Admin | `roles/firebaserules.admin` | Only if the workflow also deploys `firestore.rules` |

Download a JSON key and add it as a GitHub secret under **Settings → Secrets and variables → Actions**. Avoid granting broad roles like Firebase Admin or Owner to a key that lives in GitHub.

#### Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      # Runs the Firestore emulator; ubuntu-latest ships with a JDK
      - name: Test security rules
        run: npm run test:rules

      - name: Build
        run: npm run build

      - name: Deploy Firestore rules
        run: |
          echo '${{ secrets.FIREBASE_SERVICE_ACCOUNT }}' > "$RUNNER_TEMP/sa.json"
          export GOOGLE_APPLICATION_CREDENTIALS="$RUNNER_TEMP/sa.json"
          npx firebase-tools deploy --only firestore:rules --project "${{ secrets.FIREBASE_PROJECT_ID }}"

      - name: Deploy Hosting
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          projectId: ${{ secrets.FIREBASE_PROJECT_ID }}
          channelId: live
```

Two things to know about this file:

- `action-hosting-deploy` deploys **Hosting only**. That's why there's a separate rules step — without it, the rules you tested never reach production. That step needs the Firebase Rules Admin role from the table above.
- The rules step authenticates the CLI by writing the service account JSON to a temp file and pointing `GOOGLE_APPLICATION_CREDENTIALS` at it. `action-hosting-deploy` does the same thing internally; it just accepts the JSON directly.

---

## 11. Next Steps

- Add rules tests for your app's real collections and fields — timestamps, enums, string length limits, and any cross-document references.
- Wire the Auth emulator into the seed script so you can test third-party sign-in flows (Google, Apple) offline.
- If you're on App Hosting, look at `apphosting.staging.yaml` and a second backend for a staging environment before you scale up.
- Move the email and username validators into a shared `firestore.rules` function file only if your rules grow large enough to warrant it — premature abstraction in rules makes them harder to audit.

## Sources

- [Antigravity docs: Firebase Studio Migration](https://antigravity.google/docs/firebase-studio-migration/)
- [Antigravity docs: Build with Google](https://antigravity.google/docs/build-with-google/)
- [Codelab: How to Migrate from Firebase Studio to Antigravity](https://codelabs.developers.google.com/antigravity/how-to-migrate-from-firebase-studio-to-antigravity)
- [Firebase: Studio sunset and project migration](https://firebase.google.com/docs/studio/migrating-project)
- [Firebase MCP server](https://firebase.google.com/docs/ai-assistance/mcp-server)
- [Firebase App Hosting: Get started](https://firebase.google.com/docs/app-hosting/get-started)
- [action-hosting-deploy: service account roles](https://github.com/FirebaseExtended/action-hosting-deploy/blob/main/docs/service-account.md)
