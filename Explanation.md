# Explanation

## What was the bug?

To make an API request, the client needs a valid login token. Before sending a request, the code checks: "do I have a good token? If not, go get one."

The bug is that this check was incomplete. It only knew how to handle two situations:

1. There is no token at all (`null`)
2. There is a proper token object and it has expired

It forgot about a third situation: **the token is a plain data object** (like `{ accessToken: "stale", expiresAt: 0 }`) instead of a proper `OAuth2Token`. When that happened, the code looked at it, got confused, did nothing, and sent the request with no `Authorization` header at all.

## Why did it happen?

The token variable can hold three different things: `null`, a proper `OAuth2Token`, or a plain object. The old `if` condition only had logic for the first two. The plain object case slipped through — the code saw it was not `null` (so it didn't refresh) and it was not an `OAuth2Token` (so it didn't check expiry either). It just silently skipped the refresh.

## Why does the fix solve it?

The old question was: "is there no token, OR is the token expired?"

The new question is: **"is this NOT a proper `OAuth2Token`, OR is it expired?"**

That one change catches everything in one go:

- `null` → not a proper token → refresh
- plain object → not a proper token → refresh
- expired `OAuth2Token` → is a proper token but expired → refresh
- valid `OAuth2Token` → is a proper token and not expired → do nothing, use it

## One edge case the tests still don't cover

What if the token is a proper `OAuth2Token` but it has already expired? The fix handles this correctly (it refreshes), but there is no test that actually checks this scenario. A test like "set an expired token, make a request, expect the fresh token to be used" is missing.
