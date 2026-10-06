# PM-44178 — Study of 3 past ts-strict PRs

Read before starting PM-44178. Each section covers **what the problem was → how they fixed it → code**, and the end maps it all onto PM-44178.

| Ticket | PR | Author | Merged | Size |
|---|---|---|---|---|
| PM-36404: org route guards | [#21914](https://github.com/bitwarden/clients/pull/21914) | Brandon Treston | 2026-07-17 | 6 files, +18 / −32 |
| PM-36409: catch-all simple files | [#22391](https://github.com/bitwarden/clients/pull/22391) | Jared | 2026-08-13 | 10 files, +69 / −67 |
| PM-42900: admin-console web sweep | [#23219](https://github.com/bitwarden/clients/pull/23219) | Brandon Treston | 2026-09-24 | 27 files, +1335 / −975 |

To view a diff yourself: `gh pr diff 21914 --repo bitwarden/clients`

---

## Background: what "ts-strict" means here

- The repo's tsconfig has `strict: true`. Files that weren't compliant got this header:
  ```ts
  // FIXME: Update this file to be type safe and remove this and next line
  // @ts-strict-ignore
  ```
- Removing those two lines turns on strict checking for that file. The errors that show up are almost always one of:
  1. **`strictNullChecks`**: a value typed `X | undefined` / `X | null` is passed where `X` is required.
  2. **`strictPropertyInitialization`**: a class field is declared but never assigned in the constructor.
  3. **Used before assigned**: a `let` variable is read before it's ever assigned.
  4. **Implicit `any`**: e.g. `[].concat(...)` infers `never[]`.
- The job: remove the header, then fix each error **without changing behaviour**.

---

## PR #21914: PM-36404, org route guards (closest to your ticket)

**Files:** `apps/web/src/app/admin-console/organizations/guards/`: `org-permissions.guard.ts` + spec, `is-enterprise-org.guard.ts` + spec, `is-paid-org.guard.ts` + spec.

PR description: "Fix type safety in org route guards". No review comments.

### Fix 1: Specs: just remove the marker

All three `.spec.ts` files had **only** the 2-line header removed and nothing else. They were already strict-clean.

> Correction to my earlier note: `org-permissions.guard.spec.ts` already had `state = mock<RouterStateSnapshot>()` in `beforeEach` **before** this PR (line 59 at the time). This PR didn't add it. The file is still the pattern to copy; it just isn't something this PR fixed.

### Fix 2: `userId` could be `undefined`

**Problem:** the old code pulled the userId out with `map((a) => a?.id)`, which is typed `UserId | undefined`, then passed it to `organizations$(userId)`, which needs a `UserId`. Under strict mode that's an error.

**Before:**
```ts
const userId = await firstValueFrom(accountService.activeAccount$.pipe(map((a) => a?.id)));
const org = await firstValueFrom(
  organizationService
    .organizations$(userId)
    .pipe(getOrganizationById(route.params.organizationId)),
);
```

**After:**
```ts
import { getUserId } from "@bitwarden/common/auth/services/account.service";
import { getById } from "@bitwarden/common/platform/misc";

const org = await firstValueFrom(
  accountService.activeAccount$.pipe(
    getUserId,                                                   // throws if no account → UserId (not undefined)
    switchMap((userId) => organizationService.organizations$(userId)),
    getById(route.params.organizationId),                        // replaces deprecated getOrganizationById
  ),
);
```

**Pattern:** use the `getUserId` operator instead of `map(a => a?.id)`. It narrows the type to `UserId`. `getById` is the newer helper replacing `getOrganizationById`.

### Fix 3: `title: null` in toasts

**Problem:** `ToastService.showToast({ title })`: `title` is optional (`string | undefined`), so `null` isn't allowed under strict.

**Fix:** delete the `title: null` line entirely.
```diff
 toastService.showToast({
   variant: "error",
-  title: null,
   message: i18nService.t("accessDenied"),
 });
```

---

## PR #22391: PM-36409, catch-all for simple files

**Files:** `org-redirect.guard.ts` + spec, `user-confirm.component.ts`, `organization-reporting-routing.module.ts`, `settings/account.component.ts`, `delete-organization-dialog.component.ts`, `two-factor-setup.component.ts`, `families-for-enterprise-setup.component.ts`, `bulk-collection-access.request.ts`, `organization-user-accept.request.ts`.

### Fix 1: Fields assigned later → `!` (definite assignment)

**Problem:** `strictPropertyInitialization`. The field is set in `ngOnInit` or a subscription, not the constructor.

```ts
org!: OrganizationResponse;
protected organizationId!: string;
protected publicKeyBuffer!: Uint8Array;
loaded!: boolean;
```

### Fix 2: Fields that really can be empty → `| undefined` or `?`

```ts
fingerprint: string | undefined;
formPromise?: Promise<any>;
resetPasswordKey?: string;           // request model, optional on the wire
```

### Fix 3: Arrays in request models → initialise to `[]`

```ts
export class BulkCollectionAccessRequest {
  collectionIds: string[] = [];
  users: SelectionReadOnlyRequest[] = [];
  groups: SelectionReadOnlyRequest[] = [];
}
```

### Fix 4: Form values are nullable → `?? default`

**Problem:** typed reactive form `.value.x` is `T | null | undefined`.
```ts
name: this.formGroup.value.orgName ?? undefined,
request.limitCollectionCreation = this.collectionManagementFormGroup.value.limitCollectionCreation ?? false;
```

### Fix 5: ⚠️ `encryptedString` — same problem as your ticket, different fix

**Problem:** `EncString.encryptedString` is optional, but `OrganizationKeysRequest` needs a `string`. This is exactly what you'll hit in `web-provider.service.ts`.

**What this PR did** (`settings/account.component.ts`):
```ts
const orgKeys = await this.legacyCompatKeyService.makeKeyPair(orgShareKey!);
request.keys = new OrganizationKeysRequest(orgKeys[0], orgKeys[1].encryptedString!);
```
It used the **non-null assertion `!`**, which is compile-time only. If the value were ever undefined, it would silently send `undefined` to the server.

**What your ticket asks for:** `assertNonNullish(...)`, a **runtime** check that throws a clear error. That's stricter and better. There's a precedent in the same Providers area:
`bitwarden_license/bit-web/src/app/admin-console/providers/manage/services/provider-actions/provider-actions.service.ts:77`
```ts
const key = await this.encryptService.encapsulateKeyUnsigned(providerKey, publicKey);
assertNonNullish(key.encryptedString, "No key was provided");
const request = new ProviderUserConfirmRequest(key.encryptedString);
```
→ **Follow the ticket (`assertNonNullish`), not #22391's `!`.**

### Fix 6: Behaviour fix uncovered by strict mode (with a new test)

`org-redirect.guard.ts`: the callback type was widened to `string | string[] | undefined`, and `undefined` now falls through to the default redirect. Before this, it crashed with `[...undefined]`.
```ts
if (redirectPath != null) {
  if (typeof redirectPath === "string") redirectPath = [redirectPath];
  return router.createUrlTree([state.url, ...redirectPath]);
}
```
A spec case was added for it: `"falls back to the admin console redirect when the redirect callback returns undefined"`.

### Fix 7: Method reference typing → arrow wrapper

```ts
postKey: (id: string, request: SecretVerificationRequest) =>
  this.organizationApiService.getOrCreateApiKey(id, request as OrganizationApiKeyRequest),
```

### Review feedback (from claude[bot]) — lessons

- **Delete dead fields instead of adding `!`.** `taxFormPromise`, `secret` and `formPromise` were never read or assigned. Adding `!` "asserts something that is never true." They were deleted.
- **Behaviour changes need a test.** The org-redirect fallback got a new spec because a reviewer asked for one.

---

## PR #23219: PM-42900, admin-console web sweep (current house style)

PR description: "Refactor for ts-strict, split AddEditGroupDialog into discrete add and edit dialogs to more closely match members add/edit dialogs."

⚠️ **Much bigger than your ticket.** Most of the +1335 lines are a **refactor** (the group dialog was split into `group-add-dialog`, `group-edit-dialog` and `group-add-edit.service`, and components moved to standalone/OnPush). Read it for *style*, not as a template for scope.

### Patterns worth knowing

**1. Avoid nullable class fields by making state reactive** (bulk-collections-dialog):
```ts
// before: protected organization: Organization;  protected loading = true;  (set in subscribe)
// after:
protected readonly organization$ = this.userId$.pipe(
  switchMap((userId) => this.organizationService.organizations$(userId)),
  getById(this.params.organizationId),
);
protected readonly loading$ = this.formData$.pipe(map(() => false), startWith(true));
```

**2. Explicit null checks instead of `!`:**
```ts
if (organization == null || !organization.useGroups) { return of([] as GroupView[]); }

readonly submit = async () => {
  const organization = await firstValueFrom(this.organization$);
  if (organization == null) { return; }
  ...
};
```

**3. `[].concat(...)` → spread** (`[]` infers `never[]` under strict):
```ts
return [...groups.map(mapGroupToAccessItemView), ...users.data.map(mapUserToAccessItemView)];
```

**4. Optional chaining + widening the parameter type:**
```ts
switchMap((org) => this.buildReports(org?.productTierType)),
private async buildReports(productType: ProductTierType | undefined) { ... }
```

**5. Interface fields made optional when they really are** (`add-edit-group-detail.ts`): `id?: string; externalId?: string;`

**6. Strings from search/forms:** `groupsFilter(v ?? "")`

**7. Spec cleanups while there:**
- `function createControl(input: string | null)`: widen test helper params instead of casting
- `beforeEach(async () => { await TestBed.configureTestingModule(...) })`: fixes a floating promise
- import from `@bitwarden/components` instead of `@bitwarden/components/src/...`

**8. Extras done in the same PR (out of scope for you):** `enum` → `Object.freeze({...} as const)`, OnPush, `inject()` instead of constructor DI, `*ngIf` → `@if`, NgModule declarations → standalone imports.

### Review feedback — lessons (13 review comments)

- **CRITICAL:** a component made standalone was missing `SearchModule` in `imports` → `<bit-search>` breaks at AOT.
- **CRITICAL:** `accessItems$ | async` is `null` on first render, and the child component didn't guard against it, so the dialog would throw on open.
- **IMPORTANT:** service methods returned fresh observables each call, so `shareReplay` didn't dedupe: 18 HTTP calls instead of 2.
- Takeaway: **big refactors bundled with ts-strict create real bugs.** Your ticket is intentionally small, so keep it that way.

---

## Applying this to PM-44178

### File 1: `bitwarden_license/bit-web/src/app/admin-console/providers/guards/provider-permissions.guard.spec.ts`

**Problem:** `let state: MockProxy<RouterStateSnapshot>;` (line 35) is declared but never assigned. Every test passes `undefined` as the guard's `state` arg. Strict mode reports "used before being assigned".

**Fix:** copy the org guard spec (`apps/web/.../organizations/guards/org-permissions.guard.spec.ts:57`):
```diff
-// FIXME: Update this file to be type safe and remove this and next line
-// @ts-strict-ignore
 import { TestBed } from "@angular/core/testing";
 ...
   beforeEach(() => {
     providerService = mock<ProviderService>();
     accountService = mock<AccountService>();
+    state = mock<RouterStateSnapshot>();
```

### File 2: `bitwarden_license/bit-web/src/app/admin-console/providers/services/web-provider.service.ts`

**Problem:** `EncString.encryptedString` is `SdkEncString | undefined` (`libs/legacy-crypto/src/models/enc-string.ts:16`), but the request fields need a `string`. There are **4** spots (the ticket says "three"):

| Line | Value | Goes into |
|---|---|---|
| 54 | `encryptedOrgKey.encryptedString` | `addOrganizationToProvider({ key: string })` |
| 96 | `encryptedProviderKey.encryptedString` | `CreateProviderOrganizationRequest` `key: string` |
| 97 | `encryptedPrivateKey.encryptedString` | `new OrganizationKeysRequest(publicKey, encryptedPrivateKey: string)` |
| 98 | `encryptedCollectionName.encryptedString` | `CreateProviderOrganizationRequest` `collectionName: string` |

**Fix:** remove the header, then use `assertNonNullish` (already imported at line 10, already used at lines 49, 50 and 84).

> Correction: the 2nd argument is a **name**, not a message. It throws `"<name> is null or undefined."`. Pass e.g. `"encryptedOrgKey.encryptedString"`. See `PM-44178-plan.md`.

```ts
const encryptedOrgKey = await this.encryptService.wrapSymmetricKey(orgKey, providerKey);
assertNonNullish(encryptedOrgKey.encryptedString, "encryptedOrgKey.encryptedString");
await this.providerApiService.addOrganizationToProvider(providerId, {
  key: encryptedOrgKey.encryptedString,
  organizationId,
});
```
…and the same for the other three before `new CreateProviderOrganizationRequest(...)`.

### Do / Don't (from the PRs and reviews)

- ✅ Runtime `assertNonNullish`, **not** `!` (#22391 used `!`; the ticket and the provider precedent use the assert)
- ✅ Keep scope tight. #23219's reviews show the bugs that came from bundling a refactor.
- ✅ If a strict error points at a field that's never used, delete the field rather than add `!` (#22391 review)
- ❌ Don't add `title: null`-style nulls. Omit optional props or use `undefined`.
- ❌ Don't convert to OnPush, `inject()` or standalone here. That's out of scope.

### Verify

```bash
npx jest --testPathPatterns="providers/(guards|services)"
npm run test:types      # may reveal errors the ticket didn't list — not yet checked
npm run lint:fix
npm run prettier
```
