# Ibexa DXP — Search Criteria & Sort Clauses Reference

All in `Ibexa\Contracts\Core\Repository\Values\Content\Query\Criterion\` unless noted.

## Content criteria

| Criterion | Constructor | Notes |
|---|---|---|
| `ContentId` | `(int\|int[])` | Match by content ID |
| `ContentTypeId` | `(int\|int[])` | Match by content type ID |
| `ContentTypeIdentifier` | `(string\|string[])` | Match by content type identifier (preferred) |
| `ContentTypeGroupId` | `(int\|int[])` | Match by content type group |
| `ContentName` | `(string, operator)` | Match by content name. Operator: `LIKE`, `EQ` |
| `RemoteId` | `(string\|string[])` | Match by content remote ID |
| `SectionId` | `(int\|int[])` | Match by section ID |
| `SectionIdentifier` | `(string\|string[])` | Match by section identifier |
| `FullText` | `(string)` | Full-text search (Solr/ES recommended) |
| `Field` | `(string $fieldIdentifier, string $operator, mixed $value)` | Search by field value. Operators: `EQ`, `GT`, `GTE`, `LT`, `LTE`, `IN`, `BETWEEN`, `LIKE`, `CONTAINS` |
| `FieldRelation` | `(string $identifier, string $operator, int[])` | Search by relation field |
| `DateMetadata` | `(string $target, string $operator, int\|int[])` | Target: `DateMetadata::MODIFIED` or `DateMetadata::PUBLISHED`. Operators: `EQ`, `GT`, `GTE`, `LT`, `LTE`, `BETWEEN` |
| `ObjectStateId` | `(int\|int[])` | Match by object state ID |
| `ObjectStateIdentifier` | `(string\|string[], string\|string[] $objectStateGroupIdentifiers)` | Match by state identifier |
| `LanguageCode` | `(string\|string[], bool $matchAlwaysAvailable)` | Match by language |
| `Visibility` | `(int)` | `Visibility::VISIBLE` or `Visibility::HIDDEN` |

## Location criteria

| Criterion | Constructor | Notes |
|---|---|---|
| `Location\Id` | `(int\|int[])` | Match by location ID |
| `Location\RemoteId` | `(string\|string[])` | Match by location remote ID |
| `ParentLocationId` | `(int\|int[])` | Match by parent location ID |
| `Subtree` | `(string $pathString)` | Match all content within a subtree. Use `$location->pathString` |
| `Depth` | `(string $operator, int\|int[])` | Match by location depth. Operators: `EQ`, `GT`, `GTE`, `LT`, `LTE`, `BETWEEN`, `IN` |
| `Priority` | `(string $operator, int)` | Match by location priority |
| `Location\IsMainLocation` | `(int)` | `IsMainLocation::MAIN` or `IsMainLocation::NOT_MAIN` |

## User criteria

| Criterion | Constructor |
|---|---|
| `UserId` | `(int\|int[])` |
| `UserLogin` | `(string\|string[])` |
| `UserEmail` | `(string\|string[])` |
| `UserMetadata` | `(string $target, string $operator, mixed $value)` — `UserMetadata::MODIFIER`, `OWNER`, `GROUP` |

## Logical operators

```php
new Criterion\LogicalAnd([...criteria])   // ALL must match
new Criterion\LogicalOr([...criteria])    // ANY must match
new Criterion\LogicalNot($criterion)      // invert
```

## Aggregations (`Values\Content\Query\Aggregation\`)

| Aggregation | Purpose |
|---|---|
| `ContentTypeTermAggregation` | Count by content type |
| `SectionTermAggregation` | Count by section |
| `LanguageTermAggregation` | Count by language |
| `ObjectStateTermAggregation` | Count by object state |
| `TaxonomyEntryTermAggregation` | Count by taxonomy tag |
| `Field\DateRangeAggregation` | Date field range buckets |
| `Field\IntegerRangeAggregation` | Integer field range buckets |
| `Field\FloatRangeAggregation` | Float field range buckets |
| `Field\SelectionTermAggregation` | Selection field term counts |

```php
$aggregation = new ContentTypeTermAggregation('ct');
$aggregation->setLimit(10);
$aggregation->setMinCount(1);
$query->aggregations[] = $aggregation;
// Access: $results->aggregations->get('ct') → iterable of [$contentType => $count]
```

## Sort Clauses (`Values\Content\Query\SortClause\`)

| Sort Clause | Notes |
|---|---|
| `ContentId` | |
| `ContentName` | |
| `DateModified` | By modification date |
| `DatePublished` | By publication date |
| `Field` | `new Field('content_type', 'field_identifier', Query::SORT_ASC)` |
| `MapLocationDistance` | For map location fields |
| `Depth` | Location depth |
| `Priority` | Location priority |
| `Score` | Relevance score (Solr/ES) |
| `SectionName` | |
| `SectionIdentifier` | |
| `Random` | Random ordering |

Order constants: `Query::SORT_ASC` (default), `Query::SORT_DESC`

## Product criteria (`Ibexa\Contracts\ProductCatalog\Values\Product\Query\Criterion\`)

`ProductCode`, `ProductName`, `ProductType`, `BasePrice`, `ProductAvailability`, `VatCategory`, `ProductCategory`, `IsVirtual`, `LogicalAnd`, `LogicalOr`, `LogicalNot`

## Product sort clauses (`...Query\SortClause\`)

`ProductCode`, `ProductName`, `BasePrice`, `CreatedAt`

## Pagination with BatchIterator

```php
use Ibexa\Contracts\Core\Repository\Iterator\BatchIterator;
use Ibexa\Core\Repository\Iterator\BatchIteratorAdapter\LocationSearchAdapter;

$iterator = new BatchIterator(
    new LocationSearchAdapter($this->searchService, $query)
);
// Also: ContentSearchAdapter, ContentInfoSearchAdapter, ContentFilteringAdapter, LocationFilteringAdapter
foreach ($iterator as $item) { ... } // auto-paginates in batches
```
