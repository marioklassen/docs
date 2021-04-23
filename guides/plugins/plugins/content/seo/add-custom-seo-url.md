# Add custom SEO URL

## Overview

Every good website had to deal with it already: SEO URLs.
Of course Shopware supports the usage of SEO URLs, e.g. for products or categories.

This guide however will cover the question on how you can define your own SEO URLs, e.g. for your own
custom entities.
This will include both static SEO URLs, as well as dynamic SEO URLs.

## Prerequisites

As every almost every guide in the plugins section, this guide as well is built upon the plugin base guide.

{% page-ref page="../../plugin-base-guide.md" %}

Furthermore, we're going to use a [custom storefront controller](../../storefront/add-custom-controller.md) for the static SEO URL example,
as well as [custom entities](../../framework/data-handling/add-custom-complex-data.md) for the dynamic SEO URLs.
Make sure you know and understand those two as well before diving deeper into this guide.
Those come with two different solutions:
- Using [plugin migrations](../../plugin-fundamentals/database-migrations.md) for static SEO URLs
- Using [DAL events](../../framework/data-handling/using-database-events.md) to react on entity changes and therefore generating a dynamic SEO URL

## Custom SEO URLs

As already mentioned in the overview, this guide will divided into two parts: Static and dynamic SEO URLs.

### Static SEO URLs

A static SEO URL doesn't have to change every now and then.
Imagine a custom controller, which is accessible via the link `yourShop.com/example`.

Now if you want this URL to be translatable, you'll have to add a custom SEO URL to your controller route,
so it is accessible using both `Example-Page` in English, as well as e.g. `Beispiel-Seite` in German.

#### Example controller

For this example, the controller from the [Add custom controller guide](../../storefront/add-custom-controller.md) is being used.
It creates a controller with a route like the example mentioned above: `/example`

Let's now have a look at our example controller:
{% code title="<plugin root>/src/Migration/Migration1619094740AddStaticSeoUrl.php" %}
```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Storefront\Controller;

use Shopware\Core\Framework\Routing\Annotation\RouteScope;
use Shopware\Core\System\SalesChannel\SalesChannelContext;
use Shopware\Storefront\Controller\StorefrontController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

/**
 * @RouteScope(scopes={"storefront"})
 */
class ExampleController extends StorefrontController
{
    /**
    * @Route("/example", name="frontend.example.example", methods={"GET"})
    */
    public function showExample(): Response
    {
        return $this->renderStorefront('@SwagBasicExample/storefront/page/example/index.html.twig', [
            'example' => 'Hello world'
        ]);
    }
}
```
{% endcode %}

The important information you'll need here are the route name, `frontend.example.example`, as well as the route itself: `/example`.
Make sure to remember those for the next step.

#### Example migration

Creating a SEO URL in this scenario can be achieved by creating a [plugin migration](../../plugin-fundamentals/database-migrations.md).

The migration has to insert an entry for each sales channel and language into the `seo_url` table.
For thise case, we're making use of the `ImportTranslationsTrait`, which comes with a helper method `importTranslation`.

Don't be confused here, we'll just treat the `seo_url` table like a translation table, since it also needs a `language_id`
and respective translated SEO URLs.
You'll have to pass a German and an English array into an instance of the `Translations` class, which is then the second
parameter for the `importTranslation` method.

Let's have a look at an example:

{% code title="<plugin root>/src/Migration/Migration1619094740AddStaticSeoUrl.php" %}
```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Migration;

use Doctrine\DBAL\Connection;
use Shopware\Core\Defaults;
use Shopware\Core\Framework\Migration\MigrationStep;
use Shopware\Core\Framework\Uuid\Uuid;
use Shopware\Core\Migration\Traits\ImportTranslationsTrait;
use Shopware\Core\Migration\Traits\Translations;

class Migration1619094740AddStaticSeoUrl extends MigrationStep
{
    use ImportTranslationsTrait;

    public function getCreationTimestamp(): int
    {
        return 1619094740;
    }

    public function update(Connection $connection): void
    {
        $this->importTranslation('seo_url', new Translations(
            // German array
            array_merge($this->getSeoMetaArray($connection), ['seo_path_info' => 'Beispiel-Seite']),
            // English array
            array_merge($this->getSeoMetaArray($connection), ['seo_path_info' => 'Example-Page']),

        ), $connection);
    }

    public function updateDestructive(Connection $connection): void
    {
    }

    private function getSeoMetaArray(Connection $connection): array
    {
        return [
            'id' => Uuid::randomBytes(),
            'sales_channel_id' => $this->getStorefrontSalesChannelId($connection),
            'foreign_key' => Uuid::randomBytes(),
            'route_name' => 'frontend.example.example',
            'path_info' => '/example',
            'is_canonical' => 1,
            'is_modified' => 0,
            'is_deleted' => 0,
        ];
    }

    private function getStorefrontSalesChannelId(Connection $connection): ?string
    {
        $sql = <<<SQL
            SELECT id
            FROM sales_channel
            WHERE type_id = :typeId
SQL;
        $salesChannelId = $connection->fetchOne($sql, [
            ':typeId' => Uuid::fromHexToBytes(Defaults::SALES_CHANNEL_TYPE_STOREFRONT)
        ]);

        if (!$salesChannelId) {
            return null;
        }

        return $salesChannelId;
    }
}
```
{% endcode %}

You might want to have a look at the `getSeoMetaArray` method, that we implemented here.
Most important for you are the columns `route_name` and `path_info` here, which represent the values you've defined in
your controller's route annotation.

By using the default PHP method `array_merge`, we're then also adding our translated SEO URL to the column `seo_path_info`.

And that's it! After installing our plugin, you should now be able to access your controller's route with the given SEO URLs.

{% hint style="info"%}
You can only access the German SEO URL if you've configured a German domain in your respective sales channel first.
{% endhint %}

### Dynamic SEO URLs

Dynamic SEO URLs are URLs, that have to change every now and then.

This scenario will be about a custom entity, to be specific we're going to use the entity from our guide about
[adding custom complex data](../../framework/data-handling/add-custom-complex-data.md), which then would have a custom
storefront route for each entity.

Each entity comes with a name, which should be the SEO URL. Your entity named `Foo` should be accessible using the route
`yourShop.com/Foo` or `yourShop.com/Entities/Foo` or whatever you'd like.
Now, everytime you create a new entity, a SEO URL has to be automatically created as well. When you update your entities' name,
guess what, you'll have to change the SEO URL as well.

All of this is done by subscribing to the DAL events of your entity, which are explained in depth [here](../../framework/data-handling/using-database-events.md).
Hence, you'll need a new subscriber. It then has to listen to the `written` event, which in this example will be `swag_example.written`.

Once an entity is written, created or updated that is, you'll have to write a new SEO URL.
For the latter, you can use the class `Shopware\Core\Content\Seo\SeoUrlPersister`.

Let's have a look at an example, which does all of that:

{% code title="<plugin root>/src/Service/DynamicSeoUrlsSubscriber.php" %}
```php
<?php declare(strict_types=1);

namespace Swag\BasicExample\Service;

use Cocur\Slugify\SlugifyInterface;
use Doctrine\DBAL\Connection;
use Shopware\Core\Content\Seo\SeoUrlPersister;
use Shopware\Core\Defaults;
use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\DataAbstractionLayer\EntityRepositoryInterface;
use Shopware\Core\Framework\DataAbstractionLayer\Event\EntityWrittenEvent;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\EqualsFilter;
use Shopware\Core\Framework\Uuid\Uuid;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class DynamicSeoUrlsSubscriber implements EventSubscriberInterface
{
    public const ROUTE_NAME = 'example.route.name';

    /**
     * @var SeoUrlPersister
     */
    private SeoUrlPersister $seoUrlPersister;

    /**
     * @var EntityRepositoryInterface
     */
    private EntityRepositoryInterface $salesChannelRepository;

    /**
     * @var SlugifyInterface
     */
    private SlugifyInterface $slugify;

    public function __construct(
        SeoUrlPersister $seoUrlPersister,
        EntityRepositoryInterface $salesChannelRepository,
        SlugifyInterface $slugify
    ) {
        $this->seoUrlPersister = $seoUrlPersister;
        $this->salesChannelRepository = $salesChannelRepository;
        $this->slugify = $slugify;
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'swag_example.written' => 'onEntityWritten'
        ];
    }

    public function onEntityWritten(EntityWrittenEvent $event): void
    {
        $urls = [];

        $salesChannelId = $this->getStorefrontSalesChannelId($event->getContext());
        if (!$salesChannelId) {
            // Might want to throw an error here
            return;
        }

        foreach ($event->getWriteResults() as $writeResult) {
            $urls[] = [
                'salesChannelId' => $salesChannelId,
                'foreignKey' => $writeResult->getPrimaryKey(),
                // The name of the route in the respective controller
                'routeName' => self::ROUTE_NAME,
                // The technical path of your custom route, using a given parameter
                'pathInfo' => '/example-path/' . $writeResult->getPrimaryKey(),
                'isCanonical' => 1,
                // The SEO URL that you want to use here, in this case just the name
                'seoPathInfo' => '/' . $this->slugify->slugify($writeResult->getPayload()['name']),
            ];
        }

        // You might have to create a new context using another specific language ID
        $this->seoUrlPersister->updateSeoUrls($event->getContext(), self::ROUTE_NAME, array_column($urls, 'foreignKey') , $urls);
    }

    private function getStorefrontSalesChannelId(Context $context): ?string
    {
        $criteria = new Criteria();
        $criteria->addFilter(new EqualsFilter('typeId', Defaults::SALES_CHANNEL_TYPE_STOREFRONT));

        return $this->salesChannelRepository->searchIds($criteria, $context)->firstId();
    }
}
```
{% endcode %}

{% hint style="info" %}
This guide will not cover creating an actual controller with the example route. Learn how that is done in our guide about
[creating a storefront controller](../../storefront/add-custom-controller.md). 
{% endhint %}

First of all, we're passing three services into our subscriber:
- The mentioned `SeoUrlPersister` class, which will take care of writing the SEO URL entries to the database
- The sales channel repository in order to get the storefront sales channel ID. Might be obsolete if you can access it otherwise
- An instance of the `SlugifyInterface`, which will take care of creating a clean URL

In the next step, we're listening to the `swag_example.written` event, which is basically the name of your entity, `swag_example` for this guide, plus
the `.written` suffix. It will be executed everytime your entity is written, let it be updated or created, via the DAL.

Once that happens, we're executing the `onEntityWritten` method.
Here we're searching for the storefront sales channel ID and then iterate through each write result.
That is, because multiple instances of your entities can be written at once and you want to cover them all.

Using the information from the write result, we're building an array of new URLs to be saved to the database.
Make sure to properly change the example values for the following columns:
- `routeName`: The name of your route, configured in the respective controller
- `pathInfo`: The technical non-SEO path for your route, also configured in your respective controller
- `seoPathInfo`: The actual SEO path, mostly the name of your entity

All of this information are then passed to the `SeoUrlPersister` using the `updateSeoUrls` method.

#### Reacting to deletion of the entity

If your entity is deleted, you can use the `swag_example.deleted` event and use the `updateSeoUrls` method like this:

```php
public static function getSubscribedEvents(): array
{
    return [
        'swag_example.written' => 'onEntityWritten',
        'swag_example.deleted' => 'onEntityDeleted'
    ];
}

public function onEntityDeleted(EntityDeletedEvent $event): void
{
    $foreignKeys = [];
    foreach ($event->getWriteResults() as $writeResult) {
        $foreignKeys[] = $writeResult->getPrimaryKey();
    }

    $this->seoUrlPersister->updateSeoUrls($event->getContext(), self::ROUTE_NAME, $foreignKeys, []);
}
```

This way the respective SEO URLs will be marked as `is_deleted` for the system.
However, this SEO route will remain accessible, so make sure to implement a check whether or not the entity still exists in your controller.

#### Writing SEO URLs for another language

In the example mentioned above, we're just using the `Context` instance from the event, for whichever language that is.
You can be more specific here though, in order to properly define the language ID yourself here and therefore ensuring it is written
for the right language.

```php
$context = new Context(
    $event->getContext()->getSource(),
    $event->getContext()->getRuleIds(),
    $event->getContext()->getCurrencyId(),
    [$languageId]
);
```

You can then pass this context to the `updateSeoUrls` method.