# WiseGift — Specifications

## Vision

WiseGift is a gift-discovery mobile app (iOS and Android, built with Flutter) for people who want to give thoughtful, well-chosen gifts without the guesswork. Registered users build a personal agenda of contacts and occasions, get AI-ranked product recommendations drawn from an affiliate catalog, and curate shareable gift collections ("storefronts") that anyone — with or without an account — can browse on the web. Guests can browse editorial collections and run the AI Gift Curator without signing in; creating an account unlocks the full agenda, social, and collection features.

---

## Feature: Authentication — Email and Password

### User Story
As a visitor, I want to sign in with my email address and password so that I can access my personal data and gift collections.

### Acceptance Criteria
- [ ] The login screen presents an email field and a password field.
- [ ] The email field validates that an "@" symbol is present; the password field validates a minimum length of 6 characters.
- [ ] Tapping "Login" while fields are invalid shows inline validation messages and does not submit.
- [ ] A successful sign-in clears the navigation stack and lands the user on the Home screen.
- [ ] An unsuccessful sign-in (wrong credentials or network error) shows a snackbar with the error message; the form stays open.
- [ ] While the sign-in request is in flight, the form controls are disabled and a loading indicator is visible.
- [ ] A "Don't have an account?" link navigates to the registration flow.

---

## Feature: Authentication — Google Sign-In

### User Story
As a visitor, I want to sign in with my Google account so that I can access WiseGift without creating a separate password.

### Acceptance Criteria
- [ ] The login screen presents a "Sign in with Google" button.
- [ ] Tapping the button opens the platform Google account picker.
- [ ] A successful Google sign-in clears the navigation stack and lands the user on the Home screen.
- [ ] A failed or cancelled Google sign-in shows a snackbar with a descriptive error message.
- [ ] While the flow is in progress, the button is disabled and a loading indicator is shown.

---

## Feature: Registration — Four-Step Account Creation

### User Story
As a visitor, I want to create a WiseGift account through a guided four-step form so that my profile contains enough information for personalised recommendations from the start.

### Acceptance Criteria

**Step 1 — Credentials**
- [ ] The user enters first name, last name, email address, and password.
- [ ] The step label reads "Step 1 of 4" with a progress bar at 25 %.
- [ ] On success, a Firebase Auth account is created and the user advances to Step 2.
- [ ] On failure (e.g., email already in use), a snackbar shows the error and the form remains open.

**Step 2 — Personal details**
- [ ] The user enters a username, date of birth (selected from a date picker), and an optional phone number.
- [ ] The step label reads "Step 2 of 4" with the progress bar at 50 %.
- [ ] On save, a Firestore user profile document is created with the supplied fields.
- [ ] The user advances to Step 3.

**Step 3 — Location and gender**
- [ ] The user selects a residence country from a searchable country picker and selects a gender (Male / Female / Other).
- [ ] Both fields are required; attempting to continue without selecting both shows a snackbar error.
- [ ] The step label reads "Step 3 of 4" with the progress bar at 75 %.
- [ ] On save, the country code and gender are written to the Firestore profile; the user advances to Step 4.

**Step 4 — Interests**
- [ ] The user can select zero or more interests from a predefined list.
- [ ] The step label reads the interests screen title with the progress bar at 100 %.
- [ ] Tapping "Finish setup" saves the interest list to Firestore and advances to the contact import screen.
- [ ] A "Skip for now" link advances to the contact import screen without saving interests.

---

## Feature: Onboarding — Device Contact Import

### User Story
As a newly registered user, I want to import my phone contacts at the end of registration so that WiseGift can automatically populate birthday reminders for people I care about.

### Acceptance Criteria
- [ ] After completing (or skipping) Step 4, the user sees an onboarding screen explaining the contact import benefit.
- [ ] Tapping "Import My Contacts" triggers a system permission request for contacts access.
- [ ] If permission is granted, the app reads all contacts that have a birthday event, extracts names and birthday dates, and stores them in the user's birthday agenda in Firestore.
- [ ] During import, a progress indicator is displayed; the user cannot interact with the controls.
- [ ] After import (success or failure), the user is navigated to the Home screen with the full navigation stack cleared.
- [ ] Tapping "Maybe Later" skips the import and navigates directly to the Home screen.
- [ ] If the import fails, a snackbar notifies the user and the app proceeds to the Home screen.

---

## Feature: Home Screen — Navigation Shell

### User Story
As a user, I want a persistent bottom navigation bar so that I can switch between the main sections of the app at any time.

### Acceptance Criteria
- [ ] Logged-in users see five tabs: Home, Discover, Agenda, AI Curator, and Profile (labelled with the user's username once loaded).
- [ ] Guest (not logged-in) users see four tabs: Home, Discover, AI Curator, and Sign In.
- [ ] Tapping "Sign In" from the guest navigation opens the login screen as a modal page rather than replacing the tab content.
- [ ] The active tab is visually distinguished from inactive tabs.
- [ ] A notification bell icon is shown in the app bar for logged-in users. An unread-count badge appears when there are unread notifications.
- [ ] Tapping the notification bell navigates to the Notifications screen.
- [ ] When the user is on the Agenda tab (index 2, logged in), a floating action button labelled "Add Occasion" is visible; tapping it opens the Create Occasion screen.

---

## Feature: Home Screen — Curated Collections Feed

### User Story
As a user, I want to browse editorially curated gift collections on the Home screen so that I can discover gift ideas even before I know exactly what I'm looking for.

### Acceptance Criteria
- [ ] The Home tab displays a grid of curated collection cards, each showing a cover image, title, and starting price.
- [ ] Collection cards carry a "CURATED" badge.
- [ ] A horizontal chip bar lets the user filter collections by category (All, Birthday, Wedding, Baby). Selecting a category shows only matching collections.
- [ ] Logged-out users see a "Gift Discovery" hero banner above the grid with a call-to-action button that scrolls down to the collections.
- [ ] Logged-in users see an "Upcoming" banner above the grid that shows their next chronological occasion (name, date, and days remaining). Tapping the banner navigates to that occasion's detail page.
- [ ] Logged-in users can tap a heart icon on any curated collection card to save or unsave it. Saved collections appear in a "Saved Collections" section on the Profile tab.
- [ ] Logged-out users who tap the heart are shown a bottom sheet prompting them to join or sign in.

---

## Feature: Product Discovery (Explore / Search)

### User Story
As a user, I want to search and browse products from the catalog so that I can find specific gifts and save them to my wish list.

### Acceptance Criteria
- [ ] The Discover tab shows a text search bar. On load, a default set of products is fetched and displayed.
- [ ] Typing in the search bar triggers a debounced search (500 ms) against the backend catalog. A minimum of 2 characters is required before a search fires.
- [ ] Clearing the search field resets the results to the default recommendations.
- [ ] Filter controls allow the user to narrow results by budget range, occasion, age range, and gender. Changing a filter immediately re-fetches results.
- [ ] Each product card shows the product image, name, price, and a toggle-wishlist button.
- [ ] Logged-in users can tap the wishlist toggle to add or remove a product from their wish list. The toggle state reflects the current wish-list state.
- [ ] Logged-out users can view product cards but cannot save them (no wishlist toggle action for unauthenticated users).
- [ ] A loading indicator is shown while a search or filter is in progress.

---

## Feature: AI Gift Curator (Gift Agent)

### User Story
As a user, I want an AI-powered gift finder that walks me through a short questionnaire so that I get a curated list of personalised, budget-appropriate gift ideas without needing to search.

### Acceptance Criteria

**Step 1 — Recipient**
- [ ] The user optionally enters the recipient's name.
- [ ] The user selects the relationship type: Friend, Partner, Parent, or Other.
- [ ] The user selects an age range from a predefined list (Baby, Toddler, Child, Teen, Adult, Senior) or types an exact age.
- [ ] The user selects a gender preference: Any, Female, or Male.
- [ ] A step indicator shows "Recipient" as the active step.

**Step 2 — Occasion**
- [ ] The user selects one occasion type from a 4-column grid: Birthday, Christmas, Wedding, Anniversary, Baby Shower, Graduation, Housewarming, Other.
- [ ] A step indicator shows "Occasion" as the active step.

**Step 3 — Budget and Interests**
- [ ] The user enters a maximum budget in euros. The default hint is 100.
- [ ] The user can add free-text interest tags (e.g., "hiking", "coffee"). Tags can be removed with a chip delete button.
- [ ] A summary card recaps the selections from Steps 1 and 2.
- [ ] A step indicator shows "Budget" as the active step.
- [ ] Tapping "Find Gifts" sends the input to the backend AI recommendation endpoint.

**Results**
- [ ] While the AI pipeline runs, a full-screen loading state is shown with copy indicating the AI is working.
- [ ] On success, the user is navigated to a results list showing each recommended gift as a card with product image, name, and source (e.g., eBay).
- [ ] Tapping a gift card opens the product's affiliate URL in the device's default browser.
- [ ] On error, a snackbar describes the failure and the user remains on the curator screen.
- [ ] The AI Curator is accessible to both logged-in and logged-out users.
- [ ] The recommendation request uses the logged-in user's `residenceCountry` from their Firestore profile. For guests, the country is resolved via IP geolocation, falling back to device locale, then `US`.

---

## Feature: Agenda — Calendar View

### User Story
As a logged-in user, I want to see all my upcoming occasions on a calendar so that I can plan gift purchases in advance.

### Acceptance Criteria
- [ ] The Agenda tab renders a calendar view that streams the user's upcoming occasions from Firestore in real time.
- [ ] Occasions are visually marked on their respective calendar dates.
- [ ] The Agenda tab is only available to logged-in users.

---

## Feature: Agenda — Create Occasion

### User Story
As a logged-in user, I want to create a custom occasion with a name, type, date, and optional wish-list hints so that I can track any gift-giving event that matters to me.

### Acceptance Criteria
- [ ] The "Add Occasion" FAB on the Agenda tab opens a creation form.
- [ ] The form has a required "Occasion Name" text field and a required date picker (future dates only, up to 2 years ahead).
- [ ] Quick-select chips for the most common occasion types (Birthday, Christmas, Wedding, Anniversary, Baby Shower) pre-fill the name field if it is empty when a chip is tapped.
- [ ] The user can add free-text wish-list hint items (e.g., "Coffee Maker") as chips. Individual hints can be removed.
- [ ] Tapping "Create Occasion" with a valid name and date saves the occasion to Firestore and returns to the Agenda, showing a success snackbar.
- [ ] Attempting to save without a name or date shows a snackbar error; the form stays open.

---

## Feature: Agenda — Occasion Detail

### User Story
As a logged-in user, I want to view the details of a saved occasion so that I can see curated gift recommendations associated with it and check who is attending.

### Acceptance Criteria
- [ ] Tapping an occasion from the Agenda or the Home upcoming banner opens the Occasion Detail screen.
- [ ] The detail screen header shows the occasion name and date.
- [ ] If AI-curated gift recommendations have been generated for the occasion, they appear in a horizontal scrollable "The Selection" list, each shown as a product card.
- [ ] If no recommendations exist yet, a "Get Curated Recommendations" button is shown. Tapping it triggers the backend RAG pipeline and populates the recommendation list on success.
- [ ] While recommendations are generating, a full-screen loading state is shown.
- [ ] For birthday occasions, a "Muse Profile" section shows the recipient's name and interests.
- [ ] If the occasion has a linked recipient (a WiseGift user), their name is tappable and navigates to their public profile.
- [ ] For non-birthday occasions, a "Guest List" section lists invited guests by username. Tapping a guest navigates to their public profile.
- [ ] On error during recommendation generation, a snackbar describes the failure.

---

## Feature: Birthday Agenda — Contact Import and Management

### User Story
As a logged-in user, I want to manage a list of contacts with their birthdays so that I get timely reminders and gift ideas for the people I care about.

### Acceptance Criteria
- [ ] From the Profile tab, a "Birthday Import" menu item opens the Birthday Agenda screen.
- [ ] The Birthday Agenda screen shows a list of all saved birthdays (name and date).
- [ ] A "Sync Phone Contacts" action requests contacts permission and presents a contact selector for picking which phone contacts to import, including their birthday dates.
- [ ] For contacts without a birthday already stored, the user can tap the entry to open a date picker and assign a birthday manually.
- [ ] Existing birthdays can be updated by tapping the entry and picking a new date.
- [ ] Each birthday entry has a delete icon; tapping it removes that entry from the agenda after confirmation.
- [ ] The birthday list updates in real time via a Firestore stream.

---

## Feature: My Account — Profile Overview

### User Story
As a logged-in user, I want a profile screen that summarises my identity, collections, and settings so that I can navigate to any personal area of the app from one place.

### Acceptance Criteria
- [ ] The Profile tab shows a collapsible header with the user's display name (first + last name, or username as fallback) and their @username.
- [ ] An edit icon in the app bar opens the Edit Profile screen.
- [ ] An "About Me" section shows birthday, country, gender, and interests read from Firestore in real time.
- [ ] A "My Gift Collections" section shows a preview of up to two of the user's named collections (with a mosaic thumbnail and gift count). A "See all" link opens the full My Collections screen. A "New collection" button opens the create-collection dialog.
- [ ] A "Saved Collections" section appears when the user has saved at least one editorial collection. Each saved card shows an image, title, and an unsave button.
- [ ] An "Agenda" section contains a "Birthday Import" menu item.
- [ ] A "Settings" section contains a "Preferences" menu item.
- [ ] "Log out" and "Delete Account" actions are presented at the bottom; each requires explicit confirmation in a dialog before executing.
- [ ] Logging out clears the session and returns the user to the root (unauthenticated) state.
- [ ] Deleting the account removes the Firebase Auth account; the user is returned to the root state.

---

## Feature: My Account — Edit Profile

### User Story
As a logged-in user, I want to edit my profile details so that my information stays accurate and my recommendations improve over time.

### Acceptance Criteria
- [ ] The Edit Profile screen loads the current username, birthday, country, interests, and profile photo from Firestore.
- [ ] The user can change their username (required field).
- [ ] The user can update their date of birth and residence country.
- [ ] The user can add new interest tags or remove existing ones using a chip-based editor.
- [ ] The user can pick a new profile photo from the device gallery. The selected image is previewed immediately.
- [ ] Saving uploads the new photo to Firebase Storage (if changed), then writes all field updates to Firestore.
- [ ] On success, a snackbar confirms the update and the screen closes.
- [ ] On error, a snackbar describes the failure and the form remains open.

---

## Feature: Gift Collections — My Collections

### User Story
As a logged-in user, I want to organise my saved wishlist products into named collections so that I can share curated lists for specific occasions or themes.

### Acceptance Criteria
- [ ] The My Collections screen lists all named gift collections, each showing a mosaic thumbnail (up to 4 product images) and the gift count.
- [ ] Tapping a collection opens the Wish List screen filtered to that collection's products.
- [ ] A "New Collection" button opens a dialog where the user types a name; confirming creates the collection in Firestore and opens it immediately.
- [ ] Each collection card shows a link icon that, when tapped, copies the collection's public URL (format: `https://wisegift.web.app/u/<username>/<slug>`) to the clipboard and shows a confirmation snackbar.

---

## Feature: Gift Collections — Wish List (Personal Storefront)

### User Story
As a logged-in user, I want to manage the products in my wish list and collections so that I can keep my storefront curated and control what is publicly visible.

### Acceptance Criteria
- [ ] The Wish List screen displays all saved products in a two-column editorial grid.
- [ ] A horizontal category bar allows filtering by collection. An "All" chip shows every saved product; collection name chips show only products in that collection. An "+" icon opens the create-collection dialog.
- [ ] Each product card shows the product image, name, and price. A "See This Gift" button opens the affiliate link in the browser.
- [ ] Own-wishlist view: a lock toggle on each card switches a product between public and private. Private products are hidden when the wishlist is viewed by others.
- [ ] Own-wishlist view: a folder icon on each card lets the user move the product to a different collection via a bottom-sheet picker.
- [ ] A publishing banner shows the user's public collection URL with a "Copy Link" button.
- [ ] A share icon (top bar) opens the system share sheet with the public URL.
- [ ] An optional banner image is shown in the header. The owner can tap a camera icon to replace it from the device gallery; the image is uploaded to Firebase Storage.
- [ ] Products saved from the Discover / Search tab, or from another user's public storefront, are added to the General (default) collection unless a category was specified.

---

## Feature: Public Storefront (Unauthenticated Browsing)

### User Story
As any user (authenticated or not), I want to browse another person's public gift collections at a shareable URL so that I can discover what they want and possibly add items to my own list.

### Acceptance Criteria
- [ ] The URL `https://wisegift.web.app/u/<username>` displays a public overview page of all of that user's non-private collections, showing a mosaic tile for each collection with name and gift count.
- [ ] Tapping a collection tile navigates to `https://wisegift.web.app/u/<username>/<slug>`, which shows the products in that collection in the same two-column editorial grid.
- [ ] The header on both pages displays the user's username and their optional banner image.
- [ ] Each product card has a "Want This" heart button. Authenticated users who tap it have the product added to their own wish list; unauthenticated users see a snackbar prompting them to join.
- [ ] Private products (where `isPrivate = true`) are never shown on the public storefront.
- [ ] A footer section on both pages contains a "Join WiseGift" call-to-action.
- [ ] If the username does not exist, the page shows "Collection Not Found".

---

## Feature: User Profile (Viewing Others)

### User Story
As a logged-in user, I want to view another user's public profile so that I can see their interests and access their gift storefront.

### Acceptance Criteria
- [ ] Visiting another user's profile shows their @username, birthday, country, gender, and interests.
- [ ] A "Storefront" tile links to that user's public wish list screen (in-app, showing only their public items).
- [ ] Logged-in users who view another user's profile can access a moderation menu (three-dot icon) with two options:
  - **Report**: opens a text field dialog; on confirmation, submits a report to the backend and shows a "User reported" snackbar.
  - **Block**: shows a confirmation dialog; on confirmation, blocks the user and navigates back.
- [ ] The moderation menu is not shown when viewing one's own profile.

---

## Feature: Notifications

### User Story
As a logged-in user, I want an in-app notification centre so that I know when I am invited to an event.

### Acceptance Criteria
- [ ] The notification bell in the app bar shows a real-time unread count badge when there are unread notifications.
- [ ] The Notifications screen lists all notifications in reverse-chronological order. Each item shows a type-specific icon, title, body text, and formatted timestamp.
- [ ] Notification types currently supported: **eventInvite** (orange icon). All other types show a generic icon.
- [ ] Unread notifications are visually distinguished from read ones (border colour and background tint).
- [ ] Tapping any notification marks it as read.
- [ ] Tapping an **eventInvite** notification also navigates to the relevant Occasion Detail screen.
- [ ] If the user has no notifications, an empty-state illustration and message are shown.

---

## Feature: Preferences

### User Story
As a logged-in user, I want to configure my app language, birthday reminder lead time, and per-relationship spending limits so that WiseGift behaves according to my personal habits.

### Acceptance Criteria
- [ ] The Preferences screen is accessible from the "Settings" section on the Profile tab.
- [ ] **Language selector**: currently supports English and Spanish. Changing the language immediately applies the locale throughout the app without requiring a restart.
- [ ] **Reminder lead time**: a slider from 1 to 30 days controls how many days before an occasion the user wants to be reminded. The current value is shown next to the slider.
- [ ] **Per-relationship budget limits**: separate sliders for Partner, Close Family, Close Friend, and Other, each ranging from €10 to €500.
- [ ] Tapping "Save Preferences" or the checkmark in the app bar persists the settings to Firestore and closes the screen with a success snackbar.
- [ ] On error, a snackbar describes the failure and the screen remains open.

---

## Open Questions

None.

---

## Resolved Questions

- **Occasion wishlist hints vs. product catalog** *(closed 2026-07-01)*: `WishlistItem` always references a real catalog `productId`. The `hint` field is used only for AI-generated item descriptions. The `AddGiftWishSheet` surfaces two tabs — CATALOG search and MY COLLECTIONS — so users always select real products. Free-text chips described in an earlier spec draft were never implemented.

- **Guest save button on product cards** *(closed 2026-07-01)*: The save/wishlist button is **visible and tappable for guests**. Tapping stores a pending save via `PendingSaveService` and shows a `RegistrationGateSheet` prompting sign-up. No no-op behaviour.

- **Occasion types vs. quick-select chips** *(closed 2026-07-01)*: All 8 `OccasionType` values (`birthday`, `christmas`, `wedding`, `anniversary`, `babyShower`, `graduation`, `housewarming`, `other`) are already rendered as chips on the Create Occasion screen via `_occasionIcons`. The spec's reference to "only 5" was out of date.

- **Language support scope** *(closed 2026-07-01)*: `app_en.arb` and `app_es.arb` have identical coverage (469 lines each). Spanish localisation is complete for all currently implemented strings.

- **Notification delivery mechanism** *(closed 2026-07-02)*: Background push (FCM/APNs) is implemented. Flutter registers the device FCM token to `users/{uid}.fcmTokens` on sign-in. The Spring `user` service runs a `@Scheduled` job daily at 09:00 UTC that reads upcoming occasions per user, respects `reminderLeadTimeDays` (default 7), and sends both an FCM push and an in-app Firestore notification. Web push (VAPID) is explicitly out of scope. **Pending manual step:** APNs key must be uploaded in Firebase Console → Project Settings → Cloud Messaging before iOS push delivery works.
