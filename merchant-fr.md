# Intégration du point d'accès marchand Shwary

Cette note se concentre uniquement sur l'API marchand Shwary.

## Création de la clé marchande

1. Créez un compte sur app.shwary.com.
2. Rendez-vous dans les paramètres.
3. Générez une clé marchande.
4. Copiez immédiatement la clé marchande et l'identifiant marchand.

**Important** : la clé n'est affichée qu'une seule fois. Une fois la fenêtre fermée, elle ne peut plus être révélée. Assurez-vous de la sauvegarder.

## Vue d'ensemble

- **Fournisseur** : Shwary (`https://api.shwary.com/`)
- **But** : initier des paiements Mobile Money pour les portefeuilles **RDC**, **Kenya** et **Ouganda**.
- **Endpoint marchand** : `POST https://api.shwary.com/api/v1/merchants/payment/{countryCode}`
- **Montant minimum** : `100` (arrondi côté backend).

Les marchands initient une demande de paiement à débiter sur le compte mobile money du client final. Shwary :

1. Valide les identifiants marchand fournis dans les headers.
2. Appelle votre `callbackUrl` à chaque changement d'état.

## Authentification & En-têtes

Chaque point de terminaison marchand est protégé par `MerchantGuard`. Incluez :

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

`clientPhoneNumber` doit commencer par l'indicatif du pays.

## Corps de requête partagé

```jsonc
{
  "amount": 5000,
  "clientPhoneNumber": "+243812345678",
  "callbackUrl": "https://merchant.exemple.com/hooks/shwary" // optionnel
}
```

| Champ               | Obligatoire | Description                                                      |
| ------------------- | ----------- | ---------------------------------------------------------------- |
| `amount`            | ✔           | Montant dans la devise du pays ciblé. > 0 (≥ 2900 pour la RDC). |
| `clientPhoneNumber` | ✔           | Numéro E.164 avec indicatif.                                     |
| `callbackUrl`       | ✖           | URL HTTPS recevant les mises à jour asynchrones.                 |

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
  "txHash": null,
  "completedAt": null,
  "createdAt": "2025-01-16T10:15:00.000Z",
  "updatedAt": "2025-01-16T10:15:00.000Z",
  "isSandbox": false
}
```

- `status` commence à `pending`. Voir [Statuts de transaction](#statuts-de-transaction) pour toutes les valeurs possibles.
- `isSandbox` permet d'identifier les transactions simulées.
- `txHash` est le hash de la transaction on-chain, renseigné lorsque la transaction est complétée.
- `failureReason` fournit des détails lorsqu'une transaction échoue ou est annulée.

## Statuts de transaction

| Statut      | Description                                                                                          |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| `pending`   | Transaction créée, en attente de traitement.                                                         |
| `submitted` | Transaction envoyée au fournisseur de paiement, en attente de confirmation.                          |
| `completed` | Transaction traitée avec succès. `txHash` et `completedAt` sont renseignés.                          |
| `failed`    | La transaction a échoué. Consultez `failureReason` pour les détails.                                 |
| `cancelled` | Transaction annulée (ex. : l'utilisateur a annulé l'invite mobile money). Consultez `failureReason`. |

## Contrat de callback

Si `callbackUrl` est défini, Shwary envoie un `POST` contenant la transaction complète à chaque changement d'état. Votre endpoint recevra des callbacks pour les transitions suivantes :

| Déclencheur du callback    | Statut dans le payload |
| -------------------------- | ---------------------- |
| Transaction créée          | `pending`              |
| Transaction réussie        | `completed`            |
| Transaction échouée        | `failed`               |
| Transaction annulée        | `cancelled`            |

**Exemple de payload callback (completed) :**

```json
{
  "id": "c0fdfe50-24be-4de1-9f66-84608fd45a5f",
  "userId": "merchant-uuid",
  "amount": 5000,
  "currency": "CDF",
  "type": "deposit",
  "status": "completed",
  "recipientPhoneNumber": "+243812345678",
  "referenceId": "merchant-6c661f48-0c39-4474-9621-931d4419babb",
  "txHash": "0xabc123...",
  "failureReason": null,
  "completedAt": "2025-01-16T10:15:30.000Z",
  "createdAt": "2025-01-16T10:15:00.000Z",
  "updatedAt": "2025-01-16T10:15:30.000Z",
  "isSandbox": false,
  "callbackUrl": "https://merchant.exemple.com/hooks/shwary"
}
```

**Exemple de payload callback (failed/cancelled) :**

```json
{
  "id": "c0fdfe50-24be-4de1-9f66-84608fd45a5f",
  "status": "failed",
  "failureReason": "Solde mobile money insuffisant",
  "amount": 5000,
  "currency": "CDF",
  "...": "..."
}
```

**Notes importantes :**

- Il n'y a **pas** de retry automatique : assurez-vous que votre endpoint soit hautement disponible et idempotent.
- Les callbacks ont un timeout de 10 secondes. Si votre endpoint ne répond pas dans ce délai, le callback est considéré comme échoué.
- Vérifiez toujours le champ `status` pour déterminer le résultat de la transaction. Utilisez `failureReason` pour diagnostiquer les problèmes sur les transactions `failed` ou `cancelled`.

## Endpoints

### 1. Paiement marchand standard

`POST /merchants/payment/:countryCode`

Effectue une transaction via le compte mobile money du client et crédite le wallet Shwary du marchand.

- **Cycle de statut** : `pending` → `completed` (après confirmation du fournisseur) ou `failed` / `cancelled`.

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

Pensé pour l'intégration et les tests. Aucun appel blockchain ou mobile money n'est effectué ; la transaction est créée avec `is_sandbox = true`.

- **Cycle de statut** : `pending` → `completed` (après ~5 secondes), simulant le cycle de vie réel d'une transaction.
- **Callback** : déclenché à chaque changement de statut pour tester votre gestion des webhooks de bout en bout.

**Réponse exemple**

```json
{
  "id": "6a6bb0e6-7400-4bd9-9b1e-5b518c352da9",
  "status": "pending",
  "currency": "CDF",
  "referenceId": "merchant-sandbox-5f6b8f7a-4430-4e2a-ab46-0b3bf63503d4",
  "isSandbox": true,
  "...": "..."
}
```

Utilisez le sandbox pour valider :

1. L'authentification via headers.
2. Les règles de validation (montants, format des numéros, etc.).
3. Le comportement de votre endpoint de callback — vous recevrez un callback `pending` immédiatement, puis un callback `completed` après ~5 secondes avec un `txHash` fictif.

### 3. Récupérer une transaction marchande par ID

`GET /merchants/transactions/{id}`

Retourne une transaction unique à partir de son ID.

**Paramètres**

| Nom  | Type   | Obligatoire | Description            |
| ---- | ------ | ----------- | ---------------------- |
| `id` | string | ✔           | UUID de la transaction |

**Exemple de requête**

```http
GET /merchants/transactions/c0fdfe50-24be-4de1-9f66-84608fd45a5f HTTP/1.1
Host: api.shwary.com
x-merchant-id: f5a9f5db-1b33-4d76-9168-0035a6f71170
x-merchant-key: shwary_live_merchant_secret
```

**Réponse typique (200)**

```json
{
  "id": "c0fdfe50-24be-4de1-9f66-84608fd45a5f",
  "userId": "merchant-uuid",
  "amount": 5000,
  "currency": "CDF",
  "type": "deposit",
  "status": "pending",
  "recipientPhoneNumber": "+243972345678",
  "referenceId": "merchant-6c661f48-0c39-4474-9621-931d4419babb",
  "metadata": null,
  "failureReason": null,
  "txHash": null,
  "completedAt": null,
  "createdAt": "2025-01-16T10:15:00.000Z",
  "updatedAt": "2025-01-16T10:15:00.000Z",
  "isSandbox": false
}
```

## Gestion des erreurs

| Statut             | Signification                                  | Exemple                                                 |
| ------------------ | ---------------------------------------------- | ------------------------------------------------------- |
| `400 Bad Request`  | Erreur de validation (montant, pays, numéro).  | `{ "message": "Amount must be greater than 2900 CDF" }` |
| `401 Unauthorized` | En-têtes marchands manquants/invalides.        | `{ "message": "Invalid merchant key" }`                 |
| `404 Not Found`    | Client ou marchand absent de Shwary.           | `{ "message": "Client not found" }`                     |
| `502 Bad Gateway`  | Échec renvoyé par le gateway vers le marchand. |                                                         |

Vérifiez toujours `failureReason` (et `error` dans les callbacks) pour diagnostiquer.

## Checklist d'intégration

1. Appelez l'endpoint sandbox pour valider auth, payloads et callback.
2. Écoutez les webhooks et reconciliez via `transactionId`.
3. Gérez tous les statuts terminaux dans votre handler de callback : `completed`, `failed` et `cancelled`.
4. Passez aux endpoints standard ou Shwary lorsque vous êtes prêts.

Besoin d'aide ? Communiquez les identifiants de requête et les horodatages à l'équipe Shwary pour accélérer l'investigation côté logs.

## Exemple de requête (cURL)

L'extrait ci-dessous déclenche un paiement en CDF pour un portefeuille en RDC. Remplacez `DRC` par `KE` ou `UG`, et ajustez le montant/la devise pour correspondre au marché ciblé.

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
