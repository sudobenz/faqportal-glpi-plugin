# FAQ Portal — plugin GLPI 11.0.11+

Version du plugin : **1.0.3**  
Version minimale : **GLPI 11.0.11**  
Compatibilité déclarée : **GLPI 11.x** (GLPI 12.x non validé).

## Objectif

Le plugin affiche les résultats de recherche FAQ Portal dans deux interfaces GLPI : la recherche instantanée de l'accueil Self-Service et la recherche du Catalogue de services. Il ne crée, ne synchronise et n'utilise aucun article de la base de connaissances native GLPI.

Cette version ajoute un mécanisme réversible de désactivation de la base native GLPI pour **tous les profils**, y compris Self-Service, Hotliner, Technicien, Superviseur, Admin et Super-Admin.

## Recherche instantanée de l'accueil Self-Service (v1.0.3)

La recherche d'accueil GLPI 11 utilise `#search-input` et `#search-results`, puis remplace le contenu de `#search-results` après chaque réponse native `/Helpdesk/Search`. La v1.0.3 ajoute les résultats FAQ Portal dans une section distincte `Articles FAQ Portal` au bas de cette liste.

Le JavaScript observe les changements directs des enfants de `#search-results` et réinsère la section FAQ après chaque remplacement HTML natif. Il ne remplace pas la réponse native, ne modifie pas le contrôleur GLPI et ne touche pas aux événements des formulaires.

La requête utilise le même relais PHP `front/search.php` que l'intégration du Catalogue de services. La clé API reste côté serveur. Le panneau n'apparaît qu'à partir de deux caractères ; les résultats natifs restent disponibles même si l'API FAQ est indisponible.

L'intégration existante dans le Catalogue de services (`display_service_catalog`) est conservée sans changement de comportement.

## Désactivation de la base native GLPI

Dans **Configuration > Plugins > FAQ Portal**, une section « Base de connaissances native GLPI » permet de déclencher explicitement la désactivation.

L'action :

1. Enregistre une sauvegarde des droits `knowbase` de chaque profil existant dans la configuration du plugin (`plugin:faqportal`).
2. Met le masque de droits `knowbase` à `0` pour tous les profils existants, via l'API GLPI `ProfileRight::updateProfileRights()`.
3. Met le paramètre global `use_public_faq` à `0` pour empêcher le contournement des droits par l'accès FAQ anonyme.
4. Réapplique le verrouillage à chaque initialisation du plugin : tout droit `knowbase` réintroduit sur un profil existant ou nouvellement créé est remis à `0` au prochain chargement GLPI.
5. Met le droit `knowbase` à `0` dans la session courante. Les autres sessions sont corrigées à leur prochaine requête.

Le masque `knowbase` regroupe les droits de lecture, création, mise à jour, suppression, administration de la base, lecture FAQ, publication FAQ et commentaires (selon les droits définis par GLPI 11.0.11).

### Ce que l'action ne fait pas

- Elle ne supprime aucun article GLPI.
- Elle ne supprime aucune pièce jointe, traduction, révision ou commentaire.
- Elle ne modifie pas les tickets, formulaires, réservations ou le Catalogue de services.
- Elle ne touche pas aux autres profils et paramètres en dehors du droit `knowbase` et de `use_public_faq`.
- Elle ne modifie pas la base FAQ Portal.

### Restauration

La même page affiche un bouton de restauration lorsque le verrouillage est actif.

La restauration :

- réapplique les droits `knowbase` enregistrés pour les profils présents au moment de la désactivation ;
- restaure la valeur initiale de `use_public_faq` ;
- désactive ensuite le mécanisme de verrouillage et supprime la sauvegarde interne du plugin.

Un profil créé après la désactivation ne possède pas de droits initiaux dans la sauvegarde ; il reste donc sans droits `knowbase` après restauration. C'est volontaire : le plugin ne lui attribue pas de privilèges qui n'existaient pas avant le verrouillage.

Si le plugin est désinstallé alors que le verrouillage est actif, `plugin_faqportal_uninstall()` tente d'abord de restaurer les droits et le paramètre public avant de supprimer sa configuration. Il est néanmoins recommandé d'utiliser le bouton **Restaurer les droits** avant toute désinstallation.

## Installation / mise à jour

1. Faire une sauvegarde de la base GLPI et du répertoire `plugins/faqportal`.
2. Remplacer le répertoire `plugins/faqportal` par celui de l'archive.
3. Ne pas désinstaller le plugin pour une mise à jour : cela déclencherait volontairement la restauration des droits natifs.
4. Conserver le même nom de dossier : `faqportal`.
5. Ouvrir **Configuration > Plugins > FAQ Portal**.
6. Vérifier l'URL FAQ Portal et la clé API déjà configurées.
7. Dans la section « Base de connaissances native GLPI », cliquer sur **Désactiver la base de connaissances native pour tous les profils** et confirmer.

Aucune migration SQL spécifique n'est requise : la sauvegarde des droits est stockée dans `glpi_configs` sous le contexte du plugin.

## Configuration FAQ Portal

Dans le `.env` du conteneur FAQ Portal, définir une clé dédiée :

```dotenv
GLPI_API_TOKEN=SECRET_ALEATOIRE
GLPI_API_MAX_RESULTS=8
```

L'API utilisée par le relais PHP GLPI est :

```http
GET /api/integrations/glpi/v1/search?q=connexion%20vpn&limit=8
Authorization: Bearer <GLPI_API_TOKEN>
```

La clé API reste côté serveur GLPI et n'est jamais transmise au navigateur.

## Validation / limites

Contrôles effectués pour cette archive :

- Syntaxe PHP de tous les fichiers du plugin (`php -l`).
- Syntaxe JavaScript vérifiée avec Node.js (`node --check`).
- Vérification statique des sélecteurs natifs GLPI 11 : `#search-input` et `#search-results`.
- Vérification statique des appels GLPI utilisés : `ProfileRight::getProfileRights`, `ProfileRight::updateProfileRights`, `Config::getConfigurationValue`, `Config::setConfigurationValues` et `Config::deleteConfigurationValues` existent dans GLPI 11.0.11.
- Vérification du nom du droit `knowbase`, des bits `KNOWBASEADMIN`, `READFAQ`, `PUBLISHFAQ`, `COMMENTS` et du contrôle d'accès anonyme `use_public_faq` dans le code GLPI 11.0.11.

Une validation fonctionnelle finale nécessite l'application sur une instance GLPI 11.0.11 : contrôler la liste des profils avant/après, les accès directs `/front/knowbaseitem.php` et `/front/helpdesk.faq.php`, puis effectuer une restauration de test. Cette archive n'a pas été exécutée dans une instance GLPI complète.
