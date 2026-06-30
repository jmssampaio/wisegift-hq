# QA Re-validation Report
**Date:** 2026-06-30
**Scope:** Bug fixes from qa-validation.md

## Summary
| Bug | Status | Verdict |
|---|---|---|
| BUG-1 | FIXED | Snackbar shown when guest tries to toggle wishlist |
| BUG-2 (main) | FIXED | Delete now goes through AlertDialog; only proceeds on confirm == true |
| BUG-2 (worktree) | FIXED | Same AlertDialog pattern confirmed |
| BUG-3 | FIXED | Guard is `_currentUserId != null && _currentUserId != widget.userId` |
| BUG-4 | FIXED | No TextFormField for birthday; InkWell + InputDecorator calls `_pickBirthDate` via `showDatePicker` |
| BUG-5 (main) | FIXED | country_picker imported, InkWell calls `_pickCountry`, code stored in Firestore, initialised on load |
| BUG-5 (worktree) | FIXED | Same as main; identical pattern confirmed |
| BUG-6 | FIXED | No old `followers/{uid}/following` paths remain; docs now match implementation |

---

## Detail

### BUG-1 — Guest wishlist toggle
**File:** `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/.claude/worktrees/agent-a6df74a5a80ad3a86/lib/presentation/screens/search/search_page.dart`
**Verdict:** FIXED
**Evidence (lines 150-156):**
```dart
Future<void> _toggleWishlist(Product product) async {
  if (_currentUserId == null) {
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Sign in to save gifts to your wishlist')),
    );
    return;
  }
```
**Notes:** The snackbar message is clear and actionable. No dialog is shown — snackbar is acceptable and matches the claimed fix. The early return correctly prevents any wishlist write.

---

### BUG-2 — Birthday deleted without confirmation
**Files:**
- Main: `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/lib/presentation/screens/myaccount/contact_list_screen.dart`
- Worktree: `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/.claude/worktrees/agent-a6df74a5a80ad3a86/lib/presentation/screens/myaccount/contact_list_screen.dart`

**Verdict (main):** FIXED
**Evidence (main, lines 237-255):**
```dart
onPressed: () async {
  final confirm = await showDialog<bool>(
    context: context,
    builder: (ctx) => AlertDialog(
      title: const Text('Remove birthday'),
      content: Text('Remove ${data['name'] ?? 'this contact'} from your agenda?'),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(ctx, false),
          child: const Text('Cancel'),
        ),
        TextButton(
          onPressed: () => Navigator.pop(ctx, true),
          child: const Text('Remove', style: TextStyle(color: Colors.red)),
        ),
      ],
    ),
  );
  if (confirm == true) _deleteBirthday(data['id']);
},
```

**Verdict (worktree):** FIXED
**Evidence (worktree, lines 230-249):** Identical `showDialog<bool>` + `AlertDialog` pattern with the same Cancel/Remove actions and the same `if (confirm == true) _deleteBirthday(data['id'])` guard.

**Notes:** Both files are in sync. The delete action is fully gated behind user confirmation.

---

### BUG-3 — Moderation menu shown on own profile
**File:** `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/.claude/worktrees/agent-a6df74a5a80ad3a86/lib/presentation/screens/user_profile/user_profile_page.dart`
**Verdict:** FIXED
**Evidence (line 121):**
```dart
if (_currentUserId != null && _currentUserId != widget.userId)
  IconButton(
    icon: const Icon(Icons.more_vert),
    onPressed: () => _showModerationOptions(l10n),
  ),
```
**Notes:** The condition correctly excludes both guest users (`_currentUserId == null`) and the profile owner (`_currentUserId == widget.userId`). The moderation button will not appear when a signed-in user views their own profile.

---

### BUG-4 — Edit Profile birthday uses free-text field (worktree)
**File:** `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/.claude/worktrees/agent-a6df74a5a80ad3a86/lib/presentation/screens/myaccount/edit_profile_page.dart`
**Verdict:** FIXED
**Evidence — no TextFormField for birthday.** The birthday UI is (lines 347-365):
```dart
// Birthday — date picker
InkWell(
  onTap: _pickBirthDate,
  borderRadius: BorderRadius.circular(4),
  child: InputDecorator(
    decoration: InputDecoration(
      labelText: l10n.birthDateLabel,
      border: const OutlineInputBorder(),
      prefixIcon: const Icon(Icons.cake_outlined),
      suffixIcon: const Icon(Icons.calendar_today_outlined, size: 18),
    ),
    child: Text(
      _displayBirthDate ?? '',
      style: const TextStyle(fontSize: 15),
    ),
  ),
),
```
**Evidence — `_pickBirthDate` method (lines 61-86) calls `showDatePicker`.**
```dart
Future<void> _pickBirthDate() async {
  final now = DateTime.now();
  final initial = _selectedBirthDate ?? DateTime(now.year - 25);
  final picked = await showDatePicker(
    context: context,
    initialDate: initial,
    firstDate: DateTime(1900),
    lastDate: DateTime(now.year - 5),
    ...
  );
```
**Notes:** Free-text entry is fully replaced. The stored value is also initialised correctly from Firestore (lines 203-217).

---

### BUG-5 — Edit Profile country uses free-text field
**Files:**
- Main: `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/lib/presentation/screens/myaccount/edit_profile_page.dart`
- Worktree: `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/.claude/worktrees/agent-a6df74a5a80ad3a86/lib/presentation/screens/myaccount/edit_profile_page.dart`

**Verdict (main):** FIXED
**Verdict (worktree):** FIXED

**Checklist — both files:**

1. `country_picker` imported: YES — line 1 of both files:
   ```dart
   import 'package:country_picker/country_picker.dart';
   ```

2. No `TextFormField` for country: CONFIRMED — no `TextFormField` for the country field exists in either file.

3. `InkWell` calling `_pickCountry` / `showCountryPicker`: CONFIRMED.
   Main (lines 287-299) and worktree (lines 368-384):
   ```dart
   InkWell(
     onTap: _pickCountry,
     ...
   )
   ```
   `_pickCountry` in both files:
   ```dart
   void _pickCountry() {
     showCountryPicker(
       context: context,
       showPhoneCode: false,
       onSelect: (Country country) {
         setState(() {
           _selectedCountryCode = country.countryCode;
           _selectedCountryName = country.name;
         });
       },
     );
   }
   ```

4. Country code included in Firestore updates: CONFIRMED.
   Main `_updateProfile` (lines 119-121):
   ```dart
   if (_selectedCountryCode != null) {
     updates[AppUser.fieldResidenceCountry] = _selectedCountryCode;
   }
   ```
   Worktree (lines 146-148): identical pattern.

5. Country initialised from Firestore on load: CONFIRMED.
   Main (lines 188-200) and worktree (lines 219-233):
   ```dart
   final storedCode = userData[AppUser.fieldResidenceCountry] as String?;
   if (storedCode != null && storedCode.isNotEmpty && _selectedCountryCode == null) {
     _selectedCountryCode = storedCode;
     try {
       _selectedCountryName = CountryParser.parseCountryCode(storedCode).name;
     } catch (_) {
       _selectedCountryName = storedCode;
     }
   }
   ```

**Notes:** Both files are consistent. The `CountryParser.parseCountryCode` fallback (using the raw code if the parser throws) is a sensible defensive pattern.

---

### BUG-6 — Architecture.md documents wrong Firestore social graph paths
**File:** `/Users/joao.dasilva/Workspace/wisegift/wisegift-hq/docs/architecture.md`
**Implementation reference:** `/Users/joao.dasilva/Workspace/wisegift/wisegift_flutter/lib/data/repositories/user_repository_impl.dart`
**Verdict:** FIXED

**Evidence — no old `followers/{uid}/following` paths found.** A grep for `followers/{uid}/following` and `following/{targetUid}` returned zero matches.

**Documented paths in architecture.md (lines 81-82 and 163-164):**
```
`users/{uid}/following/{targetId}` — social graph: who this user follows
`users/{uid}/followers/{targetId}` — social graph: who follows this user
```

**Actual paths in `user_repository_impl.dart` (lines 77-81):**
```dart
batch.delete(_db.collection('users').doc(currentUserId).collection('following').doc(targetId));
batch.delete(_db.collection('users').doc(targetId).collection('followers').doc(currentUserId));
batch.delete(_db.collection('users').doc(currentUserId).collection('followers').doc(targetId));
batch.delete(_db.collection('users').doc(targetId).collection('following').doc(currentUserId));
```

The documented paths `users/{uid}/following/{targetId}` and `users/{uid}/followers/{targetId}` match the Firestore subcollection names `following` and `followers` used in the implementation.

**Notes:** The implementation does not have a dedicated `followUser` / `unfollowUser` method in `user_repository_impl.dart` — the `following`/`followers` subcollections are only written to indirectly via `blockUser` (which deletes edges). There is no `followUser` write path visible in this file. This is a pre-existing observation, not a regression from the current fix, and is outside the scope of BUG-6.

---

## Remaining issues

All six bugs reported in the previous QA pass are confirmed fixed. No regressions were introduced.

One pre-existing observation worth flagging to the product owner (not a regression):

- **Social graph write path absent:** `user_repository_impl.dart` contains no `followUser` or `unfollowUser` implementation — only deletion of follow edges inside `blockUser`. If follow/unfollow functionality is expected, it is not implemented in this repository file. This should be investigated separately to confirm whether it lives in another file or is missing entirely.
