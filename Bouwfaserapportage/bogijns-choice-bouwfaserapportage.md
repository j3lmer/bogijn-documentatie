# Bogijns choice Bouwfaserapportage

Om de Bogijns choice adapter mogelijk te maken, zal er in `framework.yaml` het volgende geconfigureerd moeten worden:

```yaml
    assets:
        packages:
            bogijns_choice:
                base_path: '%env(BOGIJNS_CHOICE_BASE_URL)%'
```

En in .env `BOGIJNS_CHOICE_BASE_URL=/media/image/bogijns_choice`.

Dit zorgt er voor dat we een plek hebben om bogijns choice product afbeeldingen op te slaan en uit op te halen.

Hierna schrijven we de adapter zelf.

De adapter komt in `Bogijn3.0/src/Adapter` te staan, en implementeert het `AdapterInterface` uit de TecDoc plugin in `storesyliustecdocplugin/src/Adapter`. Dit zorgt er voor dat het uitwisselbaar gebruikt kan worden met andere adapters.


Een adapter die de `storesyliustecdocplugin/src/Adapter/AdapterInterface` implementeert vereist de volgende functies:

`public function getProducts(array $products): array`,

`public function getLiveProductStock(array $products): array`,

`public function getProduct(ProductModel $product): ?ProductModel`.

Deze functies zijn nodig om een regulier product / lijst aan producten te 'adapteren'. Dit is exact wat er bereikt moet worden.

In de constructor van de `BogijnAdapter` gebruiken we dependency injection om `Symfony\Component\Asset\Packages` te injecteren, zodat we gebruik kunnen maken van de eerder benoemde configuratie in `framework.yaml`.

`getProducts` en `getLiveProductsStock` geven hetzelfde terug als `getProduct`, enkel in lijstvorm.

`getProduct` zit dus de logica achter. `getProduct`  kijkt naar of het functie-argument product data supplier id gelijk is aan de constante waarde ASHUKIID (442), mocht dit niet zo zijn word het product zoals hij is teruggegeven. Als dit wel het geval is voeren we een functie uit om het meegegeven product te manipuleren naar een Bogijns choice product.

`private function replaceAshukiWithBogijn(ProductModel $ashukiProduct): ProductModel`

Hierin manipuleren we handmatig het meegegeven productmodel

```php
$product->setCode($product->getCode());
		$product->setBrandName('BOGIJNS CHOICE');
		$product->setbrandNo('999999');
		$product->setArticleNumber(strrev($product->getArticleNumber()));

		$product->setImages([
			'typeKey' => 1,
			'imageURL3200' => $this->packages->getUrl($product->getArticleNumber().'.jpg', 'bogijns_choice'),
			'imageURL800' => $this->packages->getUrl($product->getArticleNumber().'.jpg', 'bogijns_choice'),
			'imageURL400' => $this->packages->getUrl($product->getArticleNumber().'.jpg', 'bogijns_choice'),
		]);

		return $product;
```

We veranderen dus de brandname, het brandnummer en het artikelnummer van de product.

Verder zetten we de afbeeldingen naar Bogijns choice afbeeldingen door middel van `$this->packages`, Deze afbeeldingen handmatig moeten worden toegevoegd op de locatie beschreven in .env.

Tenslotte zullen alle services welke een adapter injecteren in de services, de bogijns choice adapter moeten gebruiken.