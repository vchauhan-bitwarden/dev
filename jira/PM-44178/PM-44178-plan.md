# PM-44178 — Implementation plan

Related notes:
- `PM-44178.md`: ticket summary and checklist
- `PM-44178-pr-study.md`: what past ts-strict PRs did

Base path: `bitwarden_license/bit-web/src/app/admin-console/providers/`

---

## ⚠️ Read first: `assertNonNullish` takes a NAME, not a message

`libs/common/src/auth/utils/assert-non-nullish.util.ts`:
```ts
export function assertNonNullish<T>(val: T, name: string, ctx?: string): asserts val is NonNullable<T> {
  if (val == null) {
    throw new Error(`${name} is null or undefined.${ctx ? ` ${ctx}` : ""}`);
  }
}
```

- The existing calls in `web-provider.service.ts` misuse it:
  `assertNonNullish(orgKey, "Organization key not found")` throws **"Organization key not found is null or undefined."**
- The tests only pass because `toThrow("Organization key not found")` matches a substring.
- For the new calls, pass a **name**, e.g. `"encryptedOrgKey.encryptedString"`.

---

## Main decision: how to handle `encryptedString` being optional

`EncString.encryptedString` is optional (`libs/legacy-crypto/src/models/enc-string.ts:16`), but the request fields need a `string`.

| Option | Looks like | Pros | Cons |
|---|---|---|---|
| **A. `assertNonNullish` at each spot** ✅ | `assertNonNullish(encryptedOrgKey.encryptedString, "encryptedOrgKey.encryptedString")` | Checks at runtime and throws before a bad request reaches the server. Narrows the type, so the next line compiles. Same as `provider-actions.service.ts:77`. It's what the ticket asks for. | 4 extra lines |
| B. Non-null `!` | `encryptedOrgKey.encryptedString!` | Smallest diff. #22391 did this. | Only silences the compiler. If the value were ever missing, it would send `undefined` to the server. Goes against the ticket. |
| C. `?? ""` or loosen the request types | `key: x ?? ""`, or `key?: string` | Compiles | Would send an empty key, or push the problem into shared `libs/common` models. Wrong. |
| D. Fix it at the source | Make `EncString.encryptedString` non-optional / add a helper | Fixes it for everyone | Changes `@bitwarden/legacy-crypto` (Key Management owns it; CLAUDE.md flags crypto changes). Way out of scope. |

**→ Recommendation: A.** B is the fallback only if a reviewer prefers the smaller diff.

---

## Smaller decisions

### 1. What to pass to the assert
- ✅ **A name, as designed:** `"encryptedOrgKey.encryptedString"` → "encryptedOrgKey.encryptedString is null or undefined."
- ❌ Copying the file's message-style calls just spreads the misuse.
- 🚫 **Don't** rewrite the 3 existing calls (lines 49, 50, 84). That changes error text existing tests match against, and it's outside the ticket. Mention it in the PR instead.

### 2. Where to put the asserts
- ✅ **Right after each value is created.** Same style as line 84 (`providerKey` asserted right after it's fetched), so the guard sits next to where the value comes from.
- Also fine: group the 3 asserts for `createClientOrganization` just before `new CreateProviderOrganizationRequest(...)`. Nothing before that line calls the server, so the behaviour is the same.

### 3. Tests

| Option | Pros | Cons |
|---|---|---|
| None (the ticket only asks to run jest) | Smallest PR | The new throw paths have no tests. In #22391, a reviewer asked for tests on a similar behaviour change. |
| **4 tests, one per assert** ✅ | Each behaviour is tested on its own. The existing setup makes each test about 5 lines. | About 30 lines of tests |
| 1 test per method | Fewer tests | Can't show which of the 3 asserts in `createClientOrganization` fired |

- How to trigger the missing value in a test: make the mock return `new EncString(undefined as unknown as string)`, then `await expect(...).rejects.toThrow("…encryptedString is null or undefined")`.
- The guard spec needs **no test**. The guard ignores `_state` (`guards/provider-permissions.guard.ts:39`), so the fix only affects compilation.

---

## Step-by-step (each step has a gate)

### Step 0: Prep
- [ ] `git rebase origin/main` (the branch is 7 commits behind; none touch these files)
- [ ] **Gate:** `npx jest --testPathPatterns="providers/(guards|services)"` passes *before* any change, so you have a baseline

### Step 1: Find the real error list
- [ ] Remove the 2-line marker from **both** files:
  ```ts
  // FIXME: Update this file to be type safe and remove this and next line
  // @ts-strict-ignore
  ```
- [ ] Run `npm run test:types`
- [ ] **Gate:** the errors should be only the unassigned `state` in the spec plus the 4 `encryptedString` spots in the service. **If there are more, stop and reassess the scope.** I haven't checked this.

### Step 2: Guard spec (`guards/provider-permissions.guard.spec.ts`)
- [ ] In `beforeEach`, add:
  ```ts
  state = mock<RouterStateSnapshot>();
  ```
  (copy of `apps/web/src/app/admin-console/organizations/guards/org-permissions.guard.spec.ts:57`)
- [ ] **Gate:** the spec compiles and passes

### Step 3: Service (`services/web-provider.service.ts`)
Add 4 asserts, each right after its value is created:

| Line (today) | Value | Goes into |
|---|---|---|
| 54 | `encryptedOrgKey.encryptedString` | `addOrganizationToProvider({ key })` |
| 96 | `encryptedProviderKey.encryptedString` | `CreateProviderOrganizationRequest` `key` |
| 97 | `encryptedPrivateKey.encryptedString` | `new OrganizationKeysRequest(publicKey, …)` |
| 98 | `encryptedCollectionName.encryptedString` | `CreateProviderOrganizationRequest` `collectionName` |

Shape:
```ts
const encryptedOrgKey = await this.encryptService.wrapSymmetricKey(orgKey, providerKey);
assertNonNullish(encryptedOrgKey.encryptedString, "encryptedOrgKey.encryptedString");
```

- [ ] **Gate:** `npm run test:types` is clean
- [ ] **Gate:** the existing `services/web-provider.service.spec.ts` still passes. It should: the mocks use `new EncString("encrypted-…")`, and the constructor sets `encryptedString` first (`enc-string.ts:95`).

### Step 4: Tests (`services/web-provider.service.spec.ts`)
- [ ] `addOrganizationToProvider`: throws when `encryptedOrgKey.encryptedString` is missing
- [ ] `createClientOrganization`: throws when `encryptedPrivateKey.encryptedString` is missing
- [ ] `createClientOrganization`: throws when `encryptedCollectionName.encryptedString` is missing
- [ ] `createClientOrganization`: throws when `encryptedProviderKey.encryptedString` is missing
- [ ] **Gate:** each new test **fails** if you temporarily remove its assert, which proves it catches the right thing

### Step 5: Finish
- [ ] `npm run lint:fix`
- [ ] `npm run prettier`
- [ ] `npx jest --testPathPatterns="providers/(guards|services)"`
- [ ] Commits: 2 (spec / service + tests) or 1, whichever your team prefers
- [ ] In the PR description, mention:
  - the ticket says "three" values but there are **four**
  - the 3 existing message-style `assertNonNullish` calls were left as they are on purpose (they pass a message where a name is expected)

---

## Files involved

**You edit:**
- `guards/provider-permissions.guard.spec.ts`
- `services/web-provider.service.ts`
- `services/web-provider.service.spec.ts` (new tests, if you take the recommendation)

**Read-only context:**
- `guards/provider-permissions.guard.ts`: code under test; already strict; ignores `_state`
- `libs/legacy-crypto/src/models/enc-string.ts`: `encryptedString?` is the cause
- `libs/common/src/admin-console/models/request/create-provider-organization.request.ts`: `key` / `collectionName: string`; already throws on a missing `key` (lines 35–39)
- `libs/common/src/admin-console/models/request/organization-keys.request.ts`
- `libs/common/src/auth/utils/assert-non-nullish.util.ts`

**Callers (where a new assert failure would show up for users):**
- `clients/add-existing-organization-dialog.component.ts:65`: `addOrganizationToProvider`
- `clients/create-client-dialog.component.ts:205`: `createClientOrganization`

---

## Risks
- **Step 1 may show more errors than the ticket lists.** That's the main unknown.
- **Crypto review:** this only adds null checks around existing encryption output, with no new crypto logic. Key Management review shouldn't be needed, but the file imports `EncryptService` / `LegacyCompatKeyService`, so a reviewer might flag it.
