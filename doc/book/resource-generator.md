# Generating Resources from PHP Objects

In the previous chapter, [we discussed links and resources](links-and-resources.md).
The primitive objects allow us to create representations easily, but do not
answer one critical question: how can we create resources based on existing PHP
object types?

To answer that question, we provide two related features: metadata, and a
resource generator.

## Metadata

Metadata allows you to detail the requirements for generating a HAL
representation of a PHP object. Metadata might include:

- The PHP class name to represent.
- A URI to use for the generated resource's self relational link.
- Alternately, routing information to use with the `LinkGenerator`.
- A zend-hydrator extractor to use to serialize the object to a representation.
- Whether or not the resource is a collection, and, if so, whether pagination is
  handled as a path parameter or a query string argument, the name of the
  parameter, etc.

All metadata types inherit from `Hal\Metadata\AbstractMetadata`, which defines a
single method, `getClass()`, for retrieving the name of the PHP class to
represent; all metadata are expected to inherit from this class.

The component also provides four concrete metadata types, requiring the
following information:

- `Hal\Metadata\RouteBasedCollectionMetadata`:
    - string `$class`
    - string `$collectionRelation`
    - string `$route`
    - string `$paginationParam = 'page'` (name of the parameter indicating the
      current page of results)
    - string `$paginationParamType = self::TYPE_QUERY` (one of "query" or "placeholder")
    - array `$routeParams = []` (associative array of substitutions to use with
      the designated route)
    - array `$queryStringArguments = []` (associative array of query string
      arguments to include in the generated URI)
- `Hal\Metadata\RouteBasedResourceMetadata`:
    - string `$class`
    - string `$route`
    - string `$extractor` (string service name of the zend-hydrator hydrator to
      use for extracting data from the instance)
    - string `$resourceIdentifier = 'id'` (name of the property uniquely
      identifying the resource)
    - string `$routeIdentifierPlaceholder = 'id'` (name of the routing parameter
      that maps to the resource identifier)
    - array `$routeParams = []` (associative array of additional routing
      parameters to substitute when generating the URI)
- `Hal\Metadata\UrlBasedCollectionMetadata`:
    - string `$class`
    - string `$collectionRelation`
    - string `$url`
    - string `$paginationParam = 'page'` (name of the parameter indicating the
      current page of results)
    - string `$paginationParamType = self::TYPE_QUERY` (one of "query" or "placeholder")

We aggregate metadata in a `Hal\Metadata\MetadataMap` instance:

```php
$bookMetadata = new RouteBasedResourceMetadata(
    Book::class,
    'book',
    ObjectPropertyHydrator::class
);
$booksMetadata = new RouteBasedCollectionMetadata(
    BookCollection::class,
    'book',
    'books',
);

$metadataMap = new MetadataMap();
$metadataMap->add($bookMetadata);
$metadataMap->add($booksMetadata);
```

### Configuration-based metadata

To automate generation of the `MetadataMap`, we provide
`Hal\Metadata\MetadataMapFactory`. This factory may be used with any
[PSR-11](http://www.php-fig.org/psr/psr-11/) container. It utilizes the `config`
service, and pulls its configuration from a key named after the
`Hal\Metadata\MetadataMap` class.

Each item in the map will be an associative array. The member `__class__` will
describe which metadata class to create, and the remaining properties will then
be used to generate an instance. As an example, the above could be configured as
follows:

```php
use Api\Books\Book;
use Api\Books\BookCollection;
use Hal\Metadata\MetadataMap;
use Hal\Metadata\RouteBasedCollectionMetadata;
use Hal\Metadata\RouteBasedResourceMetadata;
use Zend\Hydrator\ObjectProperty;

return [
    'Hal\Metadata\MetadataMap' => [
        [
            '__class__' => RouteBasedResourceMetadata::class,
            'resource_class' => Book::class,
            'route' => 'book',
            'extractor' => ObjectProperty::class,
        ],
        [
            '__class__' => RouteBasedCollectionMetadata::class,
            'collection_class' => BookCollection::class,
            'collection_relation' => 'book',
            'route' => 'books',
        ],
    ],
];
```

## ResourceGenerator

Once you have defined the metadata for the various objects you will represent in
your API, you can start generating resources.

`Hal\ResourceGenerator` has the following constructor:

```php
public function __construct(
    Hal\Metadata\MetadataMap $metadataMap,
    Psr\Container\ContainerInterface $hydrators,
    Hal\LinkGenerator $linkGenerator
) {
```

We described the `MetadataMap` in the previous section, and the `LinkGenerator`
in the [previous chapter](links-and-resources.md#route-based-link-uris).

Hydrators are defind in the [zend-hydrator component](https://docs.zendframework.com/zend-hydrator/),
and are objects which can _hdyrate_ associative arrays to object instances and
_extract_ associative arrays from object instances. Generally speaking, the
`$hydrators` instance may be any PSR-11 container, but you will generally want
to use the `Zend\Hydrator\HydratorPluginManager`.

Once you have your instance created, you can start generating resources:

```php
$resource = $resourceGenerator->fromObject($book, $request);
```

(Where `$request` is a `Psr\Http\Message\ServerRequestInterface` instance; the
instance is passed along to the `LinkGenerator` in order to generate route-based
URIs for `Link` instances.)

### Customizing resource generation

The `ResourceGenerator` allows composing `Hal\ResourceGenerator\Strategy`
instances. The `Strategy` interface defines the following:

```php
namespace Hal\ResourceGenerator;

use Hal\HalResource;
use Hal\Metadata;
use Hal\ResourceGenerator;
use Psr\Http\Message\ServerRequestInterface;

interface Strategy
{
    /**
     * @param object $instance Instance from which to create Resource.
     * @throws Exception\UnexpectedMetadataTypeException for metadata types the
     *     strategy cannot handle.
     */
    public function createResource(
        $instance,
        Metadata\AbstractMetadata $metadata,
        ResourceGenerator $resourceGenerator,
        ServerRequestInterface $request
    ) : HalResource;
}
```

When you register a strategy, you will map a metadata type to the strategy; the
`ResourceGenerator` will then call your strategy whenever it encounteres
metadata of that type.

```php
$resourceGenerator->addStrategy(CustomMetadata::class, CustomStrategy::class);

// or:
$resourceGenerator->addStrategy(CustomMetadata::class, $strategyInstance);
```

If a strategy already is mapped for the given metadata type, this method will
override it.

To facilitate common operations, this library provides two traits,
`Hal\ResourceGenerator\ExtractCollection` and
`Hal\ResourceGenerator\ExtractInstance`; inspect these if you decide to write
your own strategies.
