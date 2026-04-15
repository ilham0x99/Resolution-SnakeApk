# Resolution-Snake.apk

# Rapport de Laboratoire : Challenge Snake (PwnSec CTF)
## 1. Objectif du Challenge
L'objectif est d'exploiter une vulnérabilité de désérialisation non sécurisée dans la bibliothèque **SnakeYAML** afin d'instancier une classe cachée nommée `BigBoss`. Cette classe déclenche l'exécution d'un code natif qui génère le flag dans les journaux système (logcat).

Le challenge est protégé par plusieurs mécanismes :
* Détection de Root.
* Détection d'Émulateur.
* Détection de Frida (via librairie native).

---

## 2. Analyse Statique

### 2.1 Identification des Protections
L'analyse du code décompilé via **Jadx-GUI** a permis de localiser les fonctions de sécurité empêchant l'exécution de l'application sur un environnement de test.

<img width="1348" height="738" alt="Pasted image (2)" src="https://github.com/user-attachments/assets/1b4183f2-7a6e-4229-8094-1d1b61e44e8f" />
<img width="1292" height="262" alt="Pasted image (3)" src="https://github.com/user-attachments/assets/85675bf7-c28e-44e8-90f8-5c0a06d9cc51" />



### 2.2 Analyse du Flux d'Exécution
L'application attend un Intent spécifique pour activer la lecture du fichier de configuration YAML.

* **Intent requis :** `SNAKE` avec la valeur `BigBoss`.
* **Fichier cible :** `/sdcard/Snake/Skull_Face.yml`.

<img width="1186" height="532" alt="Pasted image (5)" src="https://github.com/user-attachments/assets/df8c31ef-c8a6-4377-bc20-48b866b7f7ac" />


### 2.3 Classe Cible (BigBoss)
La classe `com.pwnsec.snake.BigBoss` charge une bibliothèque native et attend la chaîne de caractères `"Snaaaaaaaaaaaaaake"` pour valider l'appel JNI.

<img width="1185" height="361" alt="Pasted image (6)" src="https://github.com/user-attachments/assets/2cab1008-0ee9-4547-b59f-60f4abcf1fc8" />
<img width="938" height="652" alt="Pasted image (4)" src="https://github.com/user-attachments/assets/40246c6c-7f48-45a2-afe9-56ce9bda1581" />


---

## 3. Patching et Bypassing

### 3.1 Décompilation et Modification Smali
Pour contourner les protections, l'APK a été décompilé avec `apktool`. Les méthodes de détection ont été modifiées en Smali pour retourner systématiquement `false`.

<img width="1185" height="361" alt="Pasted image (6)" src="https://github.com/user-attachments/assets/ce3aa16c-784b-4ddd-bd5b-ccca52a1096f" />


**Exemple de modification (Bypass Root) :**
```smali
# Avant : Logique de vérification complexe
# Après :
.method public static isRooted()Z
    const/4 v0, 0x0
    return v0
.end method
```
<img width="1600" height="548" alt="Pasted image (7)" src="https://github.com/user-attachments/assets/f751f457-9ae8-4d30-ad44-2afaa9bd3f7d" />
<img width="1243" height="208" alt="Pasted image (8)" src="https://github.com/user-attachments/assets/818765c2-fea4-4696-a3ce-c05631138bcb" />

### 3.2 Reconstruction et Signature
Après modification, l'APK a été reconstruit et signé pour permettre son installation.

<img width="896" height="131" alt="Pasted image (12)" src="https://github.com/user-attachments/assets/00db3c39-3969-47ce-bc48-06462104597e" />
<img width="1387" height="596" alt="Pasted image (11)" src="https://github.com/user-attachments/assets/a0d1e6e6-a109-4365-b008-2218cd151233" />
<img width="1343" height="255" alt="Pasted image (9)" src="https://github.com/user-attachments/assets/040972ca-70c0-4478-a445-aca4f78b6e43" />

---

## 4. Exploitation (CVE-2022-1471)

### 4.1 Préparation du Payload
Le payload utilise le "Global Tag" de SnakeYAML pour forcer l'instanciation de la classe `BigBoss` avec le paramètre requis.

**Contenu de `Skull_Face.yml` :**
```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```
<img width="1461" height="92" alt="Pasted image (15)" src="https://github.com/user-attachments/assets/73780e97-61f1-46f2-bc2d-781bb17a909a" />

### 4.2 Mise en place sur l'appareil
Le fichier est poussé sur le stockage externe de l'émulateur.

```bash
adb shell mkdir -p /sdcard/Snake
adb push Skull_Face.yml /sdcard/Snake/Skull_Face.yml
```

<img width="1371" height="92" alt="Pasted image (16)" src="https://github.com/user-attachments/assets/34570557-43bd-41c2-a120-c71e773dd5cd" />
<img width="1461" height="92" alt="Pasted image (15)" src="https://github.com/user-attachments/assets/897c88d0-6c62-4c15-aeaa-efdc374093a7" />

---

## 5. Récupération du Flag

### 5.1 Exécution de l'Intent
L'application est lancée en forçant l'extra Intent `SNAKE`.

```bash
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

<img width="357" height="817" alt="Pasted image (14)" src="https://github.com/user-attachments/assets/1487fd93-837b-4e29-bef8-0173f82642e2" />


### 5.2 Capture du Flag (Logcat)
Le flag est extrait en filtrant les logs avec le tag `PWNSEC`.


---

## Conclusion
Ce lab a permis de démontrer l'importance de sécuriser les processus de désérialisation. Même avec des protections anti-reverse avancées (Root/Frida detection), un patching statique combiné à une vulnérabilité logique permet de compromettre l'application.
