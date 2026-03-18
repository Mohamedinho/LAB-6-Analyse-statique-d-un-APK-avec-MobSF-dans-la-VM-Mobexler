# Rapport d'analyse statique - Web Service PHP 8

## Informations générales
- **Date d'analyse :** 17 mars 2026
- **Analyste :** Mohamed DOUASSI
- **APK analysé :** app-debug.apk (SHA-256: 810a2cb4b3d277674d5692ffee6c48ee729b2ceb1d7d361380a97013f11154eb)
- **Version :** 1.0 (Code 1)
- **SDK minimum :** 24 (Android 7.0)
- **SDK cible :** 36 (Android 13)
- **Outils utilisés :** MobSF v4.0.6 dans VM Mobexler
- <img width="1901" height="814" alt="image" src="https://github.com/user-attachments/assets/fa06556f-87a9-4ca3-adad-8bb66542f143" />


## Résumé exécutif
L'analyse statique de l'application Web Service PHP 8 révèle un niveau de risque ÉLEVÉ (score 32/100). Les principales vulnérabilités concernent la signature avec un certificat de débogage (au lieu d'un certificat de release), les communications réseau non chiffrées (HTTP avec trafic en clair), et le support de versions Android vulnérables (minSdk=24 correspondant à Android 7.0 sans mises à jour de sécurité). L'application demande 1 seule permission (INTERNET) qui est cohérente avec sa fonction principale. Des composants exportés inutilement augmentent la surface d'attaque.

## Vulnérabilités critiques
<img width="1917" height="543" alt="image" src="https://github.com/user-attachments/assets/11cd838a-a7fc-4db8-8128-3193f83798c8" />


### 1. Signature avec certificat de débogage
- **Sévérité :** CRITIQUE
- **Catégorie MASVS :** MASVS-CODE-3 (Configuration de build)
- **Description :** L'application est signée avec un certificat de débogage Android debug (CN=Android Debug, O=Android, C=US) au lieu d'un certificat de release. Les signatures v2 sont présentes mais v1, v3 et v4 sont absentes.
- **Preuve :** Rapport MobSF, section CERTIFICATE INFORMATION - Sujet: CN=Android Debug, O=Android, C=US
- **Impact :** Un attaquant peut redistribuer l'application avec des modifications malveillantes. Les composants protégés par signature deviennent vulnérables. Rejet automatique par le Google Play Store.
- **Remédiation :** Générer un certificat de release avec keystore approprié et signer l'application pour les builds de production.

### 2. Communications réseau non chiffrées (HTTP)
- **Sévérité :** CRITIQUE
- **Catégorie MASVS :** MASVS-NETWORK-1 (Sécurité réseau)
- **Description :** L'application autorise le trafic réseau en clair avec android:usesCleartextTraffic="true" dans le manifeste. Les endpoints utilisent tous le protocole HTTP non chiffré.
- **Preuve :** AndroidManifest.xml - <application android:usesCleartextTraffic="true"> et endpoints http://10.0.2.2/projet/ws/loadetudiant.php, createetudiant.php, updateetudiant.php, deleteetudiant.php
- **Impact :** Les données personnelles des étudiants sont transmises en clair, vulnérables aux attaques Man-in-the-Middle (MITM). Non-conformité RGPD.
- **Remédiation :** Migrer tous les endpoints vers HTTPS, définir android:usesCleartextTraffic="false", configurer network_security_config.xml.

### 3. Support de versions Android vulnérables
- **Sévérité :** ÉLEVÉE
- **Catégorie MASVS :** MASVS-CODE-3 (Exigences de sécurité minimales)
- **Description :** L'application déclare minSdkVersion="24" (Android 7.0) qui ne reçoit plus de mises à jour de sécurité depuis 2019.
- **Preuve :** AndroidManifest.xml - <uses-sdk android:minSdkVersion="24" android:targetSdkVersion="36">
- **Impact :** Installation possible sur des appareils sans mises à jour de sécurité, exposition aux vulnérabilités critiques connues et non patchées.
- **Remédiation :** Augmenter minSdkVersion à au moins 29 (Android 10) pour garantir des mises à jour de sécurité.

### 4. Activités exportées inutilement
- **Sévérité :** ÉLEVÉE
- **Catégorie MASVS :** MASVS-CODE-4 (Sécurité des composants)
- **Description :** Deux activités sont déclarées avec android:exported="true" sans nécessité fonctionnelle : PreviewActivity et ComponentActivity.
- **Preuve :** AndroidManifest.xml - androidx.compose.ui.tooling.PreviewActivity (exported="true") et androidx.activity.ComponentActivity (exported="true")
- **Impact :** Toute application sur l'appareil peut lancer ces activités, augmentant la surface d'attaque. PreviewActivity expose des fonctionnalités de développement en production.
- **Remédiation :** Définir android:exported="false" pour ces deux activités.

### 5. Broadcast Receiver exposé avec permission DUMP
- **Sévérité :** MOYENNE
- **Catégorie MASVS :** MASVS-CODE-4 (Sécurité des composants)
- **Description :** ProfileInstallReceiver est exporté avec la permission android.permission.DUMP.
- **Preuve :** AndroidManifest.xml - androidx.profileinstaller.ProfileInstallReceiver avec android:exported="true" et android:permission="android.permission.DUMP"
- **Impact :** Des applications malveillantes avec permission DUMP peuvent interagir avec ce receiver et manipuler les profils de performance.
- **Remédiation :** Définir android:exported="false" ou renforcer les contrôles d'accès.

## Autres observations

### Bibliothèques natives (libandroidx.graphics.path.so)
-  NX bit activé (protection contre l'exécution de code)
-  Stack Canary présent (protection buffer overflow)
-  Full RELRO activé (protection GOT)
-  Fonctions fortifiées désactivées (_FORTIFY_SOURCE=2 recommandé)
-  Symboles non supprimés (reverse engineering facilité)

### Permission personnalisée
- com.example.webservicephp8.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION (protectionLevel="signature") - Correct
<img width="1750" height="522" alt="image" src="https://github.com/user-attachments/assets/9349d664-5d25-44eb-8c58-c37e7241a567" />

### Compilateurs détectés
- classes4.dex et classes3.dex : r8 without marker (suspicious) - Potentiel faux positif

## Recommandations prioritaires

1. **URGENT : Remplacer le certificat de débogage par un certificat de release**
   - Générer un keystore de production avec keytool
   - Signer l'APK avec ce nouveau certificat
   - Tester la mise à jour depuis l'ancienne version

2. **URGENT : Migrer tous les endpoints HTTP vers HTTPS**
   - Configurer le serveur avec un certificat TLS valide
   - Mettre à jour les URLs dans le code
   - Désactiver usesCleartextTraffic dans le manifeste

3. **IMPORTANT : Augmenter minSdkVersion à 29 ou supérieur**
   - Modifier build.gradle avec minSdkVersion 29
   - Tester la compatibilité sur Android 10+
   - Mettre à jour les dépendances si nécessaire

4. **IMPORTANT : Désactiver l'export des activités inutiles**
   - Définir exported="false" pour PreviewActivity et ComponentActivity
   - Vérifier l'impact sur les fonctionnalités

5. **À PLANIFIER : Sécuriser les bibliothèques natives**
   - Compiler avec _FORTIFY_SOURCE=2
   - Stripper les symboles des bibliothèques

## Annexes

### Annexe A : Permissions demandées
| Permission | Type | Description |
|------------|------|-------------|
| android.permission.INTERNET | Normale | Accès à Internet |
| com.example.webservicephp8.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION | Normale (signature) | Permission personnalisée |

### Annexe B : Composants exportés
| Type | Nom | Exporté | Recommandation |
|------|-----|---------|----------------|
| Activity | com.example.webservicephp8.MainActivity |  true | OK (point d'entrée) |
| Activity | androidx.compose.ui.tooling.PreviewActivity |  true |  Désactiver |
| Activity | androidx.activity.ComponentActivity |  true |  Désactiver |
| Receiver | androidx.profileinstaller.ProfileInstallReceiver |  true |  Vérifier |
| Provider | androidx.startup.InitializationProvider | false | OK |

### Annexe C : Endpoints identifiés
| Endpoint | Protocole | Méthode | Risque |
|----------|-----------|---------|--------|
| http://10.0.2.2/projet/ws/loadetudiant.php | HTTP | GET |  Critique |
| http://10.0.2.2/projet/ws/createetudiant.php | HTTP | POST |  Critique |
| http://10.0.2.2/projet/ws/updateetudiant.php | HTTP | PUT/POST |  Critique |
| http://10.0.2.2/projet/ws/deleteetudiant.php | HTTP | DELETE/POST |  Critique |

**Note :** 10.0.2.2 est l'adresse de l'émulateur Android pour localhost

### Annexe D : Informations de signature
| Propriété | Valeur |
|-----------|--------|
| Type de certificat | DEBUG CERTIFICATE |
| Sujet | CN=Android Debug, O=Android, C=US |
| Empreinte SHA-256 | 03395040de7235b8992076a467fea3ffadf32268d276004a43f99c37f68151fb |

---
**Rapport généré le :** 17 mars 2026
**Analyste :** Mohamed DOUASSI
**Environnement :** Mobexler VM avec MobSF v4.0.6
