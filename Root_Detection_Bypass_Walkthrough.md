# Root Detection Bypass Lab — Walkthrough

> **Avertissement éthique** : N'appliquez ce lab que sur des appareils/apps vous appartenant ou dans un cadre d'audit autorisé.

---

## Environnement

| Élément | Valeur |
|--------|--------|
| PC Hôte | Windows 11 |
| Émulateur | Genymotion Android 11 (x86_64) |
| Frida | v17.9.1 |
| Objection | v1.12.4 |
| Medusa | v30.7 (dev) |
| App cible | com.example.rootdetectiontest |

---

## Objectif

Bypasser 4 méthodes de détection root via Frida, Objection et Medusa.

---

## App de test développée pour le lab

App Android custom développée avec Android Studio implémentant 4 checks root :

- **Build.TAGS** — vérifie si `test-keys` est présent
- **File.exists** — cherche les binaires su/busybox
- **Runtime.exec** — exécute `which su`
- **RootBeer** — bibliothèque tierce de détection root

Affichage coloré avec cartes, icônes et badge global (✅ PROPRE / ⚠️ ROOT DÉTECTÉ).

---

## État initial (sans bypass)

```
 ROOT DÉTECTÉ — 4/4 checks
🔴 Build.TAGS: test-keys
🔴 File.exists: /system/bin/su, /system/xbin/su, /sbin/su
🔴 Runtime.exec: su trouvé via which
🔴 RootBeer.isRooted: Root détecté
```

<img width="215" height="429" alt="image" src="https://github.com/user-attachments/assets/38109c63-4987-457c-8ccd-cc79b5faf35e" />

---

<img width="1600" height="838" alt="image" src="https://github.com/user-attachments/assets/96d8b1e1-ec09-4493-95a8-ba4a4b266ac3" />

---

## Méthode 1 — Frida (scripts JS manuels)

### Script bypass_root_basic.js

Hooks Java ciblés :
- `Build.TAGS.value = 'release-keys'`
- `File.exists()` → false pour chemins suspects
- `Runtime.exec()` → redirige vers `false` (exit code 1)
- `RootBeer.isRooted()` → false

```bash
frida -U -f com.example.rootdetectiontest -l bypass_root_basic.js
```

---

<img width="1600" height="780" alt="image" src="https://github.com/user-attachments/assets/2d4c1deb-e591-48b4-a77c-79615f5c7dc5" />

---

<img width="1600" height="849" alt="image" src="https://github.com/user-attachments/assets/520bd042-b484-4889-827b-63db88a0f5ce" />

---

<img width="1600" height="844" alt="image" src="https://github.com/user-attachments/assets/d2ce4a91-5d8a-4763-87a8-03badb3c6abd" />

---


### Script bypass_native.js (checks natifs libc)

Hooks natifs pour apps avec pinning natif :
- `open`, `openat`, `access`, `stat`, `lstat`

```bash
frida -U -f com.example.rootdetectiontest -l bypass_root_basic.js -l bypass_native.js
```

> Sur Genymotion, les symboles libc sont non interceptables (inline/optimisés) — comportement normal.

```
[*] open non interceptable sur cet appareil
[*] access non interceptable sur cet appareil
[*] stat non interceptable sur cet appareil
```

---

<img width="1600" height="805" alt="image" src="https://github.com/user-attachments/assets/33a4bbc9-cb9e-4501-9e5e-edf005b6cef8" />

---

<img width="1600" height="800" alt="image" src="https://github.com/user-attachments/assets/11956651-5e9e-4d8f-ab37-dd04067c2fb0" />

---

<img width="1600" height="839" alt="image" src="https://github.com/user-attachments/assets/9a20099d-784f-4c7d-9561-1f6bb4d4b70b" />

---

### frida-trace (diagnostic natif)

```bash
frida-trace -U -f com.example.rootdetectiontest -i open -i access -i stat -i openat
```

Permet de voir tous les appels natifs effectués par l'app au démarrage.


<img width="1600" height="807" alt="image" src="https://github.com/user-attachments/assets/851ca949-932e-4abd-b00e-e27692880093" />

---

## Méthode 2 — Objection
### Bypass complet (2 commandes)

```bash
# Lancement
objection -g com.example.rootdetectiontest explore --startup-command "android root disable"

# Dans la console Objection
android hooking set return_value java.lang.Runtime.exec false
```

Résultat : **4/4 ✅**

| Commande Objection | Ce qu'elle bypasse |
|--------------------|--------------------|
| `android root disable` | Build.TAGS + File.exists + RootBeer |
| `android hooking set return_value java.lang.Runtime.exec false` | Runtime.exec |


<img width="1600" height="839" alt="image" src="https://github.com/user-attachments/assets/4ad7a176-cf96-478e-b68c-09a7a3e93d28" />

---

<img width="1600" height="805" alt="image" src="https://github.com/user-attachments/assets/b1f3e226-6819-4517-8cb9-f560e3d5479d" />

---

## Méthode 3 — Medusa

### Lancement correct

```bash
cd C:\platform-tools
set PATH=%PATH%;C:\platform-tools
py medusa\medusa.py -p com.example.rootdetectiontest -d 192.168.56.102:5555
```

### Modules nécessaires

```bash
use root_detection/universal_root_detection_bypass
use root_detection/rootbeer_detection_bypass_no_obfuscation
run -f com.example.rootdetectiontest
```


| Module Medusa | Ce qu'il bypasse |
|---------------|-----------------|
| `universal_root_detection_bypass` | Build.TAGS + File.exists + Runtime.exec |
| `rootbeer_detection_bypass_no_obfuscation` | RootBeer.isRooted() |


---

<img width="1600" height="823" alt="image" src="https://github.com/user-attachments/assets/35c99654-5595-4969-8fbd-06ae04c11abc" />

---

<img width="1242" height="356" alt="image" src="https://github.com/user-attachments/assets/2264003a-702f-48cd-9d99-7aee091d8be6" />

---

<img width="1600" height="852" alt="image" src="https://github.com/user-attachments/assets/928deb99-64f9-483c-8de6-46b1e14efb96" />

---
## Méthode 4 — Magisk

| Fonctionnalité | Status sur Genymotion |
|---------------|----------------------|
| Magisk App v30.7 |  installé |
| Zygisk | ❌ non supporté |
| DenyList | ❌ nécessite Zygisk |
| Modules | ❌ nécessite boot patché |

> Magisk nécessite un appareil physique rooté pour fonctionner pleinement. Sur émulateur, Frida/Objection/Medusa sont plus adaptés.


<img width="497" height="1012" alt="image" src="https://github.com/user-attachments/assets/a915bf90-4edd-42f4-9149-e905ef7c989a" />



---

## Checklist finale

- [x] App custom avec 4 checks root développée
- [x] Bypass Java complet avec Frida (bypass_root_basic.js)
- [x] Script natif libc créé (bypass_native.js)
- [x] frida-trace diagnostic exécuté
- [x] Bypass complet avec Objection (android root disable + hooking)
- [x] Bypass complet avec Medusa (2 modules combinés)
- [x] Magisk inspecté (limites émulateur documentées)

## Auteur
**H-oubane**

