# EXPLANATION.md

## What was the bug?

In `app/http_client.py`, the `request()` method did not refresh the OAuth2 token when
`oauth2_token` was set to a plain `dict` instead of an `OAuth2Token` instance. As a
result, no `Authorization` header was ever added to API requests in that state.

## Why did it happen?

The refresh condition was:

```python
if not self.oauth2_token or (
    isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired
):
```

When `oauth2_token` is a non-empty dict:

- `not self.oauth2_token` evaluates to `False` (non-empty dict is truthy).
- `isinstance(self.oauth2_token, OAuth2Token)` is `False`, so the `expired` check is
  short-circuited.

The whole condition is `False`, so `refresh_oauth2()` is never called and the token
stays as a dict. The subsequent `isinstance` guard before `as_header()` then also
evaluates to `False`, so the `Authorization` header is silently dropped.

## Why does the fix solve it?

The condition was changed to:

```python
if (
    not self.oauth2_token
    or not isinstance(self.oauth2_token, OAuth2Token)
    or self.oauth2_token.expired
):
```

This triggers a refresh whenever the token is absent *or* is not a proper
`OAuth2Token` instance *or* has expired — covering the dict case explicitly with
the middle clause. After refresh, the token is a valid `OAuth2Token` and
`as_header()` is called correctly.

## One edge case the tests still don't cover

A race condition where `expires_at` is set to **exactly** the current second:
`OAuth2Token.expired` uses `>=`, so a token that expires at the exact current
timestamp is considered expired. If the system clock ticks between the `expired`
check and the API call, the freshly-refreshed token could itself expire before use
, though this is unlikely in practice, it is not tested.
