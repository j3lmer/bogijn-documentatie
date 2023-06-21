# Import commandos

Er zijn een aantal commandos om data uit de bestaande database (Bogijn2.0) naar de huidige database (Bogijn3.0) over te zetten vereist, dit betreft;

- Accounts

- Categorieën (Taxons)

- Universele producten / Auto accessoires

- Prijs en voorraad

Verder moet er ook nog een commando komen om de universele producten aan de bijbehorende categorieën te koppelen.

Ook maak ik een commando die al deze commandos achter elkaar aan uitvoert.

De basisfunctionaliteit voor al deze commandos volgt een soortgelijk systeem;

Er worden 3 variabelen gedefinieerd in de globale scope;

- `private bool $isLastRun = false;`

- `private int $offset = 0;`

- `private const CHUNK_SIZE = 100;`

Er is een `private function execute(InputInterface $input, OutputInterface $output): int`  functie,

Deze functie word door Symfony functionaliteit automatisch aangeroepen wanneer het commando word uitgevoerd.

Er worden een aantal benodigdheden geinjecteerd in de `__construct` functie, bij de meesten is dit; `private EntityManagerInterface $entityManager`, en `private string $csvPath`.

`$csvPath` is iets wat wij zelf definieëren in `config/services.yaml`, voor elk commando. In `services.yaml` benoemen we dan bij relatief pad het commando, hierbij geven we argument `$csvPath` mee, en zetten we deze naar een evaluatie van een omgevingsvariabel, gedefinieerd in een van de .env* bestanden. Uiteindelijk wijst dit dus naar een `.csv` bestand, waarvan wij data uit kunnen gaan lezen

Als voorbeeld hier de definitie van `ImportUniversalProducts`:

```yaml
App\Command\ImportUniversalProducts:
    arguments:
        $csvPath: '%env(BOGIJN_UNIVERSAL_PRODUCTS_EXPORT_LOCATION)%'
```

Verder, binnen de `execute` functie van elk commando, maken we variabel `$io`, welke een nieuwe `SymfonyStyle` bevat, welke de `$input` en `$output` variabelen neemt. Deze kunnen we gebruiken om naar de console te schrijven wanneer het commando word uitgevoerd.

Hierna definieëren we variabel `$csv`, waarvan de waarde equivalent is aan `League\Csv\Reader::createFromPath($this->csvPath)`.

We gebruiken hier dus een extern pakket om het csv bestand te gaan verwerken, dit doen we omdat het bestand vrij groot kan zijn, en we willen voorkomen dat al het geheugen van de server die dit commando uiteindelijk moet gaan draaien volledig vol komt te zitten.

Hierna initialiseren we een `while` loop, die enkel stopt wanneer `$this->isLastRun` op true staat.

Hierbinnen bereiden wij een statement voor om vanaf `$this->offset`, tot `self::CHUNK_SIZE`, de waardes uit het csv bestand te gaan ophalen. Dit statement slaan we op in `$statement`, en hierna halen we hier de records mee op door gebruik te maken van `$records = $statement->process($csv)`. Waarna we door deze records heen loopen, en er dan, per record, iets mee kunnen gaan doen. Nadat de individuele records zijn verwerkt, flushen we de `$this->entityManager`, verhogen we de waarde van `$this->offset` met de waarde van `self::CHUNK_SIZE`, en kijken we of de waarde van `$this->isLastRun` moet worden aangepast, door te kijken naar of de hoeveelheid records kleiner is dan `self::CHUNK_SIZE`. Als laatste binnen de `while` loop, plaatsen we een comment voor de terminal om weer te geven (met `$io`) dat er een chunk is verwerkt, en dat de offset op dit moment gelijk staat aan `$this->offset`.

Na de `while` loop, gebruiken we een laatste keer `$io`, om aan te geven dat het commando successvol is geexecuteerd, en `return`en we `Command::SUCCESS`, welke overeenkomt met `0`.

Nu we de algemene functionaliteit hebben besproken zal ik de individuele commandos bij langs gaan. 

## ImportAccounts

Binnen de iteratie over de individuele records, roepen wij `$this->configureAccount($record)` aan,

`configureAccount` zal er voor zorgen dat de waardes uit deze record worden gebruikt om een nieuwe `ShopUser` en `Customer` aan te maken.

Ik heb binnen de Bogijn2.0 database een testgebruiker aangemaakt, en geexporteerd om te laten zien hoe een `$record` er uit ziet:

```json
{
    "id": "1",
    "lastname": "van der Bij",
    "firstname": "jelmer",
    "company": "",
    "invoice_street": "staatnaam",
    "invoice_streetnumber": "huisnummer+toevoeging",
    "invoice_zip": "9101LZ",
    "invoice_city": "Dokkum",
    "invoice_country": "nl",
    "shipping_street": "straatnaam",
    "shipping_streetnumber": "huisnummer+toevoeging",
    "shipping_zip": "9101LZ",
    "shipping_city": "Dokkum",
    "shipping_country": "nl",
    "phone": "0611111111",
    "email": "voorbeeld@gmail.com",
    "gender": "m",
    "password": "$2y$10$qL8zz/VwzjpGCZTvdarShOmfZ4xY9IfgxRZIn9qaSzhKmGOIP7AOu",
    "salt": "",
    "token": "11f721a75f9f3339019e79b47d66ba027b32a4bb",
    "active": "yes",
    "count_logins": "3",
    "date_lastlogin": "2023-02-15 08:58:05",
    "date_lastupdate": "2023-02-08 15:13:13",
    "language": "nl",
    "mailing": "0",
    "external_id": "",
    "show_netprice": "no",
    "parent": "NULL",
    "allow_invoice_payment": "no",
    "customer_group": "0"
}
```

De service configuratie van dit commando ziet er uit als volgt:

```yaml
App\Command\ImportAccounts:
    arguments:
        $csvPath: '%env(BOGIJN_ACCOUNTS_EXPORT_LOCATION)%'
        $addressFactory: '@sylius.custom_factory.address'
```

Binnen `private function configureAccount(array $record): void` is het eerste wat we doen, variabel `$user` definieëren. Deze vullen we door te zoeken door de `$this->shopUserRepository` te doorzoeken op email, waarbij het email de `$record['email']` zou zijn. Mocht deze niet gevonden zijn, mooi! dat betekent dat we een nieuwe gebruiker aan maken, oftewel; `new ShopUser()`

Als de `$user->getId()` functie niet gelijk staat aan `null`,  dan `return`en we gelijk uit de functie, Aangezien we niet meer een nieuwe gebruiker zouden aanmaken.

We zetten `$user->setEnabled()` naar de uitkomst van `$record['active'] === "yes`, en de `user->setEmail()` naar `$record['email']`. Dan zetten we `$user->setCustomer()` naar de uitkomt van de functie `$this->getCustomer($user, $record)`. Hierna persisten we de `$user` en is de functie afgerond.

Om onvoorziene problemen te voorkomen is het eerste wat we doen binnen `private function getCustomer(Shopuser $user, array $record): CustomerInterface` checken of er al een `Customer` op deze `ShopUser` staat. Mocht dit het geval zijn word deze teruggegeven. Nu zeker wetende dat dit niet het geval is, maken we `$customer = new Customer()` aan, en roepen we hier `->setUser($user)` op aan.

Ook hier zetten we de email op d.m.v. `$customer->setEmail($user->getEmail())`, en configureren we de volgende waardes:

- `$customer->setGender($record['gender']);`

- `$customer->setFirstName($record['firstname']);`

- `$customer->setLastName($record['lastname']);`

- `$customer->setPhoneNumber($record['phone']);`

Tot slot roepen we `$customer->setDefaultAddress()` aan met de uitkomst van de functie `$this->getAddress($customer, $record)`. Dan roepen we wederom `$this->entityManager->persist()` aan op de customer en `return`en we deze.

`private function getAddress(Customer $customer, array $record): AddressInterface` is de laatste functie in dit commando bestand.

hierin zetten we `$address` naar `$this->addressFactory->createForCustomer($customer)`, configureren we de volgende waardes als volgt:

- `$address->setPhoneNumber($customer->getPhoneNumber());`

- `$address->setFirstName($customer->getFirstName());`

- `$address->setLastName($customer->getLastName());`

- `$address->setCity($record['invoice_city']);`

- `$address->setCountryCode($record['invoice_country']);`

- `$address->setPostcode($record['shipping_zip']);`

- `$address->setStreet("{$record['shipping_street']} {$record['shipping_streetnumber']}");`

En persisten we `$address` met de `entityManager`.

Nu `return`en we de `$address`, en herhaalt deze sequentie.

## ImportCategories

`services.yaml`:

```yaml
App\Command\ImportCategories:
    arguments:
        $csvPath: '%env(BOGIJN_CATEGORIES_EXPORT_LOCATION)%'
```

Globale variabelen:

```php
    private string $bogijnImagesURL = "https://bogijn.nl/assets/images/";
    private Taxon $rootTaxon;
    private array $processedRecords = [];
    private array $waitingForProcessing = [];
    private array $rootTaxonCodes = [];
```

Constructor injecties:

```php
        private TaxonRepository        $taxonRepository,
        private EntityManagerInterface $entityManager,
        private TaxonFactoryInterface  $taxonFactory,
        private SluggerInterface       $slugger,
        private FactoryInterface       $taxonImageFactory,
        private ImageUploaderInterface $imageUploader,
        private string                 $csvPath,
```

Record:

```json
{
  "id": "1",
  "title": "Gereedschap",
  "path": "gereedschap",
  "image": "ijfdoFIJzCPCoeXQHDjPTHjCsUyzofeNSAHGzR1l.webp",
  "parent_id": "0",
  "description": "",
  "synonyms": "",
  "precedence": "0",
  "assemblygroup": "0",
  "ga": "0",
  "date": "2021-08-26 08:41:22",
  "active": "no",
  "meta_title": "",
  "meta_description": "",
  "meta_keywords": "",
  "external_ids": "",
  "title_german": "Werkzeug",
  "description_german": "",
  "synonyms_german": "",
  "meta_description_german": "",
  "meta_title_german": "",
  "meta_keywords_german": ""
}
```

Verder is er wat speciafieke functionaliteit binnen `execute` welke ik eerst ga behandelen.

Na het aanmaken van `$io`  checken we eerst of er al een `Taxon` bestaat waarvan de `code` property gelijk staat aan de string "universeel". 

Als dit niet zo is, of hij bestaat wel maar deze `Taxon` zijn `isRoot()` functie geeft terug dat hij niet een root `Taxon` is, gebruiken we `$io->error()` om een error terug te geven dat deze `Taxon` niet bestaat of niet root is, en `return`en we `Command::FAILURE`, Wat gelijk staat aan 1.

Nu vullen we `$this->rootTaxonCodes` met "tecdoc", "hoofdmenu" en "universeel".

Hierna vervolgt standaard functionaliteit zoals eerder beschreven.

Binnen de loop over de `$records`, formatteren we de `$record` met `$record = $this->formatRecord($record);`, Configureren we deze record/Taxon met `$hasBeenPersisted = $this->configureCategoryTaxon($record);`

flushen we de `entityManager`, en roepen we tot slot `$this->checkIfNeededBySavedChild($record, $hasBeenPersisted);` aan.

`private function configureCategoryTaxon(array $record): bool`:

Binnen deze functie configureren we de `$record` naar een `Taxon`, enkel is dit niet altijd mogelijk. Sommige taxons vereisen een 'ouder' `Taxon`, welke pas later in het `.csv` bestand langs komen. Deze moeten we opslaan en verwerken nadat we door het hele `.csv` bestand heen zijn.

Het eerste wat we doen in deze functie is kijken of de `parent_id` waarde van deze `$record` gelijk staat aan 0, en kijken of de `path` waarde ook in de `$this->rootTaxons` array staat. Als dit zo is, roepen we `$this->addTaxon($record, $this->rootTaxon)` aan, en `return`en we true. De `addTaxon` functie word na de huidige functie behandeld.

Als de `parent_id` dus niet 0 is, of de `path` wel in de `$this->rootTaxonCodes` array zit, kijken we naar of de `parent_id` ook in de `$this->processedRecords` array zit als key, en ook wederom naar dat de `path` niet in de `rootTaxonCodes` zit. Als dit zo is, halen we de `path` op van de parent record, uit `$this->processedRecords[$record['parent_id']]['path'];`. Deze gebruiken we in de `taxonRepository` om een bestaande `Taxon` uit de database op te halen d.m.v. `$parent = $this->taxonRepository->findOneBy(["code" => $this->getCode($parentRecordPath)]);`. Hierna roepen we `$this->addTaxon($record, $parent);` aan en `return`en we true.

Mocht de `parent_id` dus niet tussen de `processedRecords` staan, voegen we aan `$this->waitingForProcessing`, op de plek `$record['parent_id']` de `$record` toe, en `return`en we false.

We roepen dus meerdere keren `$this->addTaxon` aan, `private function addTaxon(array $record, Taxon $parent): void` doet de volgende dingen:

```php
$taxon = $this->getTaxon($record);
$taxon->setParent($parent);
$this->entityManager->persist($taxon);
$this->processedRecords[$record['id']] = $record;
```

Oftewel: 

- Haal een taxon op met de `$this->getTaxon()` functie

- Zet de parent van deze `$taxon` naar de meegegeven waarde `$parent`,

- `persist` de `$taxon`

- voeg de `$record` toe aan `$this->processedRecords`, op de plek `record['id']`

`private function getTaxon(array $record): Taxon`s functionaliteit is vrij vanzelfsprekend, deze functie zoekt in de database naar een `Taxon` waarvan de `code` gelijk staat aan`$this->getCode($record['path'])`, mocht deze niet gevonden zijn, gebruiken we `$this->taxonFactory->createNew()` om een nieuwe `Taxon` aan te maken.

`private function getCode(string $path): AbstractUnicodeString` doet niks meer of minder dan `return $this->slugger->slug($path);`, welke een ingebouwde functie is.

Na het ophalen /  aanmaken van de `Taxon`, configureren we de volgende waardes:

- `$taxon->setEnabled($record['active'] === 'yes');`

- `$taxon->setCode($this->getCode($record['path']));`

- `$taxon->setName($record['title']);`

- `$taxon->setDescription($record['description']);`

- `$taxon->setSlug($this->getCode($record['path']));`

- `$taxon->setCreatedAt(new DateTime($record['date']));`

en `return`en we de `Taxon`

Terug in de `$records` loop, na het flushen roepen we dus nog `$this->checkIfNeededBySavedChild($record, $hasBeenpersisted)` aan.

`private function checkIfNeededBySavedChild(array $record, bool $hasBeenPersisted): void` checkt ten eerste of `$hasBeenPersisted` true of false is, en of de array key `record['id']` bestaat op array `$this->waitingForProcessing`.

Als deze niet bestaat, of `$hasBeenPersisted` uitkomt op false, dan `return`en we.

Mocht dit niet het geval zijn, halen we de `$parent` op met de `taxonRepository` door te zoeken naar een `Taxon` waarvan de `code` gelijk staat aan `$this->getCode($record['path'])`.

Hierna loopen we door `$this->waitingForProcessing[$record['id']]`, en roepen we `$this->addTaxon()` aan met dit overwerkte kind en de parent als parameters.

uiteindelijk roepen we `unset` aan op `$this->waitingForProcessing[$record['id]`.

## ImportUniversalProducts

`services.yaml`

```yaml
App\Command\ImportUniversalProducts:
    arguments:
        $csvPath: '%env(BOGIJN_UNIVERSAL_PRODUCTS_EXPORT_LOCATION)%'
```

Globale variabelen:

```php
private ChannelInterface $mainChannel;
private string $bogijnImagesURL = "https://bogijn.nl/assets/images/";
```

Constructor injecties:

```php
private ProductRepository                 $productRepository,
private EntityManagerInterface            $em,
private ChannelRepository                 $channelRepository,
private ProductFactoryInterface           $productFactory,
private ProductVariantRepositoryInterface $productVariantRepository,
private ProductVariantFactoryInterface    $productVariantFactory,
private TaxCategoryRepository             $taxCategoryRepository,
private ShippingCategoryRepository        $shippingCategoryRepository,
private FactoryInterface                  $channelPricingFactory,
private FactoryInterface                  $productImageFactory,
private ImageUploaderInterface            $imageUploader,
private string                            $csvPath,
ProviderRepository                        $providerRepository,
```

Record:

```json
{
    "id": "1",
    "title": "Absaar acculader 6A 12V",
    "slug": "absaar-acculader-6a-12v-0635608",
    "sku": "U001/0635606",
    "brand_id": "U001",
    "brand_name": "Absaar",
    "description": "Semi-professioneel. Netspanning 230V. 1 Laadstand. Laadtype: normaal. Afmetingen: 280x210x120mm. Gewicht: 2,6kg.<br>Laad de accu van uw auto op met deze semi-professionele Absaar acculader. De acculader is geschikt voor 12V, heeft een maximum capaciteit van 6A en is voorzien van twee laadstanden, snel en normaal.<br><br><br><br>Het artikel is Rood gekleurd en vervaardigt uit Gemengd materiaal. Wordt geleverd in een Doos.",
    "meta_title": "Koop voordelig Absaar acculader 6A 12V",
    "meta_description": "Semi-professioneel. Netspanning 230V. 1 Laadstand. Laadtype: normaal. Afmetingen: 280x210x120mm. Gew",
    "meta_tags": "",
    "meta_keywords": "Absaar,acculader,6A,12V,0635606,Absaar",
    "price_gross": "65.34",
    "price_net": "61.50",
    "price_promotion": "0",
    "deposit": "0",
    "manage_stock": "0",
    "stock": "0",
    "stock_2": "0",
    "unit": "1",
    "barcode": "4002031060646",
    "ean": "4002031060646",
    "shipping_class": "shipping_class_1",
    "images": "0635606.jpg",
    "files": "",
    "video": "",
    "date": "2022-09-12 d15:00:59",
    "active": "no",
    "article_no": "0635606",
    "data": "{\"brandid\":\"U001\",\"artnr\":\"0635606\",\"price\":\"5083\",\"stock1\":\"0\",\"delivery1\":\"5\",\"stock2\":\"1\",\"delivery2\":\"5\",\"stock3\":\"0\",\"delivery3\":\"0\",\"deposit\":\"000\"}",
    "language": "dutch"
}
```

In dit commando hebben we in de constructor nog wat extra functionaliteit aanwezig, namelijk; 

We halen met de `providerRepository` een `enabled` provider op, mocht hier niks van terug komen, gooien we een `RuntimeException`

Hierna zetten we `$this->mainChannel` naar `$this->channelRepository->findOneByCode($provider->getChannelCode())`. Mocht `$this->mainChannel` geen instantie van `Channel` zijn, gooien we ook een `RuntimeException`.

Binnnen de `$records` loop, roepen we `$this->configureProduct($record)` aan.

`private function configureProduct(array $record): void`:

Het eerste wat deze functie doet is de `$record`s `sku` en `article_no` waardes in place veranderen, zodat deze conform is.

Hierna zoeken we naar een product in de `productRepository`, waarvan de `code` gelijk staat aan `$record['sku']`, mocht deze niet bestaan gebruiken we de `productFactory->createNew()` methode om een nieuw product aan te maken.

Dan checken we of dit `$product` ook afbeeldingen heeft geconfigureerd, mocht dit niet het geval zijn, executeren we de volgende lijn code: `$product = $this->configureImage($product, $record['images']);`.

Dan configureren we een aantal waardes op dit product

- `$product->setName($record['title']);`

- `$product->setSlug($record['slug']);`

- `$product->setCode($record['sku']);`

- `$product->setDescription($record['description']);`

- `$product->setMetaDescription($record['meta_description']);`

- `$product->setMetaKeywords($record['meta_keywords']);`

- `$product->addChannel($this->mainChannel);`

- `$product->setEnabled($record['active'] === "yes");`

Hierna roepen we `$this->setProductTranslation($product, $record);` en `$this->productRepository->add($product);` aan. Het toevoegen aan de `productRepository` zorgt er onder andere voor dat we niet handmatig de EntityManagers `persist` functie aan hoeven te roepen.

Als laatste in deze functie roepen we `$this->getProductVariant($record, $product);` aan.

Dus als eerste `private function configureImage(Product $product, string $image): Product`.

Deze maakt variabele `$img` aan, en vult deze met de uitkomst van de functie `private function getImage(string $image): bool|string|null`, met als parameter de meegegeven `$image` string. 

`getImage` initialiseert een return variabele met als waarde `null`, waarna de functie de ingebouwde php-functie `ini_set` aanroept met als key om te setten `user_agent` en de waarde `'Mozilla/4.0 (compatible; MSIE 6.0)'` . Dit doen we omdat we een afbeelding willen ophalen van het internet, en dit bepaalde problemen kan voorkomen. 

Daarna checken we of de meegegeven `$image` string een volledige URL is of niet, door `filter_var($image, FILTER_VALIDATE_URL)` aan te roepen. Mocht dit het geval zijn vullen we de return variabele met de uitkomst van `@file_get_contents($image)`.

Mocht dit niet een volledige URL zijn, proberen we tevens `@file_get_contents` aan te roepen, enkel dit keer met `$this->bogijnImagesURL` voor `$image`.

Hierna geven we terug wat er in onze return variabele staat.

Terug in `configureImage` hebben we dus een mogelijke afbeelding opgehaald in `$img`, checken we of deze is geset en niet `false` is. In welk geval we gewoon het meegegeven `$product` teruggeven zoals die is.

Mocht er dus iets corrects zijn teruggekomen, maken we een tijdelijk bestandsnaam aan door middel van de volgende logica: `tempnam(sys_get_temp_dir(), $product->getCode());`. Dit genereert een string die er uit ziet als: `/tmp/U001_0635606xMxVoh`.

Waarna we de inhoud van `$img` neerzetten op deze plek door gebruik te maken van `file_put_contents`. 

Nu maken we een nieuw `ProductImage` model aan door de bijbehorende factory hiervoor, en een nieuwe `Symfony\Component\HttpFoundation\File\UploadedFile`, op de plek van de tijdelijke bestandsnaam. Voordat we hier verder iets mee gaan doen controleren we eerst of de bestandsextensie een afbeeldingbestandsextensie is, als dit niet het geval is `return`en we gewoon het meegegeven `$product` zonder er iets mee te doen, om te voorkomen dat er andere soorten bestanden worden gebruikt als afbeelding voor dit product.

Als het een afbeelding bestandsexensie heeft, zetten we het bestand op de `ProductImage`, en gebruiken we `$this->imageUploader->upload` met als parameter onze `ProductImage`, om het bestand correct te uploaden en configureren. Nu kunnen we `$product->addImage($productImage)` aanroepen en het `$product` teruggeven.

De functie `private function setProductTranslation(Product $product, array $record): void` die we hebben gebruikt in `configureProduct`, loopt over alle `Locale`s  die aan `$this->mainChannel` vastzitten. Binnen deze loop proberen we een bestaande `ProductTranslation` van het huidige `Product` op te halen, door alle translations die op het huidige `Product` staan geconfigureerd te filteren op waar de locale van de translation gelijk staat aan locale van de huidige iteratie van de loop over alle locales van `$this->mainChannel`. Mocht er geen gevonden zijn maken we hier een nieuwe `ProductTranslation` aan.

Hier zetten we dan de locale van naar de locale van de loop, en de name, slug, description, meta description, meta keywords, en short description naar de bijbehorende data uit de `$record`. Hierna roepen we `$product->addTranslation()` aan met als parameter de translation die we hebben opgehaald / aangemaakt.

Nu moeten we enkel nog een product variant aanmaken / configureren voor dit product. dat doen we dus in `private function getProductVariant(array $record, Product $product): void` in `configureProduct`.

We zoeken wederom eerst naar een bestaande `ProductVariant`, waarbij de code gelijk staat aan de `sku` waarde van de `$record` . Mocht deze niet bestaan maken we een nieuwe aan.

Hiervan zetten we de naam naar `$record['title']`, de code naar `$record['sku']`, de positie naar 1, het product naar het meegegeven product, onHand naar `(int) $record['stock']`, en tracked op false.

Nu voegen we dit productvariant toe aan de `productVariantRepository`, zodat de `EntityManager` bewust is van deze variant.

Vervolgens zoeken we naar de standaard `TaxCategory`, en maken we deze vast aan de variant, mits deze bestaat. Hetzelfde geld voor de `ShippingCategory`, enkel zetten we ook op de variant dat shipping verplicht is.

Nu roepen we `private function getChannelPricing(ProductVariantInterface $productVariant, array $record): ChannelPricingInterface` aan met parameters `$productVariant` en `$record`, de uitkomst hiervan slaan we op in `$channelPricing`.

`getChannelPricing` controleert of er al een channelpricing voor de `$this->mainChannel` staat op onze `ProductVariant`, mocht dit niet het geval zijn word er een nieuwe aangemaakt.

Hierna zetten we de prijs op de `ChannelPricing` naar `(int) $record['price_net']`, de `channelCode` naar de code die geconfigureerd staat op `$this->mainChannel`, en de `ProductVariant` naar onze `ProductVariant`. Hierna `return`en we onze `ChannelPricing`.

Nu zetten we deze `ChannelPricing`, mits hij er niet al op staat, op onze `ProductVariant`.

Nu roepen we alleen nog maar `setProductVariantTranslation($productVariant, $record)`, welke functioneel gelijk is aan `setProductTranslation`, slechts met andere parameters om te zetten, en `$this->productRepository->add($productVariant)` aan, waarna dit commando zichzelf herhaalt voor de volgende `$record`.

## ImportAvailabilityNavision

`services.yaml`

```yaml
App\Command\ImportAvailabilityNavision:
    arguments:
        $csvPath: '%env(BOGIJN_NAVISION_EXPORT_LOCATION)%'
        $variantResolver: '@sylius.product_variant_resolver.default'
```

Globale variabelen:

```php
private StockRepository $stockRepository;
private ChannelInterface $mainChannel;
private DateTime $updatedAt;
private array $columns = [
    'brandid',
    'artnr',
    'price',
    'stock1',
    'delivery1',
    'stock2',
    'delivery2',
    'stock3',
    'delivery3',
    'deposit'
];
```

Constructor injecties:

```php
private EntityManagerInterface          $em,
private SluggerInterface                $slugger,
private ProductRepository               $productRepository,
private ProductVariantResolverInterface $variantResolver,
private ChannelRepository               $channelRepository,
private TaxCategoryRepository           $taxCategoryRepository,
private ShippingCategoryRepository      $shippingCategoryRepository,
private FactoryInterface                $channelPricingFactory,
private ProductVariantRepository        $productVariantRepository,
private string                          $csvPath,
ProviderRepository                      $providerRepository,
```

Record:

```json
[
  "0030",
  "0 001 106 017",
  "18392",
  "0",
  "6",
  "0",
  "6",
  "0",
  "0",
  "000"
]
```

Correct geformatteerde record: 

```json
{
    "article_no": "0 001 106 017",
    "sku": "30_0001106017",
    "brand_id": "30",
    "price_net": 183.92,
    "deposit": 0,
    "stock": 0,
    "data": "{\"brandid\":\"0030\",\"artnr\":\"0 001 106 017\",\"price\":\"18392\",\"stock1\":\"0\",\"delivery1\":\"6\",\"stock2\":\"0\",\"delivery2\":\"6\",\"stock3\":\"0\",\"delivery3\":\"0\",\"deposit\":\"000\"}",
    "date": "2023-03-29 13:54:54",
    "active": "yes"
}
```

Verder zijn er nog een aantal checks in de constructor voordat de `execute` functie daadwerkelijk kan worden uitgevoerd;

De `ProviderRepository` word gebruikt om een `enabled` provider op te halen, als deze niet bestaat word er een `RuntimeException` gegooid.

Als de `Store\SyliusTecDocPlugin\Entity\Stock` class niet bestaat, word er een `RuntimeException` gegooid.

Door gebruik te maken van de `ChannelRepository` en de `$provider`, word `$this->mainChannel` gevuld. Als deze geen instantie is van `App\Entity\Channel\Channel`, word er een `RuntimeException` gegooid.

`$this->stockRepository` word gevuld door `$this->em->getRepository(Stock::class)`.

`$this->updatedAt` word geset naar `new DateTime()`.

De csv die we hier importeren heeft geen headers, dus de record bevat enkel data, en geen koppelingswaardes. Ook moeten we een andere delimiter instellen op onze `$csv` variabel, In deze `.csv` word namelijk in plaats van een ',' een ';' gebruikt.

Binnen de `$records` loop is het eerste wat we doen de `$record` anders formatteren, we roepen hiervoor `private function formatRecord(array $record): array` aan.  met onze `$record` als parameter.
Hierbinnen vervangen we de huidige `$record` met de uitkomst van `array_combine($this->columns, $record)` aan. Wat er voor zorgt dat onze record er als volgt uit ziet:

```json
"brandid" => "0030"
"artnr" => "0 001 106 017"
"price" => "18392"
"stock1" => "0"
"delivery1" => "6"
"stock2" => "0"
"delivery2" => "6"
"stock3" => "0"
"delivery3" => "0"
"deposit" => "000"
```

Nu hebben we een koppeling tussen de data en het type waar het onder valt.

Vervolgens maken we variabel `$articleNo` aan. We bewerken het bestaande artikelnummer voor deze `$record`, hiervoor maken we gebruik van php-functie `preg_replace`. We geven als patroon parameter de string `'/_' . $record['brandid'] . '$/'`. Dit vervangen we met een lege string, op `$record['artnr']`. Dit doen we omdat sommige artikelnummers een brand id hebben.

Nu hebben we dus enkel een artikelnummer.

de `sku` genereren we door alle spaties te vervangen met lege strings in `$record['brandid']` , en hierna er '_' en daarna het artikelnummer in hoofdletters achter te plakken.

het `brand_id` maken we van het bestaande `brandid`, enkel met alle whitespace aan het begin van de string er voor weg te halen.

`price_net` is de `price` gedeeld door 100.

`deposit` is `deposit` gedeeld door 100.

voor `stock` kijken we naar `stock1`, is deze groter dan 0? dan word `stock` `stock1`. zo niet, kijken we naar of `stock2 + stock3` groter is dan 0, en of `delivery2` groter is dan 0, is dit het geval? dan word `stock` 1, zo niet, word `stock` 0.

`data` is de hele `record` ge-encode in json formaat.

`date` is `$this->updatedAt` geformatteerd als `Jaar-maand-dag Uur:minuten:seconden`.

`active` is altijd `yes`.

Nu de `$record` geformatteerd is, geven we deze terug.

Terug in de loop zoeken we in de `productRepository` naar een `Product` met de code van `record['sku']`, als deze niet bestaat, dan slaan we deze iteratie van de loop verder over. Bestaat deze wel, roepen we `$this->updateTecDocStock($record, $product)` en `$this->updateUniversalProduct($record, $product)` aan.

Nadat de hele `.csv` is verwerkt, roepen we de functies 

```php
$this->clearStockIfNotUpdatedTecdoc();
$this->clearStockIfNotUpdatedUniversal();
```

aan.

Binnen `private function updateTecdocStock(array $record, Product $product): void` roepen we ten eerste `Store\SyliusTecDocPlugin\Entity\Stock::generateCodeFromBrandNumberAndArticleNumber()` aan met parameters`(string) $this->slugger->slug($record['brand_id'])->lower()` en `string) $this->slugger->slug($record['article_no'])->lower()`.

Dit geeft ons een `$code` die we kunnen gebruiken om in de `stockRepository` te zoeken naar een `Stock` met code `$code`. Deze `Stock` slaan we op in `$existingStockItem`.

Is `$existingStockItem` niet null en is de waarde `updatedAt` van `$existingStockItem` gelijk aan `$this->updatedAt` of `null`, dan `return`en we uit de functie, aangezien deze `Stock` zojuist is bewerkt, of er iets anders aan de hand is.

Hierna halen we van ons meegegeven `$product`, de geassocieerde `ProductVariant` op d.m.v. `$this->variantResolver->getVariant($product)`.

Dan zeggen we dat `$stock` gelijk staat aan de uitkomst van `$this->getStock($record, $code, $existingStockItem ?? new Stock())`.

`private function getStock(array $record, string $code, Stock $stock): Stock` gebruikt waardes van de `$record` om de `$stock` in / bij te vullen, tevens zet deze functie de `updatedAt` waarde van de `$stock` gelijk aan `$this->updatedAt`.

Nu word er gekeken naar of er een `TaxCategory` met de code `'DEFAULT'` bestaat, bestaat deze, dan word deze `TaxCategory` ingesteld voor onze `$variant`.

Hetzelfde geld voor `ShippingCategory`, tevens word er als er een `ShippingCategory` is gevonden, `$variant->setShippingRequired(true)` uitgevoerd.

Als er geen `ChannelPricing` word gevonden voor channel `$this->mainChannel`, word deze aangemaakt, en opgeslagen in `$channelPricing`.

De prijs van de `$channelPricing` word ingesteld naar `(int) ($record['price_net'] * 100)`, om te voorkomen dat er `float`s in de database komen.

Hetzelfde geld voor `$channelPricing`s `originalPrice` property.

`$channelPricing`s `channelCode` word ingesteld op `$this->mainChannel->getCode()`.

Dan is de `$channelPricing` voldoende ingesteld en word deze toegevoegd aan de `ProductVariant $variant`.

`$variant`s `onHand` waarde word geconfigureerd als `$record['stock']`, en `tracked` als `false`. Als laatste in deze functie word de `$stock` ge`persist`, de entitymanager ge`flush`ed, en ge`clear`ed.

De volgende functie die word uitgevoerd na `updateTecDocStock`, is `private function updateUniversalProduct(array $record, Product $product): void`.

Hierin word als eerste gechecked of het meegegeven product geset is, en of de `updatedAt` property hiervan gelijk staat aan `$this->updatedAt`, net als in `updateTecDocStock`.  Als het product niet is geset, of de `updatedAt` properties zijn gelijk, dan returnen we uit de functie.

Hierna halen we wederom de `ProductVariant`  op voor dit product, en tevens ook de `ChannelPricing` voor de `$this->mainChannel`.

op de `ChannelPricing` zetten we dan de prijs en originele prijs naar `$record['price_net'] * 100`. We zetten de `onHand` property naar `$record['stock']`,  de `enabled` property naar de uitkomst van `$record['active'] === "yes"`, en de `updatedAt` property naar `$this->updatedAt`.

Tot slot voegen we de `$this->mainChannel` toe aan het product en `flush`en we de `EntityManager`.

Zoals eerder benoemd roepen we de functies `$this->clearStockIfNotUpdatedTecdoc();` en `
$this->clearStockIfNotUpdatedUniversal();` aan nadat de csv verwerkt is.

Deze twee functies zijn vrij identiek, in de zin dat ze voor beide entities, de stock op 0 zetten, als de `updatedAt` waarde niet gelijk is aan `$this->updatedAt`, of als de `updatedAt` waarde `NULL` is.

## LinkUniversalProductsAndTaxonsImports

Voor dit commando is er niet een tabel die zomaar klaar is om te exporteren, het csv bestand wat hier getracht word om verwerkt te worden, zou een export moeten zijn van de SQL query: 

```sql
SELECT products.sku, products.brand_id, products.article_no, categories.path FROM fuel_relationships rel INNER JOIN shop_universal_products products ON rel.candidate_key=products.id INNER JOIN shop_categories categories ON rel.foreign_key=categories.id WHERE candidate_table="shop_universal_products" and foreign_table="shop_categories"
```

in de Bogijn2.0 database.

`services.yaml`

```yaml
App\Command\LinkUniversalProductsAndTaxonsImports:
    arguments:
        $csvPath: '%env(BOGIJN_TAXON_PRODUCT_RELATIONSHIP_EXPORT_LOCATION)%'
```

Globale variabelen:

standaard

Constructor injecties:

```php
private string                          $csvPath,
private ProductRepositoryInterface      $productRepository,
private TaxonRepositoryInterface        $taxonRepository,
private ProductTaxonRepositoryInterface $productTaxonRepository,
private FactoryInterface                $productTaxonFactory,
private EntityManagerInterface          $em,
```

Record:

```php
"sku" => "U001/0635606"
"article_no" => "0635606"
"brand_id" => "U001"
"path" => "acculaders"
```

In de `$records` loop, word `$this->handleRelationShip($record)` aangeroepen,

In `private function handleRelationShip(array $record): void`, word als eerste de  variabelen `$sku` en `$articleNo` ingesteld op dezelfde manier als die werden in `ImportAvailabilityNavision::formatRecord`,

Dan word er getracht een `Taxon` op te halen met de `code` `$record['path']`.

Vervolgens word er getracht een `Product` opgehaald te worden met de code `$sku`.

En niet te vergeten word de koppeltabel entity `ProductTaxon` opgehaald via de `ProductTaxonRepository` functie `findOneByProductCodeAndTaxonCode`, met parameters `$sku` en `$record['path']`.

Als de `Taxon` of het `Product` niet zijn geset, dan word er uit de functie gereturned, aangezien het niet de taak van deze functie is om deze aan te maken.

Hierna word er op het `Product` `$product` de `Taxon` `$taxon` ingesteld als `mainTaxon`.

Dan word er gekeken naar of de `ProductTaxon` `$productTaxon` is geset, als dit niet het geval is, dan word er een nieuwe `ProductTaxon` aangemaakt door middel van de `productTaxonFactory`, word hierop het `Product` `$product` gezet, en de `Taxon` `$taxon`, daarna word deze `ProductTaxon` gepersist door de `EntityManager`

Dan word er aan het `$product` de `ProductTaxon` `$productTaxon` toegevoegd, en de `EntityManager` ge`flush`ed.


