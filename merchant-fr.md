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
  x- **Montant minimum** : strictement supérieur à `2 900` (arrondi côté backend).

## Vue d’ensemble

Les marchands initient une demande de paiement à débiter sur le compte mobile money du client final. Shwary :

1. Valide les identifiants marchand fournis dans les headers.
2. Appelle éventuellement votre `callbackUrl` à chaque changement d’état.

## Authentification & En-têtes

Chaque point de terminaison marchand est protégé par `MerchantGuard`. Incluez :

| En-tête          | Description        |
| ---------------- | ------------------ |
| `x-merchant-id`  | UUID du marchand   |
| `x-merchant-key` | Secret du marchand |

Des clés manquantes ou invalides retournent `401 Unauthorized`.

## Pays pris en charge

Utilisez le paramètre de chemin `countryCode` pour cibler les bons rails.

| Code  | Pays                             | Indicatif | Devise |
| ----- | -------------------------------- | --------- | ------ |
| `DRC` | République Démocratique du Congo | `+243`    | `CDF`  |
| `KE`  | Kenya                            | `+254`    | `KES`  |
| `UG`  | Ouganda                          | `+256`    | `UGX`  |

`clientPhoneNumber` doit commencer par l’indicatif du pays.

## Corps de requête partagé

```jsonc
{
  "amount": 5000,
  "clientPhoneNumber": "+243812345678",
  "callbackUrl": "https://merchant.exemple.com/hooks/shwary" // optionnel
}
```

| Champ               | Obligatoire | Description                                                     |
| ------------------- | ----------- | --------------------------------------------------------------- |
| `amount`            | ✔           | Montant dans la devise du pays ciblé. > 0 (≥ 2900 pour la RDC). |
| `clientPhoneNumber` | ✔           | Numéro E.164 avec indicatif.                                    |
| `callbackUrl`       | ✖           | URL HTTPS recevant les mises à jour asynchrones.                |

## Structure de réponse

Tous les endpoints renvoient `TransactionResponseDto` :

```json
{
  "id": "c0fdfe50-24be-4de1-9f66-84608fd45a5f",
  "userId": "merchant-uuid",
  "amount": 5000,
  "currency": "CDF",
  "type": "deposit",
  "status": "pending",
  "recipientPhoneNumber": "+243812345678",
  "referenceId": "merchant-6c661f48-0c39-4474-9621-931d4419babb",
  "metadata": null,
  "failureReason": null,
  "completedAt": null,
  "createdAt": "2025-01-16T10:15:00.000Z",
  "updatedAt": "2025-01-16T10:15:00.000Z",
  "isSandbox": false
}
```

- `status` commence à `pending` pour les flux monétiques réels et à `completed` pour le sandbox.
- `isSandbox` permet d’identifier les transactions simulées.

## Contrat de callback

Si `callbackUrl` est défini, Shwary envoie un POST contenant la transaction à chaque changement d’état (`pending` → `completed` ou `failed`). Attendez-vous au même JSON que la réponse, avec des champs additionnels (ex. `error` en cas d’échec).

Il n’y a **pas** encore de "retry" automatique : assurez-vous que votre endpoint soit hautement disponible et idempotent.

## Endpoints

### 1. Paiement marchand standard

`POST /merchants/payment/:countryCode`

Effectue une transaction via le compte mobile money du client et crédite le wallet Shwary du marchand.

- **Cycle de statut** : `pending` → `completed` (après callback Pretium) ou `failed`.

**Exemple de requête**

```http
POST /merchants/payment/DRC HTTP/1.1
Host: api.shwary.com
x-merchant-id: f5a9f5db-1b33-4d76-9168-0035a6f71170
x-merchant-key: shwary_live_merchant_secret
Content-Type: application/json

{
  "amount": 5000,
  "clientPhoneNumber": "+243812345678",
  "callbackUrl": "https://merchant.exemple.com/hooks/shwary"
}
```

**Réponse typique**

```json
{
  "id": "e44b497a-d2d3-4b82-aad5-0bddb674ba69",
  "status": "pending",
  "currency": "CDF",
  "pretium_transaction_id": "PRT-123456",
  "isSandbox": false,
  "...": "..."
}
```

### 2. Paiement sandbox

`POST /merchants/payment/sandbox/:countryCode`

Pensé pour l’intégration. Aucun appel mobile money n’est effectué; une transaction complétée est inscrite avec `is_sandbox = true`.

- **Cycle de statut** : `completed` instantanément (ou `failed` si validation KO).
- **Callback** : déclenché pour tester la gestion des webhooks.

**Réponse exemple**

```json
{
  "id": "6a6bb0e6-7400-4bd9-9b1e-5b518c352da9",
  "status": "completed",
  "currency": "CDF",
  "referenceId": "merchant-sandbox-5f6b8f7a-4430-4e2a-ab46-0b3bf63503d4",
  "isSandbox": true,
  "...": "..."
}
```

Utilisez le sandbox pour valider :

1. L’authentification via headers.
2. Les règles de validation (montants, format des numéros, etc.).
3. Le comportement de votre endpoint de callback.

## Gestion des erreurs

| Statut             | Signification                                 | Exemple                                                 |
| ------------------ | --------------------------------------------- | ------------------------------------------------------- |
| `400 Bad Request`  | Erreur de validation (montant, pays, numéro). | `{ "message": "Amount must be greater than 2900 CDF" }` |
| `401 Unauthorized` | En-têtes marchands manquants/invalides.       | `{ "message": "Invalid merchant key" }`                 |
| `404 Not Found`    | Client ou marchand absent de Shwary.          | `{ "message": "Client not found" }`                     |
| `502 Bad Gateway`  | Échec renvoyé par Pretium vers le marchand.   |                                                         |

Vérifiez toujours `failureReason` (et `error` dans les callbacks) pour diagnostiquer.

## Checklist d’intégration

2. Appelez l’endpoint sandbox pour valider auth, payloads et callback.
3. Écoutez les webhooks et reconciliez via `transactionId`.
4. Passez aux endpoints standard ou Shwary lorsque vous êtes prêts.

Besoin d’aide ? Communiquez les horodatages à l’équipe Shwary pour accélérer l’investigation côté logs.

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
