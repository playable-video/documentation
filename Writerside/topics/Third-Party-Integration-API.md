# Third-Party Integration API Documentation

## Overview

This documentation outlines the Third-Party Integration API for Playable, enabling external platforms (e.g., marketing tools like Mailchimp) to securely authenticate users and access their video content. The API leverages AWS Cognito for token-based authentication, avoiding direct password sharing. It focuses on core functionalities: user authentication, video listing, and embed snippet retrieval.

**Key Features:**
- Secure authentication via AWS Cognito.
- Access to user-specific video lists and embed codes.

**Base URL:** `https://api.playable.video`
**Endpoint Prefix for Third-Party Features:** `/thirdparty/`  
**Request Method:** All endpoints use POST with JSON payloads.  
**Response Format:** JSON.

## Important Notes

- **Authentication:** Handled by AWS Cognito User Pools. Users must have a Playable account; credentials are synced with Cognito during signup, pw resets, and changes.
- **Access Tokens:** JWTs from Cognito, valid for 1 hour by default. Include in subsequent requests for authorization.
- **Error Handling:** Common HTTP codes include:
  - 400: Bad request (e.g., missing parameters).
  - 401: Unauthorized (e.g., invalid credentials or token).
  - 403: Forbidden (e.g., access denied to resource).
  - 404: Not found (e.g., video ID invalid).
  - 500: Internal server error.
  Error bodies contain a descriptive message.
- **Security:** All Cognito interactions use encrypted channels.
- **Out of Scope (MVP):** Video uploads, edits, or full OAuth flows. These may be added in future versions.
- **Prerequisites:** Users need an active Playable account. For testing, create one via the Playable signup process.

## Table of Contents

1. [Authentication](#authentication)
2. [Listing Videos](#listing-videos)
3. [Getting Video Snippet (Embed Code)](#getting-video-snippet-embed-code)
4. [AWS Cognito](#aws-cognito)
5. [Security Considerations](#security-considerations)
6. [Additional Resources](#additional-resources)

## Authentication

Obtain an access token to authorize subsequent API calls.

**Endpoint:** `/thirdparty/authenticate`  
**Method:** POST  
**Description:** Authenticates using the user's Playable email (as username) and password. Returns a Cognito JWT access token via the `USER_PASSWORD_AUTH` flow.

**Request Body (JSON):**
```json
{
  "username": "user@example.com",  // Playable account email
  "password": "securepassword123"   // User's password
}
```

**Successful Response (200 OK):**
```json
{
  "access_token": "eyJraWQiOiJ...exampleJWT"  // JWT token (expires in 1 hour)
}
```

**Common Errors:**
- 400: "Missing username or password"
- 401: "Invalid username or password"
- 500: "Internal server error" (e.g., Cognito outage)

### How It Works
1. The endpoint fetches Cognito credentials from AWS Secrets Manager.
2. It initializes a boto3 Cognito client.
3. Calls `cognito_client.initiate_auth` with provided credentials.
4. Returns the access token on success.
5. The token is validated in other endpoints via `cognito_client.get_user` to extract the user's email.

### Implementation Details
- File: `auth.py`
- Libraries: `boto3` for AWS interactions.
- Secrets: User Pool ID and Client ID from `playable/prod/cognito-config` in Secrets Manager.
- Logging: Errors (e.g., invalid params) are captured for debugging.

**Usage Tip:** Store the token securely and refresh it before expiration if your session exceeds 1 hour.

## Listing Videos

Retrieve a list of videos owned by the authenticated user.

**Endpoint:** `/thirdparty/listvideos`  
**Method:** POST  
**Description:** Returns video details including ID, title, and thumbnail URL.

**Request Body (JSON):**
```json
{
  "access_token": "eyJraWQiOiJ...exampleJWT"  // From authentication endpoint
}
```

**Successful Response (200 OK):**
```json
{
  "videos": [
    {
      "video_id": 12345,
      "title": "Campaign Video 1",
      "thumbnail_url": "https://cdn.playable.video/lowsrc.jpg"
    },
    {
      "video_id": 67890,
      "title": "Product Demo",
      "thumbnail_url": "https://cdn.playable.video/another-lowsrc.jpg"
    }
  ]
}
```

**Common Errors:**
- 400: "Missing access_token"
- 401: "Invalid access_token" or "Account not found for this user"
- 500: "Internal server error"

### How It Works
1. Validate token with Cognito to get the user's email.
2. Query Playable's Google Cloud NDB for the matching account.
3. Fetch videos where `account_id` matches.
4. Generate thumbnail URLs using the video's `get_cdn_url` method.

### Implementation Details
- File: `list_videos.py`
- Datastore: Uses NDB queries on `datamodels.Account` and `datamodels.Video`.
- Authorization: Ensures token links to a valid Playable account.

**Usage Tip:** This is what would be used to populate a dropdown in your plugin for video selection.

## Getting Video Snippet (Embed Code)

Fetch embed HTML for a specific video, optimized for email clients.

**Endpoint:** `/thirdparty/getvideosnippet`  
**Method:** POST  
**Description:** Returns the "inline" embed snippet for third-party tools.

**Request Body (JSON):**
```json
{
  "access_token": "eyJraWQiOiJ...exampleJWT",  // From authentication
  "video_id": 12345                          // From listvideos response
}
```

**Successful Response (200 OK):**
```json
{
  "snippet": "<div class='playable-embed'>...Embed HTML here...</div>",
}
```

**Common Errors:**
- 400: "Missing access_token or video_id"
- 401: "Invalid access_token" or "Account not found"
- 403: "Unauthorized access to video"
- 404: "Video not found"
- 500: "Internal server error"

### How It Works
1. Validate token to get email and account.
2. Query video by ID and verify ownership (`account_id` match).
3. Generate snippet via video's `snippets` method with `{'client': 'email'}` options.

### Implementation Details
- File: `get_video_snippet.py`
- Datastore: NDB for `datamodels.Account` and `datamodels.Video`.

**Usage Tip:** This snippet is what should be inserted directly into email templates for playable video embeds.

## AWS Cognito

### Cognito Setup
- **Identifiers:** Email as username.
- **Secrets:** User Pool ID/Client ID in AWS Secrets Manager (`playable/prod/cognito-config`).
- **Operations:** Use `boto3` with credentials from `settings_gen.AWS.auth`.
- **Repo:** Terraform was used to create the cognito pool set up. Accessable from: https://github.com/playable-video/playable-cognito-setup

**Note:** Sync ensures Cognito mirrors Playable's user base without exposing passwords.

## Security Considerations

- **Password Management:** Hashed in Cognito.
- **Token Expiry:** 1-hour default.
- **Rate Limiting/Logging:** Backend handles throttling and error logs (e.g., `logging.error`).
- **Best Practices:** Use HTTPS, validate inputs, and monitor for anomalies.

## Additional Resources

- **Repo:** Maintained in Playable Documentation GitHub. Published at: https://playable-video.github.io/documentation/.
