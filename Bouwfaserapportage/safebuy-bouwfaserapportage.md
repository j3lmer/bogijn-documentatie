# Safebuy Bouwfaserapportage

Kijkende in de element inspecteur van de browser, zijn de Sylius evenementen zichtbaar. Deze kunnen we gebruiken om een template te injecteren. Op de plek waar in bogijn2 het Safebuy stukje staat, is het evenement `sylius.shop.cart.suggestions`.

Om hier een template in te injecteren navigeren we naar `Bogijn3/config/packages/sylius_ui.yaml`. En voegen we het volgende toe:

```yaml
sylius_ui:
    events:
    # ...
        sylius.shop.cart.suggestions:
            blocks:
                SafeBuy:
                    template: 'Cart/safebuy.html.twig'
                    priority: 50
```

Het template bestand wat we hier refereren bestaat nog niet, dus die maken we aan op `Bogijn3/templates/Cart/safebuy.html.twig`.

Gezien we op onze template injecteren op de manier hoe we het doen, hebben we toegang tot de 'cart' variabele in twig, toch voeg ik hier een check aan toe om te kijken of deze wel is gedefinieerd, om onvoorziene errors te voorkomen. Als we dit hebben, zetten we lokale variabelen `safeBuyProductCode` op de string `0000_SAFEBUY`, en `hasSafeBuy` op `false`.

Hierna loopen we door de cart items, en checken we als `hasSafeBuy` op `false` staat, of de productcode van dit item gelijk is aan de `safeBuyProductCode`. Als dat zo is, dan zetten we `hasSafeBuy` op true.

Vervolgens renderen we, mits `hasSafebuy` op false staat en de cart op zn minst 1 item bevat, een rij met ruimte voor een afbeelding van het safebuy product, text gerelateerd aan safebuy, en een controller functie welke ons 'Voeg toe aan winkelwagen' functionaliteit verleent.

De controller aanroep ziet er als volgt uit: `{{ render(url('sylius_shop_partial_product_show_by_slug', { 'slug': 'safebuy', 'template': 'Cart/individualCartItem.html.twig'} )) }}`

De route die we meegeven is een bestaande route geconfigureerd door Sylius, welke een product weer wil geven door middel van de meegegeven `slug` en `template`. De `template` die we meegeven is wel een die we zelf aanmaken.


Binnen `template` `Cart/individualCartItem.html.twig`, hebben we toegang tot het desbetreffende (safebuy) product, deze gebruiken we om een andere controller aan te roepen om de 'Voeg toe aan winkelwagen' knop weer te geven, Dit ziet er als volgt uit;

`{{ render(url('sylius_shop_partial_cart_add_item', { 'template': 'Product/addToCartIndividualButton.html.twig', 'productId': product.id } )) }}`

Wederom is deze route al voorgeconfigureerd door Sylius, en is de `template` door ons zelf aangemaakt, Hierin hebben we door de controller waar het product aan is meegegeven, toegang tot een formulier door middel van welke wij, onze 'Voeg toe aan winkelwagen' knop aan kunnen maken.

Verder moeten we nog de safebuy afbeelding weergeven, we downloaden deze van het huidige bogijn, en zetten deze neer in `Bogijn3/themes/BootstrapTheme/assets/media/safebuy.png`, vervolgens in `Bogijn3/themes/BootstrapTheme/assets/app.js` importeren we `./js/images;`, welke we aanmaken op de genoemde locatie.

Hier importeren en exporteren we de afbeelding:
```js
import safebuy from '../media/safebuy.png';

export default {safebuy };

```

Nu is de afbeelding bruikbaar in ons `safebuy.html.twig, en gebruiken we hem als volgt:

```js
<img class="img-fluid"
    src="{{ asset('bootstrap-theme/images/safebuy.png', 'bootstrapTheme') }}"
    alt="SafeBuy"
    style="max-width: 100px"
/>
```

Verder voegen we de gewenste text toe in de vertalingen, en renderen we deze, samen met een link naar de pagina welke safebuy informatie zal bevatten (komt uit een import)

```twig
<p>
    {{ 'bogijn.ui.safebuy_text'|trans }}
    <a href="{{ path('bitbag_sylius_cms_plugin_shop_page_show', {'slug': 'safebuy'}) }}">{{ 'bogijn.ui.more_info'|trans }}</a>
</p>
```