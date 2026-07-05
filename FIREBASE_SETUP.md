# Firebase setup (one-time, ~10 min, free)

**Coupled** uses Firebase (Google sign-in + Firestore). Anyone can sign in with Google; each person gets a private space and can invite a partner by email. It runs on the free **Spark plan** — no credit card, no cost for normal use.

---

## 1. Create a Firebase project
1. Go to **https://console.firebase.google.com** → **Add project** → name it (e.g. `coupled`) → you can turn **off** Analytics → **Create project**.

## 2. Turn on Google sign-in
1. **Build → Authentication → Get started**.
2. **Sign-in method** → **Google** → **Enable** → pick a support email → **Save**.
3. **Settings → Authorized domains → Add domain** → add your live domain (e.g. `your-site.netlify.app`). `localhost` is already allowed.

## 3. Create the database
1. **Build → Firestore Database → Create database** → pick a location → **Production mode** → **Enable**.
2. **Rules** tab → replace everything with the contents of **`firestore.rules`** → **Publish**.

## 4. Get your web config
1. Project **Settings** (gear) → **General** → **Your apps** → web icon `</>` → register app → copy the `firebaseConfig`.

## 5. Paste config into the app
In **`index.html`**, fill the `FIREBASE_CONFIG` object near the top of the `<script>` with your values. (These values are **not secret** — security is enforced by the Firestore rules.)

## 6. Deploy
Host the folder on any static host (e.g. Netlify). Connect this GitHub repo for auto-deploy on push.

---

## How it works
- Open the app → **Sign in with Google**. First sign-in creates your private space.
- **Set a secret phrase** — this is your end-to-end encryption key. Everything is encrypted in the browser with it before saving.
- Gear ⚙ → **Invite your partner** by their Google email. They sign in, enter the **same secret phrase** (tell them privately), and you both share one space in real time.
- Works offline; syncs when back online.

### About the encryption
- **Encrypted:** every habit log, list item, and Vault note. The database only stores ciphertext.
- **Not encrypted:** account emails and space membership (needed for login + invites), and that records exist / roughly when.
- **If you both forget the phrase, the data is gone** — no recovery, by design.
- The Vault's "hidden until Friday" is an honor-system UI rule, not a cryptographic time-lock.
