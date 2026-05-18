# Application Bancaire Simplifiée – Architecture Microservices

Projet réalisé dans le cadre du cours SoA & Microservices – Dr. Salah Gontara  
A.U. : 2025-2026

# Structure du projet

```
bank-microservices/
│
├── 📁 proto/
│   ├── compte.proto              # Contrat gRPC – CompteService
│   ├── transaction.proto         # Contrat gRPC – TransactionService
│   └── notification.proto        # Contrat gRPC – NotificationService
│
├── 📁 microservice-compte/       # Port gRPC : 50051
│   ├── server.js                 # Serveur gRPC
│   ├── db.js                     # Base SQLite3 (sql.js)
│   └── package.json
│
├── 📁 microservice-transaction/  # Port gRPC : 50052
│   ├── server.js                 # Serveur gRPC
│   ├── db.js                     # Base SQLite3 (sql.js)
│   ├── kafka.js                  # Producteur Kafka
│   └── package.json
│
├── 📁 microservice-notification/ # Port gRPC : 50053
│   ├── server.js                 # Serveur gRPC
│   ├── db.js                     # Base RxDB (NoSQL - LokiJS)
│   ├── kafka.js                  # Consommateur Kafka
│   └── package.json
│
├── 📁 api-gateway/               # Port HTTP : 3000
│   ├── server.js                 # Express – REST + GraphQL
│   ├── grpcClients.js            # Clients gRPC vers les 3 microservices
│   ├── resolvers.js              # Résolveurs GraphQL
│   ├── schema.gql                # Schéma GraphQL
│   └── package.json
│
└── 📁 client/
    └── client.js                 # Client de test – tous les scénarios
```

---

# Installation & Lancement

## Prérequis

| Outil | Version minimale |
|-------|-----------------|
| Node.js | ≥ 18 |
| Java | ≥ 17 (pour Kafka) |
| Kafka | 4.2 |

## 1. Installer les dépendances

```bash
cd microservice-compte        && npm install 
cd microservice-transaction   && npm install 
cd microservice-notification  && npm install 
cd api-gateway                && npm install 
```

## 2. Démarrer Kafka (mode KRaft)

```bash
# Formater le stockage
KAFKA_CLUSTER_ID="$(bin/kafka-storage.bat random-uuid)"
bin/windows/kafka-storage.bat format --standalone -t "$KAFKA_CLUSTER_ID" -c config/server.properties
# Démarrer Kafka
bin/kafka-server-start. config/server.properties
# Créer les topics (dans un autre terminal)
bin/kafka-topics.bat --create --partitions 3 --replication-factor 1 \
  --topic transaction-validee --bootstrap-server localhost:9092
bin/kafka-topics.bat --create --partitions 3 --replication-factor 1 \
  --topic transaction-echouee --bootstrap-server localhost:9092
```

## 3. Lancer les microservices 

cd microservice-compte && node server.js
cd microservice-transaction && node server.js
cd microservice-notification && node server.js
cd api-gateway && node server.js


### 4. Lancer test

bank-test-ui.html

---

##  Endpoints REST

###  Comptes

| Méthode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/comptes` | Créer un compte |
| `GET` | `/api/comptes` | Lister tous les comptes |
| `GET` | `/api/comptes/:id` | Obtenir un compte |
| `PATCH` | `/api/comptes/:id/bloquer` | Bloquer un compte |
| `DELETE` | `/api/comptes/:id` | Supprimer un compte |

###  Transactions

| Méthode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/transactions` | Effectuer une transaction |
| `GET` | `/api/transactions` | Lister toutes les transactions |
| `GET` | `/api/transactions/:id` | Obtenir une transaction |
| `GET` | `/api/comptes/:id/transactions` | Transactions d'un compte |

###  Notifications

| Méthode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/api/notifications` | Toutes les notifications |
| `GET` | `/api/comptes/:id/notifications` | Notifications d'un compte |
| `PATCH` | `/api/notifications/:id/lue` | Marquer comme lue |

---

## GraphQL

### Queries

```graphql
# Lister tous les comptes
query {
  comptes {
    id
    proprietaire
    solde
    statut
  }
}

# Transactions d'un compte
query {
  transactionsDuCompte(compteId: "votre-id") {
    id
    type
    montant
    statut
    createdAt
  }
}

# Notifications d'un compte
query {
  notificationsDuCompte(compteId: "votre-id") {
    message
    type
    lue
    createdAt
  }
}
```

### Mutations

```graphql
# Créer un compte
mutation {
  creerCompte(
    proprietaire: "Ahmed Ben Ali"
    email: "ahmed@banque.tn"
    soldeInitial: 5000
    type: "courant"
  ) {
    id
    solde
  }
}

# Effectuer un virement
mutation {
  effectuerTransaction(
    compteSource: "id-source"
    compteDest: "id-dest"
    montant: 500
    type: "virement"
    description: "Remboursement"
  ) {
    id
    statut
    montant
  }
}
```

---

## Topics Kafka

| Topic | Producteur | Consommateur | Déclencheur |
|-------|------------|-------------|-------------|
| `transaction-validee` | MS Transaction | MS Notification | Transaction réussie |
| `transaction-echouee` | MS Transaction | MS Notification | Solde insuffisant / compte bloqué |

### Format des messages

**`transaction-validee`**
```json
{
  "transactionId": "uuid",
  "compteSource": "uuid",
  "compteDest": "uuid",
  "montant": 500.0,
  "type": "virement",
  "description": "Remboursement",
  "createdAt": "2025-01-01T12:00:00.000Z"
}
```

**`transaction-echouee`**
```json
{
  "transactionId": "uuid",
  "compteSource": "uuid",
  "raison": "Solde insuffisant"
}
```

---

##  Bases de données

| Microservice | SGBD | Type | Stockage |
|---|---|---|---|
| Compte | SQLite3 (sql.js) | Relationnel | `comptes.db` |
| Transaction | SQLite3 (sql.js) | Relationnel | `transactions.db` |
| Notification | RxDB + LokiJS | NoSQL | En mémoire |

---
