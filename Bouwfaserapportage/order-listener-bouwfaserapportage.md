# Order listener Bouwfaserapportage

Als eerste hernoem ik de bestaande order listener (`src/EventListenerOrderCompleteListener.php`) naar `OrderCompleteListenerSendMail.php`, en maak ik een nieuw bestand aan in dezelfde map genaamd `OrderCompleteListenerSyncErp.php`.

Hierin definieer ik functie; `onSyliusOrderPostComplete`, welke een `GenericEvent $event` neemt en `void` teruggeeft.

Ik haal de `$order` op uit het event door middel van `$event->getSubject();`, hierna check ik of dit ook wel echt een order is door `Assert::isInstanceOf($order, OrderInterface::class);` aan te roepen, welke een fout zou gooien wanneer `$order` geen instantie van de `OrderInterface` class is.

Hierna definieer ik functie `syncToErp(OrderInterface $order): void` en roep ik deze aan na het checken van de `$order`.

Binnen `syncToErp`;

- Haal ik de `Customer` op uit de order

- Haal ik het adres op door middel van dit stukje code: `$address = $customer->getDefaultAddress() ?? $order->getShippingAddress() ?? $order->getBillingAddress();`

- Haal ik de Shipping address en de Billing address los van elkaar op.

- Zet ik de `$name` variabel naar de uitkomst van het volgende stukje pseudocode:

is de bedrijfsnaamleeg? gebruik de volledige naam uit het `$address` variabel. Zo niet, gebruik het bedrijfsnaam en de volledige naam uit het `$address` variabel, gesepareerd met een komma, en pak hier de eerste 50 karakters van.

Nadien maak ik een vrij grote key-value array (`$navOrder`) aan welke data bevat uit de order om naar navision te gaan versturen. Hierbinnen hou ik rekening met de 'Bogijns choice' producten, door dezelfde data aan te passen als in de adapter, voor het word toegevoegd aan de `$navOrder`

Dan check ik de betaalmethode uit een lijst met voorgedefinieerde toegestane betaalmethodes.
estane betaalmethodes.

Nu is het tijd om deze data naar Navision te gaan versturen, het protocol wat hier gebruikt voor word is SOAP. Om alles overzichtelijk te houden maak ik in `src/Factory` het bestand `NavisionClientFactory` aan, welke in zijn construct de volgende waardes neemt:

- `string $login`

- `string $password`

- `string $wsdl`

`$login` en `$password` spreken voor zich, `$wsdl` verwijst naar een `.xml` bestand welke het formaat wat er naar navision moet worden gestuurd, alswel de url waar het heen moet worden gestuurd bevat.

Nu maken we `public function getClient(): SoapClient` aan, checken we hier in of de drie constructor variabelen niet leeg zijn, en `return`en we een `new SoapClient`, met de volgende parameters:

```php
$this->wsdl,
[
    'exceptions' => true,
    'login' => $this->login,
    'password' => $this->password,
    'cache_wsdl' => WSDL_CACHE_NONE,
]
```

Nu we aan een `SoapClient` kunnen komen met een connectie naar het ERP systeem, kunnen we deze gaan gebruiken in het synchronisatie commando. Na het checken van de betaalmethode, schrijven we de volgende syntax: `$client = $this->navisionClientFactory->getClient()`, en natuurlijk voegen we aan de constructor toe: `private NavisionClientFactory $navisionClientFactory`.

Deze `SoapClient $client` bevat functionaliteit welke gedefinieerd staat in de eerder benoemde `.xml`, waardoor we nu `$client->CreateOrder([wSOrders => ['Order' => $navOrder])` kunnen aanroepen, waarbij `$navOrder` onze key-value array is met data over de order. Mocht dit falen, word er een email gestuurd naar het contact email van het kanaal waarop de order is aangemaakt

Hiervoor configureer ik nog een nieuw email template in `config/packages/sylius_mailer.yaml`:

```yaml
sylius_mailer:
    sender:
        name: Stor-e
        name: Bogijn
        address: no-reply@stor-e.net
        address: info@bogijn.nl
    emails:
        navision_order_failed:
            subject: Aanmaken Navision order is mislukt {{ order.number }}
            template: "Email/navision_order_failed.html.twig"
```

`templates/Email/navision_order_failed.html.twig`:

```twig
{% block subject %}
	Aanmaken Navision order is mislukt {{ order.number }}
{% endblock %}
{% block body %}
	Het aanmaken van de Navision order {{ order.number }} is mislukt
	Foutmelding:
	{{ exception }}
{% endblock %}
```

Om nu het voor elkaar te krijgen dat `onSyliusOrderPostComplete` daadwerkelijk word aangeroepen wanneer er een order compleet is afgerond, voegen we het volgende toe aan `config/services.yaml` 

```yaml
    App\EventListener\OrderCompleteListenerSyncErp:
        tags:
            - { name: kernel.event_listener, event: sylius.order.post_complete }
```

Verder zorgen we er hier ook voor dat de constructor argumenten worden ingevoerd voor de `NavisionClientFactory`

```yaml
    App\Factory\NavisionClientFactory:
        arguments:
            $login: '%env(NAVISION_LOGIN)%'
            $password: '%env(NAVISION_PASSWORD)%'
            $wsdl: '%env(NAVISION_WSDL)%'
```

Dit zorgt er voor dat de .env bestanden worden doorzocht en de NAVISION_* waarden hierin worden gebruikt voor de constructor argumenten.