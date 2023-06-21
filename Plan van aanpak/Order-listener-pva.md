# Order listener Plan van aanpak

Wanneer er een order word geplaatst op de webshop van bogijn, moet deze order gesynchroniseerd worden met Bogijns ERP systeem (Navision).

Er bestaat in de basiscode al een Order listener, welke een email stuurt naar het contact email van het kanaal waar de order op is geplaatst.

Deze neem ik als voorbeeld om de order listener in te stellen zodat orders gesynchroniseerd worden naar Navision.

Wat er moet komen:

- Een email template welke bogijn informeert wanneer het aanmaken van een order in hun ERP systeem is mislukt

- De Order listener zelf

- Een SOAP client factory welke kan praten met Navision