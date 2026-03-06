# Ibexa DXP — Extensibility Reference

## Custom Query Type

```php
use Ibexa\Core\QueryType\QueryType;
use Ibexa\Contracts\Core\Repository\Values\Content\LocationQuery;
use Ibexa\Contracts\Core\Repository\Values\Content\Query\Criterion;
use Ibexa\Contracts\Core\Repository\Values\Content\Query\SortClause;

class LatestArticlesQueryType implements QueryType
{
    public static function getName(): string { return 'LatestArticles'; }

    public function getQuery(array $parameters = []): LocationQuery
    {
        $criteria = [
            new Criterion\ContentTypeIdentifier($parameters['contentType'] ?? 'article'),
            new Criterion\Visibility(Criterion\Visibility::VISIBLE),
        ];
        if (isset($parameters['parentLocationId'])) {
            $criteria[] = new Criterion\ParentLocationId($parameters['parentLocationId']);
        }
        return new LocationQuery([
            'filter' => new Criterion\LogicalAnd($criteria),
            'sortClauses' => [new SortClause\DatePublished(LocationQuery::SORT_DESC)],
            'limit' => $parameters['limit'] ?? 10,
        ]);
    }

    public function getSupportedParameters(): array
    {
        return ['contentType', 'parentLocationId', 'limit'];
    }
}
```

Service tag (auto-registered for `App\` namespace, otherwise explicit):
```yaml
services:
    App\QueryType\LatestArticlesQueryType:
        tags: [ibexa.query_type]
```

View config usage:
```yaml
content_view:
    full:
        folder:
            controller: ibexa_query::locationQueryAction   # locationQueryAction | contentQueryAction | contentInfoQueryAction | pagingQueryAction
            template: '@ibexadesign/full/folder.html.twig'
            match:
                Identifier\ContentType: folder
            params:
                query:
                    query_type: LatestArticles
                    parameters:
                        parentLocationId: '@=location.id'  # Symfony expression: @=content, @=location, @=view
                        limit: 12
                    assign_results_to: articles  # available in template as {{ articles }}
```

With pagination (`pagingQueryAction`):
```yaml
params:
    query:
        query_type: LatestArticles
        limit: 10           # items per page
        assign_results_to: articles  # PagerFanta object
```

Template: `{{ pagerfanta(articles) }}`

---

## Custom Page Builder block

```yaml
# config/packages/ibexa_fieldtype_page.yaml
ibexa_fieldtype_page:
    blocks:
        my_block:
            name: 'My Block'
            category: 'Custom'
            thumbnail: '/bundles/app/img/my_block.svg'
            views:
                default:
                    template: '@ibexadesign/blocks/my_block.html.twig'
                    name: 'Default view'
                compact:
                    template: '@ibexadesign/blocks/my_block_compact.html.twig'
                    name: 'Compact view'
            attributes:
                title:
                    type: text
                    name: 'Title'
                    validators:
                        not_blank: ~
                content_id:
                    type: embed
                    name: 'Content item'
                count:
                    type: integer
                    name: 'Number of items'
                    value: 5
                    validators:
                        not_blank: ~
                        regexp:
                            options:
                                regexp: '/[0-9]+/'
```

Attribute types: `text`, `string`, `number`, `integer`, `url`, `embed`, `select`, `multiple`, `radio`, `checkbox`, `location`, `richtext`.

```php
// EventSubscriber for pre-render logic
use Ibexa\Contracts\FieldTypePage\FieldType\Page\Block\Renderer\BlockRenderEvents;
use Ibexa\Contracts\FieldTypePage\FieldType\Page\Block\Renderer\Event\PreRenderEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class MyBlockSubscriber implements EventSubscriberInterface
{
    public function __construct(private ContentService $contentService) {}

    public static function getSubscribedEvents(): array
    {
        return [
            BlockRenderEvents::getBlockPreRenderEventName('my_block') => 'onBlockPreRender',
        ];
    }

    public function onBlockPreRender(PreRenderEvent $event): void
    {
        $blockValue = $event->getBlockValue();
        $renderRequest = $event->getRenderRequest();
        $params = $renderRequest->getParameters();

        // Get block attribute
        $contentId = (int) $blockValue->getAttribute('content_id')->getValue();
        $params['content'] = $this->contentService->loadContent($contentId);

        $renderRequest->setParameters($params);
    }
}
```

Block template `@ibexadesign/blocks/my_block.html.twig`:
```twig
{% set title = block_value.getAttribute('title').getValue() %}
<div class="my-block">
    <h2>{{ title }}</h2>
    {% if content is defined %}
        {{ ibexa_render(content, {'viewType': 'embed'}) }}
    {% endif %}
</div>
```

Clear cache after YAML changes: `php bin/console cache:pool:clear cache.app`

---

## Custom field type (generic approach)

```php
// src/FieldType/HelloWorld/Value.php
use Ibexa\Contracts\Core\FieldType\Value as ValueInterface;

final class HelloWorldValue implements ValueInterface
{
    public function __construct(private ?string $name = null) {}
    public function getName(): ?string { return $this->name; }
    public function __toString(): string { return (string) $this->name; }
}

// src/FieldType/HelloWorld/Type.php
use Ibexa\Contracts\Core\FieldType\Generic\Type as GenericType;
use Ibexa\Contracts\ContentForms\FieldType\FieldValueFormMapperInterface;
use Ibexa\Contracts\ContentForms\Data\Content\FieldData;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\Extension\Core\Type\TextType;

final class HelloWorldType extends GenericType implements FieldValueFormMapperInterface
{
    public function getFieldTypeIdentifier(): string
    {
        return 'hello_world';
    }

    public function mapFieldValueForm(FormBuilderInterface $fieldForm, FieldData $data): void
    {
        $fieldForm->add('name', TextType::class, [
            'required' => $data->fieldDefinition->isRequired,
            'label' => false,
        ]);
    }
}
```

```yaml
# config/services.yaml
services:
    App\FieldType\HelloWorld\HelloWorldType:
        public: true
        tags:
            - { name: ibexa.field_type, alias: hello_world }
            - { name: ibexa.admin_ui.field_type.form.mapper.value, fieldType: hello_world }

# config/packages/ibexa.yaml — template
ibexa:
    system:
        default:
            field_templates:
                - { template: '@ibexadesign/field_types/hello_world.html.twig', priority: 0 }
```

Field template `@ibexadesign/field_types/hello_world.html.twig`:
```twig
{% block hello_world_field %}
    {{ field.value.name }}
{% endblock %}
```

For searchable field types, also implement `Ibexa\Contracts\Core\FieldType\Indexable` and tag with `ibexa.field_type.indexable`.

---

## SiteAccess-aware custom configuration

```php
// src/DependencyInjection/Configuration.php
use Ibexa\Bundle\Core\DependencyInjection\Configuration\SiteAccessAware\Configuration as SAConfiguration;
use Symfony\Component\Config\Definition\Builder\TreeBuilder;

class Configuration extends SAConfiguration
{
    public function getConfigTreeBuilder(): TreeBuilder
    {
        $treeBuilder = new TreeBuilder('my_bundle');
        $systemNode = $this->generateScopeBaseNode($treeBuilder->getRootNode());
        $systemNode
            ->scalarNode('api_key')->defaultNull()->end()
            ->integerNode('cache_ttl')->defaultValue(3600)->end();
        return $treeBuilder;
    }
}

// src/DependencyInjection/MyBundleExtension.php
use Ibexa\Bundle\Core\DependencyInjection\Configuration\SiteAccessAware\ConfigurationProcessor;
use Ibexa\Bundle\Core\DependencyInjection\Configuration\SiteAccessAware\ContextualizerInterface;

$processor = new ConfigurationProcessor($container, 'my_bundle');
$processor->mapConfig($config, function ($scopeSettings, $scope, ContextualizerInterface $c): void {
    if (isset($scopeSettings['api_key'])) {
        $c->setContextualParameter('api_key', $scope, $scopeSettings['api_key']);
    }
    $c->setContextualParameter('cache_ttl', $scope, $scopeSettings['cache_ttl']);
});
```

Usage in config:
```yaml
my_bundle:
    system:
        site_group:
            api_key: 'abc123'
            cache_ttl: 7200
        fr_site:
            api_key: 'xyz789'
```

Retrieve in service via `ConfigResolverInterface`:
```php
use Ibexa\Contracts\Core\SiteAccess\ConfigResolverInterface;

class MyService
{
    public function __construct(private ConfigResolverInterface $configResolver) {}

    public function doSomething(): void
    {
        $apiKey = $this->configResolver->getParameter('api_key', 'my_bundle');
        $ttl = $this->configResolver->getParameter('cache_ttl', 'my_bundle');
    }
}
```

---

## Custom back-office tab

```php
use Ibexa\Contracts\AdminUi\Tab\AbstractTab;
use Ibexa\Contracts\AdminUi\Tab\OrderedTabInterface;

class MyCustomTab extends AbstractTab implements OrderedTabInterface
{
    public function getIdentifier(): string { return 'my_tab'; }
    public function getName(): string { return 'My Tab'; }
    public function getOrder(): int { return 100; }
    public function renderView(array $parameters): string
    {
        return $this->twig->render('@ibexadesign/my_tab.html.twig', $parameters);
    }
}
```

```yaml
services:
    App\Tab\MyCustomTab:
        tags:
            - { name: ibexa.admin_ui.tab, group: content }
            # groups: content, location, content-type, dashboard-everyone
```

---

## Custom back-office menu item

```php
use Ibexa\AdminUi\Menu\Event\ConfigureMenuEvent;
use Knp\Menu\ItemInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class MenuSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [ConfigureMenuEvent::MAIN_MENU => 'onConfigureMainMenu'];
    }

    public function onConfigureMainMenu(ConfigureMenuEvent $event): void
    {
        $menu = $event->getMenu();
        $myItem = $menu->addChild('my_section', [
            'label' => 'My Section',
            'uri' => '/admin/my-section',
            'extras' => ['icon' => 'system-information'],
        ]);
    }
}
```

---

## Custom permissions (policies)

```php
use Ibexa\Contracts\Core\Limitation\Type as LimitationType;

// 1. Register custom policy in the limitation service
// In services.yaml:
services:
    App\Permission\MyLimitationType:
        tags:
            - { name: ibexa.permissions.limitation_type, alias: MyLimitation }
```

```yaml
# Register policy module/function
ibexa:
    system:
        default:
            # In custom bundle's DependencyInjection or via compiler pass
```

Checking custom policy in a controller:
```php
use Ibexa\Core\MVC\Symfony\Security\Authorization\Attribute;

// In controller:
$this->denyAccessUnlessGranted(new Attribute('my_module', 'my_function'));

// Or with object check:
$hasAccess = $this->isGranted(
    new Attribute('my_module', 'view', ['valueObject' => $myObject])
);
```
