---
name: ibexa-dxp
description: >
  Expert-level Ibexa DXP (formerly eZ Platform / eZ Publish) development assistant.
  Use this skill for ANY task involving an Ibexa DXP codebase — content management,
  PHP API usage, data migrations, SiteAccess configuration, templating, search, commerce,
  PIM, permissions, custom field types, Page Builder blocks, events, workflows, and more.
  Trigger whenever the user mentions: ibexa, ibexa dxp, ibexa experience, ibexa commerce,
  ez platform, ezpublish, ContentService, LocationService, ibexa_string, ibexa_richtext,
  ibexa:migrations:migrate, siteaccess, content_view, ibexadesign, content type creation,
  ibexa repository, or any Ibexa-specific namespace (Ibexa\Contracts, Ibexa\Core, etc.).
  Even if they just say "I'm working on an Ibexa project", use this skill.
---

# Ibexa DXP Development Guide

## Reference files (read when needed)

- `references/search_criteria.md` — Full criteria, sort clauses, aggregations, BatchIterator
- `references/events.md` — All content/location/content-type/workflow events with properties
- `references/extensibility.md` — Custom query types, Page Builder blocks, field types, menus, tabs, SiteAccess-aware config
- `references/rest_api.md` — REST API endpoints, authentication, extending

---

## Architecture overview

Ibexa DXP is a Symfony-based PHP CMS/DXP.

- **Repository pattern**: All content operations go through services (`ContentService`, `LocationService`, etc.). Always inject via Symfony DI using `Ibexa\Contracts\Core\Repository\*` interfaces.
- **Value objects are immutable**: Use struct classes (`ContentCreateStruct`, `ContentUpdateStruct`) to modify, then persist via the service.
- **SiteAccess-aware config**: `ibexa.system.<scope>.*` keys are scoped to a SiteAccess, group, `default`, or `global`.
- **Design engine**: Templates live in `templates/themes/<theme>/`, referenced as `@ibexadesign/path/template.html.twig`.
- **Namespaces**: `Ibexa\Contracts\*` (public API), `Ibexa\Core\*` (internals), `Ibexa\Bundle\*` (Symfony bundles).
- **Editions**: Ibexa Headless (OSS core), Ibexa Experience (+ Page Builder, Site Factory), Ibexa Commerce (+ PIM, Cart, Checkout, Orders).

---

## Content model

| Concept | Key properties |
|---|---|
| **ContentInfo** | `id`, `name`, `contentTypeId`, `mainLocationId`, `remoteId`, `sectionId`, `mainLanguageCode`, `alwaysAvailable`, `publishedDate`, `modificationDate`, `status` (0=draft, 1=published, 2=archived) |
| **Content** | Full object with field values; loaded with `ContentService::loadContent($id)` |
| **Location** | Tree node. Content can have multiple locations. `pathString` = `/1/2/5/42/`. |
| **ContentType** | Blueprint: `identifier`, `nameSchema` (`<title>`), `urlAliasSchema`, field definitions. |
| **FieldDefinition** | `identifier`, `fieldTypeIdentifier`, `isRequired`, `isTranslatable`, `isSearchable`. |
| **VersionInfo** | `STATUS_DRAFT`, `STATUS_PUBLISHED`, `STATUS_ARCHIVED`. Default archive limit: 5. |

---

## PHP API — reading content

```php
// Inject via constructor:
// ContentService, LocationService, ContentTypeService, SearchService, UserService, PermissionResolver

// Load content
$content = $this->contentService->loadContent($contentId);
$contentInfo = $this->contentService->loadContentInfo($contentId);
$content = $this->contentService->loadContentByRemoteId('my-remote-id');
$content = $this->contentService->loadContent($contentId, ['ger-DE']); // specific language
$content = $this->contentService->loadContent($contentId, Language::ALL); // all languages

// Field value
$fieldValue = $content->getFieldValue('title');
$fieldValue = $content->getFieldValue('title', 'fre-FR'); // specific language

// Locations
$location = $this->locationService->loadLocation($locationId);
$locations = $this->locationService->loadLocations($contentInfo);
$children = $this->locationService->loadLocationChildren($location, $offset, $limit);
foreach ($children->locations as $child) { /* ... */ }
$urlAlias = $this->urlAliasService->reverseLookup($location);

// Versions
$versionInfos = $this->contentService->loadVersions($contentInfo);
$archived = iterator_to_array($this->contentService->loadVersions($contentInfo, VersionInfo::STATUS_ARCHIVED));

// Relations
$versionInfo = $this->contentService->loadVersionInfo($contentInfo);
$count = $this->contentService->countRelations($versionInfo);
$list = $this->contentService->loadRelationList($versionInfo, 0, $count);

// Content type
$ct = $this->contentTypeService->loadContentTypeByIdentifier('article');
```

---

## PHP API — creating & updating content

```php
// Set current user (important for permissions)
$user = $this->userService->loadUserByLogin('admin');
$this->permissionResolver->setCurrentUserReference($user);

// Create
$contentType = $this->contentTypeService->loadContentTypeByIdentifier('article');
$createStruct = $this->contentService->newContentCreateStruct($contentType, 'eng-GB');
$createStruct->setField('title', 'My Article');
$createStruct->remoteId = 'unique-remote-id'; // optional

$locationCreateStruct = $this->locationService->newLocationCreateStruct($parentLocationId);
$draft = $this->contentService->createContent($createStruct, [$locationCreateStruct]);
$content = $this->contentService->publishVersion($draft->versionInfo);

// Update
$contentInfo = $this->contentService->loadContentInfo($contentId);
$draft = $this->contentService->createContentDraft($contentInfo);
$updateStruct = $this->contentService->newContentUpdateStruct();
$updateStruct->initialLanguageCode = 'eng-GB';
$updateStruct->setField('title', 'New Title');
$updateStruct->setField('title', 'Neuer Titel', 'ger-DE'); // add translation
$draft = $this->contentService->updateContent($draft->versionInfo, $updateStruct);
$this->contentService->publishVersion($draft->versionInfo);

// Image field
use Ibexa\Core\FieldType\Image\Value as ImageValue;
$createStruct->setField('image', new ImageValue([
    'path' => '/path/to/image.jpg',
    'fileSize' => filesize('/path/to/image.jpg'),
    'fileName' => 'image.jpg',
    'alternativeText' => 'Alt text',
]));

// Create content type
$ctCreateStruct = $this->contentTypeService->newContentTypeCreateStruct('blog_post');
$ctCreateStruct->mainLanguageCode = 'eng-GB';
$ctCreateStruct->nameSchema = '<title>';
$ctCreateStruct->names = ['eng-GB' => 'Blog Post'];
$fieldCreateStruct = $this->contentTypeService->newFieldDefinitionCreateStruct('title', 'ibexa_string');
$fieldCreateStruct->names = ['eng-GB' => 'Title'];
$fieldCreateStruct->isRequired = true;
$fieldCreateStruct->isTranslatable = true;
$fieldCreateStruct->isSearchable = true;
$fieldCreateStruct->position = 10;
$ctCreateStruct->addFieldDefinition($fieldCreateStruct);
$group = $this->contentTypeService->loadContentTypeGroupByIdentifier('Content');
$ctDraft = $this->contentTypeService->createContentType($ctCreateStruct, [$group]);
$this->contentTypeService->publishContentTypeDraft($ctDraft);
```

---

## PHP API — locations & sudo

```php
// Move, copy, hide
$this->locationService->moveSubtree($sourceLocation, $targetLocation);
$this->locationService->copySubtree($sourceLocation, $targetLocation);
$this->locationService->hideLocation($location);
$this->locationService->unhideLocation($location);

// Trash
$this->trashService->trash($location);
$this->trashService->recover($trashItem); // restore to original location
$this->trashService->recover($trashItem, $newParentLocation);

// Delete (removes content if last location)
$this->locationService->deleteLocation($location);

// Sudo — bypass permission check
$result = $this->repository->sudo(function (Repository $repo) use ($id) {
    return $repo->getContentService()->loadContent($id);
});
```

---

## Search API

```php
use Ibexa\Contracts\Core\Repository\Values\Content\LocationQuery;
use Ibexa\Contracts\Core\Repository\Values\Content\Query;
use Ibexa\Contracts\Core\Repository\Values\Content\Query\Criterion;
use Ibexa\Contracts\Core\Repository\Values\Content\Query\SortClause;

$query = new LocationQuery();
$query->filter = new Criterion\ContentTypeIdentifier('article');  // use filter (not query) unless relevance scoring needed
$query->limit = 10;
$query->offset = 0;
$result = $this->searchService->findContentInfo($query);  // lighter — use for lists
$result->totalCount;
foreach ($result->searchHits as $hit) { $hit->valueObject; /* ContentInfo */ }

// Full content (loads fields — heavier)
$result = $this->searchService->findContent($query);

// Complex query
$query->query = new Criterion\LogicalAnd([
    new Criterion\Subtree($location->pathString),
    new Criterion\ContentTypeIdentifier(['article', 'blog_post']),
    new Criterion\FullText('search term'),
    new Criterion\Visibility(Criterion\Visibility::VISIBLE),
    new Criterion\LogicalNot(new Criterion\SectionIdentifier('Media')),
]);
$query->sortClauses = [
    new SortClause\DatePublished(Query::SORT_DESC),
    new SortClause\ContentName(Query::SORT_ASC),
];

// Repository filter (DB-direct, no search index required)
use Ibexa\Contracts\Core\Repository\Values\Filter\Filter;
$filter = (new Filter())
    ->withCriterion(new Criterion\ParentLocationId($parentId))
    ->withSortClause(new SortClause\ContentName(Query::SORT_DESC))
    ->sliceBy(10, 0);
$result = $this->contentService->find($filter);

// See references/search_criteria.md for full criteria/sort clause list
```

---

## Template configuration

```yaml
ibexa:
    system:
        site_group:
            pagelayout: '@ibexadesign/pagelayout.html.twig'
            content_view:
                full:
                    article:
                        template: '@ibexadesign/full/article.html.twig'
                        match:
                            Identifier\ContentType: article
                    folder:
                        template: '@ibexadesign/full/folder.html.twig'
                        controller: App\Controller\FolderController::viewAction
                        match:
                            Identifier\ContentType: folder
                    # Multiple matchers: ALL must match (AND logic)
                    news_article:
                        template: '@ibexadesign/full/news_article.html.twig'
                        match:
                            Identifier\ContentType: article
                            Identifier\Section: news
                    default:
                        template: '@ibexadesign/full/default.html.twig'
                        match: ~  # matches everything
                line:
                    article_line:
                        template: '@ibexadesign/line/article.html.twig'
                        match:
                            Identifier\ContentType: article
```

Built-in view types: `full`, `line`, `embed`, `embed-inline`, `text_linked`, `asset_image`.
Built-in matchers: `Identifier\ContentType`, `Identifier\Section`, `Id\Content`, `Id\Location`, `Id\ContentType`, `Depth`, `UrlAlias`.

---

## Twig — essential functions

```twig
{# Render content/location #}
{{ ibexa_render(content) }}
{{ ibexa_render(location, {'viewType': 'line'}) }}
{{ ibexa_render(content, {'method': 'esi', 'viewType': 'embed'}) }}

{# Field rendering #}
{{ ibexa_render_field(content, 'title') }}
{{ ibexa_render_field(content, 'image', {'parameters': {'alias': 'medium'}}) }}
{{ ibexa_content_name(content) }}
{{ ibexa_content_name(content, 'fre-FR') }}

{# Field checks #}
{% if not ibexa_is_field_empty(content, 'body') %}...{% endif %}
{% set value = ibexa_field_value(content, 'title') %}

{# Image variation #}
{% set imageAlias = ibexa_image_alias(content.getField('image'), content.versionInfo, 'large') %}
<img src="{{ imageAlias.uri }}" alt="{{ imageAlias.alternativeText }}">

{# URL #}
{{ ibexa_url(location) }}

{# SEO #}
{{ ibexa_seo(content) }}

{# Taxonomy #}
{{ content|ibexa_taxonomy_entries_for_content|map(e => e.name)|join(', ') }}

{# Query in template #}
{{ ibexa_render_location_query({'query': {'query_type': 'LatestArticles', 'parameters': {'limit': 5}, 'assign_results_to': 'items'}}) }}
```

---

## Design engine

```yaml
ibexa_design_engine:
    design_list:
        my_design: [my_theme, standard]  # fallback order

ibexa:
    system:
        site_group:
            design: my_design
```

Templates: `templates/themes/my_theme/` or `Resources/views/themes/my_theme/` in bundles. Reference via `@ibexadesign/`.

---

## SiteAccess configuration

```yaml
ibexa:
    siteaccess:
        list: [site, admin, fr]
        groups:
            site_group: [site, fr]
            admin_group: [admin]
        default_siteaccess: site
        match:
            Map\URI: { fr: fr, admin: admin }
            # Also: Map\Host, HostElement, URIElement, URIText, Map\Port, Compound\LogicalAnd

    system:
        site_group:
            languages: [eng-GB]
            content:
                tree_root: { location_id: 2 }
        fr:
            languages: [fre-FR, eng-GB]
            content:
                tree_root: { location_id: 60 }
```

Matching precedence: `X-Siteaccess` header > `EZPUBLISH_SITEACCESS` env var > configured matchers.

---

## Data migrations

Command: `php bin/console ibexa:migrations:migrate --file=my_migration.yaml [--siteaccess=admin]`
Auto-import: `php bin/console ibexa:migrations:import my_migration.yaml`
Files folder: `src/Migrations/Ibexa/migrations/`

```yaml
# Create content type
- type: content_type
  mode: create
  metadata:
    identifier: blog_post
    mainTranslation: eng-GB
    contentTypeGroups: [Content]
    nameSchema: '<title>'
    translations:
      eng-GB: { name: Blog Post }
  fields:
    - identifier: title
      type: ibexa_string
      required: true
      translatable: true
      searchable: true
      translations:
        eng-GB: { name: Title }
    - identifier: body
      type: ibexa_richtext
      translations:
        eng-GB: { name: Body }

# Create content with references
- type: content
  mode: create
  metadata:
    contentType: folder
    mainTranslation: eng-GB
  location:
    parentLocationId: 2
  fields:
    - fieldDefIdentifier: name
      languageCode: eng-GB
      value: 'My Folder'
  references:
    - name: folder_location_id
      type: location_id

- type: content
  mode: create
  metadata:
    contentType: article
    mainTranslation: eng-GB
  location:
    parentLocationId: 'reference:folder_location_id'  # reference from above step
  fields:
    - fieldDefIdentifier: title
      languageCode: eng-GB
      value: 'My Article'
    - fieldDefIdentifier: intro
      languageCode: eng-GB
      value:
        xml: |
          <?xml version="1.0" encoding="UTF-8"?>
          <section xmlns="http://docbook.org/ns/docbook" xmlns:xlink="http://www.w3.org/1999/xlink"
                   xmlns:ezxhtml="http://ibexa.co/xmlns/dxp/docbook/xhtml"
                   xmlns:ezcustom="http://ibexa.co/xmlns/dxp/docbook/custom"
                   version="5.0-variant ezpublish-1.0">
            <para>Intro paragraph.</para>
          </section>

# Create role with policies
- type: role
  mode: create
  metadata:
    identifier: Editor
  policies:
    - module: content
      function: read
    - module: content
      function: create
      limitations:
        - identifier: Class
          values: [article, blog_post]
    - module: content
      function: edit
      limitations:
        - identifier: Owner
          values: ['1']

# Create user
- type: user
  mode: create
  metadata:
    login: janedoe
    email: jane@example.com
    password: StrongPass123
    enabled: true
    mainLanguage: eng-GB
    contentType: user
  groups: [<group_remote_id>]
  fields:
    - fieldDefIdentifier: first_name
      languageCode: eng-GB
      value: Jane
  actions:
    - { action: assign_user_to_role, identifier: Editor }

# Repeatable steps with expressions
- type: repeatable
  mode: create
  iterations: 10
  steps:
    - type: content
      mode: create
      metadata:
        contentType: folder
        mainTranslation: eng-GB
      location:
        parentLocationId: 2
      fields:
        - fieldDefIdentifier: name
          languageCode: eng-GB
          value: '###SSS "Item " ~ i SSS###'  # Symfony expression syntax

# Delete content
- type: content
  mode: delete
  match:
    field: content_remote_id
    value: 'my-remote-id'
```

Available migration types: `content_type`, `content_type_group`, `content`, `location`, `role`, `section`, `user`, `user_group`, `attribute`, `attribute_group`, `product_price`, `product_variant`, `product_asset`, `product_availability`, `currency`, `customer_group`, `discount`, `discount_code`, `segment`, `segment_group`, `setting`, `language`, `object_state`, `object_state_group`, `payment_method`, `shipping_method`.
Expression syntax: `###SSS expression SSS###`. Built-in functions: `to_bool`, `to_int`, `to_float`, `to_string`, `reference()`, `project_dir()`, `env()`, `faker()`.

---

## Field type identifiers

`ibexa_string`, `ibexa_text`, `ibexa_richtext`, `ibexa_integer`, `ibexa_float`, `ibexa_boolean`, `ibexa_date`, `ibexa_datetime`, `ibexa_time`, `ibexa_email`, `ibexa_url`, `ibexa_isbn`, `ibexa_keyword`, `ibexa_selection`, `ibexa_country`, `ibexa_map_location`, `ibexa_matrix`, `ibexa_image`, `ibexa_image_asset`, `ibexa_binary_file`, `ibexa_media`, `ibexa_relation`, `ibexa_relation_list`, `ibexa_author`, `ibexa_user`, `ibexa_page`, `ibexa_form`, `ibexa_taxonomy_entry`, `ibexa_taxonomy_entry_assignment`, `ibexa_content_query`, `ibexa_product_specification`, `ibexa_measurement`, `ibexa_address`, `ibexa_customer_group`

---

## Permissions

```php
use Ibexa\Core\MVC\Symfony\Security\Authorization\Attribute;

// In controller (extends AbstractController):
$this->denyAccessUnlessGranted(new Attribute('content', 'read'));

// Check with value object:
$hasAccess = $this->isGranted(
    new Attribute('section', 'assign', ['valueObject' => $contentInfo, 'targets' => [$section]])
);

// Via PermissionResolver:
$this->permissionResolver->canUser('content', 'edit', $content, [$location]);
```

Key policy modules: `content` (read, create, edit, publish, manage_locations, hide, translate, remove, versionread), `section` (view, assign), `state` (administrate, assign), `role` (read, create, update, delete, assign), `setup` (administrate, system_info), `user` (login, preferences, invite, register), `cart` (view, create, edit, delete), `order` (view, create, update, cancel), `payment` (view, create, edit, delete), `checkout` (view, create, update, delete), `product` (view, create, edit, delete).

---

## Events (subscribe with Symfony EventSubscriberInterface)

```php
use Ibexa\Contracts\Core\Repository\Events\Content\PublishVersionEvent;
use Ibexa\Contracts\Core\Repository\Events\Content\BeforePublishVersionEvent;

class ContentSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            PublishVersionEvent::class => 'onPublish',
            BeforePublishVersionEvent::class => 'onBeforePublish',
        ];
    }

    public function onPublish(PublishVersionEvent $event): void
    {
        $content = $event->getContent();
        $versionInfo = $event->getVersionInfo();
    }
}
```

See `references/events.md` for full event list (content, location, content type, user, workflow, page builder).

---

## Workflow

```yaml
ibexa:
    system:
        default:
            workflows:
                editorial:
                    name: 'Editorial Workflow'
                    matchers:
                        content_type: [article]
                    stages:
                        draft: { label: Draft }
                        review: { label: In Review }
                        published: { label: Published, last_stage: true }
                    transitions:
                        to_review: { from: draft, to: review, label: 'Send to review' }
                        publish: { from: review, to: published, label: Publish }
                        reject: { from: review, to: draft, label: Reject }
```

```php
use Ibexa\Contracts\Workflow\Service\WorkflowServiceInterface;
use Ibexa\Contracts\Workflow\Registry\WorkflowRegistryInterface;

$this->workflowService->start($content, 'editorial');
$meta = $this->workflowService->loadWorkflowMetadataForContent($content, 'editorial');
if ($this->workflowService->can($meta, 'to_review')) {
    $workflow = $this->workflowRegistry->getWorkflow('editorial');
    $workflow->apply($meta->content, 'to_review', ['message' => 'Ready', 'reviewerId' => $userId]);
}
foreach ($meta->markings as $marking) { echo $marking->name; }
```

---

## Product catalog / PIM

```php
use Ibexa\Contracts\ProductCatalog\ProductServiceInterface;
use Ibexa\Contracts\ProductCatalog\Local\LocalProductServiceInterface;
use Ibexa\Contracts\ProductCatalog\Values\Product\ProductQuery;
use Ibexa\Contracts\ProductCatalog\Values\Product\Query\Criterion as PCriterion;
use Ibexa\Contracts\ProductCatalog\Values\Product\Query\SortClause as PSortClause;

$product = $this->productService->getProduct('PROD001');
$product->getCode(); $product->getName(); $product->getPrice();

$query = new ProductQuery(null, new PCriterion\ProductType(['sofa']), [new PSortClause\ProductName()]);
$products = $this->productService->findProducts($query);

// Create product
$productType = $this->productTypeService->getProductType('sofa');
$createStruct = $this->localProductService->newProductCreateStruct($productType, 'eng-GB');
$createStruct->setCode('SOFA001');
$createStruct->setField('name', 'My Sofa');
$this->localProductService->createProduct($createStruct);

// Availability
use Ibexa\Contracts\ProductCatalog\Values\Availability\ProductAvailabilityCreateStruct;
$this->productAvailabilityService->createProductAvailability(
    new ProductAvailabilityCreateStruct($product, true, false, 50) // available, infinite, stock
);
```

---

## Cart

```php
use Ibexa\Contracts\Cart\CartServiceInterface;
use Ibexa\Contracts\Cart\Value\{CartCreateStruct, EntryAddStruct, EntryUpdateStruct};

$currency = $this->currencyService->getCurrencyByCode('EUR');
$cart = $this->cartService->createCart(new CartCreateStruct('Cart', $currency));

$addStruct = new EntryAddStruct($product);
$addStruct->setQuantity(2);
$cart = $this->cartService->addEntry($cart, $addStruct);

$entry = $cart->getEntries()->first();
$cart = $this->cartService->updateEntry($cart, $entry, new EntryUpdateStruct($entry->getId()));
$cart = $this->cartService->removeEntry($cart, $entry);
$violations = $this->cartService->validateCart($cart);
$this->cartService->deleteCart($cart);
```

---

## Taxonomy

```php
use Ibexa\Contracts\Taxonomy\Service\TaxonomyServiceInterface;

$entry = $this->taxonomyService->loadEntryByIdentifier('electronics');
$children = $this->taxonomyService->loadEntryChildren($entry, 10, 0);
$this->taxonomyService->moveEntry($entry, $newParent);
$this->taxonomyService->moveEntryRelativeToSibling($entry, $sibling, TaxonomyServiceInterface::MOVE_POSITION_PREV);
```

---

## Image variations

```yaml
ibexa:
    system:
        default:
            image_variations:
                thumbnail:
                    type: alias
                    filters:
                        - { name: geometry/scalewidthdownonly, params: [100] }
                medium:
                    type: alias
                    filters:
                        - { name: geometry/scaledownonly, params: [500, 400] }
```

Twig: `{{ ibexa_render_field(content, 'image', {'parameters': {'alias': 'medium'}}) }}`

---

## Caching

```yaml
ibexa:
    http_cache:
        purge_type: varnish  # or fastly
    system:
        site_group:
            http_cache:
                purge_servers: ['http://varnish:8081']
```

Content-aware: responses are auto-tagged with content IDs and purged on content update.
Persistence cache: `php bin/console cache:pool:clear cache.ibexa.persistence`
Performance: `composer dump-autoload --optimize`, use Solr/ES, Redis with persistence disabled, `--env=prod` for CLI.

---

## Common tips

- **Use `ContentInfo` not `Content`** in lists — it's lighter (no field data).
- **Never serialize value objects** — memory limit errors.
- **RichText XML** must wrap content in `<section xmlns="http://docbook.org/ns/docbook" version="5.0-variant ezpublish-1.0">`.
- **Always use SiteAccess-scoped config** for site-specific settings.
- **Use `--siteaccess=admin`** flag on migration and other CLI commands in multisite setups.
- **sudo** for backend operations: `$this->repository->sudo(fn($repo) => ...)`.
- **Service tags**: `ibexa.field_type`, `ibexa.query_type`, `ibexa.field_type.indexable`, `ibexa.admin_ui.field_type.form.mapper.value`, `ibexa.admin_ui.tab`, `ibexa.rest.output.value_object.visitor`.
- For custom query types, page blocks, field types, menus, SiteAccess-aware config: see `references/extensibility.md`.
