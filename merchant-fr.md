# Intégration du point d’accès marchand Shwary

Cette note se concentre uniquement sur l’API marchand Shwary.

## Création de la clé marchande

1. Créez un compte sur app.shwary.com.
2. Rendez-vous dans les paramètres.
3. Générez une clé marchande.
4. Copiez immédiatement la clé marchande et l’identifiant marchand.

**Important** : la clé n’est affichée qu’une seule fois. Une fois la fenêtre fermée, elle ne peut plus être révélée. Assurez-vous de la sauvegarder.

## Vue d’ensemble

- **Fournisseur** : Shwary (`https://api.shwary.com/`)
- **But** : initier des paiements Mobile Money pour les portefeuilles **RDC**, **Kenya** et **Ouganda**.
- **Endpoint marchand** : `POST https://api.shwary.com/api/v1/merchants/payment/{countryCode}`
- **En-têtes nécessaires** :
  - `x-merchant-key`
  - `x-merchant-id`
- **Montant minimum** : strictement supérieur à `2 900` (arrondi côté backend).

## Variables d’environnement

```env
SHWARY_MERCHANT_KEY=cle-marchande-shwary
SHWARY_MERCHANT_ID=id-marchand-shwary
```

## Exemple de requête (cURL)

L’extrait ci-dessous déclenche un paiement en CDF pour un portefeuille en RDC. Remplacez `DRC` par `KE` ou `UG`, et ajustez le montant/la devise pour correspondre au marché ciblé.

```bash
curl -X POST "https://api.shwary.com/api/v1/merchants/payment/DRC" \
  -H "Content-Type: application/json" \
  -H "x-merchant-key: $SHWARY_MERCHANT_KEY" \
  -H "x-merchant-id: $SHWARY_MERCHANT_ID" \
  -d '{
    "amount": 5000,
    "clientPhoneNumber": "+243820000000",
    "callbackUrl": "https://your-app.com/api/shwary/callback"
  }'
```

## Endpoint de callback (`POST /api/shwary/callback`)

Shwary envoie des mises à jour asynchrones vers l’URL de rappel fournie. Le backend :

1. Accepte la charge utile JSON (aucun en-tête d’authentification requis).
2. Retrouve la commande via `metadata.orderId` ou `transactionId`.
3. Mappe le statut Shwary (`PENDING`, `COMPLETED`, `FAILED`) vers le statut interne (`pending`, `completed`, `cancelled`).

États attendus :

- **Pending** : l’utilisateur doit encore confirmer sur son téléphone.
- **Completed** : le client a saisi son PIN et les fonds sont libérés.
- **Failed** : le client a rejeté, le push a expiré ou Shwary a refusé.

Le service répond toujours `200` + CORS pour confirmer la réception.

## Récapitulatif des validations

1. Le montant doit être numérique et `2900 CDF` ou plus pour la RDC, `3500 UGX` pour l'Ouganda et `130 KES` pour le Kenya
2. Le numéro de téléphone est obligatoire.
3. `countryCode` doit être `DRC`, `KE` ou `UG`.
4. `metadata`, si présent, doit être un objet JSON.
5. Les en-têtes marchands (`SHWARY_MERCHANT_KEY`, `SHWARY_MERCHANT_ID`) sont obligatoires ; s’ils manquent, l’appel est refusé (401/500 selon la configuration).
