# Zoekfunctionaliteit Bouwfaserapportage


## Installatie

Om te beginnen voeren we `composer require bitbag/elasticsearch-plugin` uit, en voegen we 

```php
FOS\ElasticaBundle\FOSElasticaBundle::class => ['all' => true],
BitBag\SyliusElasticsearchPlugin\BitBagSyliusElasticsearchPlugin::class => ['all' => true],
```

toe aan onze `config/bundles.php` bestand. Hierna zorgen we er voor dat onze `ProductVariant` de `BaseProductVariantInterface` implementeerd, en de `ProductVariantTrait` gebruikt.

Tevens voegen we in `config/packages/_sylius.yaml` de configuratie voor de plugin toe.

```yaml
 - { resource: "@BitBagSyliusElasticsearchPlugin/Resources/config/config.yml" }
```

Om het pakket zelf verder te configureren zorgen we er voor dat `config/packages/fos_elastica.yaml` er als volgt uit ziet:

```yaml
fos_elastica:
    clients:
        default: { url: '%env(ELASTICSEARCH_URL)%' }
```

Dit zorgt er voor dat de sylius plugin en de lokale elasticsearch installatie met elkaar kunnen communiceren.

in `.env` zetten we vervolgens `ELASTICSEARCH_URL=http://localhost:9200/` 

Bovenaan de `config/routes.yaml` definieren we de routes die standaard worden geconfigureerd door de elasticsearch plugin, en een aantal van ons zelf:

```yaml
bitbag_sylius_elasticsearch_plugin:
    resource: "@BitBagSyliusElasticsearchPlugin/Resources/config/routing.yml"

sylius_elasticsearch_search:
    path: /search
    methods: [ POST ]
    defaults:
        _controller: App\Controller\SearchController::searchAction

sylius_elasticsearch_form_renderer:
    path: /get-elastic-search-form
    methods: [GET]
    defaults:
        _controller: App\Controller\SearchController::getElasticSearchForm

bitbag_sylius_elasticsearch_plugin_shop_list_products:
    path: /auto-accessoires/products/{slug}
    defaults:
        _controller: bitbag_sylius_elasticsearch_plugin.controller.action.shop.list_products
        template: "@SyliusShop/Taxon/universalOverviewProducts.html.twig"
    requirements:
        slug: .+

bitbag_sylius_elasticsearch_plugin_shop_auto_complete_product_name:
    path: /auto-complete/product
    defaults:
        _controller: bitbag_sylius_elasticsearch_plugin.controller.action.shop.auto_complete_product_name
    requirements:
        slug: .+

bogijn_get_elasticsearch_products_ajax:
    path: /get-elastic-search-products
    methods: [ POST ]
    defaults:
        _controller: App\Controller\SearchController::getElasticSearchProductsAjax

bogijn_get_tecdoc_products_ajax:
    path: /get-tecdoc-products
    methods: [ POST ]
    defaults:
        _controller: App\Controller\SearchController::getTecDocProductsAjax
```


Verdere standaard configuratie van de Elasticsearch plugin houdt in dat we een redirect opzetten van de standaard sylius shop pructen index pagina in `config/routes/sylius_shop.yaml`

```yaml
redirect_sylius_shop_product_index:
    path: /{_locale}/taxons/{slug}
    controller: Symfony\Bundle\FrameworkBundle\Controller\RedirectController::redirectAction
    defaults:
        route: bitbag_sylius_elasticsearch_plugin_shop_list_products
        permanent: true
    requirements:
        _locale: ^[a-z]{2}(?:_[A-Z]{2})?$
        slug: .+
```

Tenslotte draaien we de volgende twee commandos:

`bin/console assets:install` en `bin/console fos:elastica:populate`.

Hierna zijn we klaar om de Sylius Elasticsearch Plugin te gaan gebruiken.


## Configuratie

Om te beginnen maken we een nieuwe controller in `Bogijn3.0/src/Controller`, deze noemen we `SearchController`.

De constructor voor deze controller-klasse neemt de volgende klasses

- `NamedProductsFinderInterface $namedProductsFinder`

- `TecDocClient $tecDocClient`

- `AdapterInterface $adapter`

- `TaxonRepository $taxonRepository`

Ook zetten we een standaard constate waarde `DEFAULTMAXPRODUCTS`, deze zetten we voor nu op 10.

Hierna definieren we de functie die we aanroepen in onze eerste custom geconfigureerde route, `searchAction`. Deze neemt het `Request $request` object, en geeft een `Response` terug.

Het eerste wat we doen is de zoekopdracht die de gebruiker heeft ingevoerd uit het `$request` halen, Hiervoor schrijven we onze eigen functie `getSearchQuery(Request $request)`.

in `getSearchQuery` maken we een variabele `$searchQuery`, deze zetten we op `null`.

hierna halen we de sessie op uit het `$request` als volgt: `$request->getSession()`. Deze slaan we op in `$session`. Hierna checken we of het `$request` een zoekopdracht bevat, mocht dit het geval zijn vullen we `$searchQuery` hiermee en zetten we op de `$session`, dat de laatste zoekopdracht, de huidige is.

Mocht er geen zoekopdracht in het `$request` gevonden zijn, dan proberen we de laatste zoekopdracht uit de `$session` te gebruiken.

Hierna geven we de `$searchQuery` terug aan de aanroepende functie.


Na het ophalen van de `$searchQuery`, willen we een 'Voeg aan winkelwagen' template formulier genereren, voor wanneer we eenmaal de zoekresultaten hebben gevonden. Hiervoor hebben we een nieuw `Type` nodig. We navigeren ons naar `sylius-tecdoc-plugin/src/Form/Type` en creeëren hier de `AddToCartType.php`. Deze extend `Symfony\Component\Form\AbstractType`, en heeft de functie `buildForm`, welke een `FormBuilderInterface $builder` neemt, en een `array $options`. (`$options` gebruiken we hier niet. Deze is enkel vereist door Symfony).

Binnen deze functie roepen we de `$builder` 3 keer aan met zijn `add` methode.

```php
->add('quantity', IntegerType::class, ['label' => 'Aantal', 'data' => 1])
->add('code', HiddenType::class)
->add('addToCart', SubmitType::class, ['label' => 'Add to cart'])
```

Nu we de `AddToCartType` hebben gemaakt, kunnen we deze gebruiken om hier een formulier van te genereren, d.m.v. de algemene `$this->createForm` Symfony functie.

Uiteindelijk ziet het er als volgt uit: 

```php
$addToCartForm = $this->createForm(AddToCartType::class, null, [
    'action' => $this->generateUrl('store_sylius_tecdoc_add_to_cart'),
]);
```

Hierna halen we de lokale-database producten, en de TecDoc-producten, en eventuele overeenkomende taxons (productcategorieën) op en geven we deze mee aan `@StoreSyliusTecDocPlugin/Catalog/search.html.twig` om gerendered te worden, samen met het `addToCart` formulier en de zoekopdracht.


Om de lokale database producten op te halen maken en roepen we een nieuwe functie `getElasticSearchProducts(string $searchQuery, int $offset): ?array` aan, deze functie is vrij simpel, het gebruikt de `$namedProductsFinder` die we hebben geinjecteerd in de constructor, en roept hier van de functie `findByNamePart` aan, met als parameter de string zoekopdracht `$searchQuery`, hierna geeft deze functie de uitkomst van de php functie `array_slice` terug, met als hetgene wat "gesliced" word, de uitkomst van de `findByNamePart` functie,  vanaf `$offset` tot `DEFAULTMAXPRODUCTS`. 

we roepen `getElasticSearchProducts` aan als volgt vanuit `searchAction`: 

`$products = $this->getElasticSearchProducts($searchQuery, 0);`

Het enige wat we nu nog moeten doen om dit te laten werken is in `config/services.yaml` aan te geven waar symfony naar moet zoeken voor de `$namedProductsFinder`. Normaliter vind symfony zelf automatisch de meeste services wel om te injecteren, maar deze moeten we even zelf configureren. We configurerren hem als volgt:

```yaml
    App\Controller\SearchController:
        autowire: true
        arguments:
            $namedProductsFinder: '@bitbag_sylius_elasticsearch_plugin.finder.named_products'
```

Super. Nu hebben we de universele producten opgehaald, alleen, zoals ik eerder had benoemd, moeten we ook nog de TecDoc producten ophalen. We maken een nieuwe functie aan in de `SearchController`, deze noemen we: `getTecDocProducts`. Deze functie neemt dezelfde argumenten als `getElasticSearchProducts`, en geeft tevens een `?array` terug.

Het eerste wat we doen in deze functie is de TecDoc api aanroepen, speciafiek de `getArticleDirectSearchAllNumbersWithState` functie, deze functie en documentatie zijn beschikbaar op [WEB SERVICE 3.0](https://webservice.tecalliance.services/pegasus-3-0/info/).]


Na het aanroepen van dit endpoint, doen we een aantal checks of de de data die terugkwam, correct geset is, mappen we het naar een (al bestaand) `ProductModel` klasse, welke we door de adapter heen halen. (zie: Bogijns choice). Hierna sorteren we de lijst in-place en geven we de uitkomst van `array_slice` op dezelfde manier als `getElasticSearchProducts` terug.

De laatste belangrijke functie om te beschrijven die gebruikt word in deze functie is `getTaxons`. Deze neemt enkel de string zoekopdracht, en geeft tevens een `?array` terug.

In deze functie maken we gebruik van de geinjecteerde `taxonRepository`, en maken we hier een `queryBuilder` mee aan d.m.v. de `$this->taxonRepository->createQueryBuilder()` functie, als parameter geven we hier de string 'taxon' aan mee. Dit zorgt er voor dat wanneer we de (sub)string 'taxon' in onze query gebruiken, de `queryBuilder` snapt dat we refereren naar het Entiteit waar de repository bij hoort (in dit geval de Taxon entiteit). Verder is query vrij simpel, we selecteren alle taxons waarvan de code soortgelijk is aan onze zoekopdracht. Deze query voeren we dan uit en halen we het resultaat van op, welke we in een variabel genaamd `$allTaxons` opslaan.

Hierna maken  we twee nieuwe arrays aan, de lege `$sortedTaxons`, welke we in een moment gaan vullen, en `$allowedRootCodes`, welke de volgende twee strings bevat:

- "TECDOC"

- "universeel"

Dan loopen we door `$allTaxons` heen, met als variabel waar we onze huidige taxon in opslaan `$taxon` genoemd is.

Hierna halen we van deze `$taxon` de rootcode op (`$taxon->getRoot()->getCode()`).

(Taxons kunnen "ouders" hebben, een 'root' is een taxon zonder ouder.)

Hierna checken we of deze rootcode in onze lijst met `$allowedRootCodes` staat, en als dit het geval is dan voegen we deze `$taxon` toe aan `$sortedTaxons` op de plek `$rootCode`, op een nieuwe index.

na deze loop geven we `$sortedTaxons` terug aan de aanroepende functie.

Zoals ik eerder al benoemd had, word er aan het einde van de `searchAction` een twig bestand gerendered. Ik zal nog even de benodigde logica in dit bestand globaal behandelen.

Voor TecDocproducten, elasticsearchproducten en categorieen geld alle drie dat, mochten ze niet gedefinieerd of leeg zijn, er een simpele alert komt die aan de gebruiker verteld dat er geen resultaten zijn.

Anders word er door de lijst heen gelooped, en het product / de categorie weergegeven. Is het een product? dan word er weergegeven hoeveel er in voorraad zijn. Tevens is er dus een 'Voeg aan winkelwagen' formulier. Ook worden er voor de producten 2 variabelen bijgehouden, hoeveel iteraties er per productsoort (TecDoc/Elastic) zijn weergegeven

Het mocht je zijn opgevallen dat we in `searchAction` voor beide tecdoc en elasticsearch producten niet alle mogelijke producten worden meegegeven aan dit bestand. Hier is bewust voor gekozen, zodat wanneer een zoekterm honderden resultaten teruggeeft, de webpagina niet vastloopt. In plaats hiervan is er gekozen voor een ajax aanpak tot dit probleem. We maken voor beide TecDoc alswel elastic producten in dit twig bestand knoppen aan om meer producten te laden, we zetten op deze knoppen data attributen voor de meegegeven zoekterm, hoeveel producten er al zijn ingeladen voor dit producttype, en de url waar er meer producten uit kunnen worden gehaald, bij TecDoc producten is dit dus `bogijn_get_tecdoc_products_ajax`, en voor elastic is dit `bogijn_get_elasticsearch_products_ajax`.

Om hier functionaliteit aan te geven verplaatsen wij ons naar `themes/BootstrapTheme/assets/js/index.js`

In dit bestand importeren we `ConfigureLoadMoreButtonElastic`, en `ConfigureLoadMoreButtonTecDoc` van `./load-more-products-ajax.js`, welke we straks behandelen.

Onderaan `index.js` proberen we de 'meer laden' knoppen op te halen, en mochten deze bestaan op de pagina, roepen we de `ConfigureLoadMore...` functies aan.

Deze twee functies zijn vrij gelijk, dus om wat herhaalde text te voorkomen kies ik er voor om enkel `ConfigureLoadMoreElastic` te behandelen.

de `ConfigureLoadMore` functies nemen het knop element, en de tabel van het bijbehorende producttype in als parameters

Als eerste halen we de huidige offset van de knop af, en zetten we deze op een globale variabele,

hierna configureren we een `data` variabele om uiteindelijk mee te geven wanneer we meer producten willen ophalen via een HTTP request. Deze ziet er als volgt uit:

```js
const data = {
    tecdoc: {
      searchQuery: btnElem.dataset.tecdocSearchquery,
    },
  };
```

Dan halen halen we de eerste x hoeveelheid nieuwe producten alvast op, d.m.v de asynchrone functie `getProducts()`, deze neemt de url waar het request straks heen moet worden gestuurd, het zojuist-benoemde `data` variabele, en de uitkomst van `GetParams()`

`getParams()` neemt de string van het producttype (Elastic of TecDoc), en geeft op basis hiervan een object terug welke correct geformatteerd staat voor de http request, met als enige data de huidige offset.

`getProducts` stuurt dan een HTTP post request naar de meegegeven url, met de meegegeven data, en geeft hierna het `data` object terug van de response van dit request. Hierna checken we of de 'Laad meer' knop weg moet, door te checken of de data die word teruggegeven gezet of leeg is. Mocht dit zo zijn dan halen we de knop weg.

Hierna roepen we de functie `ConfigureEventListener` aan, welke het button element neemt, de response HTML, de `data` variabele, het tabel van het producttype, het producttype string, en de url waar eventuele volgende requests heen moeten.

Als een van deze parameters niet correct is geconfigureerd, dan stopt de functie gelijk.

Hierna word de knop geconfigureerd, zodat wanneer er op de knop word gedrukt, er meer producten in worden geladen via alle eerder benoemde functies, of dat de knop moet worden weggehaald. De eerste keer dat er word geklikt word de originele response HTML gebruikt, de keren daarna word deze opnieuw opgehaald, enkel die keer met een verhoogde producttype-offset, zodat er niet elke keer dezelfde producten worden opgehaald.

Zoals geimpliceerd, zijn er 2 verschillende URLs waar heen kan worden geroepen om nieuwe producten op te halen, om je herinnering te verfrissen zet ik hier weer de route configuratie neer hiervan.

```yaml
bogijn_get_elasticsearch_products_ajax:
    path: /get-elastic-search-products
    methods: [ POST ]
    defaults:
        _controller: App\Controller\SearchController::getElasticSearchProductsAjax

bogijn_get_tecdoc_products_ajax:
    path: /get-tecdoc-products
    methods: [ POST ]
    defaults:
        _controller: App\Controller\SearchController::getTecDocProductsAjax
```

de elastic search route neemt het `$request` object, haalt hier de zoekterm uit via de eerder benoemde `getSearchQuery` functie, en de offset via de volgende logica; `json_decode($request->query->get("elastic_search"))->offset`. Deze worden gebruikt om de eerder benoemde functie `getElasticSearchProducts` aan te roepen, welke dan samen met het producttype ('elastic') word meegegeven aan het teruggeven van het gerenderde twig bestand `searchDataElasticProducts.html.twig`. Dit nieuwe twig bestand bied functionaliteit om te kijken wat het type is, en op basis hiervan een lijst aan producten weer te geven, met een formulier om het product, mits het in voorraad is, toe te voegen aan de winkelwagen. De TecDoc route bied gelijke functionaliteit, enkel dan via de `getTecDocProducts` functie. Deze gebruikt `searchDataTecDocProducts` om te renderen. Deze functies geven dus HTML terug, welke dan via de javascript functies word toegevoegd aan de DOM.

Voor het renderen van dit bestand, genereren we een formulier van type `elasticSearchType`, welke TextType `searchQuery` veld, en een SubmitType `submitSearch` velden bevatten, met als actie, een redirect naar de `searchAction` (`sylius_elasticsearch_search`) url. Hierna voegen we dit formulier toe aan de parameters bij het renderen van dit twig bestand, In `vehicle_search`/`vehicle_search_sidebar` staat uiteindelijk dit formulier, welke een gebruiker kan gebruiken om te zoeken naar onderdelen zowel uit de database (universeel), of uit TecDoc.