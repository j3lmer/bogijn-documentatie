# Dynamische cataloguspagina Plan van aanpak.

Een belangrijk onderdeel van de migratie van Bogijn, is dat er uiteindelijk de mogelijkheid moet zijn om een pagina aan te maken vanuit het cms, en dan op de voorkant van de website, een lijstoverzicht te hebben van producten met een meegegeven (TecDoc) generiek artikel id, zonder dat dit ten koste gaat van algemene content te tonen op deze pagina.

Hoe ik dit voor me zie is als volgt; op de standaard pagina in het CMS waar je een pagina voor aan de voorkant van de website aanmaakt, wil ik in het 'Inhoud' veld, waar je gebruikelijk invoert wat je op de pagina wil hebben te staan, de mogelijkheid hebben om op de eerste regel van dit inhoud-invoerveld, een generiek artikel id nummer mee te geven tussen `{TECDOC}` identificators. Hierna word deze uitgelezen en op basis hiervan een TecDoc productenlijst weergegeven.