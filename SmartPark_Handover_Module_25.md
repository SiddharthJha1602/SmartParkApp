# SmartPark Handover - Module 25 (UI/UX Polish)

Baseline: SmartPark_Module_24_SecurityReview.zip. Verification: STATIC ONLY (bracket balance, package/directory match, import resolution). Not compiled, no Gradle build, no emulator. No schema change, no business-logic change, no new dependencies (versionName only; `material-icons-extended` already present).

## Survey result
Most list/detail screens already had loading, empty and error states (History, Notifications, Wallet, Vehicles, Reservations, Analytics with Denied/Error, Pricing, Admin gate). Gaps found and fixed:

| Area | Gap | Fix |
|------|-----|-----|
| Login / Register | No keyboard types or IME actions, no show/hide password, content could clip with large fonts, spinner had no spoken label | New `common/PasswordField.kt` (toggle labelled "Show/Hide password", Done submits); email keyboard + Next; `singleLine`; screens scroll; spinner described "Signing in" / "Creating account"; submit guarded while loading |
| Parking map | Slot status shown by colour only; slots had no spoken description; touch target about 40dp; summary cards read as separate fragments | Slot chip shows "Free/Reserved/Occupied" text, spoken as "Slot A01, available. Double tap to reserve.", button role, min height 52dp; summary cards merged ("3 available") |
| Add/Edit vehicle | Plate and name fields had no capitalisation/IME config; date fields not numeric; spinner unlabelled | Characters/Words capitalisation, Next actions, numeric keyboard for dates, `singleLine`, spinner described "Saving" |
| Shared | `FullScreenLoading` was not full-size and unlabelled; errors not announced | `FullScreenLoading` fills the screen and is labelled "Loading"; `ErrorText` is a polite live region |

Left as they are on purpose: icons with `contentDescription = null` that sit beside a text label (Dashboard, Entry, Receipt, Reports, Scanner, Vehicle detail) are decorative, so hiding them from screen readers is correct.

## Changed files
New: presentation/common/PasswordField.kt
Changed: presentation/common/CommonComponents.kt, auth/LoginScreen.kt, auth/RegisterScreen.kt, parking/ParkingMapScreen.kt, vehicle/AddEditVehicleScreen.kt, app/build.gradle.kts (versionName)

## Known limits
1. Vehicle list and parking map start from an empty StateFlow, so the empty text/spinner can show for a moment before data arrives (no loading flag in those ViewModels; changing that is a logic change, deferred).
2. Not visually checked on a device: the 1.2 aspect slot chip with two text lines, dark theme contrast of `labelSmall` status text, font scale 200%.
3. Everything in the M23/M24 "unverified until first build" lists is still unverified. Run a real Gradle sync/build first.

## Manual checklist
1. Login: email field shows the @ keyboard; Next moves to password; the eye icon toggles visibility; Done on the keyboard logs in; wrong password shows the error and TalkBack announces it. Same for Register.
2. Set font size to the largest: Login, Register and Add Vehicle scroll with nothing cut off.
3. Parking map: every slot shows its state as a word; with TalkBack, available slots say "Double tap to reserve", occupied/reserved ones are read but not actionable; tapping an available slot still opens Reserve.
4. Add vehicle: plate types in capitals, date fields show the number keyboard, a bad date still shows the date error.
5. Regression: vehicle CRUD, reserve/cancel, entry -> session -> exit/pay -> receipt, wallet, admin screens, reports, demo mode.

## Next action
Module 26: Final Testing - run the Part K acceptance flow end to end (static trace plus the manual checklist), regression across all modules, fix only what it finds, produce `SmartPark_Module_Final.zip`. Needs a real build to be meaningful.
