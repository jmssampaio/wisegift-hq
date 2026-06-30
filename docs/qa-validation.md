# QA Validation Report
**Date:** 2026-06-30
**Validator:** QA Tester
**Spec version:** docs/spec.md (as of 2026-06-30)

## Summary
| Status | Count |
|---|---|
| ✅ PASS | 72 |
| ❌ FAIL | 14 |
| ⚠️ UNTESTABLE | 19 |
| **Total** | **105** |

---

## Feature: Authentication — Email and Password

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Login screen presents an email field and a password field | ✅ PASS | login_page.dart:115–145 |
| 2 | Email validates "@"; password validates min 6 chars | ✅ PASS | login_page.dart:124–144 |
| 3 | Invalid form shows inline messages and does not submit | ✅ PASS | login_page.dart:32 — `_formKey.currentState!.validate()` gates submission |
| 4 | Successful sign-in clears stack and lands on Home | ✅ PASS | login_page.dart:43 — `popUntil(isFirst)` triggers AuthGatekeeper which routes to Home |
| 5 | Unsuccessful sign-in shows snackbar; form stays open | ✅ PASS | login_page.dart:45–53 |
| 6 | While in-flight, controls disabled and loading indicator visible | ✅ PASS | login_page.dart:102–104 (indicator), :123 + :138 (`enabled: !_isLoading`) |
| 7 | "Don't have an account?" link navigates to registration | ✅ PASS | login_page.dart:156–165 |

---

## Feature: Authentication — Google Sign-In

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Login screen presents "Sign in with Google" button | ✅ PASS | login_page.dart:181–191 |
| 2 | Tapping button opens platform Google account picker | ⚠️ UNTESTABLE | Code calls `_authRepository.signInWithGoogle()` (login_page.dart:64) which delegates to the Firebase Google Sign-In plugin; requires device |
| 3 | Successful Google sign-in clears stack and lands on Home | ✅ PASS | login_page.dart:67–69 — `popUntil(isFirst)` |
| 4 | Failed/cancelled Google sign-in shows snackbar | ✅ PASS | login_page.dart:70–76 |
| 5 | While in progress, button disabled and loading shown | ✅ PASS | login_page.dart:62 (`_isLoading = true`), :184 (`onPressed: _isLoading ? null : _signInWithGoogle`), :102–104 (indicator) |

---

## Feature: Registration — Four-Step Account Creation

| # | Criterion | Status | Notes |
|---|---|---|---|
| **Step 1** | | | |
| 1 | User enters first name, last name, email, and password | ✅ PASS | registration_step1_form.dart:38–77 — all four fields present |
| 2 | Step label reads "Step 1 of 4"; progress bar at 25% | ✅ PASS | registration_step1_page.dart:75–93 (`stepXofY(1,4)`, `value: 0.25`) |
| 3 | On success, Firebase Auth account is created; user advances to Step 2 | ✅ PASS | registration_step1_page.dart:35–43 — calls `createAccountWithEmailAndPassword` then pushes Step 2 |
| 4 | On failure, snackbar shows error and form stays open | ✅ PASS | registration_step1_page.dart:45–49 |
| **Step 2** | | | |
| 5 | User enters username, date of birth (date picker), optional phone | ✅ PASS | registration_step2_page.dart:23–25, 37–49 — all three fields |
| 6 | Step label reads "Step 2 of 4"; progress bar at 50% | ✅ PASS | registration_step2_page.dart:102, 119 (`value: 0.50`) |
| 7 | On save, Firestore user profile document created | ✅ PASS | registration_step2_page.dart:67–75 — `createUserProfile` with username, uuid, email, phone, birthday |
| 8 | User advances to Step 3 | ✅ PASS | registration_step2_page.dart:77–80 |
| **Step 3** | | | |
| 9 | User selects residence country from searchable country picker and selects gender | ✅ PASS | registration_step3_page.dart:28–38 (country picker), :129–148 (gender dropdown) |
| 10 | Both fields required; missing fields shows snackbar | ✅ PASS | registration_step3_page.dart:43–48 |
| 11 | Step label reads "Step 3 of 4"; progress bar at 75% | ✅ PASS | registration_step3_page.dart:89, 106 (`value: 0.75`) |
| 12 | Country code and gender written to Firestore; user advances to Step 4 | ✅ PASS | registration_step3_page.dart:56–65 |
| **Step 4** | | | |
| 13 | User can select zero or more interests from predefined list | ✅ PASS | registration_step4_page.dart:96–99; InterestsSelection widget used |
| 14 | Step label shows interests screen title; progress bar at 100% | ✅ PASS | registration_step4_page.dart:65–85 (`interestsTitle`, `value: 1.0`) |
| 15 | "Finish setup" saves interests to Firestore and advances to contact import | ✅ PASS | registration_step4_page.dart:25–55 |
| 16 | "Skip for now" advances to contact import without saving interests | ✅ PASS | registration_step4_page.dart:124–130 — navigates to OnboardingImportPage |

---

## Feature: Onboarding — Device Contact Import

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | After Step 4, user sees onboarding screen explaining contact import benefit | ✅ PASS | onboarding_import_page.dart:86–115 |
| 2 | Tapping "Import My Contacts" triggers system permission request | ⚠️ UNTESTABLE | onboarding_import_page.dart:26 — `FlutterContacts.requestPermission()` exists; requires device |
| 3 | If permission granted, app reads contacts with birthday events and stores them | ✅ PASS | onboarding_import_page.dart:27–47 — reads birthday events, calls `_birthdayService.importDeviceContacts` |
| 4 | During import, progress indicator shown; controls not interactive | ✅ PASS | onboarding_import_page.dart:74–84 — `_isImporting` flag shows CircularProgressIndicator and hides buttons |
| 5 | After import (success or failure), navigated to Home with stack cleared | ✅ PASS | onboarding_import_page.dart:56–66 — `pushAndRemoveUntil(..., (route) => false)` |
| 6 | "Maybe Later" skips import and navigates to Home | ✅ PASS | onboarding_import_page.dart:112–115 — calls `_navigateToHome()` |
| 7 | If import fails, snackbar notifies and proceeds to Home | ✅ PASS | onboarding_import_page.dart:49–55 — snackbar shown, then `_navigateToHome()` called |

---

## Feature: Home Screen — Navigation Shell

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Logged-in users see five tabs: Home, Discover, Agenda, AI Curator, Profile (username label) | ✅ PASS | home_page.dart:119–144 — five items; Profile label uses `_username ?? l10n.profile` |
| 2 | Guest users see four tabs: Home, Discover, AI Curator, Sign In | ✅ PASS | home_page.dart:146–167 |
| 3 | "Sign In" tab opens login screen as modal page (not replacing tab content) | ✅ PASS | home_page.dart:76–79 — pushed via `Navigator.push` (modal), not `setState` |
| 4 | Active tab is visually distinguished | ✅ PASS | home_page.dart:219–222 — `selectedItemColor` vs `unselectedItemColor` on BottomNavigationBar |
| 5 | Notification bell shown for logged-in users with unread-count badge | ✅ PASS | home_page.dart:178–195 — StreamBuilder with Badge widget |
| 6 | Tapping notification bell navigates to Notifications screen | ✅ PASS | home_page.dart:190–193 |
| 7 | On Agenda tab (index 2, logged in), FAB "Add Occasion" visible; tapping opens Create Occasion screen | ✅ PASS | home_page.dart:225–236 |

---

## Feature: Home Screen — Curated Collections Feed

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Home tab displays grid of curated collection cards with cover image, title, and starting price | ✅ PASS | curated_collections_view.dart:298–323 — SliverGrid of `_CollectionCard` widgets |
| 2 | Collection cards carry a "CURATED" badge | ✅ PASS | curated_collections_view.dart:571–591 — "CURATED" text overlay |
| 3 | Horizontal chip bar filters by category (All, Birthday, Wedding, Baby) | ✅ PASS | curated_collections_view.dart:33 — `_categories = ['All', 'Birthday', 'Wedding', 'Baby']`; chips at :261–291 |
| 4 | Logged-out users see "Gift Discovery" hero banner with CTA that scrolls down | ✅ PASS | curated_collections_view.dart:236–237, 328–388 — `_DiscoveryBanner` with `onExplore: _scrollToCollections` |
| 5 | Logged-in users see "Upcoming" banner showing next occasion (name, date, days remaining); tapping navigates to detail | ✅ PASS | curated_collections_view.dart:238–246, 391–463 — `_UpcomingBanner` |
| 6 | Logged-in users can tap heart icon to save/unsave curated collection | ✅ PASS | curated_collections_view.dart:196–217 — `_toggleSave` via `_usersService.saveCollection/unsaveCollection` |
| 7 | Logged-out users who tap heart see bottom sheet prompting join or sign in | ✅ PASS | curated_collections_view.dart:97–153 — `_showSaveGate` bottom sheet |

---

## Feature: Product Discovery (Explore / Search)

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Discover tab shows text search bar; on load default products fetched | ✅ PASS | search_page.dart:60, 76–88; product_search_tab.dart:100–126 |
| 2 | Typing triggers debounced search (500ms); minimum 2 chars | ✅ PASS | search_page.dart:138–148 — 500ms timer; `query.length >= 2` guard |
| 3 | Clearing search field resets results to default recommendations | ✅ PASS | search_page.dart:141–144 — empty query calls `_refreshSearch()` → `_loadDefaultRecommendations()` |
| 4 | Filter controls by budget range, occasion, age range, gender; changing filter re-fetches | ✅ PASS | search_page.dart:197–212; product_search_tab.dart:282–368 — all four filter dimensions |
| 5 | Each product card shows image, name, price, and wishlist toggle button | ✅ PASS | product_card.dart:28–147 |
| 6 | Logged-in users can toggle wishlist | ✅ PASS | search_page.dart:150–174 |
| 7 | Logged-out users cannot save (no action for unauthenticated) | ❌ FAIL | search_page.dart:150–152 — `_toggleWishlist` returns early if `_currentUserId == null` but the heart button is still rendered and tappable on `ProductCard`. The `showFavoriteButton` parameter defaults to `true` and is always passed without an auth check. Guest users see a tappable heart that silently does nothing. Spec requires either hidden toggle or clear prompting. |
| 8 | Loading indicator shown while search or filter in progress | ✅ PASS | product_search_tab.dart:143–144 — `CircularProgressIndicator` when `isLoading` |

---

## Feature: AI Gift Curator (Gift Agent)

| # | Criterion | Status | Notes |
|---|---|---|---|
| **Step 1 — Recipient** | | | |
| 1 | User optionally enters recipient name | ✅ PASS | gift_agent_page.dart:217–243 |
| 2 | User selects relationship type: Friend, Partner, Parent, Other | ✅ PASS | gift_agent_page.dart:249–254 — RelationshipType enum chips (4 values) |
| 3 | User selects age range from predefined list or types exact age | ✅ PASS | gift_agent_page.dart:258–296 — 6-item chip list plus free text field |
| 4 | User selects gender preference: Any, Female, Male | ✅ PASS | gift_agent_page.dart:299–307 |
| 5 | Step indicator shows "Recipient" as active step | ✅ PASS | gift_agent_page.dart:160 — `labels = ['Recipient', 'Occasion', 'Budget']` |
| **Step 2 — Occasion** | | | |
| 6 | User selects one occasion from 4-column grid: 8 types | ✅ PASS | gift_agent_page.dart:367–406 — GridView with 4 columns over `OccasionType.values` (8 items) |
| 7 | Step indicator shows "Occasion" as active step | ✅ PASS | gift_agent_page.dart:160 |
| **Step 3 — Budget and Interests** | | | |
| 8 | User enters max budget in euros; default hint is 100 | ✅ PASS | gift_agent_page.dart:429–450 — `hintText: '100'` |
| 9 | User adds free-text interest tags as chips; chips can be removed | ✅ PASS | gift_agent_page.dart:456–500 |
| 10 | Summary card recaps Steps 1 and 2 selections | ✅ PASS | gift_agent_page.dart:503–523 |
| 11 | Step indicator shows "Budget" as active step | ✅ PASS | gift_agent_page.dart:160 |
| 12 | Tapping "Find Gifts" sends input to backend AI recommendation endpoint | ✅ PASS | gift_agent_page.dart:71–103 — calls `_giftRecommendationService.getGiftRecommendations` |
| **Results** | | | |
| 13 | While AI pipeline runs, full-screen loading state shown | ✅ PASS | gift_agent_page.dart:110–128 — full-screen loading scaffold |
| 14 | On success, navigated to results list with product image, name, source | ✅ PASS | gift_agent_page.dart:93–96; gift_ideas_page.dart:46–83 |
| 15 | Tapping gift card opens affiliate URL in device browser | ✅ PASS | gift_ideas_page.dart:74–79 — `launchUrl(uri, mode: externalApplication)` |
| 16 | On error, snackbar describes failure; user remains on curator screen | ✅ PASS | gift_agent_page.dart:97–100 |
| 17 | AI Curator accessible to both logged-in and logged-out users | ✅ PASS | home_page.dart:92–113 — GiftAgentPage included in both logged-in and guest tab arrays |

---

## Feature: Agenda — Calendar View

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Agenda tab renders calendar view streaming user's occasions from Firestore in real time | ✅ PASS | calendar_view.dart:118–132 — StreamBuilder over `specialOccasionsStream`; TableCalendar renders events |
| 2 | Occasions visually marked on calendar dates | ✅ PASS | calendar_view.dart:162 — `eventLoader` + `markerDecoration` on CalendarStyle |
| 3 | Agenda tab only available to logged-in users | ✅ PASS | home_page.dart:91–113 — Agenda (index 2) only in logged-in pages list |

---

## Feature: Agenda — Create Occasion

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | "Add Occasion" FAB opens creation form | ✅ PASS | home_page.dart:226–235 |
| 2 | Form has required "Occasion Name" field and required date picker (future dates only, up to 2 years ahead) | ✅ PASS | create_occasion_page.dart:62–77 — `firstDate: DateTime.now()`, `lastDate: +2 years`; validator at create_occasion_view_model.dart:104 |
| 3 | Quick-select chips for 5 occasion types pre-fill name field if empty | ✅ PASS | create_occasion_page.dart:34–39 (5 types), :138–152 (pre-fills name when empty) |
| 4 | User can add free-text wish-list hint items as chips; individual hints can be removed | ✅ PASS | create_occasion_page.dart:221–255 — add/remove wishlist items as chips |
| 5 | "Create Occasion" with valid name and date saves to Firestore and returns to Agenda with success snackbar | ✅ PASS | create_occasion_page.dart:80–98 |
| 6 | Saving without name or date shows snackbar; form stays open | ✅ PASS | create_occasion_page.dart:81–86 |

---

## Feature: Agenda — Occasion Detail

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Tapping occasion from Agenda or Home banner opens detail screen | ✅ PASS | calendar_view.dart:90–96 (agenda); curated_collections_view.dart:404–409 (home banner) |
| 2 | Detail screen header shows occasion name and date | ✅ PASS | occasion_detail_page.dart:110–164 — `_buildHeader` with name and `displayDate` |
| 3 | AI-curated recommendations appear in horizontal "The Selection" list if they exist | ✅ PASS | occasion_detail_page.dart:118–119, 166–198 |
| 4 | If no recommendations, "Get Curated Recommendations" button shown; tapping triggers RAG pipeline | ✅ PASS | occasion_detail_page.dart:120–121, 201–228 |
| 5 | While recommendations generating, full-screen loading state shown | ✅ PASS | occasion_detail_page.dart:77–97 |
| 6 | For birthday occasions, "Muse Profile" section shows recipient name and interests | ✅ PASS | occasion_detail_page.dart:122, 264–322 |
| 7 | If occasion has linked recipient (WiseGift user), name is tappable and navigates to their public profile | ✅ PASS | occasion_detail_page.dart:279–298 — `InkWell` with `UserProfilePage` push when `recipientId != null` |
| 8 | For non-birthday occasions, "Guest List" section lists guests; tapping navigates to public profile | ✅ PASS | occasion_detail_page.dart:123, 231–261 |
| 9 | On error during recommendation generation, snackbar describes failure | ✅ PASS | occasion_detail_page.dart:51–54 |

---

## Feature: Birthday Agenda — Contact Import and Management

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | From Profile tab, "Birthday Import" menu item opens Birthday Agenda screen | ✅ PASS | my_account_page.dart:199–205 — navigates to `ContactListScreen` |
| 2 | Birthday Agenda screen shows list of saved birthdays (name and date) | ✅ PASS | contact_list_screen.dart:113–168 — StreamBuilder rendering birthday tiles |
| 3 | "Sync Phone Contacts" requests permission and presents contact selector | ✅ PASS | contact_list_screen.dart:38–63 — `FlutterContacts.requestPermission()`, then PhoneContactsList picker |
| 4 | For contacts without a birthday, user can tap entry to open date picker and assign one | ✅ PASS | contact_list_screen.dart:66–101, 203–238 — `_pickBirthdayForUser` with date picker |
| 5 | Existing birthdays can be updated by tapping and picking a new date | ✅ PASS | contact_list_screen.dart:84–95 — `birthdayId != null` branch calls `_birthdayService.updateBirthday` |
| 6 | Each birthday entry has a delete icon; tapping removes after no confirmation | ❌ FAIL | contact_list_screen.dart:228–231 — delete icon calls `_deleteBirthday` immediately without a confirmation dialog. Spec requires explicit confirmation before executing. |
| 7 | Birthday list updates in real time via Firestore stream | ✅ PASS | contact_list_screen.dart:29 — `_birthdayService.birthdaysStream`, rendered via StreamBuilder |

---

## Feature: My Account — Profile Overview

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Profile tab shows collapsible header with display name (first+last or username) and @username | ✅ PASS | my_account_page.dart:127–169 — SliverAppBar with expanded header, display name logic, @username |
| 2 | Edit icon in app bar opens Edit Profile screen | ✅ PASS | my_account_page.dart:177–183 |
| 3 | "About Me" section shows birthday, country, gender, interests from Firestore in real time | ✅ PASS | my_account_page.dart:210–221 — uses `watchUserProfile` StreamBuilder |
| 4 | "My Gift Collections" shows up to 2 collections (mosaic, gift count); "See all" link; "New collection" button | ✅ PASS | my_account_page.dart:192, 268–327 — shows `visible = allNames.take(2)`, "See all" nav, new collection button |
| 5 | "Saved Collections" section appears when user has at least one saved editorial collection | ✅ PASS | my_account_page.dart:502–531 — returns `SizedBox.shrink()` when `saved.isEmpty`, otherwise renders |
| 6 | "Agenda" section contains "Birthday Import" menu item | ✅ PASS | my_account_page.dart:197–206 |
| 7 | "Settings" section contains "Preferences" menu item | ✅ PASS | my_account_page.dart:224–238 |
| 8 | "Log out" and "Delete Account" each require explicit dialog confirmation | ✅ PASS | my_account_page.dart:47–74 (log out dialog), :77–107 (delete dialog) |
| 9 | Logging out clears session and returns to root unauthenticated state | ✅ PASS | my_account_page.dart:64–66 — `signOut()` then `popUntil(isFirst)` |
| 10 | Deleting account removes Firebase Auth account; returned to root state | ✅ PASS | my_account_page.dart:96–98 — `deleteAccount()` then `popUntil(isFirst)` |

---

## Feature: My Account — Edit Profile

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Edit Profile loads current username, birthday, country, interests, photo from Firestore | ✅ PASS | edit_profile_page.dart:154–159 — pre-populates all fields from Firestore data |
| 2 | User can change their username (required field) | ✅ PASS | edit_profile_page.dart:200–208 — validator present |
| 3 | User can update date of birth and residence country | ✅ PASS | edit_profile_page.dart:246–261 — both text fields present |
| 4 | User can add or remove interest tags via chip editor | ✅ PASS | edit_profile_page.dart:58–66, 214–242 |
| 5 | User can pick new profile photo from gallery; previewed immediately | ✅ PASS | edit_profile_page.dart:68–78 — `picker.pickImage`, `_imageBytes` shown in `MemoryImage` |
| 6 | Saving uploads photo to Firebase Storage (if changed) then writes fields to Firestore | ✅ PASS | edit_profile_page.dart:89–107 |
| 7 | On success, snackbar confirms and screen closes | ✅ PASS | edit_profile_page.dart:109–113 |
| 8 | On error, snackbar describes failure and form remains open | ✅ PASS | edit_profile_page.dart:115–119 |
| 9 | Birthday field is a free-text input, NOT a date picker | ❌ FAIL | edit_profile_page.dart:246–254 — birthday is a plain `TextFormField` with no date picker, contrary to step 2 registration which uses a date picker. The spec for Edit Profile says the user can "update their date of birth" but the input is unguided free text. Inconsistency with Step 2 UX is a defect. |
| 10 | Country field is a free-text input, NOT the same searchable country picker used in Step 3 | ❌ FAIL | edit_profile_page.dart:255–261 — country is a plain `TextFormField`. Spec says user can update residence country; using Step 3's country_picker would be correct. This inconsistency means a user could enter an invalid country string. |

---

## Feature: Gift Collections — My Collections

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | My Collections screen lists all named collections with mosaic thumbnail and gift count | ✅ PASS | my_collections_page.dart:244–309 — `_buildCollectionCard` with mosaic and gift count |
| 2 | Tapping a collection opens the Wish List screen filtered to that collection | ✅ PASS | my_collections_page.dart:253–255 — `WishListsPage(initialCategory: collection.name)` |
| 3 | "New Collection" button opens dialog; confirming creates collection and opens it immediately | ✅ PASS | my_collections_page.dart:49–65 — creates via `_giftCollectionService.createCollection`, then pushes WishListsPage |
| 4 | Each collection card shows link icon that copies public URL and shows confirmation snackbar | ✅ PASS | my_collections_page.dart:295–305 — `Clipboard.setData`, snackbar shown |

---

## Feature: Gift Collections — Wish List (Personal Storefront)

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Wish List screen displays saved products in two-column editorial grid | ✅ PASS | wish_lists_page.dart:247–258 — SliverGrid, crossAxisCount: 2 |
| 2 | Horizontal category bar with "All", collection chips, and "+" to create | ✅ PASS | wish_lists_page.dart:437–473 — category bar with `_showCreateCategoryDialog` on "+" |
| 3 | Each product card shows image, name, price, and "See This Gift" button opening affiliate link | ✅ PASS | wish_lists_page.dart:370–434 — `_buildEditorialCard` |
| 4 | Own-wishlist view: lock toggle switches product between public/private | ✅ PASS | wish_lists_page.dart:386–404 — lock icon toggles `isPrivate` |
| 5 | Own-wishlist view: folder icon lets user move product to different collection | ✅ PASS | wish_lists_page.dart:405–415 — `_moveProductToCategory` bottom sheet |
| 6 | Publishing banner shows public collection URL with "Copy Link" button | ✅ PASS | wish_lists_page.dart:272–325 — `_buildPublishingBanner` |
| 7 | Share icon in top bar opens system share sheet with public URL | ✅ PASS | wish_lists_page.dart:341 — share icon; `_shareWishlist` calls `Share.share` |
| 8 | Optional banner image shown in header; owner can replace from gallery (uploads to Firebase Storage) | ✅ PASS | wish_lists_page.dart:179–196 — `_updateBannerImage` + `_storageService.uploadWishlistBanner` |
| 9 | Products saved from Discover/Search added to General (default) collection unless specified | ❌ FAIL | No evidence of "General" as the default. `wishlist_repository_impl.dart` / `wishlist_service.dart` would need inspection, but in search_page.dart the `addToWishlist` call at line 161 passes only `product.toMap()` with no explicit `category` field. Whether the wishlist service defaults to "General" is not confirmed from the code read. |

---

## Feature: Public Storefront (Unauthenticated Browsing)

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | `/u/<username>` displays public overview of all non-private collections with mosaic and gift count | ✅ PASS | public_wishlist_page.dart:64–97 (`_OverviewPage`) — builds `byCollection` from public items only (`:48` filters `isPrivate == false`) |
| 2 | Tapping collection tile navigates to `/u/<username>/<slug>` showing products in two-column grid | ✅ PASS | public_wishlist_page.dart:149 — `context.push('/u/$username/${_toSlug(entry.key)}')` |
| 3 | Header on both pages shows username and optional banner image | ✅ PASS | public_wishlist_page.dart:162–219 (overview header), :327–380 (collection header) |
| 4 | "Want This" heart: authenticated adds to wishlist; unauthenticated sees snackbar prompting to join | ✅ PASS | public_wishlist_page.dart:446–462 — `userId == null` → snackbar; else `addToWishlist` |
| 5 | Private products (`isPrivate = true`) never shown on public storefront | ✅ PASS | public_wishlist_page.dart:48 — `publicItems = allItems.where((item) => !(item['isPrivate'] ?? false))` |
| 6 | Footer on both pages contains "Join WiseGift" CTA | ✅ PASS | public_wishlist_page.dart:221–253 (overview footer), :411–441 (collection footer) |
| 7 | If username does not exist, page shows "Collection Not Found" | ✅ PASS | public_wishlist_page.dart:41 — `_errorState(context, 'COLLECTION NOT FOUND')` when no data |

---

## Feature: User Profile (Viewing Others)

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Shows @username, follower count, birthday, country, gender, interests | ✅ PASS | profile_body.dart:48–85 (header with username/follower count), :152–195 (info card with birthday/country/gender/interests) |
| 2 | "Storefront" tile links to that user's public wish list screen | ✅ PASS | profile_body.dart:88–128 — `WishListsPage(userId: userData['id'], userName: username)` |
| 3 | Logged-in users can access moderation menu (three-dot) with Report and Block | ✅ PASS | user_profile_page.dart:120–126 — `IconButton` for `_showModerationOptions` shown when `_currentUserId != null` |
| 4 | Report: text field dialog; on confirmation submits report and shows snackbar | ✅ PASS | user_profile_page.dart:60–87 |
| 5 | Block: confirmation dialog; on confirmation blocks and navigates back | ✅ PASS | user_profile_page.dart:90–110 |
| 6 | Moderation menu not shown when viewing own profile | ❌ FAIL | user_profile_page.dart:121 — the guard is `_currentUserId != null`, NOT `_currentUserId != widget.userId`. If a user navigates to their own UserProfilePage (e.g., via a guest-list tap), the three-dot moderation menu will appear on their own profile. |

---

## Feature: Notifications

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Notification bell shows real-time unread count badge | ✅ PASS | home_page.dart:179–192 — StreamBuilder on `getUnreadCount`; Badge widget |
| 2 | Notifications screen lists all notifications in reverse-chronological order with icon, title, body, timestamp | ✅ PASS | notifications_page.dart:48–110 — `orderBy('createdAt', descending: true)` in repo; rendered with icon, title, body, DateFormat timestamp |
| 3 | `newFollower` shows blue icon; `eventInvite` shows orange icon; others show generic | ✅ PASS | notifications_page.dart:117–136 |
| 4 | Unread notifications visually distinguished (border colour and background tint) | ✅ PASS | notifications_page.dart:63–71 — border width, colour, and card background differ for `isRead` state |
| 5 | Tapping any notification marks it as read | ✅ PASS | notifications_page.dart:95–97 — `markAsRead` called when `!isRead` |
| 6 | Tapping `eventInvite` also navigates to Occasion Detail screen | ✅ PASS | notifications_page.dart:99–107 |
| 7 | If no notifications, empty-state illustration and message shown | ✅ PASS | notifications_page.dart:34–44 — icon and "No notifications yet" text |

---

## Feature: Preferences

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Preferences screen accessible from "Settings" section on Profile tab | ✅ PASS | my_account_page.dart:230–237 — navigates to PreferencesPage |
| 2 | Language selector: English and Spanish; changing applies locale immediately without restart | ✅ PASS | preferences_page.dart:107–127 — `MyApp.setLocale(context, Locale(value))` on change |
| 3 | Reminder lead time: slider 1–30 days; current value shown next to slider | ✅ PASS | preferences_page.dart:140–165 — slider min:1 max:30; value shown in Row label |
| 4 | Per-relationship budget sliders: Partner, Close Family, Close Friend, Other; each €10–€500 | ✅ PASS | preferences_page.dart:92, 168–200 — iterates `RelationshipType.values`; min:10 max:500 |
| 5 | "Save Preferences" or checkmark persists to Firestore and closes with success snackbar | ✅ PASS | preferences_page.dart:34–57 — `updateUserProfile`, snackbar, `Navigator.pop` |
| 6 | On error, snackbar describes failure and screen remains open | ✅ PASS | preferences_page.dart:50–54 |

---

## Feature: Social Graph — Contacts and Following

| # | Criterion | Status | Notes |
|---|---|---|---|
| 1 | Firestore social graph stores follows under `followers/{uid}/following/{targetUid}` | ❌ FAIL | user_repository_impl.dart:74–78 — the actual Firestore paths used are `users/{uid}/following/{targetId}` and `users/{uid}/followers/{targetId}`. The spec describes `followers/{uid}/following/{targetUid}` as a top-level collection, but the implementation uses a sub-collection under `users`. This is a data schema discrepancy. |
| 2 | Follower count surfaced on public profile page | ✅ PASS | profile_body.dart:51 — `followerCount` from Firestore data displayed |
| 3 | `newFollower` notification generated when someone starts following | ⚠️ UNTESTABLE | `NotificationType.newFollower` exists (notification_type.dart:2); notifications_page.dart handles it. However no explicit "follow" write path was found in the client code — the follow relationship appears to be managed only via block cleanup. Cannot confirm from code alone that a `newFollower` notification is auto-generated on follow. |

---

## Bugs Found

- **[BUG-1]** Unauthenticated wishlist toggle (Search/Discover tab): The heart/wishlist button is rendered and tappable for guest users in `ProductCard` but silently no-ops (`_currentUserId == null` check is in `search_page.dart:150–152` not in the widget itself). Spec requires guests cannot save, and implies a clear affordance. — `lib/presentation/screens/search/search_page.dart:150` / `lib/presentation/widgets/search/product_card.dart:59`

- **[BUG-2]** Birthday entry deletion skips confirmation dialog: `contact_list_screen.dart:228–231` — `_deleteBirthday` is called immediately on icon tap. Spec requires confirmation. — `lib/presentation/screens/myaccount/contact_list_screen.dart:228`

- **[BUG-3]** Moderation menu shown on own profile: `user_profile_page.dart:121` — guard is `_currentUserId != null` but should also check `_currentUserId != widget.userId`. A logged-in user navigating to their own `UserProfilePage` will see the Report/Block menu. — `lib/presentation/screens/user_profile/user_profile_page.dart:121`

- **[BUG-4]** Edit Profile — birthday uses free-text field, not a date picker: `edit_profile_page.dart:246–254`. Registration step 2 correctly uses `showDatePicker`; the edit screen does not. Inconsistent UX and allows entry of invalid date strings. — `lib/presentation/screens/myaccount/edit_profile_page.dart:246`

- **[BUG-5]** Edit Profile — country uses free-text field, not country picker: `edit_profile_page.dart:255–261`. Registration step 3 uses the `country_picker` package. Edit Profile accepts any raw string, allowing invalid or non-standard country codes/names to be stored. — `lib/presentation/screens/myaccount/edit_profile_page.dart:255`

- **[BUG-6]** Social graph Firestore path mismatch: Spec specifies `followers/{uid}/following/{targetUid}` (top-level `followers` collection) but implementation uses `users/{uid}/following/{targetId}` subcollection. — `lib/data/repositories/user_repository_impl.dart:74`

---

## Recommendations

Priority order for resolution before launch:

1. **[HIGH] BUG-3 — Own-profile moderation menu**: One-line fix; `_currentUserId != null && _currentUserId != widget.userId`. Security/trust risk if users can report/block themselves.

2. **[HIGH] BUG-1 — Guest wishlist toggle**: Guest users should either see the toggle hidden or receive a prompt to sign in (as the curated-collections save gate already does). Silent no-op is poor UX and inconsistent with the spec.

3. **[HIGH] BUG-2 — Birthday delete confirmation**: Spec explicitly requires confirmation before deletion. Easy to add an `AlertDialog` gate identical to logout/delete-account dialogs.

4. **[MEDIUM] BUG-4 / BUG-5 — Edit Profile date/country inputs**: Replace the plain `TextFormField` for birthday with `showDatePicker` (matching Step 2) and the country field with the `country_picker` package (matching Step 3). Without this, users can corrupt their own profile data after initial registration.

5. **[MEDIUM] BUG-6 — Social graph Firestore schema**: Align the implementation path with the spec or update the spec to reflect the actual `users/{uid}/following/{targetId}` path. Mismatches between spec and implementation are a maintenance and onboarding risk.

6. **[LOW] Follow/Unfollow UI gap (Open Question #1)**: No explicit Follow button was found in any screen. The `followerCount` is displayed and `newFollower` notification type exists, but the UI action to follow a user is missing. Requires product owner decision on where to surface this.

7. **[LOW] Hardcoded `residenceCountry: 'ES'` in Gift Agent** (Open Question #2): `gift_agent_page.dart:84` and `calendar_view.dart:55` both hardcode Spain. Logged-in users' actual country is ignored. This limits recommendation accuracy for all non-Spain users.
