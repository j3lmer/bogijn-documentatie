# Assortiment pagina Bouwfaserapportage

Er zijn drie routes nodig om de assortimentpagina-functionaliteit te implementeren, dus laten we daar maar mee beginnen.

```yaml
bogijn_assortment:
    path: /assortiment
    methods: [GET]
    defaults:
        _controller: App\Controller\AssortmentController::indexAction

bogijn_assortment_child:
    path: /assortiment/{genericArticleId}
    defaults:
        _controller: App\Controller\AssortmentController::childAction

bogijn_assortment_brand:
    path: /assortiment/{genericArticleId}/brand-selection
    defaults:
        _controller: App\Controller\AssortmentController::brandSelectionAction
```

We definiëren hier drie routes,

- `bogijn_assortment`

- `bogijn_assortment_child`

en

- `bogijn_assortment_brand`

Deze drie routes wijzen naar controller functies uit een controller welke we aan gaan maken genaamd `AssortmentController`

op `Bogijn3.0/src/Controller` maken we dit bestand (`AssortmentController.php`) aan.

We weten al dat we hier Merk, Generiek artikel en TecDoc data nodig hebben dus injecteren we in de controller de services hiervoor:

```php
private GenericArticleRepository|ObjectRepository|EntityRepository $articleRepository;
	private BrandRepository|ObjectRepository|EntityRepository $brandRepository;

	/**
	 * @param EntityManagerInterface $em
	 * @param TecDocClient           $tecDocClient
	 * @param SluggerInterface       $slugger
	 */
	public function __construct(private EntityManagerInterface $em, private TecDocClient $tecDocClient, private SluggerInterface $slugger)
	{
		$this->articleRepository = $this->em->getRepository(GenericArticle::class);
		$this->brandRepository = $this->em->getRepository(Brand::class);
	}
```

Gezien we iets moeten doen met routes heb ik ook alvast de `SluggerInterface` geinjecteerd, welke functionaliteit bevat om routes URL-vriendelijk te maken.

Laten we beginnen met de functie die de `bogijn_assortment` route verwacht; `indexAction`.

```php
/**
 * @return Response
 */
public function indexAction(): Response
{}
```
We willen hier natuurlijk `assemblyGroups` ophalen. Maar deze staat nergens in onze database. Dit moeten we eerst oplossen

We voegen aan het generiek artikel Entity het volgende property toe:

```php
/**
 * @ORM\Column(type="string", length=255, nullable=true)
 */
private ?string $assemblyGroup;
```

met simpele getters en setters.

Hierna voegen we aan de `importGenericArticles` functie in de `TecDocDataManager` uit de `stor-e-sylius-tecdoc-plugin` een enkele lijn functionaliteit toe welke er voor zal zorgen dat de `assemblyGroup` waarde op de `GenericArticle` ingevuld zal zijn:

```php
$genericArticle->setAssemblyGroup($item['assemblyGroup'] ?? null);
```

Ten slotte voegen we aan de `GenericArticleRepository` een functie toe welke ons alle ingeschakelde niet-null `GenericArticle`s teruggeeft:

```php
public function findByEnabledNotNullAssemblyGroup(): array
{
    $qb = $this->createQueryBuilder('g');
    return $qb->select('g')
        ->where($qb->expr()->isNotNull('g.assemblyGroup'))
        ->andWhere($qb->expr()->eq('g.enabled', ':isEnabled'))
        ->groupBy('g.assemblyGroup')
        ->setParameter('isEnabled', true)
        ->getQuery()
        ->getResult();
}
```

Nu we deze functie hebben (en een migratie voor deze veranderingen hebben gegenereerd) kunnen we teruggaan naar de `indexAction` functie in de `AssortmentController`.

We moeten uiteindelijk de naam van de `assemblyGroup` weergeven, als link naar `bogijn_assortment_child` route, verder hebben we de data van elk generiek artikel niet nodig. Om dit te bewerkstelligen mappen we de data die we terugkrijgen van deze functie zoals zojuist beschreven;

```php
$assemblyGroups = array_map(function ($genericArticle) {
    /** @var GenericArticle $genericArticle */

    return [
        'name' => $genericArticle->getAssemblyGroup(),
        'url' => $this->generateUrl('bogijn_assortment_child', ['genericArticleId' => $genericArticle->getId()]),
    ];
}, $this->articleRepository->findByEnabledNotNullAssemblyGroup());
```

Dit zorgt er voor dat we een tweedimensionele array terugkrijgen welke namen en URLs bevat van elk opgehaalde generiek artikel.

Nu hebben we alle data wat we nodig gaan hebben voor de `bogijn_assortment` route. Het enige wat nog ontbreekt is dat het nog niet op alfabetische volgorde is gegroepeerd.

Gezien de groepeerfunctionaliteit niet gelimiteerd is tot alleen deze route, schrijf ik hier een losse functie voor in de controller;

```php
/**
 * @param iterable $data
 *
 * @return array
 */
private function getGrouped(iterable $data): array
{
    $grouped = [];

    foreach ($data as $group) {
        $grouped[substr(strtoupper($group['name']), 0, 1)][] = $group;
    }
    ksort($grouped);

    return $grouped;
}
```

Deze functie lijkt weliswaar wat verwarrend, maar laat me het je uitleggen;

We maken een lege array aan genaamd `$grouped`.
We itereren over de meegegeven `$data` variabele, en voor elk item in deze array nemen we de de waarde op de locatie `name`, en veranderen alles hierin naar hoofdletters. Hierna snijden we alles van deze waarde af zodat we enkel het eerste (hoofd)letter overhouden, en gebruiken we deze als sleutelwaarde in de array, met als inhoudelijke waarde op deze plek, de hele waarde van uit deze iteratie, we zorgen er ook voor dat deze niet de vorige waarde overschrijft maar er aan toevoegt, door een extra paar `[]` toe te voegen na het definieren van de sleutelwaarde.

Hierna gebruiken we php's ingebouwde functie `ksort` om de array te sorteren (https://www.php.net/manual/en/function.ksort.php).

Hierna zou `$grouped` er iets als dit uit moeten zien;

```php
[
    'A' => [
        [
            'name' => 'MontageGroepNaam-1',
            'url' => 'MontageGroepUrl-1'
        ],
        [
            'name' => 'MontageGroepNaam-2',
            'url' => 'MontageGroepUrl-2'
        ]
    ],
    'B' => [
        [
            'name' => 'MontageGroepNaam-3',
            'url' => 'MontageGroepUrl-3'
        ],
        [
            'name' => 'MontageGroepNaam-4',
            'url' => 'MontageGroepUrl-4'
        ]
    ],  //... enzovoort
]
```

Nu deze functie in plaats is kunnen we een template gaan renderen:

```php
return $this->render("assortment/assortment.html.twig", ['assemblyGroups' => $this->getGrouped($assemblyGroups)]);
```

Dit bestand staat nog niet en maken we aan, op `Bogijn3.0/templates/assortment/assortment.html.twig`.


```twig
{% extends '@SyliusShop/layout.html.twig' %}

{% block content %}
	<div class="row">
		{% set middle = (assemblyGroups|length / 2)|round %}

		<div class="col col-6">
			{% for letter, chunk in assemblyGroups %}
				{% if loop.index <= middle %}
					<div class="row">
						<div class="col">
							<h1>{{ letter }}</h1>
							{% for assemblyGroup in chunk %}
								<div class="row">
									<div class="col">
										<a href="{{ assemblyGroup.url }}">{{ ssemblyGroup.name }}</a>
									</div>
								</div>
							{% endfor %}
						</div>
					</div>
				{% endif %}
			{% endfor %}
		</div>
		<div class="col col-6">
			{% for letter, chunk in assemblyGroups %}
				{% if loop.index > middle %}
					<div class="row">
						<div class="col">
							<h1>{{ letter }}</h1>
							{% for assemblyGroup in chunk %}
								<div class="col">
									<a href="{{ assemblyGroup.url }}">{{ assemblyGroup.name }}</a>
								</div>
							{% endfor %}
						</div>
					</div>
				{% endif %}
			{% endfor %}
		</div>
	</div>
{% endblock %}
```

Zoals je kan zien berekenen we het midden van de array, als we nog niet of op de helft zijn, renderen we onze content aan de linkerhelft van de pagina, zijn we over de helft? dan renderen we aan de rechter helft van de pagina.

Dit lijkt mij voldoende voor de index pagina.

We gaan verder met de `bogijn_assortment_child` route in de `AssortmentController`.

```php
/**
 * @param string $genericArticleId
 *
 * @return Response
 */
public function childAction(string $genericArticleId): Response
{}
```

De data die we hier willen ophalen zijn alle generieke artikelen gekoppeld aan de `assemblyGroup` (waarvan we het Generieke artikel id als parameter nemen), verder word deze functie vrij gelijk aan de vorige controllerfunctie;

```php
$genericArticles = array_map(
    function ($genericArticle) {

        return [
            'name' => $genericArticle->getDesignation(),
            'url' => $this->generateUrl('bogijn_assortment_brand', ['genericArticleId' => $genericArticle->getId()]),
        ];
    },
    // Zoek naar het generieke artikel met het meegegeven generiek artikel id, pak hiervan de montagegroep-waarde,
    // en gebruik deze om alle generieke artikelen op te halen welke ook deze montagegroep-waarde hebben
    // map deze wederom naar een tweedimensionale array met namen en URLs.
    $this->articleRepository->findBy(['assemblyGroup' => $this->articleRepository->find($genericArticleId)->getAssemblyGroup()])
);

return $this->render('assortment/assortment-child.html.twig', ['genericArticles' => $this->getGrouped($genericArticles)]);
```
`Bogijn3.0/templates/assortment/assortment-child.html.twig` is verder ook vrij gelijk aan `Bogijn3.0/templates/assortment/assortment.html.twig`.

```twig
{% extends '@SyliusShop/layout.html.twig' %}

{% block content %}
	<div class="row">
		{% set equal = (genericArticles|length % 2) is same as (0) %}
		{% set middle = (genericArticles|length / 2)|round %}

		<div class="col col-6">
			{% for letter, chunk in genericArticles %}
				{% if loop.index <= middle %}
					<div class="row">
						<div class="col">
							<h1>{{ letter }}</h1>
							{% for genericArticle in chunk %}
								<div class="row">
									<div class="col">
										<a href="{{ genericArticle.url }}">{{ genericArticle.name }}</a>
									</div>
								</div>
							{% endfor %}
						</div>
					</div>
				{% endif %}
			{% endfor %}
		</div>
		<div class="col col-6">
			{% for letter, chunk in genericArticles %}
				{% if loop.index > middle %}
					<div class="row">
						<div class="col">
							<h1>{{ letter }}</h1>
							{% for assemblyGroup in chunk %}
								<div class="col">
									<a href="{{ assemblyGroup.url }}">{{ assemblyGroup.name }}</a>
								</div>
							{% endfor %}
						</div>
					</div>
				{% endif %}
			{% endfor %}
		</div>
	</div>
{% endblock %}
```

`bogijn_assortment_brand` is wel een stukje anders dan de vorige twee controller functies. We halen hier data over de merken op vanuit TecDoc.
TecDoc functies staan hier beschreven: https://webservice.tecalliance.services/pegasus-3-0/info/

Wederom halen we wel eerst het bijbehorende generieke artikel Entity op via het generieke artikel id wat we meekrijgen als parameter.

Hierna vragen we TecDoc om het artikel wat we meekregen, en vragen we ook om alle `dataSupplierFacets` Dit geeft ons de merken welke we nodig hebben.

```php
$tecDocBrands = $this->tecDocClient->getArticles([
    'genericArticleIds' => $genericArticle->getGenericArticleId(),
    'includeDataSupplierFacets' => true,
    'perPage' => 0,
])['dataSupplierFacets']['counts'];
```
Hierna halen we alles weg behalve de `dataSupplierId` waardes;

```php
$brandIds = array_map(function ($tecDocBrand) {
    return $tecDocBrand['dataSupplierId'];
}, $tecDocBrands);
```

en halen we onze eigen Entities op via deze Ids;

```php
$brands = $this->brandRepository->findBy([
    'enabled' => true,
    'brandId' => $brandIds,
]);
```

nu `slug`-en we de generiek artikel designations en geven we deze mee aan onze template;

```php
$genericArticleSlug = (string) $this->slugger->slug($genericArticle->getDesignation())->lower();

return $this->render('assortment/assortment-brand.html.twig', [
    'genericArticleSlug' => $genericArticleSlug,
    'brands' => $brands,
]);
```

tenslotte maken we nog even de bijbehorende template aan:

```twig
{% extends '@SyliusShop/layout.html.twig' %}

{% block content %}
	{% for brand in brands %}
		{% if loop.index0 is divisible by(4) %}
			<div class="row my-4">
		{% endif %}
		<div class="col">
			<div class="card">
				<img src="/media/image/brands/{{ brand.image }}" class="card-img-top" alt="{{ brand.brandName }}"/>
				<div class="card-text d-flex justify-content-center">
					<a href="{{ path('store_sylius_tecdoc_brand_generic_article', {'brandSlug': brand.slug, 'genericArticleSlug': genericArticleSlug}) }}">{{ brand.brandName }}</a>
				</div>
			</div>
		</div>
		{% if loop.index is divisible by(4) or loop.last %}
			</div>
		{% endif %}
	{% endfor %}
{% endblock %}
```

als de huidige iteratie-index deelbaar is door vier, maken we een nieuwe rij aan, verder maken we een simpele bootstrap-card met een afbeelding en een link naar de (bestaande) bijbehorende pagina.