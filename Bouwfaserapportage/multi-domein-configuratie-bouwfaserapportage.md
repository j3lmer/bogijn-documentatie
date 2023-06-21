# Multi domein configuratie Bouwfaserapportage

Multi domein configuratie brengt het probleem met zich mee dat er in de CartController word gezocht naar een vaste waarde van het kanaal. enkel zijn er nu meerdere kanalen. Om dit op te lossen schrijf ik deze functie.

```php
private function getCurrentChannelFromRequest(Request $request): ChannelInterface
```
De functie zelf is niet per se heel spannend;

```php
return $this->channelRepository->findOneByCode($request->cookies->get('_channel_code'));
```

Hierna pas ik overal in dit bestand aan waar de default taxon word opgehaald naar een call naar deze functie, en verander ik de naamgeving van de variabelen waar het kanaal in word opgeslagen van `mainChannel` naar `currentChannel`.

Dit blijkt het probleem te verhelpen.
