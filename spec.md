# Specification

v0.0.0 (versioning coming soon)

**This is a draft version of the spec, and is subject to change.**

Feedback on any part of this is extremely welcome, please create a GitHub issue, or comment on an existing one.

## Table of Contents
- [Specification](#specification)
  - [Table of Contents](#table-of-contents)
  - [Preamble](#preamble)
  - [Terminology](#terminology)
  - [Usernames](#usernames)
  - [Global Usernames](#global-usernames)
  - [User Data or Status](#user-data-or-status)
    - [`avatar`](#avatar)
    - [`name`](#name)
    - [`status`](#status)
    - [`emoji`](#emoji)
    - [`media`](#media)
    - [`media_type`](#media_type)
  - [String cleaning](#string-cleaning)
  - [Unicode Code Points](#unicode-code-points)
  - [Unicode Normalization](#unicode-normalization)
  - [HTTP API](#http-api)
    - [Headers](#headers)
    - [Cross-Origin Resource Sharing (CORS)](#cross-origin-resource-sharing-cors)
    - [Status Codes and Errors](#status-codes-and-errors)
    - [Authentication](#authentication)
    - [Status API](#status-api)
      - [Status Query](#status-query)
      - [Set Status Field(s)](#set-status-fields)
      - [Set Avatar](#set-avatar)
    - [Following API](#following-api)
      - [Get Following](#get-following)
      - [Set Following](#set-following)
  - [Client Behavior](#client-behavior)
    - [Status Storage](#status-storage)
    - [Automatic Retrieval of Statuses](#automatic-retrieval-of-statuses)
    - [Manual Update](#manual-update)
  - [Server Behavior](#server-behavior)


## Preamble

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] when, and only when, they appear in all capitals, as shown here.

## Terminology

**Clients** and **servers** are pieces of software that serve different roles for fmrl. Clients retrieve certain user statuses and do something with them, usually displaying to the user. They can also provide ways to set statuses. Servers implement an API for retrieving and setting statuses.

**Following** a user refers to when a client regularly retrieves updates for that user.

An **update** is when part of a user's data has changed.

## Usernames

Usernames are the only thing that identify one account from another on the same server.

A valid username consists only of ASCII lowercase letters, numbers, and the following characters: `_.` All together, the valid character set is `abcdefghijklmnopqrstuvwxyz0123456789_.` This character set keeps usernames URL-safe, with only one canonical representation, and avoids bringing complexity of Unicode where it is not necessary.

Usernames are limited to 40 characters. Due to the ASCII limitation, this is equivalent to 40 bytes.

Servers MUST reject attempts for account creation with usernames that are invalid, and clients MUST NOT attempt to follow, retrieve updates, or upload data for a user with an invalid username. This includes invalid length.

## Global Usernames

To refer to an fmrl user, their full username must be used. Like email, this includes the server. The canonical way of doing this is as follows: `@username@example.com`, where `example.com` is the fmrl server that the user `username` has an account on. Note the leading `@`, which differentiates it from email, and is more similar to social media handles.

The other valid way of doing this is: `fmrl:username@example.com`. Note the leading `@` is dropped. This is a URI, so this can be used to link to fmrl users in Web pages or other hyperlinked media. fmrl clients can then choose to support opening these kinds of URIs, to display the user's status, and/or ask if you'd like to follow them.

Clients MUST accept these kinds of strings for following or viewing users, and MUST NOT accept other methods.

## User Data or Status

"User data", or the status, is all the information needed to display a full status for that user. It's encoded in JSON. Every part of the the API sends out this kind of data, whether on its own for one user, or in an array, etc.

In lieu of a proper JSON schema, here is an example layout of the data, with every valid field filled.

```json
{
    "avatar": {
        "original": "/path/to/avatar.jpg"
    },
    "name": "Jacob Jingleheimer",
    "status": "Just grooving",
    "emoji": "🤓",
    "media": "Lord of The Rings",
    "media_type": 2
}
```

Any fields that do not appear here have an undefined meaning and purpose and SHOULD be ignored by clients.

Missing fields are equivalent to the zero/empty value for the field. For example, having no `"status"` field is equivalent to `"status": ""`. The same goes for the JSON `null` value, it is equivalent to an empty string, or number zero, etc.

Any field can be missing, empty, or null and the user data will still be valid. Therefore, `{}` is a valid status, there just isn't much going on.

Clients MUST support setting and getting all the fields above.

Servers MAY choose to not support some, like avatars or emoji. So any of the field-specific requirements below for servers only apply if servers choose to support that field. Servers SHOULD support all fields, however.

If a server chooses not to support a field, it MUST silently drop the field, and MUST NOT simply set the field without validation or something similar.

### `avatar`

The avatar is the profile picture for the user. If the `avatar` key exists and is not `null`, the `original` key MUST exist, which points to the largest and canonical version of the avatar. Servers MUST only serve avatar images which are square. Servers MUST only serve either JPEG or PNG avatar files.

Other valid keys in the `avatar` dictionary provide lower resolution versions of the avatar. Here's an example:

```json
{
    "avatar": {
        "original": "/path/to/avatar.jpeg",
        "48": "/path/to/avatar-48",
        "360": "/path/to/avatar-large2.png"
    },
    ...
}
```

Other keys MUST be in the format of `N`, where `N` is the width and height of the image in pixels.

Clients MUST NOT rely on any keys other than `original` existing however.

Note that the value for all these keys is an absolute path that uses the same domain as the fmrl server. A value like `https://other-server.com/avatar.jpg` is not allowed. Servers MUST only serve absolute paths, and clients MUST NOT accept anything else. In other words, the values of keys in `avatar` MUST always begin with `/` and be resolved relative to the domain of the server. Any avatar strings not starting with `/` MUST be ignored.

Clients MUST NOT rely on entries in the dictionary ending with a file extension. The only valid way to determine the image is JPEG or PNG, if required, is by the magic number in the first few bytes of the file.

Servers MUST change all avatar strings when the avatar image changes. The only valid way for clients to know if the avatar has changed is to compare the avatar string to previously cached one. If the avatar *path* must stay the same for simplicity, the server can simply add a query string like `/avatar.png?341` that will be ignored by the server. This still changes the avatar *string*, so it works. Previous avatar strings SHOULD NOT be reused within a reasonable length of time for a client to be open without updating, perhaps 30 minutes, so that clients are aware of all avatar updates. Not reusing also works fine.

### `name`

The display name of the user, not their username.

Servers MUST limit this to 40 UTF-8 code points, returning code 400 to clients that try to set a longer one. Clients MUST NOT attempt to set a longer `name`. Clients SHOULD truncate any received `name` that is longer.

### `status`

This is the main part of the data, the actual status line of the user.

Servers MUST limit this to 100 UTF-8 code points, returning code 400 to clients that try to set a longer one. Clients MUST NOT attempt to set a longer `status`. Clients SHOULD truncate any received `status` that is longer.

### `emoji`

A single [fully-qualified](https://www.unicode.org/reports/tr51/#def_qualified_emoji_character) emoji, as defined by Unicode.

Servers MUST validate this. Clients MUST validate this, both when setting and getting statuses.

The only way to validate emojis is to compare them to a list of all emojis. A list of all valid emojis with code points can be found [here](https://unicode.org/Public/emoji/14.0/emoji-test.txt). Note the emoji version number in the URL, that can be changed to find the new file as new versions are released. Also note that list contains emojis that are not fully-qualified.

### `media`

Media the user is consuming.

Servers MUST limit this to 100 UTF-8 code points, returning code 400 to clients that try to set a longer one. Clients MUST NOT attempt to set a longer `media`. Clients SHOULD truncate any received `media` that is longer.

### `media_type`

The kind of media the `media` string defines. This can be used to display an icon beside the `media` string.

Registered types of media:

1. Text (book, article, etc)
2. Movie
3. TV Show
4. Music
5. Speech (podcast, radio program, etc)
6. Game (video game, board game, card game, etc)

Number 0 means the media type is not specified by the user. Perhaps no icon should be displayed.

Servers MUST return code 400 to clients that try to set a `media_type` value outside of the range above.

## String cleaning

Certain characters are banned in user data strings. Servers MUST reject attempts to set data strings that contain these characters, and clients MUST NOT upload data strings that contain these characters. Clients MAY strip out these characters when user attempts to upload them, instead of returning an error to the user. Clients MUST strip these out from statuses it downloads.

Characters that are banned are those that attempt to control presentation and/or spacing. This are the ASCII and Unicode control characters. Note that this includes commonplace characters like TAB (U+0009) and LINE FEED (U+000A).

Banned code points:

- U+0000—U+001F (C0 controls)
- U+007F DELETE
- U+0080—U+009F (C1 controls)


## Unicode Code Points

The string limits of fmrl are expressed in Unicode UTF-8 code points. These are not equivalent with bytes, and you must make sure you are counting them correctly.

Counting code points is biased depending on the language, as some languages take more code points to express an idea than others. One possible solution to this would be to count Unicode graphemes instead, but that is also not equal for all languages, and is not a trivial thing to do in many programming languages.

## Unicode Normalization

Sometimes in Unicode there are multiple ways of writing the same grapheme. For example, `é` can be written as `U+00E9`, or `U+0065 U+0301`, which combines `e` with the accent alone: `◌́`.

Unicode has several methods of normalizing these different styles, and the one that reduces the code points used is called NFKC. Applying NFKC normalization to a string will combine code points when possible.

Servers MAY do this to incoming user strings before length validation and storage. Clients MAY do this to user strings before sending them to a server, but SHOULD NOT do it to incoming strings from other users.

It's a nice thing to do if there is a library available and you don't mind increasing your application size somewhat.

## HTTP API

The only officially defined API for fmrl is the one defined in this document, that operates over HTTP. Servers and clients MUST support it to work within the fmrl ecosystem. Some parts of the API are optional, and this is noted for those sections.

Servers and clients MUST support HTTP/1.1, and MAY support later versions such as HTTP/2 or HTTP/3.

Clients MUST support HTTP and HTTPS, supporting at least TLS 1.2. Servers MUST support TLS 1.2 at least. Servers SHOULD only serve HTTPS, but serving unsecure HTTP is allowed.

Clients SHOULD try HTTPS first, then HTTP if it fails, to prevent downgrade attacks.

Clients MUST NOT send request URLs longer than 2048 bytes. Servers MAY support URLs longer than this, and shouldn't need to add code that discriminates against longer URLs.

The API MUST be available on port 80 (HTTP) and/or port 443 (HTTPS). Other ports are not supported, as there is no way to specify them in a global username.

Clients MUST NOT follow redirects, and servers MUST NOT offer them.

### Headers

This section only applies to GET requests.

Servers MUST include [`Last-Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified) in all API responses. Servers also MUST follow the rules of [`If-Modified-Since`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Modified-Since) if the client provides it in the request.

Clients SHOULD include `If-Modified-Since` with every request where the the previous update time is known. The returned `Last-Modified` value from the server can be used for the next request.

Clients MUST be able to handle status code 304 in accordance with the rules of `If-Modified-Since`, if they intend to send it.

### Cross-Origin Resource Sharing (CORS)

> Cross-origin resource sharing (CORS) is a mechanism that allows restricted resources on a web page to be requested from another domain outside the domain from which the first resource was served.

- [Wikipedia](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)

To allow in-browser fmrl clients to make requests to servers, fmrl servers MUST support CORS. CORS is acheived through setting certain headers. You can (and should!) read more about CORS [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), but everything a server needs to do to be compliant is explained below.

These instructions apply to any request that is made with GET. Requests that are for updating data (using PATCH or PUT) SHOULD NOT set up CORS like below, if at all.

The GET request path MUST also support OPTIONS. When OPTIONS requests are made, the response is the same every time. Status code 204 with the following headers and no body:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Headers: If-Modified-Since
Access-Control-Max-Age: 86400
```

And then responses to GET requests MUST always have the following header included:

```
Access-Control-Allow-Origin: *
```

Following these requirements is all that is needed for in-browser clients to work with your server, which is required.

### Status Codes and Errors

Servers may encounter errors when a client sets or gets data, and these errors should be able to be communicated back to the user.

If the request was successful, servers MUST return 200 or 304.

If there was an error with the client request, the server MUST return a 4xx status code. A server-side error MUST cause 5xx status code to be returned.  For example, servers SHOULD return 405 (Method Not Allowed) for requests using methods not defined in this document.

All other status codes are banned and MUST NOT be used.

In the case of any error, plain text (UTF-8) SHOULD be returned that gives a brief explanation of the error. For example:

```
Image not recognized as JPEG or PNG.
```

The client SHOULD then display this error to the user, indicating that something failed. Clients shouldn't expect the error description to be only one line long, or to exist at all.

### Authentication

APIs that allow updating data on the server must be protected with authentication. To keep things simple, [basic authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization#basic_authentication) is used for each request that changes a user's status data.

Servers MUST return status code 401 for requests that require authentication but don't have it, or if the password is incorrect.

The username in the basic auth header and the username for which a request is being made MUST match.

Clients SHOULD strongly warn users before sending their password over HTTP, as opposed to HTTPS.

Creating an account is handled by each server independently and is not defined here. Servers SHOULD allow users to change their password by proving they have access to another service, like email. This helps keep their account secure if their password is discovered or leaked.

Servers MAY accept "tokens" instead of the actual account password for the user, still using basic auth. A token system allows for each application to have its own token, which can be revoked individually by the user if the application is hacked or misbehaves, or if the token is leaked.

It also allows for tokens with limited scope, for example a token that can only update the media fields. This could be given to a service that tracks what you're listening to and updates your status with it.


### Status API

The Status API is made up of three API calls for working with statuses: Status Query, Set Status, and Set Avatar. These are defined below.

Servers and client MUST support the Status API.

#### Status Query

A status query returns data for one or more users.

The client makes a GET request that looks like this:

```
https://example.com/.well-known/fmrl/users?user=username1&user=username2
```

Here is an example output:

```json
[
    {
        "username": "bob",
        "code": 200,
        "data": {
            "avatar": {
                "original": "/path/to/avatar.jpg"
            },
            "name": "Jacob Jingleheimer",
            "status": "Just grooving",
            "emoji": "🤓",
            "media": "Lord of The Rings",
            "media_type": 2
        }
    },
    {
        "username": "alice",
        "code": 404,
        "msg": "User not found",
    },
    {
        "username": "example",
        "code": 304
    }
]
```

The `username` field MUST be included.

The `code` field MUST be included, as the HTTP status code that is appropriate. If that status code represents an error, the `msg` field SHOULD be included with a description of the error. See [§Status Codes and Errors](#status-codes-and-errors) for details.

The `data` field is the same format as [§User data](#user-data). If the `code` is not an error, the field MUST exist and be of the dictionary type. If the `code` is an error, `data` MUST NOT exist.

Clients MUST NOT include usernames more than once, such as `?user=bob&user=alice&user=bob`. Servers MAY deduplicate usernames to avoid setting duplicate keys in the returned JSON. Servers MAY also generate the JSON normally, including duplicate entries.

All non-duplicated usernames the client requested MUST appear in the returned JSON. They MAY appear out-of-order as compared to the order in the URL query.

If the client makes the request with an `If-Modified-Since` header, the server MUST only return `data` for users that have been updated since the time in that header. Users whose status was updated previous to that time MUST have the `username` field, `"code": 304`, and nothing else. The `msg` field MUST NOT appear in the instance of `code` 304.

The client can ensure all (non-error) users will have `data` simply by not including the `If-Modified-Since` header.

The `Last-Modified` header MUST be included in the response from the server, and MUST be set to the most recent updated time out of all the statuses. If none of the requested users can be retrieved successfully (no 200 or 304 codes), then `Last-Modified` MUST be set to the same value as the `If-Modified-Since` header the client set. If `If-Modified-Since` does not exist, the `Last-Modified` header MUST be set to the Unix epoch. Clients SHOULD NOT use the `Last-Modified` value from servers if all the users return errors, but these precautions are set in place for any clients that might be doing that accidentally.

A request to `/.well-known/fmrl/users` with no `user` query fields is invalid and SHOULD cause code 400 to be returned.

Since the `code` field exists, servers MUST always return 200 OK, unless no usernames are specified or the URL path is incorrect.

#### Set Status Field(s)

This request sets one or more fields of the status. It is a PATCH request. The body of the request is a JSON document of the same schema as the status. The server MUST replace any fields of the user's status with the fields included in the request body. Any fields not included in the request body MUST remain the same. Any fields in the request body that the server doesn't recognize MUST NOT be set in the JSON.

Servers SHOULD return code 400 with error text if the client body tries to set fields the server doesn't support.

An empty JSON document like `{}` is valid, but of course will do nothing. It SHOULD NOT update the last modified time of the status.

An empty body is not valid, and servers MUST return an error, like status code 400.

If the server is hosted at `example.com`, the URL to set the status of the user `bob` looks like

```
https://example.com/.well-known/fmrl/user/bob
```

To change the `status` of Bob, the PATCH body would look like:

```json
{
    "status": "My new status"
}
```

The only field this doesn't apply to is the `avatar` field. Servers MUST reject requests that try to set the `avatar` field, and clients MUST NOT send them. This is because there's no way to upload the avatar with this kind of request.

#### Set Avatar

To set the avatar for bob, the client makes a PUT request to the following URL:

```
https://example.com/.well-known/fmrl/user/bob/avatar
```

If the server doesn't support setting the avatar, it SHOULD return 404 to all requests on this kind of path. Clients can use this to inform users that setting their avatar won't work due to lack of server support.

The body of the PUT request MUST be either a JPEG or PNG image. Empty bodies are invalid.

Assuming the image was received and processed correctly, the server MUST return 200, and then MUST set the `original` key of the user JSON so that it points to where this avatar will be served from.

This processing MUST include verifying the image is either JPEG or PNG. The server MUST ensure the image dimensions are square as well. If these requirements are too intensive for the server they should not support avatars at all.

The server MAY also derive alternate resolutions of the avatar at this point and set those keys as well.

Servers SHOULD limit the request body to 4 MiB, rejecting larger images with code 413. Clients SHOULD NOT try to set avatars with images larger than 4 MiB.

A DELETE request can also be made, and the server MUST remove any existing avatar image, remove any avatar keys from the status, and return 200.

### Following API

The Following API allows the accounts a user follows to be synchronized across clients. This is useful for users that use multiple clients, such as one on their mobile device and one on their desktop.

Clients and servers MAY support this. It's a nice feature, but not a big deal in most cases as users likely won't have a large following list like with traditional social media.

If the client supports the Following API, it can assume the server does not support it if the Get Following API call returns a 404 error. Clients SHOULD still make the Get Following request every so often in case the server adds the API.

All requests in the Following API MUST be authenticated, as defined in [§Authentication](#authentication).

The Following API is defined in the subsections below.

#### Get Following

This GET request returns all the accounts the user is following. The path for `bob` on `example.com` looks like:

```
https://example.com/.well-known/fmrl/user/bob/following
```

The response is a JSON array of global usernames as the body.

```json
[
    "@alice@her-server.com",
    "@test@bigbox.net",
    "@mallory@wack.cat"
]
```

Only global usernames in the format of `@username@example.com` are allowed.

If the server has no information on who the user follows but still supports this API, the response MUST be an empty array: `[]`.

This server MUST follow the rules in [§Headers](#headers) for this API as well, seeing as it's GET. That means the last updated time of the entire follower list as whole must be tracked.

This request path SHOULD NOT implement CORS, despite being GET, because it is authenticated. The CORS from [§CORS](#cross-origin-resource-sharing-cors) does not work with authenticated requests.

#### Set Following

This is a PATCH request that adds or removes accounts the user follows from the list. The URL is the same as [§Get Following](#get-following).

Here is an example body:

```json
{
    "add": [
        "@newuser@myhost.com"
    ],
    "remove": [
        "@badfriend@bad.io",
        "@worsefriend@superbad.com"
    ]
}
```

Servers apply these instructions against the following list.

Servers MUST respond with 200 if the JSON was parsed and set properly. The new `Last-Modified` header for the following list MUST also be included, which should of course just be the current time.

## Client Behavior

This section provides guidelines for how clients should behave.

### Status Storage

Clients SHOULD NOT retain statuses in long term storage. For example, no statuses should be available immediately after opening a GUI desktop client. The client would need to request statuses from the servers again. This prevents stale statuses from staying too long, or staying even if the user's account is gone.

Clients MUST NOT keep previous statuses for users after receiving an update.

### Automatic Retrieval of Statuses

Some clients may be designed to automatically request status updates in the background. Clients SHOULD stop automatically requesting a user that returns a 4xx status code. Other status codes such as 5xx SHOULD NOT cause this. The client MAY check the 4xx user again after some longer interval, such as when the application is re-opened.

If the client automatically retrieve statuses, it SHOULD also retrieve the status of the user using the client regularly, possibly more regularly than other statuses.. This is because there may be other clients running that have changed that user's status.

For clients that support it, automatic retrieval of the following list SHOULD also happen.

### Manual Update

If the user clicks some sort of manual update or "refresh" button, that SHOULD update: the statuses of the accounts the user follows, the user's status, and the following list of the user.


## Server Behavior

Servers SHOULD NOT have a list of usernames available, to protect the privacy of user accounts.
