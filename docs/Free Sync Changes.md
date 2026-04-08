# Free Sync Changes

## Overview

Two changes were made to remove the payment gate from S3, WebDAV, and Local File System sync providers, making them available to all users.

## Changes

### 1. Backend — `kernel/model/conf.go`

**Function:** `IsPaidUser()`

The function now always returns `true`, bypassing the subscription and one-time payment status checks. The original logic is preserved as comments.

```go
func IsPaidUser() bool {
    return true
    // if IsSubscriber() {
    //     return true
    // }
    //
    // u := Conf.GetUser()
    // if nil == u {
    //     return false
    // }
    // return 1 == u.UserSiYuanOneTimePayStatus
}
```

**Affected backend checks (5 locations in `kernel/model/repository.go`):**
- `CreateSnapshot()`
- `TagSnapshot()`
- `RollbackAssetsRepo()`
- `RollbackDocHistoryRepo()`
- `OpenRepoSnapshotDoc()`

**Plus 1 location in `kernel/model/sync.go`:**
- `syncEnabledCheck()` — no longer disables sync for unpaid users

### 2. Frontend — `app/src/util/needSubscribe.ts`

**Function:** `isPaidUser()`

Same approach — always returns `true` with the original check commented out.

```ts
export const isPaidUser = () => {
    return true;
    // return window.siyuan.user && (0 === window.siyuan.user.userSiYuanSubscriptionStatus || 1 === window.siyuan.user.userSiYuanOneTimePayStatus);
};
```

**Affected frontend files (all import from `needSubscribe.ts`):**
- `app/src/config/repos.ts` — sync provider config UI
- `app/src/sync/syncGuide.ts` — sync guide flow
- `app/src/protyle/wysiwyg/transaction.ts` — auto-sync after edits
- `app/src/card/openCard.ts` — flashcard sync

## What is NOT affected

- **SiYuan Cloud** (Provider 0) uses `IsSubscriber()` / `needSubscribe()` instead, which remains unchanged. Official cloud sync still requires a subscription.
