# Ibexa DXP — Full Event Reference

All events are in `Ibexa\Contracts\Core\Repository\Events\*` unless otherwise noted.
Every action dispatches a `Before*Event` before and a `*Event` after.

## Content events (`Events\Content\`)

| Event | Dispatched by | Key Properties |
|---|---|---|
| `BeforeCreateContentEvent` / `CreateContentEvent` | `ContentService::createContent` | `ContentCreateStruct`, `array $locationCreateStructs`, `Content` |
| `BeforeCreateContentDraftEvent` / `CreateContentDraftEvent` | `ContentService::createContentDraft` | `ContentInfo`, `VersionInfo`, `User $creator`, `Content $contentDraft` |
| `BeforeUpdateContentEvent` / `UpdateContentEvent` | `ContentService::updateContent` | `VersionInfo`, `ContentUpdateStruct`, `Content` |
| `BeforeUpdateContentMetadataEvent` / `UpdateContentMetadataEvent` | `ContentService::updateContentMetadata` | `ContentInfo`, `ContentMetadataUpdateStruct`, `Content` |
| `BeforePublishVersionEvent` / `PublishVersionEvent` | `ContentService::publishVersion` | `VersionInfo`, `Content`, `string[] $translations` |
| `BeforeCopyContentEvent` / `CopyContentEvent` | `ContentService::copyContent` | `ContentInfo`, `LocationCreateStruct`, `VersionInfo`, `Content` |
| `BeforeDeleteContentEvent` / `DeleteContentEvent` | `ContentService::deleteContent` | `ContentInfo`, `array $locations` |
| `BeforeDeleteVersionEvent` / `DeleteVersionEvent` | `ContentService::deleteVersion` | `VersionInfo` |
| `BeforeHideContentEvent` / `HideContentEvent` | `ContentService::hideContent` | `ContentInfo` |
| `BeforeRevealContentEvent` / `RevealContentEvent` | `ContentService::revealContent` | `ContentInfo` |
| `BeforeAddRelationEvent` / `AddRelationEvent` | `ContentService::addRelation` | `VersionInfo $sourceVersion`, `ContentInfo $destinationContent`, `Relation` |
| `BeforeDeleteRelationEvent` / `DeleteRelationEvent` | `ContentService::deleteRelation` | `VersionInfo`, `ContentInfo $destinationContent` |
| `BeforeDeleteTranslationEvent` / `DeleteTranslationEvent` | `ContentService::deleteTranslation` | `ContentInfo`, `string $languageCode` |

## Location events (`Events\Location\`)

| Event | Dispatched by | Key Properties |
|---|---|---|
| `BeforeCreateLocationEvent` / `CreateLocationEvent` | `LocationService::createLocation` | `ContentInfo`, `LocationCreateStruct`, `Location` |
| `BeforeUpdateLocationEvent` / `UpdateLocationEvent` | `LocationService::updateLocation` | `Location`, `LocationUpdateStruct`, `Location $updatedLocation` |
| `BeforeDeleteLocationEvent` / `DeleteLocationEvent` | `LocationService::deleteLocation` | `Location` |
| `BeforeHideLocationEvent` / `HideLocationEvent` | `LocationService::hideLocation` | `Location`, `Location $hiddenLocation` |
| `BeforeUnhideLocationEvent` / `UnhideLocationEvent` | `LocationService::unhideLocation` | `Location`, `Location $revealedLocation` |
| `BeforeMoveSubtreeEvent` / `MoveSubtreeEvent` | `LocationService::moveSubtree` | `Location`, `Location $newParentLocation` |
| `BeforeCopySubtreeEvent` / `CopySubtreeEvent` | `LocationService::copySubtree` | `Location $subtree`, `Location $targetParentLocation`, `Location $location` |
| `BeforeSwapLocationEvent` / `SwapLocationEvent` | `LocationService::swapLocation` | `Location $location1`, `Location $location2` |

## Content type events (`Events\ContentType\`)

| Event | Key Properties |
|---|---|
| `BeforeCreateContentTypeEvent` / `CreateContentTypeEvent` | `ContentTypeCreateStruct`, `array $groups`, `ContentTypeDraft` |
| `BeforeCreateContentTypeDraftEvent` / `CreateContentTypeDraftEvent` | `ContentType`, `ContentTypeDraft` |
| `BeforeUpdateContentTypeDraftEvent` / `UpdateContentTypeDraftEvent` | `ContentTypeDraft`, `ContentTypeUpdateStruct` |
| `BeforePublishContentTypeDraftEvent` / `PublishContentTypeDraftEvent` | `ContentTypeDraft` |
| `BeforeCopyContentTypeEvent` / `CopyContentTypeEvent` | `ContentType`, `User $creator`, `ContentType $contentTypeCopy` |
| `BeforeDeleteContentTypeEvent` / `DeleteContentTypeEvent` | `ContentType` |
| `BeforeAddFieldDefinitionEvent` / `AddFieldDefinitionEvent` | `ContentTypeDraft`, `FieldDefinitionCreateStruct` |
| `BeforeUpdateFieldDefinitionEvent` / `UpdateFieldDefinitionEvent` | `ContentTypeDraft`, `FieldDefinition`, `FieldDefinitionUpdateStruct` |
| `BeforeRemoveFieldDefinitionEvent` / `RemoveFieldDefinitionEvent` | `ContentTypeDraft`, `FieldDefinition` |

## User events (`Events\User\`)

| Event | Dispatched by |
|---|---|
| `BeforeCreateUserEvent` / `CreateUserEvent` | `UserService::createUser` |
| `BeforeUpdateUserEvent` / `UpdateUserEvent` | `UserService::updateUser` |
| `BeforeDeleteUserEvent` / `DeleteUserEvent` | `UserService::deleteUser` |
| `BeforeCreateUserGroupEvent` / `CreateUserGroupEvent` | `UserService::createUserGroup` |
| `BeforeUpdateUserGroupEvent` / `UpdateUserGroupEvent` | `UserService::updateUserGroup` |
| `BeforeDeleteUserGroupEvent` / `DeleteUserGroupEvent` | `UserService::deleteUserGroup` |

## Section events (`Events\Section\`)
`BeforeCreateSectionEvent` / `CreateSectionEvent`, `BeforeUpdateSectionEvent` / `UpdateSectionEvent`, `BeforeDeleteSectionEvent` / `DeleteSectionEvent`, `BeforeAssignSectionEvent` / `AssignSectionEvent`

## Role events (`Events\Role\`)
`BeforeCreateRoleEvent` / `CreateRoleEvent`, `BeforeUpdateRoleEvent` / `UpdateRoleEvent`, `BeforeDeleteRoleEvent` / `DeleteRoleEvent`, `BeforeAssignRoleToUserEvent` / `AssignRoleToUserEvent`, `BeforeAssignRoleToUserGroupEvent` / `AssignRoleToUserGroupEvent`

## Workflow events (`Ibexa\Contracts\Workflow\Event\`)
`WorkflowTransitionEvent` — dispatched after every workflow transition. Properties: `content`, `workflowMetadata`, `transition`.

## Page Builder events (`Ibexa\Contracts\FieldTypePage\...`)
`BlockRenderEvents::getBlockPreRenderEventName('block_identifier')` → `PreRenderEvent` with `BlockValue` and `RenderRequest`.

## Event subscriber pattern

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Ibexa\Contracts\Core\Repository\Events\Content\PublishVersionEvent;

class MySubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [PublishVersionEvent::class => 'onPublish'];
    }

    public function onPublish(PublishVersionEvent $event): void
    {
        $content = $event->getContent(); // Content object
        $versionInfo = $event->getVersionInfo();
        // contentTypeIdentifier:
        $typeId = $content->contentInfo->contentTypeId;
    }
}
```

Auto-tagged if in `App\` namespace with `Symfony\Component\EventDispatcher\EventSubscriberInterface`.
