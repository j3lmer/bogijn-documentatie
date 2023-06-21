# Assortiment pagina Plan van aanpak

De bedoeling van dit onderdeel is dat de pagina https://bogijn.nl/assortiment word geimplementeerd in Bogijn3.0

op het eerste gezicht is de pagina een Categorieën-lijst. Door te kijken in de codebase van Bogijn2.0 ondervind ik dat de data welke naar de gerenderde pagina word meegegeven bestaat uit iets genaamd `assemblyGroups`, ofwel Montage groepen. In de basisfunctionaliteit van Stor-e (Bogijn3.0) bestaan deze niet. Deze data moet worden verkregen

Er moet een route (/assortiment) komen om de pagina weer te geven

De `assemblyGroups` gaan moeten worden gegroepeerd op eerste letter

Na het klikken op een van de `assemblyGroups`, word er nog een letter-gegroepeerde pagina weergegeven met data. Kijkende in de oude codebase blijkt het dat de data welke hier word getoont bestaat uit `genericArticles` ofwel generieke artikelen.

Doorklikkende word ik gepresenteerd met de naam van het (generieke) artikel welke ik heb geselecteerd. Naar verdere inspectie blijkt het dat wat hier hoort te staan zijn de merken welke de gekozen selectie aanbieden.

Na een ander montagegroep te hebben geselecteerd blijkt dit inderdaad het geval te zijn.

er zijn dus minimaal drie routes nodig, alfabetisch gegroepeerde lijsten aan montage groepen en generieke artikelen, en een route om merken weer te geven welke de selectie bieden.

Verder logica om dit weer te geven en templates om de data aan mee te geven zijn ook vrij vereist.