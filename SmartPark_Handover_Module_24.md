# SmartPark Handover - Module 24 (Security & Validation Audit)

Baseline: SmartPark_Module_23_DemoMode.zip. Verification: STATIC ONLY (bracket balance with comments/strings/char-literals stripped, package-vs-directory check, import resolution for every `com.smartpark.*` import, grep of every changed signature's call sites). Kotlin not compiled, no Gradle build, no emulator. No schema change (DB version 1), no new DAO queries, no Gradle dependency changes (versionName only).

## Audit result: critical business rules

| # | Rule | Enforced in | Result |
|---|------|-------------|--------|
| 1 | Vehicle number unique | `VehicleRepositoryImpl.addVehicle/updateVehicle` (now one `withTransaction`; normalized plate; add forces a new row id) | GAP FIXED. Was an unguarded check-then-insert. There is still no unique index (schema change deferred), so the transaction is the only guard |
| 2 | One ACTIVE session per vehicle | `ParkingRepositoryImpl.allocateAndStartSession` (`findActiveByVehicle`, inside the transaction) | OK |
| 3 | RESERVED/OCCUPIED slots not re-allocated | `allocateAndStartSession` takes only AVAILABLE slots or the vehicle's own RESERVED slot (`claimReservedSlot`); `ReservationRepositoryImpl.reserveSlot` needs AVAILABLE and no active reservation | OK. Reserve additionally hardened (see below) |
| 4 | Balance never negative | `WalletRepositoryImpl.debitForParking` (`balance < fee` rejected), `recharge` validated by `WalletRules` | OK |
| 5 | Every balance change has a ledger row | `recharge` and `debitForParking` write the TransactionEntity in the same transaction | OK. Only exception: Demo reset sets balance to 0 and deletes that wallet's ledger rows in the same guarded transaction |
| 6 | Failed payment never completes a session / releases a slot | `completeSessionWithPayment`: debit result checked before anything else is written | OK. GAP FIXED: removed the unused public `completeSessionAndReleaseSlot` (completed a session with NO payment) |
| 7 | No duplicate / partial payment-completion-release | One outer transaction; session status check; `countForSession` guard | OK |
| 8 | Compliance derived from stored dates | `ComplianceRules`, `refreshCompliance` | OK. GAP FIXED: entry gate used the caller's `VehicleEntity` copy; it now re-reads the vehicle. Add/Edit vehicle silently replaced an unparsable date with "today"; now rejected (strict calendar parsing) |
| 9 | Timers from persisted entry time | `ActiveSessionScreen` derives elapsed/fee from `session.entryTime` | OK |
| 10 | Fee / reservation rules preserved | `ParkingFeeCalculator`, `PricingValidator` (min Rs 1, so no zero payments) unchanged | OK |

## Access-control gaps fixed
- Reservation cancel had no ownership check (open item from M23): `cancelReservation(reservationId, requesterUserId)` now checks inside the transaction; the failure is shown as a snackbar.
- `reserveSlot` accepted any vehicleId: now requires the vehicle to exist, belong to the user, be compatible with the slot's vehicle type, and not be parked.
- Vehicle update/delete had no ownership check and update could move a vehicle to another user: use cases take the signed-in user (from SessionManager via VehicleViewModel); the repository keeps the stored owner. VehicleViewModel.getVehicle/getCompliance return null for vehicles the signed-in user does not own (Vehicle Detail shows "Vehicle not found").
- Entry: `allocateAndStartSession(vehicleId)` reads the vehicle type from the database (parameter removed). `completeSessionWithPayment` verifies the payer owns the session's vehicle.
- Admin screens' denied states (open item from M23): new `AdminGate` (presentation/common/AccessGate.kt) wraps all six admin routes in the NavHost; role is re-read from the stored user, fails closed, denied screen has "Back to sign in". Admin use cases already re-check the role (`AdminAccessChecker` in Recheck/Add/Remove slot, UpdatePricing, GenerateReport, ObserveAnalytics).
- `UserRepositoryImpl.register` now refuses any role except USER and runs check+insert in one transaction (closes the M23 "register role argument" note). Email format and name/password length checked in `RegisterUserUseCase`.
- Splash restores a session only if the account still exists, and takes the role from the database.
- Removed unused rule-bypassing mutators from `ParkingRepository`: `updateSlot`, `addSlot`, `removeSlot`, `createSession`, `updateSession`.
- `VehicleRules` (new) validates plate, owner name and type in the add/update use cases.
- Manifest: `allowBackup="false"` (the database holds password hashes and the ledger).

## Changed files
New: presentation/common/AccessGate.kt, domain/usecase/vehicle/VehicleRules.kt
Changed: VehicleRepositoryImpl, UserRepositoryImpl, ParkingRepositoryImpl, ReservationRepositoryImpl, ParkingRepository, ReservationRepository, AddVehicleUseCase, UpdateVehicleUseCase (+requesterUserId), DeleteVehicleUseCase (+requesterUserId), CancelReservationUseCase (+requesterUserId), StartParkingSessionUseCase (re-reads vehicle; injects VehicleRepository), RegisterUserUseCase, SplashViewModel, VehicleViewModel, ReservationViewModel, ReservationListScreen, AddEditVehicleScreen, VehicleDetailScreen, SmartParkNavHost, AndroidManifest.xml, app/build.gradle.kts (versionName)

## Accepted / open risks (not fixed, by design or by scope)
1. No unique indexes on vehicles.vehicleNumber, users.email, wallets.userId. Enforced by transactions only; an index needs a migration (deferred, per the plan).
2. Passwords: SHA-256 + static salt, no login throttling, known demo password, admin credentials shown on the login screen (prototype).
3. User screens take `userId` from the nav argument (Wallet, History, Notifications, Dashboard, Scanner). The app declares no deep links, so it cannot be altered from outside, and data queries filter by it; session/receipt/exit/entry/vehicle/reservation actions are additionally checked in the domain layer. Wallet recharge relies on the nav argument only.
4. `DeleteVehicleUseCase` checks active session/reservation and then deletes in separate steps (tiny race; single-device app).
5. Reserve screen lists all AVAILABLE slots; an incompatible type is rejected with a message rather than hidden.
6. Demo data still mixes into admin totals until deleted (M23 trade-off).

## Unverified until first real build
Everything from M23's list, plus: `Result.failure<T>` inference inside the new if/else expressions (VehicleViewModel), `AdminGate` with `hiltViewModel()` default parameter followed by a trailing lambda, `SimpleDateFormat.isLenient = false` strictness on the emulator's locale, Hilt graph with the new constructor parameters (ParkingRepositoryImpl +VehicleDao, ReservationRepositoryImpl +VehicleDao/+ParkingSessionDao, UserRepositoryImpl +db, StartParkingSessionUseCase +VehicleRepository, SplashViewModel +UserRepository, VehicleViewModel +SessionManager).

## Manual regression checklist (new behaviour)
1. Register with `abc@x` -> "Enter a valid email". Register `demo.user@smartpark.demo` -> refused. Normal register still lands on the dashboard.
2. Add vehicle with plate `AB1` -> rejected; with date `2026-02-31` or `abc` -> date error, nothing saved; valid vehicle saves. Add the same plate twice -> "already registered".
3. Edit a vehicle: change owner name/type/dates -> saved, compliance refreshed. Delete a parked vehicle -> blocked with message.
4. Reserve a free slot -> OK; reserve a second slot for the same vehicle -> rejected; reserve while that vehicle is parked -> rejected. Cancel -> slot AVAILABLE again.
5. Entry with an expired-insurance vehicle (Demo plate KA01DM0002) -> denied; edit its date to the future -> entry approved.
6. Exit with Rs 0 wallet -> insufficient balance, session stays ACTIVE and slot OCCUPIED; recharge -> exit succeeds, one PARKING_PAYMENT ledger row, receipt opens.
7. Sign in as admin: all six admin screens open. Kill the app and relaunch -> returns to the right dashboard. Demo: Start, walk through, Reset, Delete -> still works, then Login.
8. Regression: Login, vehicle CRUD, parking map, scanner -> entry, live timer, wallet recharge, history/receipt, notifications, analytics, reports export.

## Next action
Module 25: UI/UX Polish - empty/loading/error states and accessibility across screens. No new features, no business-rule changes. First real Gradle sync/build should happen before or at the start of Module 25 so the unverified items above are confirmed.
