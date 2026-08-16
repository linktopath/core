# Testing Specification

In order to ensure testing consistency across the different distributions, we
will define the expected functionality in this document.

## Rationale

1. Emojis shouldn't be allowed in the `slug` because the URL-encoded version
   decreases readability
2. Spaces shouldn't be allowed in the `slug` because URL highlighting in
   browsers and social media would exclude the parts after the first space
3. Underscores should be allowed as they don't break URL highlighting, even if
   it generally isn't recommended for being visually obstructed by the underline
4. All other non-ASCII characters (accented letters, CJK, etc.) shouldn't be
   allowed in the `slug` because, like emojis, they require URL-encoding that
   decreases readability
5. Dots shouldn't be allowed in the `slug` because a dot-separated or leading
   dot segment (e.g. `..`) could be interpreted as a path component, opening up
   path-traversal and caching ambiguities
6. Slugs are case-sensitive: `MySlug` and `myslug` are distinct shortcuts

## Allowed slug characters

The `slug` may only contain alphanumeric characters (`a-z`, `A-Z`, `0-9`),
hyphens (`-`), and underscores (`_`), and must be between 1 and 100 characters
long. Anything else must be rejected with a 422 Unprocessable Content. The
`slug` may not be `shortcut`, as it collides with the `/shortcut` endpoint.

## Allowed source_url schemes

The `source_url` must use one of the following schemes: `http`, `https`, or
`mailto`. Anything else (including `ftp`, `file`, `javascript`, or a missing
scheme) must be rejected with a 422 Unprocessable Content.

## API Endpoints

### `/:slug`

- Happy path tests
  1. Requests without authentication should succeed
  2. GET requests should NOT return a 405 Method Not Allowed
  3. HEAD requests should succeed and return the same status code as GET,
     without a response body
  4. All tests still pass with a JSON request body
  5. Fetching a previously-created shortcut should return the associated
     `source_url` in the response body
- Sadge path tests
  1. Requests to nonexistent shortcuts should return a 404 Not Found
  2. Requests to expired shortcuts should return a 404 Not Found
  3. PATCH, PUT, POST, DELETE requests should return 405 Method Not Allowed

### `/shortcut`

- Happy path tests
  1. Requests with the following `slug` values should NOT return 422
     Unprocessable Content:
     - short
     - numbers123
     - with_underscore
     - long-with-hyphen
     - LongWithCapitalization
     - ReallyLongButStillWithin100Characters
  2. Requests with a valid `source_url` using the following schemes should NOT
     return a 422 Unprocessable Content:
     - <http://example.com>
     - <https://example.com>
     - <mailto:user@example.com>
  3. Requests with an `expiry_date` in the future should NOT return a 422
     Unprocessable Content
  4. Attempting to create a shortcut with a valid request body should:
     1. Return the newly created shortcut
     2. Succeed with a 201 Created
  5. Attempting to create multiple shortcuts to the same `source_url` should
     succeed with 201 Created
  6. Creating a shortcut with a `slug` previously used by an expired shortcut
     should succeed with 201 Created
  7. Creating a shortcut without an `expiry_date` should succeed with 201
     Created, and the shortcut should NOT expire
  8. Creating a shortcut with a `slug` that differs only by capitalization from
     an existing one (e.g. `MySlug` after `myslug`) should succeed with 201
     Created, since slugs are case-sensitive
  9. A shortcut created via `/shortcut` should be retrievable via `GET /{slug}`
     with the correct `source_url` in the response body (round-trip)
  10. The `id` of the returned shortcut should be a valid UUID
- Sadge path tests
  1. Requests with the following `slug` values should fail with a 422
     Unprocessable Content:
     - with multiple spaces
     - uses\backslashes\
     - uses/slashes/
     - with-an-🥼
     - café
     - リンク
     - my.slug
     - ..
     - (empty string)
  2. Requests with a `slug` longer than 100 characters should fail with a 422
     Unprocessable Content
  3. Attempting to create a shortcut with an existing unexpired `slug` should
     fail with 409 Conflict
  4. Attempting to create a shortcut with the reserved `slug` `shortcut` should
     fail with 409 Conflict
  5. Requests with malformed JSON bodies should return 400 Bad Request
  6. Requests with no JSON body at all should return a 400 Bad Request
  7. Requests with missing JSON body fields should return 422 Unprocessable
     Content
  8. Requests with an `expiry_date` in the past or present should fail with a
     422 Unprocessable Content
  9. Requests with a malformed `expiry_date` (e.g. `not-a-date` or
     `2026-13-45T00:00:00Z`) should fail with a 422 Unprocessable Content
  10. Requests with a `source_url` that uses an unsupported scheme or no scheme
      at all (e.g. `example.com`, `ftp://example.com`, `javascript:alert(1)`)
      should fail with a 422 Unprocessable Content
  11. GET, POST, DELETE, and PATCH requests to `/shortcut` should return a 405
      Method Not Allowed
  12. Requests with a `Content-Type` other than `application/json` should return
      a 415 Unsupported Media Type
  13. Requests with a valid JSON body that is not an object (e.g. an array or a
      primitive) should fail with a 422 Unprocessable Content
