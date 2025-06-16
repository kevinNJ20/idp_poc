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
- Accepte des fichiers images (PNG) de chèques
- Envoie le document à l'API IDP MuleSoft pour analyse
- Stocke temporairement le fichier sur un serveur SFTP
- Retourne un ID d'exécution pour le suivi

### 2. **Récupération des Résultats** (`/execution/{id}`)
- Vérifie le statut du traitement via l'ID d'exécution
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

## ⚙️ Configuration

### 1. **Configuration IDP**
Modifiez les variables dans `DocAnalyzer.xml` :
```xml
<!-- Remplacez par votre token d'accès IDP -->
<set-variable value="VOTRE_TOKEN_IDP" variableName="accessToken"/>

<!-- ID de votre organisation MuleSoft -->
<!-- ID de votre action IDP -->
<!-- Ces valeurs sont dans l'URL de votre requête HTTP -->
```

### 2. **Configuration SFTP**
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

### 3. **Structure des Dossiers SFTP**
Créez la structure suivante sur votre serveur SFTP :
```
/muletest/idp-poc/
├── processing/          # Fichiers en cours de traitement
├── proceeded/
│   ├── valid/          # Chèques valides
│   └── invalid/        # Chèques invalides
```

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
2. Configurez les paramètres (tokens, SFTP, etc.)
3. Exécutez le projet (Run As > Mule Application)

## 📝 Utilisation

### **1. Envoyer un Document pour Analyse**

**Endpoint :** `POST http://localhost:8083/sendFile`

**Headers :**
```
Content-Type: multipart/form-data
```

**Body :**
```
Form Data:
- file: [fichier image du chèque - PNG]
```

**Réponse :**
```json
{
    "success": true,
    "timestamp": "2025-06-16T10:30:00Z",
    "correlationId": "abc-123-def",
    "executionID": "execution-id-from-idp"
}
```

### **2. Vérifier le Résultat du Traitement**

**Endpoint :** `GET http://localhost:8083/execution/{executionID}`

**Réponse (Chèque Valide) :**
```json
{
    "success": true,
    "correlationId": "abc-123-def",
    "timestamp": "2025-06-16T10:35:00Z",
    "executionId": "execution-id-from-idp",
    "Result": "This check is valid",
    "datas": {
        "id": "execution-id",
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

## 🔍 Détails Techniques

### **Flux de Traitement**

1. **Reception du fichier** → Stockage temporaire en mémoire
2. **Appel API IDP** → Envoi du document pour analyse
3. **Stockage SFTP** → Sauvegarde dans le dossier "processing"
4. **Attente de traitement** → L'IDP analyse le document (asynchrone)
5. **Récupération des résultats** → Via l'endpoint de consultation
6. **Validation** → Vérification de la signature détectée
7. **Classification** → Déplacement vers "valid" ou "invalid"

### **Critères de Validation**

Un chèque est considéré comme **valide** si :
- Le traitement IDP s'est terminé avec succès (`status: "SUCCEEDED"`)
- Une signature a été détectée (`signature: "SIGNATURE_DETECTED"`)

### **Gestion d'Erreurs**

- **Erreurs de connectivité** → Logged et propagées
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
├── pom.xml                          # Dépendances Maven
└── mule-artifact.json              # Configuration Mule
```

## 🔧 Dépendances

- **mule-http-connector** (1.10.3) - Communication HTTP
- **mule-sftp-connector** (2.4.4) - Gestion SFTP
- **mule-validation-module** (2.0.6) - Validation des données

## 🚨 Sécurité

⚠️ **Important :** 
- Ne commitez jamais les tokens d'accès dans le code
- Utilisez des variables d'environnement ou des propriétés externes
- Sécurisez les accès SFTP avec des credentials appropriés
- Configurez HTTPS pour les endpoints en production

## 📈 Monitoring et Logs

Les logs sont configurés dans `log4j2.xml` :
- **Fichier de log :** `logs/idp_poc.log`
- **Rotation :** 10 MB par fichier, 10 fichiers max
- **Pattern :** Inclut le correlationId pour le tracing

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

---

**Note :** Ce README suppose que vous avez accès à l'API MuleSoft IDP et à un serveur SFTP configuré. Adaptez les configurations selon votre environnement.
