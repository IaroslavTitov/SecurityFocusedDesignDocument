# Security-focused Sample Design Document

This is a sample design document that I wrote recently for a hackathon challenge. I'm posting it publically to help anyone interested evaluate my skills and knowledge as a software engineer. Below you will find the full challenge prompt I was provided before writing the document. Click [here](/DesignDocument.md) to view the final design document.

# Challenge Prompt

In this challenge, you will write a design document for an app that allows a user to browse
directory content on a remote server. Think Google Drive, Dropbox, or even
GitHub's file browser.

![Google Drive Screenshot](https://github.com/gravitational/careers/raw/main/challenges/fullstack/assets/google.jpg)

![GitHub Screenshot](https://github.com/gravitational/careers/raw/main/challenges/fullstack/assets/github.jpg)

## Requirements

Implement an application that allows a user to browse directory content on a remote server.

This application should have the following functionality:

-   A Go backend that serves the webapp and an API
-   The UI, which should include client-side filtering and sorting capabilities
    and URL-based navigation.
-   Strong authentication

Additionally, put extra emphasis on security. The solution needs to have a strong security posture as it pertains to authentication, encryption, and overall web security.

### API

Your API needs to encrypt all data in transit.

Your API should serve file information for files under a specific directory.

Your API only needs to return the contents of the requested directory and does not need to recurse into subdirectories.

The API will also needs to support session management (logging in and out).

### UI

The UI should allow a user to view the contents of a single directory. Clicking
on a subdirectory should navigate to that directory and refresh the contents.
Unlike other commercial tools, _file preview is not required_. Clicking on a
file should not do anything.

The following features are required:

-   [ ] Display the filename, type (file or directory), and human-readable size for files.
-   [ ] Add support for filtering the directory contents based on filename.
        Filtering should be performed client-side, and a simple substring match is
        sufficient.
-   [ ] Add support for sorting directory contents based on filename, type, and
        size.
-   [ ] Include breadcrumbs that show the current location in the directory. The
        breadcrumbs should be clickable for easy navigation to parent directories.
-   [ ] Implement URL navigation. The state of the app should be encoded in the
        URL. No state should be lost upon a page refresh.

### Authentication

The app should also present directory information only to authenticated users.
This will require changes to both the API and the UI.

The following features are required:

-   [ ] The API should reject requests from unauthenticated users
-   [ ] The UI should redirect unauthenticated users to a login page. After a
        successful login, users should be directed back to the page they initially
        requested.
-   [ ] The UI should provide a way for users to log out.

User sessions can be stored in memory, there is no need for a database of any
kind.

User registration / enrollment is not required. You are welcome to hard-code one
or two valid users, just let us know what their credentials are for testing
purposes.

Please do not design to use a third party solution that provides authentication out of the box.
