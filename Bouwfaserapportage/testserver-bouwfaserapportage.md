# Testserver Bouwfaserapportage

Na het inloggen op de testserver, welke gebruik maakt van webmin, heb ik een nieuwe virtuele server aangemaakt.
Ik ben bij de andere testservers langsgegaan en heb daar gekeken naar hoe deze ingesteld waren, de configuratie heb ik hiervan overgenomen.

Hierna heb ik een tweede virtuele server aangemaakt, als alias van de eerste. Na een korte periode wachten waren de testservers klaar om te gebruiken.

Vanaf mijn lokale computer heb ik het deployment script ingesteld en gedraait, en voila! De testbranch staat op de testserver.

Nadien heb ik in de testserver ingelogd via SSH, en ingelogd in de gebruiker van mijn nieuwe virtuele top level server.
Ik heb de .env.local ingesteld, en de csv testbestanden, welke nodig zijn voor het importeren van alle data, ge-SCPd naar de testserver.

Hierna heb ik het installatiecommando van sylius aangeroepen, welke de standaard migraties uitvoert, en een admin gebruiker en currency insteld.

Ondertussen heb ik ingelogd op de admin pagina van sylius, Als eerste heb ik hier de duitse taal ingesteld,
en hierna de hoofd taxons handmatig ingesteld;

- hoofdmenu

- auto_accessoires

- tecdoc

Vervolgens keek ik naar de kanalen, als eerst het hoofdkanaal

naam en code: default
thema: sylius bootstrap theme
taal: nederlands
menu categorie: hoofdmenu
hostnaam: bogijn3.stor-e.net

Hierna het Duitse kanaal
code: german
naam: Duits
taal: Duits
menu categorie: hoofdmenu
hostnaam: bogijn3-de.stor-e.net

Nu stel ik de TecDoc provider in met bogijns provider id en api key, deze zet ik op enabled en de channel code zet ik op default

Vervolgens importeer ik alle manufacturers, brands, generic articles en criteria.

Nu alles is geconfigureerd, draai ik het importeer commando, via nohup, gezien deze vrij lang duurt.

Nadat alles is geimporteerd, roep ik de commandos aan uit de documentatie van de bitbag elasticsearch plugin

Ik zet de volgorde van de hoofdmenu taxons naar hetzelfde als deze van wat er nu op bogijn.nl staat.
