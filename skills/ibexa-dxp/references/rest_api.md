# Ibexa DXP — REST API Reference

Base prefix: `/api/ibexa/v2`
Browse spec: visit `/api/ibexa/v2/doc` in dev environment.
Generate spec: `php bin/console ibexa:openapi --output=openapi.json`

## Authentication

```http
# Basic auth
GET /api/ibexa/v2/content/objects/1
Authorization: Basic base64(login:password)

# Session auth (obtain session cookie first)
POST /api/ibexa/v2/user/sessions
Content-Type: application/vnd.ibexa.api.SessionInput+json
{"SessionInput": {"login": "admin", "password": "pass"}}
# Then use X-CSRF-Token header with returned csrfToken
```

## Key REST endpoints

### Content

```http
# Get content
GET /api/ibexa/v2/content/objects/{contentId}
Accept: application/vnd.ibexa.api.Content+json

# Get content by remoteId
GET /api/ibexa/v2/content/objects?remoteId={remoteId}

# Create content
POST /api/ibexa/v2/content/objects
Content-Type: application/vnd.ibexa.api.ContentCreate+json
{
  "ContentCreate": {
    "ContentType": {"_href": "/api/ibexa/v2/content/types/{typeId}"},
    "mainLanguageCode": "eng-GB",
    "LocationCreate": [{"_href": "/api/ibexa/v2/content/locations/1/2"}],
    "fields": {
      "field": [
        {"fieldDefinitionIdentifier": "title", "languageCode": "eng-GB", "fieldValue": "My Title"}
      ]
    }
  }
}

# Publish version
PUBLISH /api/ibexa/v2/content/objects/{contentId}/versions/{versionNo}

# Update content
PATCH /api/ibexa/v2/content/objects/{contentId}
```

### Locations

```http
GET /api/ibexa/v2/content/locations/{locationPath}   # e.g. /1/2/5
GET /api/ibexa/v2/content/locations/{locationId}/children
```

### Search

```http
POST /api/ibexa/v2/views
Content-Type: application/vnd.ibexa.api.ViewInput+json
{
  "ViewInput": {
    "identifier": "my-view",
    "public": false,
    "LocationQuery": {
      "Filter": {
        "ContentTypeIdentifierCriterion": "article"
      },
      "limit": 10,
      "offset": 0
    }
  }
}
```

### Content types

```http
GET /api/ibexa/v2/content/types
GET /api/ibexa/v2/content/types/{typeId}
GET /api/ibexa/v2/content/typegroups
```

### Users

```http
GET /api/ibexa/v2/user/users/{userId}
GET /api/ibexa/v2/user/groups/{groupPath}
POST /api/ibexa/v2/user/users
```

## SiteAccess in REST

Use `X-Siteaccess: siteaccess_name` header to target a specific SiteAccess.

## Extending the REST API

Create a custom endpoint:

```php
// Controller
use Ibexa\Bundle\Rest\Controller as BaseController;
use Symfony\Component\HttpFoundation\Request;

class MyController extends BaseController
{
    public function myAction(Request $request): Response
    {
        return new CreatedResponse(new Values\MyCustomValue($data));
    }
}
```

```yaml
# routing
my_rest_route:
    path: /api/ibexa/v2/my/resource
    controller: App\REST\Controller\MyController::myAction
    methods: [GET]
```

```php
// Value object
class MyCustomValue extends Value
{
    public function __construct(public readonly array $data) {}
}

// ValueObjectVisitor
use Ibexa\Contracts\Rest\Output\Generator;
use Ibexa\Contracts\Rest\Output\ValueObjectVisitor;

class MyCustomValueVisitor extends ValueObjectVisitor
{
    public function visit(Visitor $visitor, Generator $generator, $data): void
    {
        $generator->startObjectElement('MyCustomValue');
        $generator->valueElement('item', $data->data['key']);
        $generator->endObjectElement('MyCustomValue');
    }
}
```

```yaml
services:
    App\REST\Output\MyCustomValueVisitor:
        tags:
            - { name: ibexa.rest.output.value_object.visitor, type: App\REST\Values\MyCustomValue }
```
