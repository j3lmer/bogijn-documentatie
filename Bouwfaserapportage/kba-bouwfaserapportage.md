# KBA Bouwfaserapportage

We beginnen met een nieuw formuliertype voor symfony om weer te geven; `KbaType` in `storesyliustecdocplugin/src/Form/Type`

Dit type bevat 2 componenten, kba1 (TextType), kba2 (TextType) met bijbehorende labels.

In `storesyliustecdocplugin/src/Controller/SearchController` voeg ik hier dan een use statement voor aan toe.

In de `vehicleSearchAction` functie voeg ik dan een variabel toe (`$kbaForm`) welke een `FormInterface` bevat gebaseerd op het zojuist aangemaakte `KbaType`.

Hierna roep ik `$kbaForm->handleRequest($request)` aan. Dit zorgt er voor dat wanneer de gebruiker het formulier heeft ingevoerd, het hier gaat worden afgehandeld. Dan bekijk ik of het formulier correct is.

Voordat we de normale pagina renderen, kijken we of het verzoek wat op dit moment word afgehandeld door deze (`vehicleSearchAction`) functie, het KBA formulier is, en of het valide is, d.m.v. `$kbaForm->isSubmitted() && $kbaForm->isValid()`, mocht dit het geval zijn, halen we de data er uit, slaan we deze op in variabel `$data` en sturen we deze op naar de functie `searchVehiclesByKba`, samen met de eventueel non-kba-specfiake variabel `$taxon`.

`private function searchVehiclesByKba(array $data, Taxon $taxon = null)`

is een functie welke de losse kba-formulier data-array neemt en de TecDoc client gebruikt om voertuigdata op te halen bij TecDoc.

Mocht er geen data zijn teruggekomen, geven we een foutmelding, en redirecten we naar `store_sylius_tecdoc_onderdelen` of `store_sylius_tecdoc_onderdelen_taxon`, gebaseerd op of `$taxon` en zijn slug zijn gedefinieerd.

Als er wel data is teruggekomen uit tecdoc, halen we alle auto IDs op uit de response, en gebruiken we deze om de volledige voertuigdata op te halen bij deze IDs

Als er meerdere autos terug zijn gekomen uit de response, voegen we aan de voertuigen een gegenereerde URL toe, en renderen we een pagina waar de gebruiker kan kiezen welke auto ze bedoelden.

Uiteindelijk redirecten we de gebruiker naar `store_sylius_tecdoc_onderdelen_vehicle` of `store_sylius_tecdoc_onderdelen_vehicle_taxon`, gebaseerd op of de `$taxon` gedefinieerd staat of niet, beide keren geven we het voertuig ID en het voertuig slug mee.

Terug in de `vehicleSearchAction`, als het niet het geval was dat het bezoek aan deze controller functie was om de kba zoekfunctionaliteit te initialiseren, dan renderen we het standaard template (meegegeven in parameters), en geven we het kba formulier hier aan mee.

In beide `vehicle_search` en `vehicle_search_sidebar` in beide Bogijn3 en in de tecdocplugin, checken we of de taal van het huidige request 'de_DE' is, als dit het geval is renderen we het kbaForm.

het lijkt nu klaar te zijn, maar omdat er geen submit knop is meegegeven op het formulier (wat de bedoeling is), werkt het hele formulier niet. Om dit probleem op te lossen hebben we wat custom javascript nodig.

In `themes/BootstrapTheme/assets/js/index.js` checken we of er in de body van de huidig weergegeven pagina de twee invoervelden zijn voor KBA, als deze beide present zijn, roepen we een functie aan die we importeren uit `./kba-form-handler`.

Binnen deze functie halen we het element met het id 'kba_kba1' op, en gebruiken we jquery om een keyup event listener te zetten op alle elementen met de class `kba-input`.

Op het moment dat het keyup evenement word afgevuurd, checken we de keycode van het evenement, is dit 13? (enter), dan stoppen we het standaardgedrag, en submitten we het formulier.

Als het het geval is dat de currentTarget van het evenement het eerste invoerveld is (`kba_kba1`), en de lengte van dit invoerveld 4 is, dan verplaatsen we de focus naar `kba_kba2`.