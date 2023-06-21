# Route prioriteit Bouwfaserapportage

Na flink wat zoekwerk kwam ik uit dat de route verantwoordelijk voor de paginas de route `bitbag_sylius_cms_plugin_shop_page_show` is, Deze heb ik overgenomen en onderin mijn custom routes geplaatst. Hierbij ook gelijk het pad aangepast dat het nu alleen nog maar /{slug} is:
```yaml
bitbag_sylius_cms_plugin_shop_page_show:
    path: /{slug}
    methods: [GET]
    defaults:
        _controller: bitbag_sylius_cms_plugin.controller.page.overriden:showAction
        _sylius:
            template: "@BitBagSyliusCmsPlugin/Shop/Page/show.html.twig"
            repository:
                method: findOneEnabledBySlugAndChannelCode
                arguments:
                    - $slug
                    - "expr:service('sylius.context.locale').getLocaleCode()"
                    - "expr:service('sylius.context.channel').getChannel().getCode()"

```