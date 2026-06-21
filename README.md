# AntiTheft 🔒

![Build APK](https://github.com/<TON_PSEUDO>/AntiTheftApp/actions/workflows/build.yml/badge.svg)

Application Android **antivol** à installer **sur son propre téléphone**.

1. **Changement de SIM détecté** → SMS de localisation au contact de confiance **+ photo (caméra frontale) et localisation par e-mail**
2. **Protection contre la désinstallation** (Device Admin + code)
3. **Batterie sous un seuil** → SMS de localisation
4. **Identifiants chiffrés** via l'Android Keystore (code + mot de passe e-mail)
5. **APK signé** produit automatiquement par la CI sur chaque tag `v*`

> ⚠️ Destiné à votre propre appareil. Respectez la loi locale et les règles du Play Store si vous distribuez l'application.

---

## 🚀 Démarrage rapide

```bash
git clone https://github.com/<TON_PSEUDO>/AntiTheftApp.git
cd AntiTheftApp
```

Ouvrez le dossier dans **Android Studio** (*File → Open*).

### ⚙️ Le wrapper Gradle (à générer la 1re fois)

Ce dépôt ne contient pas le binaire `gradle/wrapper/gradle-wrapper.jar` ni les scripts
`gradlew`. Ils se régénèrent automatiquement :

- **Android Studio** les crée à la première synchronisation Gradle, **ou**
- en ligne de commande : `gradle wrapper --gradle-version 8.7`

> La CI n'a pas besoin du wrapper : elle installe Gradle 8.7 directement.

---

## 🔏 Signature & release d'un APK installable

Un APK de **debug** s'installe pour tester, mais pour un APK **release** distribuable, il
faut le signer. Le projet gère ça automatiquement.

### 1. Générer un keystore (une fois)

```bash
keytool -genkeypair -v \
  -keystore antitheft-release.jks \
  -alias antitheft \
  -keyalg RSA -keysize 2048 -validity 10000
```

> Conserve ce fichier et ses mots de passe précieusement : sans eux, tu ne pourras plus
> publier de mise à jour signée avec la même identité.

### 2a. Build signé en local

Copie `keystore.properties.example` → `keystore.properties` (ignoré par git) et renseigne
le chemin du `.jks` et les mots de passe, puis :

```bash
gradle assembleRelease
# → app/build/outputs/apk/release/app-release.apk
```

### 2b. Release automatique via GitHub Actions

Ajoute 4 **secrets** au dépôt (*Settings → Secrets and variables → Actions*) :

| Secret | Contenu |
|---|---|
| `KEYSTORE_BASE64` | le `.jks` encodé en base64 (voir ci-dessous) |
| `KEYSTORE_PASSWORD` | mot de passe du keystore |
| `KEY_ALIAS` | `antitheft` |
| `KEY_PASSWORD` | mot de passe de la clé |

Encodage du keystore en base64 :

```bash
base64 -w0 antitheft-release.jks      # Linux
base64 -i  antitheft-release.jks      # macOS
```

Puis pose un tag pour déclencher le build signé + la Release :

```bash
git tag v1.0.0
git push origin v1.0.0
```

Le workflow `.github/workflows/release.yml` compile, signe, et **attache l'APK à une Release
GitHub** (onglet *Releases*). Le keystore est reconstitué uniquement le temps du build puis supprimé.

---

## 🔐 Sécurité des identifiants

Le code de désinstallation et le mot de passe d'application e-mail sont **chiffrés** (AES-256-GCM)
via une clé de l'**Android Keystore** (matériel quand disponible, non exportable). Voir
`util/Crypto.kt`. `EncryptedSharedPreferences` est volontairement évité car **déprécié depuis
avril 2025**. Migration transparente : une ancienne valeur en clair est relue puis ré-écrite chiffrée.

---

## 📧 Configuration e-mail (pour la photo)

Le SMS ne transporte pas d'image : la photo part par **e-mail** (lien de localisation dans le corps).

**Avec Gmail :** active la validation en 2 étapes → `myaccount.google.com` → **Sécurité** →
**Mots de passe des applications** → génère un mot de passe (16 caractères) → renseigne dans
l'appli e-mail destinataire / expéditeur / ce mot de passe. Le mot de passe normal ne marche pas.

---

## 🤖 Intégration continue

- `build.yml` : compile l'APK **debug** à chaque push / pull request sur `main`.
- `release.yml` : compile l'APK **signé** et crée une **Release** sur chaque tag `v*`.

---

## 🧱 Versions & dépendances

| Outil | Version |
|---|---|
| Android Gradle Plugin | 8.5.2 |
| Gradle | 8.7 |
| Kotlin | 2.0.0 |
| compileSdk / targetSdk | 34 |
| minSdk | 26 (Android 8.0) |
| JDK | 17 |
| CameraX | 1.3.4 |
| JavaMail (com.sun.mail) | 1.6.7 |
| Chiffrement | Android Keystore (framework, 0 dépendance) |

---

## 📂 Structure du projet

```
AntiTheftApp/
├── .github/workflows/
│   ├── build.yml                    ← CI : APK debug (push/PR)
│   └── release.yml                  ← CI : APK signé + Release (tag v*)  (NOUVEAU)
├── keystore.properties.example      ← modèle de signature locale         (NOUVEAU)
├── build.gradle · settings.gradle · gradle.properties
├── gradle/wrapper/gradle-wrapper.properties
├── .gitignore · LICENSE · README.md
└── app/
    ├── build.gradle                 ← config + signature release
    ├── proguard-rules.pro
    └── src/main/
        ├── AndroidManifest.xml
        ├── java/com/example/antitheft/
        │   ├── MainActivity.kt
        │   ├── admin/MyDeviceAdminReceiver.kt
        │   ├── receiver/{BootReceiver,SimChangeReceiver}.kt
        │   ├── service/{BatteryMonitorService,CaptureService}.kt
        │   └── util/{AlertHelper,Crypto,LocationFetcher,Mailer,Prefs,SimChecker}.kt
        └── res/  (icône, strings, device_admin.xml)
```

---

## 📲 Utilisation

1. Renseigner : numéro de confiance, code de désinstallation, seuil batterie,
   e-mail destinataire / expéditeur / mot de passe d'application → **Enregistrer**.
2. **Demander les permissions** (SMS, téléphone, **caméra**, localisation).
3. *Réglages Android* → localisation **« Toujours »** + **exclure des optimisations batterie**.
4. **Activer la protection désinstallation**, puis **Démarrer la surveillance**.

---

## ⚠️ Limites importantes (Android moderne)

- **Photo depuis l'arrière-plan : restreinte sur Android 11+** (cas d'un changement de SIM).
  Fiable sur Android ≤ 10 et sur beaucoup d'appareils si la permission caméra est accordée
  **et** l'appli exclue des optimisations batterie — mais pas garanti. Parade robuste : réveil
  par push FCM (nécessite un backend).
- **Numéro de série de la SIM : bloqué depuis Android 10.** L'empreinte se base sur l'opérateur ;
  une SIM du même opérateur peut ne pas être détectée.
- **Batterie : déclenchez à ~5 %, pas 1 %.**
- **Play Store** : `SEND_SMS` très encadré. Aucun souci en installation perso (APK).

---

## 🛠️ Pistes d'amélioration

- **Réveil par push FCM** pour fiabiliser la capture caméra en arrière-plan.
- **Plusieurs photos** (avant + arrière) ou courte rafale.
- **Upload cloud** (Firebase Storage) + lien par SMS, en alternative à l'e-mail.

---

## 📄 Licence

MIT — voir [LICENSE](LICENSE).
