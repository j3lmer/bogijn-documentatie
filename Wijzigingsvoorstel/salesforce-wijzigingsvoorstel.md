# Salesforce wijzigingsvoorstel

- In plaats van de landen versturen als `$address->getCountryCode();`, versturen als `Countries::getName($address->getCountryCode());`.
(Symfony intl component).

- Guzzle client vervangen met Symfony Http client.