# ON-1489: First Issues (sorted by complexity)

**Onboarding task:** [ON-1489](https://bitwarden.atlassian.net/browse/ON-1489), "Talk to your buddy about first issues"
**Buddy:** Vincent Salucci
**Expectation:** by the end of week 2, have at least a branch and a commit pushed publicly.
**Repo:** `bitwarden/clients`
**All work lives in:** `bitwarden_license/bit-web/src/app/admin-console/providers/`
**Team:** Admin Console. **Labels:** `ac-tech-debt`, `platform-initiative`. **Component:** Web

> Status as of 2026-10-06: all 7 tickets are **To Do** and assigned to Vitul Chauhan.
> On `main`, nothing has started yet: every target file still has `// @ts-strict-ignore` or `standalone: false`.

---

## The two kinds of work

### Type A: TypeScript strict mode (3 tickets)

Some files start with `// @ts-strict-ignore`, which turns off TypeScript's strict null checks for that file.
The job:

1. Delete that comment.
2. Fix the type errors that appear. These are mostly values that could be `null` or `undefined`. Typical fixes: optional fields (`foo?: X`), optional chaining (`a?.b`), defaults (`x ?? 0`), explicit null checks, `assertNonNullish(...)`.
3. **Don't change behavior.** The app must work exactly as before.

### Type B: Convert to standalone components (4 tickets, epic [PM-43110](https://bitwarden.atlassian.net/browse/PM-43110))

These components are currently declared inside `ProvidersModule` (`providers.module.ts`). The job, for each component:

1. Change `standalone: false` to `true` in the `@Component` decorator.
2. Add everything its template uses (modules, pipes, child components) to the component's own `imports: []`.
3. Remove the component from `ProvidersModule.declarations`.
4. Check that the page or dialog still renders and works.

Standard acceptance criteria for every Type B ticket: **lint, type-check and tests pass**.

---

## Tickets ranked from easiest to hardest

| Rank | Ticket | Type | Files (lines of TS / HTML) | Tests? | Complexity |
|---|---|---|---|---|---|
| 1 | PM-44178 | Strict | 2 files: 151 (spec) + 112 | ✅ both have specs | ⭐ Very low |
| 2 | PM-43890 | Standalone | 2 files: 163/41 + 36/29 | ❌ manual | ⭐ Low |
| 3 | PM-43891 | Standalone | 2 files: 153/42 + 60/15 | ❌ manual | ⭐ Low |
| 4 | PM-44179 | Strict | 2 files: 127/63 + 153/42 | ❌ probably none | ⭐⭐ Medium |
| 5 | PM-43892 | Standalone | 3 files: 384/231 + 79/39 + 147/72 | ❌ manual | ⭐⭐ Medium |
| 6 | PM-43889 | Standalone | 4 dialogs: 98/73 + 263/100 + 90/22 + 222/60 | ❌ manual | ⭐⭐⭐ Medium-high |
| 7 | PM-44180 | Strict | 1 file: 222/60 (18+ errors) | ⚠️ ticket says to run jest, but no spec found under `clients/` | ⭐⭐⭐ High |

How I ranked them: the number of files and lines, how much of the typing actually changes, whether automated tests exist or you have to test by hand, and the risk of breaking something at runtime.

---

## 1. PM-44178: guard spec + shared server calls, strict mode ⭐

**Link:** https://bitwarden.atlassian.net/browse/PM-44178
**Reporter:** Brandon Treston. The ticket itself says: *"Good first ticket."*

**Files**
- `guards/provider-permissions.guard.spec.ts` (test only, 151 lines)
- `services/web-provider.service.ts` (112 lines)

**What's asked**
- **Guard spec:** remove the marker and add `state = mock<RouterStateSnapshot>()` in `beforeEach`.
- **web-provider.service.ts:** remove the marker and call `assertNonNullish(...)` on the `.encryptedString` results before they go into request objects whose fields can't be null.

**Note:** the ticket says "the three `.encryptedString` results" but lists **four**. The code has four uses (lines 54, 96, 97, 98): `encryptedOrgKey`, `encryptedProviderKey`, `encryptedPrivateKey`, `encryptedCollectionName`. Handle all four.

**Verify:** `npx jest --testPathPatterns="providers/(guards|services)"`

**Why it's easiest:** few changes, one file is test-only, and both files have specs, so you get automatic feedback.

---

## 2. PM-43890: provider setup components to standalone ⭐

**Link:** https://bitwarden.atlassian.net/browse/PM-43890
**Reporter:** Jared Scarito. **Epic:** PM-43110

**Files**
- `setup/setup.component.ts` (163 / 41 html)
- `setup/setup-provider.component.ts` (36 / 29 html)

**What's asked:** set both to standalone, move their template dependencies into `imports: []`, and remove both from `ProvidersModule.declarations`.

**Watch out:** `SetupBusinessUnitComponent` (under `billing/providers/setup/`) belongs to the **Billing team**. It must **stay** declared in `ProvidersModule`. Don't touch it.

**Acceptance criteria**
- Both components are standalone.
- Neither one is in `ProvidersModule.declarations`.
- The provider setup flow shows the invite-acceptance screen and the setup form.
- Lint, type-check and tests pass.

---

## 3. PM-43891: AccountComponent + VerifyRecoverDeleteProviderComponent to standalone ⭐

**Link:** https://bitwarden.atlassian.net/browse/PM-43891
**Reporter:** Jared Scarito. **Epic:** PM-43110. **Story points:** 1. **Sprint:** AC 2026.20

**Files**
- `settings/account.component.ts` (153 / 42 html)
- `verify-recover-delete-provider.component.ts` (60 / 15 html)

**What's asked:** set both to standalone, move their template dependencies into `imports: []`, and remove both from `ProvidersModule.declarations`.

**Acceptance criteria**
- Both components are standalone.
- Neither one is in `ProvidersModule.declarations`.
- Provider Settings → Account renders and saves, and the recover-delete-provider verification link renders.
- Lint, type-check and tests pass.

⚠️ **Overlaps with PM-44179:** both change `settings/account.component.ts`.

---

## 4. PM-44179: providers-layout + settings/account, strict mode ⭐⭐

**Link:** https://bitwarden.atlassian.net/browse/PM-44179
**Reporter:** Brandon Treston

**Files**
- `providers-layout.component.ts` (127 / 63 html)
- `settings/account.component.ts` (153 / 42 html)

**What's asked**

*providers-layout.component.ts*
- Make `provider$`, `logo$`, `canAccessBilling$`, `clientsTranslationKey$`, `subscriber$` and `getTaxIdWarning$` optional.
- Add `filter((p): p is Provider => p != null)` after `providerService.get$` so TypeScript knows `p` isn't null.
- Change the return type of `getTaxIdWarning$` to `Observable<TaxIdWarningType | null>`, to match `ProviderWarningsService.getTaxIdWarning$`.
- Save `this.provider$` in a local variable before the `getTaxIdWarning$` closure uses it.

*settings/account.component.ts*
- Make `provider` and `providerId` optional.
- Delete the unused field `taxFormPromise: Promise<any>`.
- Use `route.parent?.parent?.params`.
- In `submit()` and `deleteProvider()`, check that the id exists before using it.
- Default form values with `?? ""`.
- Change `title: null` to `undefined`.
- Wrap errors with `String(e)` in log calls.

**Verify:** `npx jest --testPathPatterns="providers/(settings|providers-layout)"`. This may find no tests, so test it manually too.

**Why it's medium:** two files, and RxJS type narrowing and Observable typing are involved.

⚠️ **Overlaps with PM-43891** (same `account.component.ts`).

---

## 5. PM-43892: Members, AcceptProvider, AddEditMemberDialog to standalone ⭐⭐

**Link:** https://bitwarden.atlassian.net/browse/PM-43892
**Reporter:** Jared Scarito. **Epic:** PM-43110. **Story points:** 1. **Sprint:** AC 2026.20

**Files**
- `manage/members.component.ts` (**384 / 231 html**, the largest template in this set)
- `manage/accept-provider.component.ts` (79 / 39 html)
- `manage/dialogs/add-edit-member-dialog.component.ts` (147 / 72 html)

**What's asked:** set all 3 to standalone, move their template dependencies into `imports: []`, and remove all 3 from `ProvidersModule.declarations`.

**Watch out:** **don't touch** `ProvidersModule.providers`. `MemberActionsService` and `MemberDialogManagerService` must keep coming from there, or from a root-level provider if the org team's "Ticket 10" ships first (ask your buddy about that one).

**Acceptance criteria**
- All 3 components are standalone.
- None are in `ProvidersModule.declarations`.
- The provider members list loads, the add/edit member dialog opens and saves, and the accept-invite flow renders.
- **No `NullInjectorError`** for `MemberActionsService` or `MemberDialogManagerService`.
- Lint, type-check and tests pass.

**Why it's medium:** the large members template means lots of imports to track down, plus the dependency-injection risk above.

---

## 6. PM-43889: provider client dialogs to standalone ⭐⭐⭐

**Link:** https://bitwarden.atlassian.net/browse/PM-43889
**Reporter:** Jared Scarito. **Epic:** PM-43110

**Files** (all under `clients/`)
- `add-existing-organization-dialog.component.ts` (98 / 73 html)
- `create-client-dialog.component.ts` (263 / 100 html)
- `manage-client-name-dialog.component.ts` (90 / 22 html)
- `manage-client-subscription-dialog.component.ts` (222 / 60 html)

**What's asked:** set all 4 to standalone, move their template dependencies into each component's `imports: []`, and remove all 4 from `ProvidersModule.declarations`.

**Acceptance criteria**
- All 4 dialogs are standalone.
- None are in `ProvidersModule.declarations`.
- On the provider Clients page, the add-existing-org, create-client, manage-name and manage-subscription dialogs all open and submit.
- Lint, type-check and any provider client tests pass.

**Why it's harder:** 4 components, and you have to click through 4 dialogs by hand.

⚠️ **Overlaps with PM-44180** (same `manage-client-subscription-dialog.component.ts`).

---

## 7. PM-44180: manage-client-subscription-dialog, strict mode ⭐⭐⭐

**Link:** https://bitwarden.atlassian.net/browse/PM-44180
**Reporter:** Brandon Treston. The ticket calls it *"the biggest single provider fix"*: 18+ errors appear once the marker is removed.

**File:** `clients/manage-client-subscription-dialog.component.ts` (222 / 60 html)

**What's asked**
- Remove the marker.
- Change to `providerPlan?: ProviderPlanResponse` (it loads asynchronously). Set the seat counters (`assignedSeats`, `openSeats`, `purchasedSeats`, `seatMinimum`) to `0` where they're declared.
- Use `new FormControl<number>(seats ?? 0, { nonNullable: true, validators: [...] })` instead of the untyped two-argument form.
- In `ngOnInit`, stop early if `this.providerPlan == null` before using it.
- Validators should take `(control: AbstractControl)` so they match `ValidatorFn`. Default `control.value ?? 0` and `dialogParams.organization.seats ?? 0` where they're read.
- In `submit`, use `formGroup.value.assignedSeats ?? 0` and `dialogParams.organization.organizationName ?? ""`.
- In `isServiceUserWithPurchasedSeats`, check `this.providerPlan != null` explicitly.

**Verify:** `npx jest --testPathPatterns="providers/clients"`. Then **manually** open a client subscription dialog as a Service User and confirm the validation behaves exactly as before.

**Why it's hardest:** the most type errors, typed forms and validator signatures, seat-calculation logic that affects billing, and testing that is mostly manual.

---

## Overlapping files: plan the order

| File | Tickets |
|---|---|
| `settings/account.component.ts` | PM-44179 (strict) + PM-43891 (standalone) |
| `clients/manage-client-subscription-dialog.component.ts` | PM-44180 (strict) + PM-43889 (standalone) |

Doing both tickets of a pair as parallel PRs will cause merge conflicts. Either do them one after the other (merge the first before starting the second) or ask Vincent whether to combine them.

---

## Commands to run before opening a PR (from the repo `CLAUDE.md`)

```bash
npm run lint:fix
npm run prettier
npm run test:types
npm test -- bitwarden_license/bit-web/src/app/admin-console/providers
```

## Suggested order

1. **PM-44178**: first PR, small and safe, and it has tests.
2. **PM-43890** or **PM-43891**: learn the standalone pattern on small components.
3. **PM-44179**: then PM-43891 if it isn't done yet (shared file).
4. **PM-43892**
5. **PM-43889**: then PM-44180 (shared file), or the other way round.
6. **PM-44180**

## Open questions for your buddy

- What is the org team's "Ticket 10" that PM-43892 mentions, and has it shipped?
- Should the overlapping pairs be separate PRs or combined?
- How do I get a local Provider / Service User account for manual testing?
