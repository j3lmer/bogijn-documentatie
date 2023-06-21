# Import commandos Plan van aanpak

Er moeten een aantal commandos komen om data uit de bestaande database (Bogijn2.0) naar de huidige database (Bogijn3.0) over te zetten, dit betreft;

- Accounts

- Categorieën (Taxons)

- Universele producten / Auto accessoires

- Prijs en voorraad

Deze commandos worden globaal bekeken vrij gelijk in hun werking, ze volgen een soortgelijk systeem. Ze definiëren allemaal de volgende variabelen:

- `private bool $isLastRun = false;`

- `private int $offset = 0;`

- `private const CHUNK_SIZE = 100;`

Ze hebben allemaal een `private function execute(InputInterface $input, OutputInterface $output): int` functie

Deze functie word door Symfony functionaliteit automatisch aangeroepen wanneer het commando word uitgevoerd.

Er worden een aantal benodigdheden geinjecteerd in de `__construct` functie, bij de meesten zal dit; `private EntityManagerInterface $entityManager`, en `private string $csvPath` worden. Per importeer commando zijn hier natuurlijk verschillen naar behoefte van het commando.

Verder lezen ze allemaal een CSV bestand (een simpel bestand welke waarde gesepareerd door kommas en witregels). Dit zijn resultaten van sql queries uit de bestaande (Bogijn2.0) database.

Ook moet er een commando komen universele producten aan hun bijbehorende categorieën te koppelen, en een commando om al deze commandos achter elkaar aan uit te voeren.

Alle commandos zullen gebruik maken van het derde partij pakket; `league/csv:^9.0`. Dit doen we omdat dit makkelijker en efficienter is dan een eigen CSV verwerker te schrijven.

en alle `execute` functies zullen het volgende patroon volgen:

```php
// Blijf itereren totdat dit variabel op true word gezet.
while (!$this->isLastRun) {

    // Bereid voor om een aantal rijen uit de CSV op te halen om te verwerken.
    $statement = Statement::create()
        ->offset($this->offset)
        ->limit(self::CHUNK_SIZE);

    // Haal deze rijen op.
    $records = $statement->process($csv);

    //Itereer over deze rijen.
    foreach ($records as $record) {
        //Doe iets met de data van een individuele rij.
    }

    //Sla alles op naar de database zo nodig.
    $this->entityManager->flush();

    //verhoog de offset met de hoeveelheid rijen we zojuist hebben.verwerkt
    $this->offset += self::CHUNK_SIZE;

    //Zet het 'isLastRun' variabel op true wanneer er minder rijen zijn dan verwacht (het einde is bereikt).
    $this->isLastRun = count($records) < self::CHUNK_SIZE;

    //Log naar de output waar we zijn met het verwerken van de CSV.
    $io->comment("Chunk verwerkt. Offset: {$this->offset}");
}
```