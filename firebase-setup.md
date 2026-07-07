# Vac Ex Sucker — Firebase (public leaderboard) setup

The game works device-only until you connect Firebase. Once configured, scores go
to a **public leaderboard** and captured **company emails** go to a separate,
private `operators` collection (never shown on the board).

## 1. Create the project
1. Go to <https://console.firebase.google.com> → **Add project** (any name, e.g. `vacex-sucker`). Google Analytics is optional.
2. In the project, click **Build → Firestore Database → Create database**.
   - Start in **Production mode** (we set explicit rules below).
   - Pick a location close to your users (e.g. `europe-west2` for the UK).

## 2. Register a Web App and copy the config
1. Project Overview → the **`</>`** (Web) icon → give it a nickname → **Register app**.
2. Copy the `firebaseConfig` object it shows you.
3. Open `index.html`, find the `firebaseConfig = { ... }` block near the top (inside
   the `<script type="module">`), and replace the `YOUR_...` placeholders with your values.

## 3. Paste the security rules
Firestore → **Rules** tab → replace everything with the block below → **Publish**.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Public leaderboard: anyone may read; anyone may add a validated row.
    match /leaderboard/{doc} {
      allow read: if true;
      allow create: if request.resource.data.keys().hasOnly(
                        ['first','last','company','score','level','distance','ts'])
                     && request.resource.data.score is number
                     && request.resource.data.score >= 0
                     && request.resource.data.score <= 100000000
                     && request.resource.data.first is string
                     && request.resource.data.first.size() <= 40
                     && request.resource.data.last is string
                     && request.resource.data.last.size() <= 40
                     && request.resource.data.company is string
                     && request.resource.data.company.size() <= 60;
      allow update, delete: if false;
    }

    // Lead capture: write-only. Emails are NOT publicly readable.
    match /operators/{doc} {
      allow create: if request.resource.data.email is string
                     && request.resource.data.email.size() <= 120;
      allow read, update, delete: if false;
    }
  }
}
```

These rules:
- let the browser read the board and submit a score (no login needed),
- block editing/deleting other people's scores,
- keep the captured emails **write-only** — you read them from the Firebase
  console (Firestore → `operators`) or export them, but the public anon key cannot.

## 4. Done
Re-host the site. Sign-ups (name, company, **company email**) land in `operators`;
each finished shift adds a row to `leaderboard`; the in-game board and Game Over
screen read the live top 25.

### Notes
- The `apiKey` in the config is **safe to publish** — it only identifies the project;
  Firestore access is controlled entirely by the rules above.
- Personal-email domains (gmail, outlook, yahoo, icloud, btinternet, etc.) are rejected
  on the sign-up screen. To add/remove domains, edit the `PERSONAL_DOMAINS` set in `index.html`.
- To view or export captured leads: Firestore → `operators` collection.
