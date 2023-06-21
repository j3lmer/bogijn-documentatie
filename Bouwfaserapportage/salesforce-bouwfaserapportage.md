# Salesforce Bouwfaserapportage


Om dit commando te implementeren begin ik met het opzetten van omgevingsvariabelen

SALESFORCE_AUTH_URL
SALESFORCE_INSTANCE_URL
SALESFORCE_USERNAME
SALESFORCE_PASSWORD
SALESFORCE_CLIENT_ID
SALESFORCE_CLIENT_SECRET

Vanzelfsprekend documenteer ik niet de bijbehorende data

Hierna maak ik in src/Command het bestand SalesforceSyncCommand.php aan en definieer ik hier de service voor in `Bogijn3.0/config/services.yaml`.;

```
   App\Command\SalesforceSyncCommand:
        arguments:
            $salesforceUsername: '%env(SALESFORCE_USERNAME)%'
            $salesforcePassword: '%env(SALESFORCE_PASSWORD)%'
            $salesforceClientId: '%env(SALESFORCE_CLIENT_ID)%'
            $salesforceClientSecret: '%env(SALESFORCE_CLIENT_SECRET)%'
            $salesforceInstanceUrl: '%env(SALESFORCE_INSTANCE_URL)%'
            $salesforceAuthUrl: '%env(SALESFORCE_AUTH_URL)%'

```

Dit maakt het mogelijk om gebruik te maken van de aangegeven omgevingsvariabelen in dit commando.

De constructor van dit commando neemt dus de meegegeven omgevingsvariabelen als parameters, en de entitymanager.
Hierna maken we een nieuwe Guzzle Http client aan met als `base_uri` de `salesforceInstanceUrl`.

Om straks data naar salesforce te sturen hebben we eerst een access token nodig.
Hiervoor roepen we in de constructor na het initialiseren van de HTTPClient, `$this-_accessToken = $this->getAccesstoken` aan.

`private function getAccessToken(): string`
gebruikt de Client om een POST request te doen naar de `salesforceAuthUrl`. We geven als query parameters het volgende mee:
```
'username' => $this->salesforceUsername,
'password' => $this->salesforcePassword,
'grant_type' => 'password',
'client_id' => $this->salesforceClientId,
'client_secret' => $this->salesforceClientSecret,
```

De response die we van dit Http request krijgen, slaan we de body contents van op.
hierna `json_decode`-en we de response en `return`en we de `access_token` property.

Tevens willen we wat opties meegeven voor het commando, debug, en cron.
Wanneer de optie `debug` word meegegeven, is het de bedoeling dat het commando meer data en waardes print naar de aanroepende CLI.
Wanneer de optie `cron` word meegegeven, is het de bedoeling dat niet ieder account word gesynchroniseerd, enkel de accounts welke niet zijn bewerkt in het afgelopen uur, om te voorkomen dat er heel erg veel Http requests worden verstuurd naar Salesforce.

De execute functie, welke automatisch word aangeroepen wanneer het commando word uitgevoerd, haalt alle `ShopUser`s op uit de database, en mits `cron` gelijk staat aan `true`, voegt hij een `where` clause toe, welke checkt of de `dateLastUpdated` property groter of gelijk aan de tijd van een uur geleden is, of of de `dateLastUpdated` property gelijk staat aan NULL.

Hierna loopen we in batches volgens de Doctrine documentatie over al deze gebruikers, checken we of hun `salesForceId` property leeg is of niet, mocht deze leeg zijn roepen we `$this->createSalesforceUser($account)` aan, zo niet, roepen we `$this->updateSalesforceUser($account)` aan, waarbij `$account` de gebruiker van de huidige loop iteratie is.

In beide `createSalesforceUser` en `updateSalesForceUser` maken we gebruik van 2 'helper' functies; `getData`, en `sendRequest`.

`getData` neemt een `ShopUser`, en geeft een `array` terug.
Deze `ShopUser` word gebruikt om een data array te configureren welke overeenkomt met de bijgeleverde prive documentatie.

`sendRequest` neemt `array $data`, `string $method`, en `string extUrl`, en geeft een nullable `array` terug.

Hierin definieren we `$url` naar de uitkomst van `"{$this-salesforceInstanceUrl}{$urlExt}"`, dit zal het adres zijn naar waar deze functie een request heen zal sturen.
`$method`, word verwacht een typische HTTP request method te zijn (POST, PATCH).

Als `headers` geven we het volgende mee om te authenticeren; `'Authorization' => "Bearer {$this->_accessToken}"`.
als `json` geven we de `$data` variabele mee.

We zetten dit HTTP verzoek in een Try-catch blok, wanneer het verzoek faalt, word de code in het catch blok uitgevoerd, deze logt een bericht naar de CLI en returned uit de functie.

Als alles goed is gegaan returnen we de response body contents, nadat deze associatief zijn ge-json-decode.

Terug binnen `createSalesforceUser`, halen we de `$data` op via de `$this-getData($account)` functie,
en zetten we `$responseData` naar `$this-sendRequest($data, 'POST', 'Account')`.
Hierna checken we of de 'success' waarde bestaat op de `$responseData` variabele, en mocht dit zo zijn, roepen we op `$account`, de functie `setSalesForceId($responseData['id'])` aan, zodat deze voor toekomstige referentie kan worden gebruikt.

Hierna word er gechecked of de `_debug` variabele gelijk staat aan `true`, als dit zo is word de `$data` gelogged.

`updateSalesforceUser` volgt een soortgelijk schema, het verschil is dat er als HTTP methode niet 'POST' maar 'PATCH' word meegegeven, en de salesforce Id niet word geupdate op het `$account`.

Na elke x iteraties word de entitymanager geflushed en gecleared, zodat deze niet vol raakt.

Ik maak een aantal properties aan op de ShopUser welke we gebruiken in het command, zoals `DateTime $dateLastUpdated`, en `string $salesForceId`, met hun respectieve getters en setters.

Verder genereer ik migraties voor de `dateLastUpdated` property, en de `salesForceId` property.