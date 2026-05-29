# Firebase Chat Application (FriendlyChat)

A group chat Android app backed by Firebase Auth, Realtime Database, and Storage.

Built in October 2020 as a college-era project, following along with Google's Udacity "Firebase in a Weekend" course. Package name still reflects that origin (`com.google.firebase.udacity.friendlychat`).

## What it does

Users sign in through FirebaseUI (Google, Twitter, Microsoft, Yahoo, Apple, Email, or Anonymous), then land in a single room where everyone's messages stream in live. They can send text or pick a JPEG from the gallery, which uploads to Firebase Storage and posts as an inline image. Sign out is in the overflow menu.

## Architecture

```
+--------------------+         +----------------------+
|  Android Client    |         |   Firebase Backend   |
|  (MainActivity)    |         |                      |
|                    |         |                      |
|  [Sign in UI]------|--Auth-->| FirebaseAuth +       |
|                    |  flow   | FirebaseUI providers |
|                    |         |   (Google, Email...) |
|                    |<--user--|                      |
|                    |         |                      |
|  [Send text]-------|--push-->| Realtime Database    |
|                    |         |   /messages/<id>     |
|                    |         |   { text, name,      |
|                    |         |     photoUrl }       |
|  [MessageAdapter]<-|--child- |                      |
|   ListView render  |  Added  |                      |
|                    | events  |                      |
|                    |         |                      |
|  [Pick photo]------|--upload>| Firebase Storage     |
|                    |         |   /chat_photos/<f>   |
|                    |<--URL---|                      |
|  push msg w/ URL---|--------->| Realtime Database   |
+--------------------+         +----------------------+
```

Auth gate first. Once `FirebaseAuth.AuthStateListener` reports a user, the app attaches a `ChildEventListener` to `/messages` and every new message fans out to all connected clients in real time. Photos are a two-step write: upload bytes to Storage, then push a `FriendlyMessage` containing the resulting download URL into the database.

## Tech stack

| Layer | Choice |
|-------|--------|
| Language | Java |
| Platform | Android (`compileSdkVersion 28`, `minSdkVersion 16`) |
| Auth | `com.google.firebase:firebase-auth:19.4.0` + `com.firebaseui:firebase-ui-auth:6.3.0` |
| Database | `com.google.firebase:firebase-database:19.4.0` (Realtime DB, not Firestore) |
| Storage | `com.google.firebase:firebase-storage:19.2.0` |
| Image loading | `com.github.bumptech.glide:glide:3.6.1` |
| Material UI | `com.google.android.material:material:1.0.0` |
| Build | Gradle + `com.google.gms.google-services` plugin |

## Notable techniques

- `MainActivity.attachDatabaseReadListener()` registers a `ChildEventListener` on `databaseReference = FirebaseDatabase.getInstance().getReference().child("messages")`. Every `onChildAdded` callback deserializes the snapshot directly into a `FriendlyMessage` POJO via `snapshot.getValue(FriendlyMessage.class)` — Firebase auto-mapping.
- The `FriendlyMessage` POJO has a no-arg constructor and JavaBean getters/setters specifically so Firebase deserialization works without custom code.
- Listeners are paired with lifecycle: `onResume` adds the `AuthStateListener`, `onPause` removes it. `attachDatabaseReadListener` / `detachDatabaseReadListener` mirror sign-in / sign-out so the app doesn't leak callbacks or stream messages while signed out.
- `MessageAdapter extends ArrayAdapter<FriendlyMessage>` switches the cell layout based on whether `getPhotoUrl() != null` — text messages show the `TextView`, photo messages hide it and load the URL into an `ImageView` via Glide.
- Photo upload chain: `mChatPhotosStorageReference.child(uri.getLastPathSegment()).putFile(uri).addOnSuccessListener(...)` — on success, push a new `FriendlyMessage` with `photoUrl = taskSnapshot.get()` (the download URI).
- FirebaseUI sign-in flow lists seven providers (Google, Twitter, Microsoft, Yahoo, Apple, Email, Anonymous) declaratively in one `Arrays.asList(...)` call.

## How to run

1. Create a Firebase project at <https://console.firebase.google.com>.
2. Enable **Authentication** (turn on whichever providers you want — at minimum Email or Anonymous).
3. Enable **Realtime Database** and set rules to allow authenticated reads/writes (start in test mode for local development).
4. Enable **Storage** and set rules similarly.
5. Register an Android app with package name `com.google.firebase.udacity.friendlychat`, download `google-services.json`, drop it in `app/`.
6. Open in Android Studio, Gradle sync, run on an emulator or device with API 16+.

## What I'd do differently today

- The `MainActivity` `onActivityResult` block has an unbalanced brace — the photo-upload `else if` is missing its closing brace before the next `@Override`. The code as committed shouldn't compile cleanly; needs fixing before anyone tries to build it.
- One `setOnClickListener` is registered on `mSendButton` and then immediately overwritten by a second one — the first (clear-text-only) listener is dead code.
- `compileSdkVersion 28` and `buildToolsVersion "24.0.1"` are mismatched and stale. Bump to current.
- Realtime Database is fine for a tutorial; Firestore would scale better and give you offline persistence and queries for free.
- Move Firebase calls into a repository layer; let the Activity render UI only. Use a `RecyclerView` instead of `ListView`.
- Glide 3.6.1 is years out of date — Glide 4.x or Coil.
- Migrate to Kotlin + coroutines + Flow.

## License

Apache 2.0 — see `LICENSE` (carried over from the original Google/Udacity starter).
