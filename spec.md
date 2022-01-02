# Specification

v0.1.0, December 2021

**This is a draft version of the spec, and is subject to change.**

Feedback on any part of this is extremely welcome, please create a GitHub issue, or comment on an existing one.

## Table of Contents
- [Specification](#specification)
  - [Table of Contents](#table-of-contents)
  - [Preamble](#preamble)
  - [Terminology](#terminology)
  - [Usernames](#usernames)
  - [Global Usernames](#global-usernames)
  - [User Data](#user-data)
    - [`avatar`](#avatar)
    - [`name`](#name)
    - [`status`](#status)
    - [`emoji`](#emoji)
    - [`media`](#media)
    - [`media_type`](#media_type)
  - [String cleaning](#string-cleaning)
  - [A note on Unicode](#a-note-on-unicode)
  - [HTTP API](#http-api)
    - [Headers](#headers)
    - [Single User Query](#single-user-query)
    - [Batch Query](#batch-query)
    - [Delta Query](#delta-query)
    - [Authentication](#authentication)
    - [Errors](#errors)
    - [Set Status Field(s)](#set-status-fields)
    - [Set Avatar](#set-avatar)


## Preamble

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://tools.ietf.org/html/bcp14) [[RFC2119](https://tools.ietf.org/html/rfc2119)] [[RFC8174](https://tools.ietf.org/html/rfc8174)] when, and only when, they appear in all capitals, as shown here.

## Terminology

**Clients** and **servers** are pieces of software that serve different roles for fmrl. Clients retrieve certain user statuses and do something with them, usually displaying to the user. They can also provide ways to set statuses. Servers implement an API for retrieving and setting statuses.

**Following** a user refers to when a client regularly retrieves updates for that user.

An **update** is when part of a user's data has changed.

## Usernames

Usernames are the only thing that identify one account from another on the same server.

A valid username consists only of ASCII lowercase letters, numbers, and the following characters: `_.` All together, the valid character set is `abcdefghijklmnopqrstuvqxyz0123456789_.` This character set keeps usernames URL-safe, with only one canonical representation, and avoids bringing complexity of Unicode where it is not necessary.

Usernames are limited to 40 characters. Due to the ASCII limitation, this is equivalent to 40 bytes.

Servers MUST reject attempts for account creation with usernames that are invalid, and clients MUST NOT attempt to follow or retrieve updates for a user with an invalid username.

## Global Usernames

To refer to an fmrl user, their full username must be used. Like email, this includes the server. The canonical way of doing this is as follows: `@username@example.com`, where `example.com` is the fmrl server that the user `username` has an account on.

Clients MUST accept these kinds of strings for following or viewing users, and MUST NOT accept other methods.

## User Data

"User data" is all the information needed to display a full status for that user. It's encoded in JSON. Every part of the the API sends out this kind of data, whether on its own for one user, or in an array, etc.

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

Missing fields are equivalent to the zero/empty value for the field. For example, having no `"status"` field is equivalent to `"status": ""`.

All fields are optional.

### `avatar`

The avatar is the profile picture for the user. If the `avatar` key exists, the `original` key MUST exist, which points to the largest and canonical version of the avatar. Servers MUST only serve avatar images which are square. Servers MUST only serve either JPEG or PNG avatar files.

Other valid keys in the `avatar` dictionary provide lower resolution versions of the avatar. Here's an example:

```json
{
    "avatar": {
        "original": "/path/to/avatar.jpg",
        "48x48": "/path/to/avatar-48.jpg",
        "360x360": "/path/to/avatar-large2.png"
    },
    ...
}
```

Other keys MUST be in the format of `NxN`, where `N` is the width and height of the image in pixels.

Clients cannot rely on any keys other than `original` existing however.

Note that the value for all these keys is an absolute path that uses the same domain as the fmrl server. A value like `https://other-server.com/avatar.jpg` is not allowed. Servers MUST only serve absolute paths, and clients MUST NOT accept anything else.

### `name`

The display name of the user, not their username. It is RECOMMENDED clients and servers limit this to 40 UTF-8 codepoints.

### `status`

This is the main part of the data, the actual status line of the user. It is RECOMMENDED clients and servers limit this to 80 UTF-8 codepoints.

### `emoji`

A single [fully-qualified](https://www.unicode.org/reports/tr51/#def_qualified_emoji_character) emoji, as defined by Unicode.

### `media`

Media the user is consuming. It is RECOMMENDED clients and servers limit this to 80 UTF-8 codepoints.

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


## String cleaning

Certain characters are banned in general data strings. Servers MUST reject attempts to set data strings that contain these characters, and clients MUST NOT upload data strings that contain these characters. Clients MAY strip out these characters when they are included, instead of returning an error to the user.

Characters that are banned are those that attempt to control presentation and/or spacing. This are the ASCII and Unicode control characters. Note that this includes commonplace characters like TAB (U+0009) and LINE FEED (U+000A).

Banned code points:

- U+0000—U+001F (C0 controls)
- U+007F DELETE
- U+0080—U+009F (C1 controls)


## A note on Unicode

The string limits of fmrl are expressed in Unicode UTF-8 codepoints. They are not equivalent with bytes, and you must make sure you are counting them correctly.

Counting codepoints is biased depending on the language, as some languages take more codepoints to express an idea than others. One possible solution to this would be to count Unicode graphemes instead, but that is also not equal for all languages, and is not a trivial thing to do in many programming languages.

## HTTP API

The only officially defined API for fmrl is the one defined in this document, that operates over HTTP.

Clients MUST support HTTP and HTTPS, supporting at least TLS 1.2. Servers SHOULD only serve HTTPS, but serving unsecure HTTP is allowed.

Servers MUST support all the GET API calls listed below.

Clients MUST NOT send request URLs longer than 2048 bytes. Servers MAY support URLs longer than this, and shouldn't need to add code that discriminates against longer URLs.

### Headers

Servers MUST include [`Last-Modified`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Last-Modified) in all API responses. Servers also MUST follow the rules of [`If-Modified-Since`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Modified-Since) if the client provides it in the request. Clients MUST be able to handle status code 304 in accordance with the rules of `If-Modified-Since`.

Clients SHOULD include `If-Modified-Since` with every request where the the previous update time is known. The returned `Last-Modified` value from the server can be used for the next request.

### Single User Query

If the server is hosted at `example.com`, the URL to get the status of a the user `bob` looks like

```
http://example.com/fmrl/users/bob
```

Only a GET request is valid.

### Batch Query

A batch query returns data for multiple users.

The client makes a GET request that looks like this:

```
http://example.com/fmrl/users?user=username1&user=username2
```


The server returns the schema described in [§User data](#user-data), but multiple of them, enclosed in a JSON list:

```json
[
    {...},
    {...}
]
```

### Delta Query

A delta query returns data for multiple users, but only the users that have new data. The client MUST make the request with an `If-Modified-Since` header, and the server MUST only return data for users that have been updated since the time in that header.

The GET request looks like:

```
http://example.com/fmrl/new?user=username1&user=username2
```

And the response is the same layout as [§Batch Query](#batch-query).


### Authentication

APIs that allow updating your status on your server must be protected with authentication. To keep things simple, [basic authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization#basic_authentication) is used for each request that changes a user's status data.

Servers MUST return status code 401 for requests that require authentication but don't have it.

Creating an account is handled by each server independently and is not defined here. Servers SHOULD allow users to change their password by proving they have access to another service, like email. This helps keep their account secure if their password is leaked.

### Errors

PUT requests (defined below) change the user's status, and have the potential for errors, on the client or server side.

The server MUST return 200 OK if the request data was set successfully. If there was an error with the client request body, the server MUST return 400. A server error MUST cause 500 to be returned.

In the case of either error, plain text (UTF-8) must be returned that gives a brief explanation of the error. For example:

```
Image not recognized as JPEG or PNG.
```

The client SHOULD then display this error to the user, indicating that the status update failed. Clients shouldn't expect the error description to be only one line long.


### Set Status Field(s)

This request sets one or more fields of the status. It is a PUT request. The body of the request is a JSON document of the same schema as the status. The server MUST replace any fields of the user's status with the fields included in the request body. Any fields not included in the request body MUST remain the same. Any fields in the request body that the server doesn't recognize MUST be ignored, instead of being set in the JSON.

An empty JSON document like `{}` is valid, but of course will do nothing.

If the server is hosted at `example.com`, the URL to set the status of a the user `bob` looks like

```
http://example.com/fmrl/users/bob
```

To change the `status` of Bob, the PUT body would look like:

```json
{
    "status": "My new status"
}
```

The only field this doesn't apply to is the `avatar` field. Servers MUST reject requests that try to set the `avatar` field, and clients MUST NOT send them. This is because there's no way to upload the avatar with this kind of request.

### Set Avatar

To set the avatar for bob, the client makes a PUT request to the following URL:

```
http://example.com/fmrl/users/bob/avatar
```

The body of the PUT request MUST be either a JPEG or PNG image.
