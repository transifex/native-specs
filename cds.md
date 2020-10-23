# 3. Content Delivery Service

[Content Delivery Service](https://docs.transifex.com/transifex-native-sdk-overview/hosting-translations) (CDS) is a web service that retrieves and caches translations from Transifex, in order to later serve them to the Native SDK. It exposes a set of HTTPS endpoints to retrieve and push content or invalidate the cache.

* [3.1 Authentication](#31-authentication)
* [3.2 Languages](#32-languages)
* [3.3 Pull content](#33-pull-content)
* [3.4 Push content](#34-push-content)
* [3.5 Invalidate cache](#35-invalidate-cache)
* [3.6 Purge cache](#36-purge-cache)
* [3.7 Analytics](#37-analytics)

## 3.1 Authentication

Authentication happens through a public project token, that identifies a resource and allows serving of content, and a secret, that can be used for pushing content or other operations that might cause alterations.

All API endpoints authenticate using a Bearer token, provided through an Authorization header:

```bash
# Read-only access to resources
Authorization: Bearer <project-token>

# Read-write access to resources
Authorization: Bearer <project-token>:<secret>
```

## 3.2 Languages

Get a list of languages associated with a content resource.

```bash
GET /languages
Authorization: Bearer <project-token>
Content-Type: application/json; charset=utf-8
```

```bash
Response status: 202
- Content not ready, queued for download from Transifex
- try again later

Response status: 302
- Get content from URL

Response status: 200
Response body:
{
  "data": [
    {
      "name": "<locale_name>",
      "code": "<locale_code>",
      "localized_name": "<localized_version_of_name>",
      "rtl": <bool>
    },
    { ... }
  ],
  "meta": {
    ...
  }
}
```

## 3.3 Pull content

Get localized content for a specific language code.

```bash
GET /content/<locale_code>
Authorization: Bearer <project-token>
Content-Type: application/json; charset=utf-8
```

```bash
Response status: 202
- Content not ready, queued for download from Transifex
- try again later

Response status: 302
- Get content from URL

Response status: 200
Response body:
{
  "data": {
    "<key>": {
      "string": "<string>"
    }
  },
  "meta": {
    ...
  }
}
```

## 3.4 Push content

Push source content.

If purge: true in meta object, then replace the entire resource content with the pushed content of this request.

If purge: false in meta object (the default), then append the source content of this request to the existing resource content.

```bash
POST /content
Authorization: Bearer <project-token>:<secret>
Content-Type: application/json; charset=utf-8
```

```bash
Request body:
{
  "data": {
    "<key>": {
      "string": "<source_string>",
      "meta": {
        "context": [<string>, ...]
        "developer_comment": "<string>",
        "character_limit": "<int>",
        "tags": ["<string>", ...],
      }
    }
    "<key>": { ... }
  },
  "meta": {
    "purge": <bool>
  }
}

Response body:
{
  "created": <int>,
  "updated": <int>,
  "skipped": <int>,
  "deleted": <int>,
  "failed": <int>,
  "errors": ["<string>", ...],
}
```

## 3.5 Invalidate cache

Endpoint to force cache invalidation for a specific language or for
all project languages. Invalidation triggers background fetch of fresh
content for languages that are already cached in the service.

```bash
POST /invalidate
POST /invalidate/<locale_code>
Authorization: Bearer <project-token>:<secret>
Content-Type: application/json; charset=utf-8
```

```bash
Request body:
{}

Response body (success):
{
  "status": "success",
  "token": "<project-token>",
  "count": "<number_of_resources_invalidated>",
}

Response body (fail):
{
  "status": "failed",
}
```

## 3.6 Purge cache

Endpoint to purge cache for a specific resource content.

```bash
POST /purge
POST /purge/<locale_code>
Authorization: Bearer <project-token>:<secret>
Content-Type: application/json; charset=utf-8
```

```bash
Request body:
{}

Response body (success):
{
  "status": "success",
  "token": "<project-token>",
  "count": "<number_of_resources_purged>",
}

Response body (fail):
{
  "status": "failed",
}
```

## 3.7 Analytics

Endpoint to get usage analytics, per language, SDK and unique anonymized clients.

```bash
GET /analytics?filter[since]=<YYYY-MM-DD>&filter[until]=<YYYY-MM-DD>
Authorization: Bearer <project-token>:<secret>
Content-Type: application/json; charset=utf-8
```

```bash
Response body:
{
  "data": [{
    "languages": {
      "<locale_code>": "<number_of_hits>",
      ...
    },
    "sdks": {
      "<sdk-version>": "<number_of_hits>",
      ...
    },
    "clients": "<number_of_unique_clients>",
    "date": "<YYYY-MM-DD or YYYY-MM>",
  }, ...],
  "meta": {
    "total": {
      "languages": {
        "<locale_code>": "<total_number_of_hits>",
        ...
      },
      "sdks": {
        "<sdk-version>": "<total_number_of_hits>",
        ...
      },
      "clients": "<total number_of_unique_clients>",
    },
  },
}
```

<div class="article-links">
    <a href="implementation_guide.html">← Implementation guide</a>
</div>
