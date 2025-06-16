# IDP POC - Document Analysis with MuleSoft IDP

## 🎯 Description

Ce projet est une **Proof of Concept (POC)** qui utilise l'**Intelligence Document Processing (IDP)** de MuleSoft pour analyser des documents, spécifiquement des chèques. L'application Mule 4 permet d'envoyer des images de documents à l'API IDP de MuleSoft, de traiter les résultats d'analyse, et de valider automatiquement les chèques selon des critères spécifiques.

## 🏗️ Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Client App    │ ── │   Mule App      │ ── │  MuleSoft IDP   │
│  (Upload File)  │    │  (Processing)   │    │   (Analysis)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                              │
                              ▼
                       ┌─────────────────┐
                       │   SFTP Server   │
                       │ (File Storage)  │
                       └─────────────────┘
```

## 🚀 Fonctionnalités

### 1. **Upload et Traitement de Documents** (`/sendFile`)
- Accepte des fichiers de documents (PNG, JPG, PDF) de chèques
- **Formats supportés :** PNG, JPG, PDF
- **Taille maximum :** 10 MB par fichier
- **Limite :** Aucune limite sur le nombre de fichiers (contrairement à l'interface IDP d'Anypoint Platform qui limite à 10 fichiers de 8 MB max chacun)
- **Authentification :** Token d'accès IDP fourni en paramètre de requête
- Envoie le document à l'API IDP MuleSoft pour analyse
- Stocke temporairement le fichier sur un serveur SFTP
- Retourne un ID d'exécution pour le suivi

### 2. **Récupération des Résultats** (`/execution/{id}`)
- Vérifie le statut du traitement via l'ID d'exécution
- **Authentification :** Token d'accès IDP fourni en paramètre de requête
- Extrait les informations du chèque (montant, date, signature, etc.)
- Valide automatiquement le chèque selon la présence de signature
- Déplace les fichiers vers des dossiers "valid" ou "invalid"

## 📋 Prérequis

- **Java 17+**
- **Apache Maven 3.6+**
- **Anypoint Studio 7.x** (recommandé)
- **Mule Runtime 4.9.0+**
- **Accès à MuleSoft IDP API** (token d'authentification requis)
- **Serveur SFTP** pour le stockage des fichiers

### 📄 **Exigences des Fichiers**

- **Formats acceptés :** PNG, JPG, PDF
- **Taille maximum :** 10 MB par fichier
- **Nombre de fichiers :** Illimité via cette API

> **💡 Avantage par rapport à l'interface IDP :** L'interface web d'Anypoint Platform limite à 10 fichiers maximum de 8 MB chacun, tandis que cette API permet un traitement illimité avec des fichiers jusqu'à 10 MB.

## ⚙️ Configuration

### 1. **Configuration SFTP**
Modifiez la configuration dans `global.xml` :
```xml
<sftp:config name="SFTP_Config">
    <sftp:connection 
        host="VOTRE_SERVEUR_SFTP" 
        username="VOTRE_USERNAME" 
        password="VOTRE_PASSWORD" 
        workingDir="/chemin/vers/dossier"/>
</sftp:config>
```

### 2. **Structure des Dossiers SFTP**
Créez la structure suivante sur votre serveur SFTP :
```
/muletest/idp-poc/
├── processing/          # Fichiers en cours de traitement
├── proceeded/
│   ├── valid/          # Chèques valides
│   └── invalid/        # Chèques invalides
```

### 3. **Configuration IDP**
- **ID Organisation :** `47d02840-e4c0-4f9d-ba31-357b8e00e857`
- **ID Action :** `9cbacc42-a2ea-4db8-8e2b-e58f88f5a491`
- **Version :** `1.0.0`

> **⚠️ Sécurité :** Le token d'accès IDP est maintenant fourni dynamiquement via le paramètre de requête `token`, éliminant le besoin de le hard-coder dans l'application.

## 🛠️ Installation et Déploiement

### 1. **Clone du Projet**
```bash
git clone <repository-url>
cd idp_poc
```

### 2. **Installation des Dépendances**
```bash
mvn clean install
```

### 3. **Déploiement Local**
```bash
mvn clean package
# Puis déployez le JAR généré dans votre runtime Mule
```

### 4. **Déploiement avec Anypoint Studio**
1. Importez le projet dans Anypoint Studio
2. Configurez les paramètres SFTP dans `global.xml`
3. Exécutez le projet (Run As > Mule Application)
4. L'application démarrera sur le port `8083`

## 📝 Utilisation

### **1. Envoyer un Document pour Analyse**

**Endpoint :** `POST http://localhost:8083/sendFile?token={IDP_TOKEN}`

**Paramètres de requête :**
- `token` (requis) : Token d'accès à l'API MuleSoft IDP

**Headers :**
```
Content-Type: multipart/form-data
```

**Body :**
```
Form Data:
- file: [fichier document - PNG, JPG ou PDF - max 10 MB]
```

**Exemple cURL :**
```bash
curl --location 'http://localhost:8083/sendFile?token=3380cc7c-df7c-47e1-b45c-8b5b9d136196' \
--form 'file=@"/C:/Users/kevin/Downloads/Image-cheque-rempli.png"'
```

**Réponse :**
```json
{
    "success": true,
    "timestamp": "2025-06-16T10:30:00Z",
    "correlationId": "abc-123-def",
    "executionID": "2f9051e5-6920-4398-a27b-ae3dc04b2d06"
}
```

### **2. Vérifier le Résultat du Traitement**

**Endpoint :** `GET http://localhost:8083/execution/{executionID}?token={IDP_TOKEN}`

**Paramètres de requête :**
- `token` (requis) : Token d'accès à l'API MuleSoft IDP

**Paramètres d'URL :**
- `executionID` : ID d'exécution retourné par l'endpoint `/sendFile`

**Exemple cURL :**
```bash
curl --location 'http://localhost:8083/execution/2f9051e5-6920-4398-a27b-ae3dc04b2d06?token=3380cc7c-df7c-47e1-b45c-8b5b9d136196'
```

**Réponse (Chèque Valide) :**
```json
{
    "success": true,
    "correlationId": "abc-123-def",
    "timestamp": "2025-06-16T10:35:00Z",
    "executionId": "2f9051e5-6920-4398-a27b-ae3dc04b2d06",
    "Result": "This check is valid",
    "datas": {
        "id": "2f9051e5-6920-4398-a27b-ae3dc04b2d06",
        "documentName": "Image-cheque-rempli.png",
        "status": "SUCCEEDED",
        "check_infos": {
            "date": "2025-06-15",
            "numeroCompte": 1234567890,
            "ville": "Paris",
            "signature": "SIGNATURE_DETECTED",
            "montant": 150.50,
            "numeroCheque": 1001,
            "montantLong": "Cent cinquante euros et cinquante centimes",
            "destinataire": "Jean Dupont"
        }
    }
}
```

**Réponse (Chèque Invalide) :**
```json
{
    "success": true,
    "correlationId": "def-456-ghi",
    "timestamp": "2025-06-16T10:35:00Z",
    "executionId": "2f9051e5-6920-4398-a27b-ae3dc04b2d06",
    "Result": "This check is invalid",
    "datas": {
        "id": "2f9051e5-6920-4398-a27b-ae3dc04b2d06",
        "documentName": "Image-cheque-rempli.png",
        "status": "SUCCEEDED",
        "check_infos": {
            "date": "2025-06-15",
            "numeroCompte": 1234567890,
            "ville": "Paris",
            "signature": "NO_SIGNATURE_DETECTED",
            "montant": 150.50,
            "numeroCheque": 1001,
            "montantLong": "Cent cinquante euros et cinquante centimes",
            "destinataire": "Jean Dupont"
        }
    }
}
```

**Réponse (Traitement en Cours) :**
```json
{
    "success": false,
    "message": "Execution 2f9051e5-6920-4398-a27b-ae3dc04b2d06 is processing"
}
```

## 🔍 Détails Techniques

### **Flux de Traitement**

1. **Reception du fichier** → Stockage temporaire en mémoire
2. **Extraction du token** → Récupération depuis le paramètre de requête
3. **Appel API IDP** → Envoi du document pour analyse avec authentification
4. **Stockage SFTP** → Sauvegarde dans le dossier "processing"
5. **Attente de traitement** → L'IDP analyse le document (asynchrone - ~10 secondes)
6. **Récupération des résultats** → Via l'endpoint de consultation
7. **Validation** → Vérification de la signature détectée
8. **Classification** → Déplacement vers "valid" ou "invalid"

### **Critères de Validation**

Un chèque est considéré comme **valide** si :
- Le traitement IDP s'est terminé avec succès (`status: "SUCCEEDED"`)
- Une signature a été détectée (`signature: "SIGNATURE_DETECTED"`)

### **Gestion de l'Authentification**

- Le token IDP est fourni dynamiquement via le paramètre de requête `token`
- Plus besoin de modifier le code pour changer de token
- Sécurité améliorée : pas de token hard-codé dans l'application

### **Gestion d'Erreurs**

- **Erreurs de connectivité** → Logged et propagées
- **Token invalide/expiré** → Erreur HTTP 401/403
- **Chèques invalides** → Déplacés vers le dossier "invalid"
- **Échecs de validation** → Gérés par le bloc `try/error-handler`

## 📁 Structure du Projet

```
idp_poc/
├── src/
│   ├── main/
│   │   ├── mule/
│   │   │   ├── DocAnalyzer.xml      # Flux principaux
│   │   │   └── global.xml           # Configurations globales
│   │   └── resources/
│   │       ├── application-types.xml
│   │       └── log4j2.xml           # Configuration logging
│   └── test/
│       └── resources/
│           └── log4j2-test.xml      # Configuration logging tests
├── exchange-docs/
│   └── home.md                      # Documentation Exchange
├── README.md                        # Ce fichier
├── pom.xml                          # Dépendances Maven
├── mule-artifact.json              # Configuration Mule
└── .gitignore                       # Fichiers ignorés par Git
```

## 🔧 Dépendances

- **mule-http-connector** (1.10.3) - Communication HTTP
- **mule-sftp-connector** (2.4.4) - Gestion SFTP
- **mule-validation-module** (2.0.6) - Validation des données

## 🚨 Sécurité

⚠️ **Important :** 
- Les tokens d'accès sont fournis dynamiquement via les paramètres de requête
- Ne loggez jamais les tokens dans les fichiers de log
- Sécurisez les accès SFTP avec des credentials appropriés
- Configurez HTTPS pour les endpoints en production
- Utilisez HTTPS pour les appels vers l'API IDP MuleSoft

## 📈 Monitoring et Logs

Les logs sont configurés dans `log4j2.xml` :
- **Fichier de log :** `logs/idp_poc.log`
- **Rotation :** 10 MB par fichier, 10 fichiers max
- **Pattern :** Inclut le correlationId pour le tracing
- **Niveaux :** INFO pour les opérations principales, WARN pour les erreurs HTTP

## 🧪 Tests avec Postman

### Collection Postman recommandée :

**1. Upload Document**
```
POST http://localhost:8083/sendFile?token={{idp_token}}
Content-Type: multipart/form-data
Body: file (binary)
```

**2. Check Execution Status**
```
GET http://localhost:8083/execution/{{execution_id}}?token={{idp_token}}
```

### Variables d'environnement Postman :
- `idp_token` : Votre token d'accès IDP
- `execution_id` : ID retourné par l'upload

## 🤝 Contribution

1. Fork le projet
2. Créez une branche feature (`git checkout -b feature/nouvelle-fonctionnalite`)
3. Commitez vos changements (`git commit -am 'Ajoute nouvelle fonctionnalité'`)
4. Push vers la branche (`git push origin feature/nouvelle-fonctionnalite`)
5. Créez une Pull Request

## 📄 Licence

Ce projet est un POC à des fins de démonstration et d'apprentissage.

## 📞 Support

Pour toute question ou problème :
- Vérifiez les logs dans `logs/idp_poc.log`
- Consultez la documentation MuleSoft IDP
- Vérifiez la connectivité vers l'API IDP et le serveur SFTP
- Assurez-vous que votre token IDP est valide et non expiré

## 🚀 Améliorations Futures

- Ajout de la validation des tokens en amont
- Support de formats de documents supplémentaires
- Interface web pour faciliter les tests
- Intégration avec des bases de données pour l'historique
- Notifications en temps réel pour les résultats de traitement

---

**Note :** Ce README suppose que vous avez accès à l'API MuleSoft IDP et à un serveur SFTP configuré. Adaptez les configurations selon votre environnement.
