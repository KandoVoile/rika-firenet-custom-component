# Revue du code

## Cohérence générale
- L'intégration repose sur une `DataUpdateCoordinator`, mais plusieurs opérations réseau sont encore lancées directement depuis les entités (ex. changements de consigne), ce qui peut bloquer la boucle d'événements Home Assistant.
- Certaines conversions de types peuvent empêcher des fonctionnalités (pilotage du mode HVAC) de fonctionner comme prévu.

## Problèmes repérés
1. **Appels réseau bloquants depuis les entités** : `set_stove_controls` envoie des requêtes HTTP synchrones et effectue un `time.sleep` dans une boucle d'attente. Cette méthode est appelée par les entités (climat, nombre, interrupteurs) depuis le thread principal de Home Assistant, ce qui peut bloquer l'interface et les automatisations pendant plusieurs secondes.【F:custom_components/rika_firenet/core.py†L114-L127】【F:custom_components/rika_firenet/core.py†L245-L387】
2. **Conversion incorrecte du mode HVAC** : le climat appelle `self._stove.set_hvac_mode(str(hvac_mode))`, ce qui transforme l'`HVACMode` en chaîne et fait échouer les comparaisons contre les enums dans `RikaFirenetStove.set_hvac_mode`. Les changements de mode risquent donc de ne rien faire.【F:custom_components/rika_firenet/climate.py†L72-L83】【F:custom_components/rika_firenet/core.py†L314-L333】

## Pistes d'amélioration
- Encapsuler les appels réseau et la boucle d'attente de `set_stove_controls` dans un exécuteur ou les convertir en appels asynchrones (utiliser `hass.async_add_executor_job` et remplacer `time.sleep` par une logique non bloquante). Centraliser les écritures pour éviter de bloquer le thread principal.
- Passer les `HVACMode` tels quels au poêle (supprimer la conversion en chaîne) et, côté poêle, valider/normaliser les valeurs reçues pour éviter les états inattendus.
- Ajouter des délais explicites (`timeout`) aux requêtes de connexion et de contrôle, et journaliser les codes d'erreur pour faciliter le diagnostic.
- Enrichir la documentation (README/manifest) avec les prérequis, les plateformes exposées et les limitations connues afin d'aider les utilisateurs à configurer et dépanner l'intégration.
