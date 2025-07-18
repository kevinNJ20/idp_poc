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
- Accepte des fichiers de documents (PDF, PNG, JPG, TIFF) de chèques
- **Formats supportés :** PDF, PNG, JPG, TIFF (150 DPI ou plus recommandé)
- **Taille maximum :** 10 MB par fichier (via API)
- **Pages maximum :** 50 pages par fichier
- **Limite :** 1 fichier par requête API (upload illimité via requêtes multiples)
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

- **Formats acceptés :** PDF, PNG, JPG, TIFF (150 DPI ou plus recommandé)
- **Pages maximum :** 50 pages par fichier
- **Taille et nombre de fichiers :**

| **Mode d'Upload** | **Limite Fichiers** | **Taille Max** | **Pages Max** |
|------------------|---------------------|----------------|---------------|
| Interface Web UI | 10 fichiers par upload | 8 MB par fichier | 50 pages par fichier |
| **API (ce projet)** | **1 fichier par requête** | **10 MB par fichier** | **50 pages par fichier** |

> **💡 Avantage de l'API :** Fichiers plus volumineux (10 MB vs 8 MB) et upload illimité via requêtes multiples, contrairement à l'interface web limitée à 10 fichiers simultanés.

> **⚠️ Note TIFF :** Les fichiers TIFF sont supportés pour l'extraction de données mais ne peuvent pas être prévisualisés dans l'interface MuleSoft.

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
- file: [fichier document - PDF, PNG, JPG, TIFF - max 10 MB, max 50 pages]
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
5. **Attente de traitement** → L'IDP analyse le document (asynchrone - ~10 secondes, polling minimum requis)
6. **Récupération des résultats** → Via l'endpoint de consultation (respecter l'intervalle de polling de 10 secondes)
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
- **Fichiers non conformes** → Vérifiez format (PDF/PNG/JPG/TIFF), taille (≤10 MB), pages (≤50)
- **Rate limiting** → L'API IDP limite à 100 jobs concurrents par minute
- **Polling trop fréquent** → Respecter l'intervalle minimum de 10 secondes entre les requêtes
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

## 📊 Limites et Quotas MuleSoft IDP

D'après la documentation officielle MuleSoft, voici les limites importantes à connaître :

### **Limitations de Fichiers**
- **Formats acceptés :** PDF, PNG, JPG, TIFF (150 DPI minimum recommandé)
- **Pages maximum :** 50 pages par fichier
- **Taille maximum :** 10 MB par fichier (API) / 8 MB (Interface Web)

### **Limitations de Requêtes**
- **API :** 1 fichier par requête
- **Interface Web :** 10 fichiers par upload simultané
- **Jobs concurrents :** 100 maximum par minute
- **Polling minimum :** Intervalle de 10 secondes entre les vérifications de statut

### **Limitations de Prompts**
- **Maximum :** 30 prompts par action de document
- **Langues :** Prompts en anglais (autres langues non totalement supportées)
- **Tokens :** 128,000 tokens maximum par requête Einstein

### **Support Linguistique**
Le support des langues varie selon le modèle LLM sélectionné. Consultez :
- [Langues supportées par OpenAI](https://platform.openai.com/docs/supported-languages)
- [Langues supportées par Gemini](https://ai.google.dev/gemini-api/docs/language-support)

> **⚠️ Important :** Cette application respecte automatiquement les limites de polling (10 secondes) et de fichiers (1 par requête) imposées par MuleSoft IDP.

## 🚨 Sécurité 
- Les tokens d'accès sont fournis dynamiquement via les paramètres de requête
- Ne loggez jamais les tokens dans les fichiers de log
- Sécurisez les accès SFTP avec des credentials appropriés
- Configurez HTTPS pour les endpoints en production
- Utilisez HTTPS pour les appels vers l'API IDP MuleSoft

⚠️ **Important :**

Les logs sont configurés dans `log4j2.xml` :
- **Fichier de log :** `logs/idp_poc.log`
- **Rotation :** 10 MB par fichier, 10 fichiers max
- **Pattern :** Inclut le correlationId pour le tracing
- **Niveaux :** INFO pour les opérations principales, WARN pour les erreurs HTTP

## 📈 Monitoring et Logs

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

## 🧪 Tests avec Postman

> **⚠️ Respect du Polling :** Attendez au minimum 10 secondes entre les appels de vérification du statut pour respecter les limites MuleSoft IDP.
- `idp_token` : Votre token d'accès IDP
- `execution_id` : ID retourné par l'upload

### Variables d'environnement Postman :

1. Fork le projet
2. Créez une branche feature (`git checkout -b feature/nouvelle-fonctionnalite`)
3. Commitez vos changements (`git commit -am 'Ajoute nouvelle fonctionnalité'`)
4. Push vers la branche (`git push origin feature/nouvelle-fonctionnalite`)
5. Créez une Pull Request

## 🤝 Contribution

Ce projet est un POC à des fins de démonstration et d'apprentissage.

## 📞 Support

Pour toute question ou problème :
- Vérifiez les logs dans `logs/idp_poc.log`
- Consultez la documentation MuleSoft IDP
- Vérifiez la connectivité vers l'API IDP et le serveur SFTP
- Vérifiez que vos fichiers respectent les limites : format (PDF/PNG/JPG/TIFF), taille (≤10 MB), pages (≤50)
- Respectez l'intervalle de polling de 10 secondes minimum

## 🚀 Améliorations Futures

- Ajout de la validation des tokens en amont
- Support de formats de documents supplémentaires
- Interface web pour faciliter les tests
- Intégration avec des bases de données pour l'historique
- Notifications en temps réel pour les résultats de traitement

---

**Note :** Ce README suppose que vous avez accès à l'API MuleSoft IDP et à un serveur SFTP configuré. Adaptez les configurations selon votre environnement.
