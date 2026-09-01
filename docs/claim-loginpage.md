# Claim: `frontend/src/pages/Login.vue` is the guest authentication entry page for Project Pulse

`Login.vue` is the unauthenticated entry point for the SPA. It is responsible for collecting the user's email/username and password, validating the form, sending the credentials to the backend authentication endpoint, storing the returned JWT and user profile in client state, and redirecting the user into the authenticated app.

## What it is responsible for

In `frontend/src/pages/Login.vue`, the page:

- renders the branded login form for Project Pulse
- accepts the username (email) and password fields
- validates both fields before submit
- calls the login API when the user clicks Login or presses Enter
- stores the token and profile information in Pinia stores
- sets default course/section context for the user's role
- redirects to the original destination or `/`
- routes the user to the password-reset flow when needed

This matches the architecture described in `docs/design/architectural-design.md`, where the SPA authenticates through the shared API layer and stores a JWT in client state before any protected request is made.

## How a request goes through it

1. The route `/login` is defined in `frontend/src/router/routes.ts` with:
   - `requiresAuth: false`
   - `visitorOnly: true`

   This means the page is only for logged-out users.

2. The user enters values into the form in `Login.vue`:
   - `loginData.username`
   - `loginData.password`

3. `async function login()` runs when the button is clicked.
   - it calls `loginForm.value.validate()`
   - if validation passes, it calls `loginUser(loginData.value)`

4. `loginUser` is exported from `frontend/src/apis/login/index.ts`:

   ```ts
   request.post<any, LoginResponse>(
     '/users/login',
     {},
     { auth: loginData }
   )
   ```

   The shared Axios instance in `frontend/src/utils/request.ts` handles the HTTP request. It injects the token when present and unwraps the backend `Result` envelope from the Axios response.

5. The shared request layer is the true transport boundary. In `request.ts`:
   - base URL is configured (`/api/v1` in production or a dev override)
   - the request interceptor adds `Authorization` when a token exists
   - the response interceptor unwraps `response.data`
   - `401` responses trigger logout and redirect back to `/login`

6. On success, `Login.vue` stores the returned data:
   - `tokenStore.setToken(result.data.token)`
   - `userInfoStore.setUserInfo(result.data.userInfo)`

7. Based on the role, it also sets defaults:
   - instructor: default course and default section
   - student: default section

8. It then redirects with:

   ```ts
   let redirect: any = route.query.redirect
   router.push({ path: redirect || '/' })
   ```

## What it depends on

The page depends on several parts of the app:

- `frontend/src/apis/login/index.ts`
  - wraps the backend login endpoint
- `frontend/src/utils/request.ts`
  - shared Axios client with auth headers and global error handling
- `frontend/src/stores/token.ts`
  - persists the JWT string
- `frontend/src/stores/userInfo.ts`
  - persists the decoded user profile and exposes role helpers (`isInstructor`, `isStudent`, etc.)
- `frontend/src/stores/settings.ts`
  - stores default course/section selections
- `frontend/src/router/routes.ts`
  - registers the `/login` route as a visitor-only page
- `frontend/src/router/guards.ts`
  - redirects unauthenticated users to login and prevents logged-in users from revisiting visitor-only pages

## Relevant project documentation

The design and requirements docs confirm this is the intended auth flow:

- `docs/design/architectural-design.md`
  - says the SPA authenticates through `POST /api/v1/users/login (HTTP Basic)`
  - stores the JWT in Pinia and sends it on subsequent authenticated calls
  - uses a shared Axios instance and authorization enforcement

- `docs/requirements/software-requirements-specification.md`
  - states `CO-single-auth`: Project Pulse shall provide a single JWT-based authentication mechanism for all users across both capability areas.
  - states `FR-SEC-authentication`: the system shall authenticate users via its JWT-based authentication mechanism before granting access to protected resources.

## Bottom line

`Login.vue` is not just a form. It is the front-end authentication gateway for the app: it captures credentials, calls the authentication API, stores the issued JWT and profile, initializes the user's context, and moves the user from anonymous to authenticated state.
