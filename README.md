# GetMax's API documentation

This repository contains the documentation for [GetMax](https://getmax.ai)’s API.

#### Contents

- [Overview](#1-overview)
- [Authentication](#2-authentication)
- [Resources](#3-resources)
  - [Users](#31-users)
  - [Articles](#32-articles)
  - [Webhook](#33-webhook)
- [Testing](#4-testing)

## 1. Overview

GetMax’s API is a JSON-based OAuth2 API. All requests are made to endpoints beginning:
`https://app.getmax.com/api`

All requests must be secure, i.e. `https`, not `http`.

#### Developer agreement

By using GetMax’s API, you agree to our [terms of service](./TERMS.md).

## 2. Authentication

In order to publish on behalf of a GetMax account, you will need an access token. An access token grants limited access to a user’s account. We offer browser-based OAuth authentication access tokens.
```
https://app.getmax.ai/oauth2?client_id={{clientId}}
    &scope={{scope}}
    &state={{state}}
    &response_type=code
    &redirect_uri={{redirectUri}}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `client_id`     | string   | required   | The clientId we will supply you that identifies your integration. |
| `scope`         | string   | required   | The access that your integration is requesting, comma separated. Currently, there are three valid scope values, which are listed below. Most integrations should request `/api/open-apis/me` |
| `state`         | string   | required   | Arbitrary text of your choosing, which we will repeat back to you to help you prevent request forgery. |
| `response_type` | string   | required   | The field currently has only one valid value, and should be `code`.  |
| `redirect_uri`  | string   | required   | The URL where we will send the user after they have completed the login dialog. This must exactly match one of the callback URLs you provided when creating your app. This field should be URL encoded. |

The following scope values are valid:

| Scope                   | Description                                                             | Extended |
| ------------------------| ----------------------------------------------------------------------- | -------- |
| /api/open-apis/me       | Grants basic access to a user’s profile                                 | No       |
| /api/open-apis/webhook  | Grants the ability to manage webhook subscriptions                      | No       |
| /api/open-apis/articles | Grants the ability to query articles                                    | No       |

Integrations are not permitted to request extended scope from users without explicit prior permission from GetMax. Attempting to request these permissions through the standard user authentication flow will result in an error if extended scope has not been authorized for an integration.

If the user grants your request for access, we will send them back to the specified `redirect_uri` with a state and code parameter:

```
https://example.com/callback/getmax?state={{state}}
    &code={{code}}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `state`         | string   | required   | The state you specified in the request.         |
| `code`          | string   | required   | A short-lived authorization code that may be exchanged for an access token. |

If the user declines access, we will send them back to the specified `redirect_uri` with an error parameter:

```
https://example.com/callback/getmax?error=access_denied
```

Once you have an authorization code, you may exchange it for a long-lived access token with which you can make authenticated requests on behalf of the user. To acquire an access token, make a form-encoded server-side POST request:

```
POST /api/oauth2/access-token
Host: app.getmax.com
Content-Type: application/x-www-form-urlencoded
Accept: application/json
Accept-Charset: utf-8

code={{code}}&client_id={{client_id}}&client_secret={{client_secret}}&grant_type=authorization_code&redirect_uri={{redirect_uri}}
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `code`          | string   | required   | The authorization code you received in the previous step. |
| `client_id`     | string   | required   | Your integration’s `clientId` |
| `client_secret` | string   | required   | Your integration’s `clientSecret` |
| `grant_type`    | string   | required   | The literal string "authorization_code" |
| `redirect_uri`  | string   | required   | The same redirect_uri you specified when requesting an authorization code. |

If successful, you will receive back an access token response:

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "token_type": "Bearer",
  "access_token": {{access_token}},
  "refresh_token": {{refresh_token}},
  "scopes": {{scope}},
  "expires_in": {{expires_at}}
}
```

With the following parameters:

| Parameter       | Type         | Required?  | Description                                     |
| -------------   |--------------|------------|-------------------------------------------------|
| `token_type`    | string       | required   | The literal string "Bearer"                     |
| `access_token`  | string       | required   | A token that is valid for 60 days and may be used to perform authenticated requests on behalf of the user. |
| `refresh_token` | string       | required   | A token that does not expire which may be used to acquire a new `access_token`.                            |
| `scopes`        | string array | required   | The scopes granted to your integration.         |
| `expires_in`    | int64        | required   | Remaining time until the access token expires, in seconds. |

Each access token is valid for 2 houres. When an access token expires, you may request a new token using the refresh token. Each refresh tokens is valid for 30 days. Both access tokens and refresh tokens may be revoked by the user at any time. **You must treat both access tokens and refresh tokens like passwords and store them securely.**

Both access tokens and refresh tokens are consecutive strings of hex digits, like this:

```
xxxxx.xxxx.xxxxxxxxxxxxxxx.xxxxxxxxx
```

To acquire a new access token using a refresh token, make the following form-encoded request:

```
POST /api/oauth2/refresh-access-token
Host: app.getmax.com
Content-Type: application/x-www-form-urlencoded
Accept: application/json
Accept-Charset: utf-8

refresh_token={{refresh_token}}&grant_type=refresh_token
```

With the following parameters:

| Parameter       | Type     | Required?  | Description                                     |
| -------------   |----------|------------|-------------------------------------------------|
| `refresh_token` | string   | required   | A valid refresh token.                          |
| `grant_type`    | string   | required   | The literal string "refresh_token"              |

## 3. Resources

The API is RESTful and arranged around resources. All requests must be made with an integration token. All requests must be made using `https`.

Typically, the first request you make should be to acquire user details. This will confirm that your access token is valid, and give you a user id that you will need for subsequent requests.

### 3.1. Users

#### Getting the authenticated user’s details
Returns details of the user who has granted permission to the application.

```
GET https://app.getmax.com/api/open-apis/me
```

Example request:

```
GET /api/open-apis/me
Host: app.getmax.com
Authorization: Bearer {{access_token}}
Content-Type: application/json
Accept: application/json
Accept-Charset: utf-8
```

The response is a User object within a data envelope.

Example response:

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "id": "42d4491b-976a-47f7-8430-1ef05afc4aaa",
  "user_id": "b024d1f9-ddc5-4d02-95dc-8ab073c2962c",
  "email": "pengfei@talentorg.com",
  "scopes": [
    "/api/open-apis/me",
    "/api/open-apis/webhook",
    "/api/open-apis/articles"
  ]
}
```

Where a User object is:

| Field      | Type   | Description                                     |
| -----------|--------|-------------------------------------------------|
| id         | string | A unique identifier for the authorization.      |
| user_id    | string | The user’s ID on GetMax.                        |
| email      | string | The user’s email on GetMax.                     |

Possible errors:

| Error code           | Description                                     |
| ---------------------|-------------------------------------------------|
| 401 Unauthorized     | The `access_token` is invalid or has been revoked. |


### 3.2. Articles

#### Get recent/example article list

```
GET https://app.getmax.ai/api/open-apis/articles/example
```

The response is a list of article objects. Will return recently reviewed but unpublished articles. If the user has no articles, a set of example data will be returned, with the IDs of the example data being negative integers.

Example response:

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "data": [
    {
      "id": -1,
      "title": "GetMax Example Title",
      "keywords": "keyword1, keyword2",
      "description": "This is an example description",
      "cover": "https://example.com/cover.png",
      "date": "2023-11-29T07:29:54.621Z",
      "content": "This is an example content"
    }
  ]
}
```

Where a Article object is:

| Field       | Type   | Description                                     |
| ------------|--------|-------------------------------------------------|
| id          | string |   Article ID      |
| keywords        | string |   Article keywords          |
| description | string |  Brief description/introduction    |
| cover    | string |  Cover image url   |
| date    | string |   Publication time     |
| content    | string |  Article content     |

Possible errors:

| Error code           | Description                                                                           |
| ---------------------|---------------------------------------------------------------------------------------|
| 401 Unauthorized     | The access token is invalid or has been revoked.    |
| 403 Forbidden        | Resource access denied, possibly due to insufficient scope.  |


### 3.3. Webhook

#### Create a new webhook subscription
Create a new webhook subscription to monitor events.

```
POST https://app.getmax.ai/api/open-apis/webhook/subscription
```

Where authorId is the user id of the authenticated user.

Example request:

```
POST /api/open-apis/webhook/subscription
Host: app.getmax.com
Authorization: Bearer {{access_token}}
Content-Type: application/json
Accept: application/json
Accept-Charset: utf-8

{
  "hook_url": "xxxx-xxxx-xxx-xxxx-xxxxx",
  "type": "new_article_ready_for_publish"
}
```

With the following fields:

| Parameter       | Type         | Required?  | Description                                     |
| -------------   |--------------|------------|-------------------------------------------------|
| hook_url  | string       | required   | Address for webhook push   |
| type   | string       | required   | Type of event the webhook is monitoring. Currently, there is only one option, "new_article_ready_for_publish" |

The response will return subscription information. Example response:

```
HTTP/1.1 201 OK
Content-Type: application/json; charset=utf-8

{
  "id": "xxxxx-xxxx-xxxx-xxxx-xxxxx"
}
```

订阅的信息如下:

| Field         | Type         | Description                                     |
| --------------|--------------|-------------------------------------------------|
| id            | string       | ID of the webhook subscription             |

Possible errors:

| Error code           | Description                                                                                                          |
| ---------------------|----------------------------------------------------------------------------------------------------------------------|
| 400 Bad Request      | Required fields were invalid, not specified.                                                                         |
| 401 Unauthorized     | The access token is invalid or has been revoked.                                                                     |
| 403 Forbidden        | The user does not have permission to publish, or the authorId in the request path points to wrong/non-existent user. |

#### Remove a webhook subscription
This API allows the cancellation of previously initiated subscriptions:

```
DELETE https://app.getmax.ai/api/open-apis/webhook/subscription
```

Here, id is the subscription ID returned when creating the subscription.

Example request:

```
POST /api/open-apis/webhook/subscription
Host: app.getmax.com
Authorization: Bearer {{access_token}}
Content-Type: application/json
Accept: application/json
Accept-Charset: utf-8

{
  "id": "xxxxx-xxxx-xxxx-xxxx-xxxxx"
}
```

With the following fields:

| Field         | Type         | Description                                                                                                            |
| --------------|--------------|------------------------------------------------------------------------------------------------------------------------|
| id | string       | ID returned when initiating the subscription |

Possible errors:

| Error code           | Description                                                                                        |
| ---------------------|----------------------------------------------------------------------------------------------------|
| 400 Bad Request      | Required fields were invalid, not specified.                                                       |
| 401 Unauthorized     | The access token is invalid or has been revoked.                                                   |
| 403 Forbidden        | The `ID` in request body doesn’t point to a webhook subscription that the user owned. |


## 4. Testing

We do not have a sandbox environment yet. To test, please feel free to create a testing account. *We recommend you do this by registering using an email address rather than GitHub or Google, as registering with the latter two automatically creates follower relationships on GetMax between your connections on those networks.*

These endpoints will perform actions on production data on `getmax.com`. **Please test with care.**