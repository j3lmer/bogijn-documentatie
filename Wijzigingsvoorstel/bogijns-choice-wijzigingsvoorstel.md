# Bogijns choice wijzigingsvoorstel

Om te voorkomen dat producten die via de BogijnAdapter gaan, geen stock en prijs hebben, stel ik het volgende voor als aanpassing in de `getProducts` functie, samen met de `$stockRepository` te injecteren:

```php
if (empty($products)) {
    return [];
}

$codes = array_map(static function ($product) {
    return $product->getCode();
}, $products);

/**
 * @var array<string, ProductModel> $products
 */
$products = array_combine($codes, $products);
/**
 * *
 * @var Stock[] $result
 */
$result = $this->stockRepository
    ->createQueryBuilder('s')
    ->andWhere('s.code IN (:codes)')
    ->andWhere('s.price > 0')
    //->andWhere('s.stock > 0')
    ->setParameter('codes', $codes)
    ->getQuery()
    ->getResult()
;


return array_map(function (Stock $stock) use ($products) {
    $product = $this->getProduct($products[$stock->getCode()]);

    $product->setStock($stock->getStock());
    $product->setStock2($stock->getStock2());
    $product->setPrice($stock->getPrice());
    $product->setOriginalPrice($stock->getPrice());
    //			$product->setExternalIdentification($stock->getExternalIdentification());

    return $product;
}, $result);
```

Ik loop je even door de code heen:

Als de meegegeven producten leeg zijn; geef een lege array terug.

Hierna sla je alle codes van de producten op in een array de producten te mappen.

Dan maken we een key-value array aan waarbij de key de code van de product is en de value het product zelf.

Vervolgens gebruiken we de `$stockRepository`, om alle `Stock` entities op te halen van welke we de codes hebben in de `$codes` array, welke meer dan 0 stock hebben.

Tenslotte mappen we het resultaat hiervan, en gebruiken we de producten array hierbij.

Binnen de map:
    Het huidige product is de uitkomst van `$this->getProduct()`, waarbij het meegegeven product het product is uit de productenarray, op de plek van de code van de stock die we op dit moment mappen.

    vervolgens zetten we de stock, stock2, prijs, en originele prijs naar die welke op de `Stock` entity staat gedefinieerd.

    Tenslotte geven we het product terug.

Het resultaat van deze map word teruggegeven aan de aanroepende functie.