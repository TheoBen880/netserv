# Web Access Control

How `netserv web` decides whether a request is allowed. Everything here is
derived from the route registration order in `server/web/index.ts`, the auth
controller, and the JWT helper.

## Two independent switches

Authentication and write access are separate and neither implies the other.

- **Authentication** is on only when the server was started with `--password`.
  The controller stores this as `authEnabled`.
- **Write access** is on only when the server was started with `--writable`.

A server can therefore be password-protected but read-only, or writable but
completely open. The client learns both from `GET /auth/init` before it renders,
which is how the UI decides whether to show the login screen and the upload,
rename, delete, and new-folder controls.

## Registration order determines what is protected

Routes are registered in this order, and `checkAuthMiddleware` is inserted part
way through:

1. `/auth/login` (only when `authEnabled`) and `/auth/init`
2. `/@` — the React bundle, with an `index.html` fallback
3. `/` — a redirect to `/@`
4. **`checkAuthMiddleware`**
5. `/api/*` — the JSON filesystem API
6. `/fs` — raw file content served statically

Anything registered before step 4 is reachable without a session. That is
deliberate: the browser has to be able to load the app and call `/auth/init`
before it has a cookie. Everything after step 4 requires a valid session, so
both the metadata API and the raw file bytes are protected by the same gate.

When `authEnabled` is false the middleware calls `next()` immediately and no
route is protected.

## Session lifecycle

Sessions are a signed JWT carried in an `httpOnly` cookie named `jwt`.

- `POST /auth/login` compares the submitted password against the configured one
  by direct equality. On a match it issues a token and sets the cookie with a
  one-year `maxAge`.
- `GET /auth/init` verifies any existing cookie. A valid cookie is rotated: a
  fresh token is issued and the cookie is reset. An invalid cookie is cleared and
  the response reports `jwt: null`.
- `checkAuthMiddleware` rejects with 401 when the cookie is absent or fails
  verification.

Two properties are worth knowing when operating this:

- The signing secret is regenerated randomly on every process start, so
  restarting the server invalidates every outstanding session.
- The token itself expires after `1h`, while the cookie is sent with a one-year
  `maxAge`. The cookie therefore outlives the token it carries; the stale cookie
  is cleared the next time `/auth/init` runs.

The token payload is empty. It proves only that the correct password was
presented at some point — there are no users, roles, or per-path permissions.

## Path handling

Every `/api/*` route is a wildcard mount, so the portion of the URL after the
route prefix is treated as a path. Handlers URI-decode it and join it onto the
configured root directory. The raw `/fs` mount is served with directory indexes
disabled and dotfiles allowed.

## Operational guidance

The intended deployment is a trusted local network: the server prints a QR code
of its LAN URL so a phone can join. Given that shape:

- Start with `--password` on any network you do not fully control. Without it,
  `/api/*` and `/fs/*` are anonymous.
- Add `--writable` only when you actually need uploads or deletion. Deletion is
  recursive and permanent — there is no trash or confirmation on the server side.
- Serve the narrowest directory that works, since the root is the only boundary
  on what the API can reach.
