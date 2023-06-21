# Zoekfunctionaliteit Wijzigingsvoorstel

Het schijnt dat elasticsearch moeite heeft met taxons waar een '-' in voorkomt, in de import commandos dus alle taxons die worden geimporteerd waar een '-' voorkomt, vervangen met een `_`, en verder code die er van uit gaat dat er een '-' tussen de taxons kan staan veranderen om er van uit te gaan dat het een `_` is.