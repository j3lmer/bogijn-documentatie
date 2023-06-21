# Migratie Bogijn 2.0 -> Bogijn3.0

## Codeigniter -> Sylius

Op het heden bestaat er een website voor [Bogijn](https://www.bogijn.nl), welke gemaakt is met Codeigniter 3 + Fuel cms. Deze website gaat worden gemigreerd naar Symfony + Sylius. Het is aan mij om deze migratie succesvol te voltooien.

Er bestaat een basis repository voor mij om van af te werken. Deze bevat basisfunctionaliteit die gedeeld word over meerdere webshops.

Voor beide de content in de codeigniter omgeving en de sylius omgeving worden de paginas aangemaakt door de gebruiker in het cms.

Er bestaat een basis repository voor mij om van af te werken, deze bevat basisfunctionaliteit en word al gebruikt door meerdere webshops.

### Multi domein functionaliteit / Functionaliteit voor meerdere kanalen

Bogijn heeft 2 websites geconfigureerd. [Bogijn.nl](https://bogijn.nl/) en [Bogijn.de](https://bogijn.de/). Het moet mogelijk zijn om vanuit dezelfde repo dit op te zetten, de taal moet automatisch geconfigureerd worden op basis van welke website word bezocht.

### Dynamische cataloguspagina (Widget/Placeholder)

In het CMS moet er een speciale soort pagina kunnen worden aangemaakt voor tecdoc taxons, waar een gebruiker zijn eigen text kan neerzetten, maar ook een widget/placeholder waarin een Generiek TecDoc Artikel Id in kan worden geplaatst

Als je dan als een gebruiker deze pagina zou bekijken word je gepresenteerd met TecDoc producten onder dat Generiek TecDoc Artikel Id vallen. Dit gebeurt nu ook op een soortgelijke manier in bogijn 2.0. zie [Accus](https://bogijn.nl/accus), en [Ruitenwissers](https://bogijn.nl/ruitenwissers).

### Zoeken op KBA nummer

Enkel in de duitse variant van de website moet er niet op kenteken worden gezocht, maar op KBA nummer.

### Bogijns choice

In huidig bogijn worden Ashuki producten omgetoverd naar Bogijns choice producten, deze functionaliteit moet ook vertaald worden naar Bogijn 3.0.

### Commando / listener voor nieuwe bestellingen

 Bogijn maakt gebruik van een ERP systeem. Wanneer er een nieuwe bestelling is moet er code worden uitgevoerd om deze naar dit ERP systeem toe te sturen.

### Zoekfunctionaliteit

In huidig bogijn is er speciale zoekfunctionaliteit, deze maakt het mogelijk om door beide TecDoc producten heen te zoeken, maar ook door de lokale database verbonden aan bogijn. Deze functionaliteit moet worden geimplementeerd in Bogijn3.0

### Safebuy artikelen / Accessoires op winkelwagenpagina

Een safebuy artikel is een speciaal soort artikel waarbij het retourtermijn niet 14 dagen maar 100 dagen heeft, bevat ook kosteloos retourlabel. Deze moeten kunnen worden toegevoegd aan de winkelwagen. Zie [Winkelwagen](https://bogijn.nl/cart).

Onderaan de winkelwagenpagina staan een aantal assecoires. Deze moeten ook worden weergegeven op de bogijn 3.0 winkelwagenpagina. Zie [Winkelwagen](https://bogijn.nl/cart).

### Layout en styling

De huidige Stor-e basis repo ziet er significant anders uit dan bogijn 2.0. Dit moet worden rechtgetrokken.

### Import commandos/scripts voor de huidige database / Prijs en voorraad

De database van bogijn2.0 bevat een hoop data welke mee moet worden genomen naar de nieuwe versie. Hier importeer commandos/scripts voor maken. Tevens word er ook een bestand naar de server waarop bogijn draait toe ge-FTPd, deze moet ook kunnen worden uitgelezen doormiddels van een commando en op basis hiervan moeten de prijs en stock worden ingesteld.

### Auto-accessoires

In de menu balk van bogijn bevind zich onder andere de knop "auto accessoires". Wanneer hier op is geklikt word de gebruiker doorverwezen naar een overzicht met categorieën. Deze functionaliteit zo 1-op-1 mogelijk recreëren

### Salesforce

Recentelijk is er een salesforce integratie commando aangemaakt in bogijn2.0, deze stuurt gebruikersdata door naar Salesforce. Dit commando moet ook beschikbaar zijn in Bogijn3.0

### Assortiment

Vergelijkbaar met de sitemap welke benodigd is voor bogijn3, is er ook een assortiment pagina, deze geeft alle ingeschakelde montage groepen weer, functionaliteit vereist hetzelfde te werken als bogijn.nl/assortiment.


### Order listener

Wanneer er een product word besteld via de bogijn2.0 webshop, word de order die hier aan vast zit gekoppeld gesynchroniseerd met het ERP systeem van bogijn, dit is Microsoft Navision, en dit gebeurt via een SOAP client.

### Testserver omgeving opzetten

Voor testgemak en controleerwerk is het praktisch om een testserver te hebben waarop aanpassingen kunnen worden gedeployed. Deze omgeving gaat moeten worden opgezet en geconfigureerd.

### Route prioriteit veranderen
Urls voor paginas in bogijn zien er uit als /bougie. Urls in Sylius zien er uit als /pages/bougie. Een manier vinden om Sylius Urls er uit te gaan laten zien als die van Bogijn.