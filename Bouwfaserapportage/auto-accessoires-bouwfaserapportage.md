# Auto-accessoires Bouwfaserapportage

Als eerste maken we een nieuwe route waar je op kan landen voor de auto-accessoires;

```yaml
sylius_shop_car_accessories:
    path: /auto-accessoires/{taxon}
    methods: [ GET ]
    defaults:
        _controller: App\Controller\CarAccessoriesController::overviewAction
        taxon: 'universeel'
```

Hierna maken we de controller aan die we hier net hebben ingesteld.

Binnen deze controller maken we de functie die aangewezen staat in de route; `overviewAction`, deze neemt een `string $taxon = 'universeel'`, en geeft een `Response` terug.

We zetten 'universeel' als standaardwaarde omdat dit de code is van de taxon op de buitenste laag in de taxon auto-accessoires familie.

nu halen we de taxon op van welke de code is meegegeven;

```php
/** @var Taxon $rootTaxon */
$rootTaxon = $this->taxonRepository->findOneBy(["code" => $taxon]);
```

Hierna moeten we kijken of de huidige `$rootTaxon` verder nog kinderen heeft of niet, als deze geen kinderen meer heeft, moeten we een overzicht met producten weergeven, heeft ie nog wel kinderen? Dan moeten deze route redirecten naar zichzelf.

De functie is niet groot maar lijkt wel wat ingewikkeld;

```php
public function overviewAction(string $taxon = "universeel"): Response
	{
		/** @var Taxon $rootTaxon */
		$rootTaxon = $this->taxonRepository->findOneBy(["code" => $taxon]);

		// check if there aren't any enabled children to the root taxon given, if not redirect
		if (!(count(array_filter($rootTaxon->getChildren()->toArray(), fn($child) => $child->isEnabled())) > 0)) {
			return $this->redirectToRoute('bitbag_sylius_elasticsearch_plugin_shop_list_products', [
				'slug' => $rootTaxon->getSlug(),
			]);
		}

		return $this->render("@SyliusShop/Taxon/universalOverview.html.twig", [
			'root' => $rootTaxon,
			'isChild' => !($taxon === "universeel"),
		]);
	}
```

Ik loop je er even doorheen vanaf na het ophalen van de `$rootTaxon`.

Eerst filteren we alle kinderen van de `$rootTaxon`, zodat we alleen maar de kinderen overhouden welke enabled zijn.
hierna tellen we de uitkomst hiervan, mocht de uitkomst hiervan niet meer zijn dan 0, redirecten we naar de route `bitbag_sylius_elasticsearch_plugin_shop_list_products`, met als parameter 'slug': de slug van de `$rootTaxon`.

Deze route komt uit de elasticsearch plugin, maar hebben we aangepast voor onze benodigdheden:
Het originele pad hiervoor was `/product-list/{slug}`, dit veranderen we naar `/auto-accessoires/products/{slug}`.
verder was het originele template `@BitBagSyliusElasticsearchPlugin/Shop/Product/index.html.twig`, deze passen we aan naar `@SyliusShop/Taxon/universalOverviewProducts.html.twig`.

Deze template komen we later op.

Als code executie voorbij de if-statement komt, renderen we het template `@SyliusShop/Taxon/universalOverview.html.twig`, met parameters;
- root: `$rootTaxon`,
- isChild: `!($taxon === "universeel")`.
Ischild kijkt dus of de meegegeven taxon string gelijk is aan 'universeel', zo niet, geeft hij true mee, anders false.

`universalOverview.html.twig`, loopt over de kinderen en de kleinkinderen van de meegegeven taxon, en geeft deze weer met linkjes naar `sylius_shop_car_accessories`, met als parameter `taxon`, de code van de huidige (klein)kind-taxon.

`universalOverviewProducts.html.twig`  heeft een overzicht van producten, met de naam van de categorie en de hoeveelheid producten er zijn gevonden bovenaan. Per product word er (als ze zijn gedefinieerd) de eerste afbeelding van dit product weergegeven aan de linker kant van de productrij, in het midden de titel en de korte beschrijving van het product, en aan de rechter kant de hoeveelheid producten er nog beschikbaar zijn, de prijs, en een formulier om dit product toe te voegen aan de winkelwagen (mits er op zn minst 1 product nog beschikbaar is om te bestellen). Als er geen producten zijn gevonden dan word dit ook weergegeven.
