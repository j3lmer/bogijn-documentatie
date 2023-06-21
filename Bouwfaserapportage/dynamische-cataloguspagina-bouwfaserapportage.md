# Dynamische cataloguspagina Bouwfaserapportage

Als eerste willen we een `{TECDOC}` indicator kunnen uitzoeken en uitlezen, hiervoor zoeken we het twig bestand op wat standaard word gerendered wanneer er een sylius pagina word weergegeven. Dit is `Bogijn3.0/templates/bundles/BitBagSyliusCmsPlugin/Shop/Page/show.html.twig`

Hierin zetten we twig variabel `shortcodeIdentifier` op `{TECDOC}`, `code` op `null` en `content` op een lege string.

Daarna checken we of de `shortcodeIdentifier` zich bevind in de content van de huidige pagina vertaling, maken we twig variabel `break` aan met waarde `false`, en variabel `split` naar de uitkomst van de pagina vertaling content, welke gesplit is op de `{TECDOC}` tags. Dit resulteert in een array.

Daarna loopen we over deze array in twig, mits `break` op false staat. Als de huidige loop iteratie waarde niet leeg is en numeriek is, dan hebben we de waarde tussen de `TECDOC` tags gevonden. Deze slaan we op in `code`

Nu zetten we de `content` variabele naar de uitkomst van `split`, welke gefilterd is zodat alle lege waardes en waardes die gelijk zijn aan `code` of welke leeg zijn er uit zijn, en weer samengevoegd is tot string.

We veranderen `{{ bitbag_cms_render_content(page) }}` met `{{ content|raw is not same as("") ? content|raw : bitbag_cms_render_content(page) }}`, zodat de tecdoc code zich niet meer bevind in de pagina.

Na de standaard opmaak van de pagina, checken we in twig of `code` null is of niet, mocht dit niet het geval zijn, dan halen we uit de url parameters het pagina nummer, de sorteermodus en de sorteer directie, welke, als deze niet gezet zijn, resulteren naar `"1"`, `"score"` en `"asc"` respectief. Na het ophalen van deze waardes geven we deze mee aan een `render(url())` call, welke aanroept naar een nieuwe route die we nu gaan maken; `bogijn_sylius_tecdoc_render_shortcode`.

```yaml
bogijn_sylius_tecdoc_render_shortcode:
    path: /shortcode/{shortCode}/{page}/{sort}/{direction}
    defaults:
        _controller: Store\SyliusTecDocPlugin\Controller\TecDocController::renderShortCode
    requirements:
        shortCode: .+(?<!/)
        page: .+(?<!/)
        sort: .+(?<!/)
        direction: .+(?<!/)

```

Nu kunnen we dus functionaliteit maken om binnen de `show.html.twig` pagina een tecdoc productenlijst weer te geven.

`public function renderShortCode(string $shortCode, int $page, string $sort, string $direction): Response`

Hierbinnen returnen we gelijk een render naar `@StoreSyliusTecDocPlugin/Product/shortCode.html.twig`, met als parameters:

- addToCartForm

- pagination

AddToCartForm is een resultaat van de volgende expressie:

```php
$this->createForm(AddToCartType::class, null, [
    'action' => $this->generateUrl('store_sylius_tecdoc_add_to_cart'),
]);
```

Dit is bestaande functionaliteit en ga ik verder niet op in.

pagination is hetzelfde verhaal:
```php
$this->paginator->paginate(
    $this->tecDocProductManager->getProductsByGenericArticles(explode(',', trim($shortCode))),
    $page,
    25,
    [
        'defaultSortFieldName' => $sort,
        'defaultSortDirection' => $direction,
    ]),
```
We 'explode'-en dus de shortcode op een comma, zodat we overblijven met een array van nummers, in plaats van een comma gesepereerde string van nummers.

Deze array geven we dus mee aan `getProductsByGenericArticles`. Deze functie bestaat nog niet, laten we deze gaan maken.

In de TecDocProductManager:

`public function getProductsByGenericArticles(array $genericArticleIds): array`

Eerst halen we TecDoc artikelen op:

```php
$result = $this->tecDocClient->getArticles([
    'page'                   => 1,
    'perPage'                => 1000,
    'genericArticleIds'      => $genericArticleIds,
    'includeArticleCriteria' => true,
    'includeImages'          => true,
    'includeMisc'            => true,
    'includeGenericArticles' => true,
    'dataSupplierIds'        => $this->brandIds,
]);
```
(this->brandIds zijn de brand ids van alle brands in de database)

dan checken we of er artikelen zijn teruggekomen, mocht dit niet zo zijn returnen we een lege array.

Dan mappen we alle artikelen naar `ProductModel`en,
Halen we ze door de product adapter en de priceRuleApplicator, en returnen we ProductModellen

Het resultaat van deze functie is dus wat er gepagineerd gaat worden, Verder geven we de hoeveelste pagina het is mee, en de sorteermodus en sorteer directie mee.

Tenslotte dus nog het twig bestand:

`@StoreSyliusTecDocPlugin/Product/shortCode.html.twig`

Deze pagina gebruikt het meegegeven paginatie variabel om sorteerfunctionaliteit te leveren, De hoeveelheid gepagineerde items weer te geven, de producten zelf weer te geven, deze toe te voegen aan de winkelwagen, en de pagineringsfunctionaliteit zelf te gebruiken.

