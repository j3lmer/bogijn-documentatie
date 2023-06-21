# Multi domein configuratie Plan van aanpak

Bogijn2.0 heeft twee verschillende domeinen, bogijn.nl, en bogijn.de

Bogijn.de is ingesteld op een duitse vertaling.

Bogijn3.0 moet net zo worden geconfigureerd.

Na wat onderzoek naar hoe ik dit zou kunnen bewerkstelligen, ben ik er achter gekomen dat sylius meerdere 'kanalen' ondersteunt.

In de admin pagina van sylius kan je navigeren naar /channels. Hier kan je kanalen aanmaken en bewerken

Als je een kanaal bewerkt of aanmaakt, kan je een Hostnaam instellen.

Ik heb twee kanalen aangemaakt, bogijn3.stor-e.net, en bogijn3-de.stor-e.net, ingesteld op Nederlands en duitse vertalingen respectief.

In de adminpagina van de server heb ik bogijn3-de ingesteld een alias van bogijn3 te zijn. Dit werkt zoals bedoeld.

Het enige probleem waar we nog eens mee zitten is dat we iets moeten verzinnen op het feit dat er in de code op sommige plekken speciafiek word gechecked op het kanaal, bij naam. Aangezien we nu een tweede kanaal gebruiken, is het vereist dat er hier iets op bedacht word.

Het probleem komt voor in de storesyliustecdocplugin repository, in de CartController.

Er moet een functie komen om het kanaal op te halen uit het huidige request.