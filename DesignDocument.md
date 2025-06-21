# Web File Viewer Design

## Background

This document contains implementation design for Web File Viewer application, which provides a web and API interface to navigate through files and folders within a given directory in the server's filesystem. For full requirements, refer to the README of this repository.

## Proposed Design

### Backend

Backend of the application will be a service written in Go using standard `net/http` package. The backend will embed frontend files into the binary.

#### Configuration

Backend will handle loading configuration from server's environment variables in a `config` package. All variables will use prefix `WFV` (standing for Web File Viewer) to minimize potential name collision. Here's a list of what will be needed:

-   `WFV_ROOT_DIRECTORY` (default `~`) - The directory to expose via API and web intefaces
-   `WFV_PORT` (default `443`) - Which port to serve the app at
-   `WFV_TLS_KEYFILE` (default `tls/localhost.key`) - TLS private key filepath
-   `WFV_TLS_CERTFILE` (default `tls/localhost.crt`) - TLS certificate filepath
-   `WFV_AUTH_KEY` (default `[]byte("insecure-localhost-key")`) - Private key used to sign JWTs

#### Web server

Once configuration is loaded, a HTTPS-only web server will start and serve the API and the frontend using provided TLS certificate. Golang server will call `http.ListenAndServeTLS`, passing in the location of certificate and key files loaded by the `config` package, ensuring all endpoints are served using HTTPS only. For local development, a script will be provided to generate a localhost TLS certificate, and in production we will be able upload our certificates as part of the CI/CD pipeline and point to them via environment variables.

All API handling logic will be located in the `api` package. The backend will handle only specific routes under `/api`, serving the frontend for all other paths, and letting React router redirect further if necessary.

#### Authentication

The backend server will feature authentication logic in `auth` package. The package will store hardcoded set of credentials for 2 test users, specifically a username and a hashed password. The package will verify credentials by hashing passed in plaintext password and comparing it to the hardcoded set. Standard library `crypto/sha256` will be used for hashing. On successful login, we will use `golang-jwt/jwt` package to issue a JWT valid for 8 hours (this can be a different or configurable value, depending on user access patterns).

All other API endpoints will be behind an authentication middleware provided by the `auth` package, ensuring a valid JWT is passed in as the `Authentication` header. On logout, we will keep an in-memory blacklist of JWTs that haven't expired yet. Normally, the blacklist would be persisted with a TTL, but requirements assert this is an acceptable solution for the purposes of the exercise.

The hardcoded users' credentials will be:

-   `user1` - `wfv1`
-   `user2` - `wfv2`

#### Authorization

All users will have the same read access to all directories, and there will not be any write actions users can take, so WFV will basically have no authorization.

### API

There will be 3 API endpoints: Login, Logout and GetDirectory. Here's a proposed design for these endpoints:

-   Login: `/api/login` POST
    -   Request model: `{ username: string, password: string }`
    -   Response model: `{ jwt: string, expirationTimestamp: DateTime }`
    -   Returns `200` on successful login
    -   Returns `400` if request model is invalid
    -   Returns `404` if username and password combo doesn't exist
    -   Returns `500` on server error
-   Logout: `api/logout` POST
    -   Requires `Authentication` header with JWT token
    -   No request body necessary
    -   Returns `200` on successful logout
    -   Returns `403` if authorization token is missing, invalid or expired
    -   Returns `500` on server error
-   GetDirectory: `api/directory/*` GET
    -   Requires `Authentication` header with JWT token
    -   Request URL past `api/directory/` will be parsed for the path within the folder
    -   The URL path will be URL encoded, to ensure support for spaces and non-latin characters in directory and file names
    -   Response model: `Directory` model in JSON format
    -   Returns `200` on successful response
    -   Returns `403` if authorization token is missing, invalid or expired
    -   Returns `404` if directory in path doesn't exist
    -   Returns `500` on server error

Models used in the API design:

```
Directory:
{
    path: string // Full path of this directory
    contents: DirectoryItem[] // All contents of the directory
}

DirectoryItem:
{
    name: string // Name of this item
    type: enum( directory | file ) // Type of this item
    size: int // Filesize of this item in bytes. 0 for directories
}
```

GetDirectory endpoint will only support directories. If a path ending with a filename is requested, a `404` response will returned, even if the path is valid and file exists in the server directory.

### Frontend

The frontend UI will be a simple React app. It have 2 pages: Login and Directory, located at `/login` and `/directory/*` routes. Any route apart from these will be redirected to `/login` via React Router in Declarative mode. The app will have a basic shared layout for all pages. The mockups below are for rough element positioning only and are not reflective of the intended design.

#### Shared Layout

Shared layout will feature a header logo/text and a logout button. The logout button will be shown only when the user is logged in.

On logout button click, Logout API endpoint will be called, local JWT token will be erased and user will be redirected to the Login page.

![image](/design/shared_layout.PNG)

#### Login page

Login page will display a simple form with inputs for username and password, with a submit button to send the request (you will also be able to submit via Enter button). The form will require both inputs so as to not send useless requests.

If login fails due to invalid credentials, an explanation message will be displayed.

On successful login, JWT returned by the login API will be stored in browser's local storage. Then, frontend will check browser storage for an original URL, stored upon redirect from the Directory page. If such original URL exist, we will load it, clean it up from storage and redirect user there. If no original URL found, user will be redirected to the root Directory page.

If a user who is already logged in navigates to the login page, he will be automatically redirected to the root Directory page.

![image](/design/login.PNG)

#### Directory page

Directory page will parse current URL to retrieve the intended path within server directory. Then, it will use that path to call `/api/directory/` endpoint to retrieve the information about this directory. For example, the URL for the page mocked up in the image below would be `/directory/MyStuff/Videos/Movies` and it will call `api/directory/MyStuff/Videos/Movies` to retrieve the directory information.

If request fails with a 403, user will be redirected to the Login page. The current URL will be stored in browser's local storage, so that login page can send the user back to the originally requested URL.

If request fails with a 404, user will be redirected to `/directory`, which is guaranteed to exist.

On a successful response, the page will render a table displaying directory information. It will feature:

-   Clickable breadcrumbs to easily navigate to any directory in the current path
-   Clickable folder names to easily navigate deeper
-   Filter input for simple "contains string" filtering
-   Clickable column headers to order by column in ascending or descending order

![image](/design/directory.PNG)

### Security Considerations

A potentially dangerous vulnerability for an application that reads files and folders from a disk directory is path traversal attack. To mitigate it, GetDirectory API will use `filepath.Clean` method from Golang's standard library, and then ensure that cleaned path points to a directory that start with our `WFV_ROOT_DIRECTORY`, effectively preventing malicious users from snooping in other directories.

Apart from the path traversal attack, here's some thoughts on top 10 common vulterabilities as a checklist to ensure security is well thought through.

-   SQL Injection
    -   N/A, as we don't use a SQL database.
-   Cross Site Scripting
    -   Reflected XSS - N/A, since we don't use query parameters.
    -   Stored XSS - N/A, since no user input is stored by the system.
    -   DOM-based XSS - frontend will not insert user input into the DOM.
-   Broken Authentication and Session Management
    -   Authentication JWTs will be blacklisted on logout, and auto-expire in 8 hours from login. The only concern would be that blacklist is in-memory only, meaning it would be cleared on server restart. This is known and accepted risk for purposes of exercise, in a real system blacklist would be persisted.
-   Insecure Direct Object References
    -   The directory data endpoint will be guarded and only authorized users will be able to acess it. All users will have the same access rights, so reading data of another user is not a concern.
-   Cross Site Request Forgery
    -   N/A, as there are no write action that can be performed by users.
-   Security Misconfiguration
    -   Would be a problem if this application was actually hosted, but since it will likely never be run outside of development environment, this is not a concern.
-   Insecure Cryptographic Storage
    -   The only sensitive data stored by the service are the hardcoded user passwords, and they will be hashed, not stored in plaintext. Directory and file names are assumed not sensitive.
-   Failure to restrict URL Access
    -   All API endpoints except for `api/login` will be behind authorization middleware. All users will have the same access, so no escalation of priviledge is possible. All frontend pages will be retrieving data via API calls and thus be useless for non-authorized users.
-   Insufficient Transport Layer Protection
    -   The service will only serve HTTPS requests.
-   Unvalidated Redirects and Forwards
    -   All redirects will be handled by React Router, and will not use query parameters or user input in the redirect logic.
