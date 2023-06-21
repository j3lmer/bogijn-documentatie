# Zoekfunctionaliteit Plan van aanpak

Bogijn2.0 heeft een zoekbalk welke door beide lokale producten in de database, en TecDoc API producten zoekt. Het is de bedoeling dat in Bogijn3.0 deze functionaliteit niet ontbreekt.

Elasticsearch versie 7.10.0 staat al geinstalleerd en gelocked in mijn development omgeving, deze hoef ik dus niet te installeren. Er bestaat voor Sylius een plugin welke Elasticsearch implementatie versimpelt. Vanzelfsprekend gebruiken we deze.

Het plan verder is om de documentatie te volgen van deze plugin, en deze samen te knopen met een custom zoekfunctie voor tecdoc producten, en dan een route/template maken om de resultaten weer te geven.
