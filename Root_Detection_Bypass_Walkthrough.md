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
⚠️ ROOT DÉTECTÉ — 4/4 checks
🔴 Build.TAGS: test-keys
🔴 File.exists: /system/bin/su, /system/xbin/su, /sbin/su
🔴 Runtime.exec: su trouvé via which
🔴 RootBeer.isRooted: Root détecté
```

> 📸 **Capture 1** — App affichant 4/4 ROOT DÉTECTÉ
> `[insérer screenshot ici]`

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

> 📸 **Capture 2** — Terminal Frida avec hooks confirmés + app 4/4 ✅
> `[insérer screenshot ici]`

### Découverte clé — Runtime.exec

`echo` retourne exit code 0 → app croit que `su` existe.
`false` retourne exit code 1 → app croit que `su` est introuvable.

```javascript
// ❌ Mauvais
return this.exec('echo');

// ✅ Correct
return this.exec('false');
```

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

> 📸 **Capture 3** — Terminal bypass combiné Java + Natif
> `[insérer screenshot ici]`

### frida-trace (diagnostic natif)

```bash
frida-trace -U -f com.example.rootdetectiontest -i open -i access -i stat -i openat
```

Permet de voir tous les appels natifs effectués par l'app au démarrage.

> 📸 **Capture 4** — frida-trace montrant les appels open/access/stat
> `[insérer screenshot ici]`

---

## Méthode 2 — Objection

### Commande simple

```bash
objection -g com.example.rootdetectiontest explore --startup-command "android root disable"
```

Résultat : **3/4** — `android root disable` ne couvre pas `Runtime.exec`.

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

> 📸 **Capture 5** — Console Objection + app ✅ APPAREIL PROPRE
> `[insérer screenshot ici]`

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

> ⚠️ `run` seul ne fonctionne pas — il faut `run -f <package>`

| Module Medusa | Ce qu'il bypasse |
|---------------|-----------------|
| `universal_root_detection_bypass` | Build.TAGS + File.exists + Runtime.exec |
| `rootbeer_detection_bypass_no_obfuscation` | RootBeer.isRooted() |

> 📸 **Capture 6** — Console Medusa + app 4/4 ✅
> `[insérer screenshot ici]`

---

## Méthode 4 — Magisk

| Fonctionnalité | Status sur Genymotion |
|---------------|----------------------|
| Magisk App v30.7 | ✅ installé |
| Zygisk | ❌ non supporté |
| DenyList | ❌ nécessite Zygisk |
| Modules | ❌ nécessite boot patché |

> Magisk nécessite un appareil physique rooté pour fonctionner pleinement. Sur émulateur, Frida/Objection/Medusa sont plus adaptés.

> 📸 **Capture 7** — Interface Magisk sur Genymotion
> `[insérer screenshot ici]`

---

## Comparaison des méthodes

| Check | Frida | Objection | Medusa |
|-------|-------|-----------|--------|
| Build.TAGS | ✅ | ✅ | ✅ |
| File.exists | ✅ | ✅ | ✅ |
| Runtime.exec | ✅ | ✅ (hook manuel) | ✅ |
| RootBeer | ✅ | ✅ | ✅ (module dédié) |
| **Total** | **4/4** | **4/4** | **4/4** |

---

## Checklist finale

- [x] App custom avec 4 checks root développée
- [x] Bypass Java complet avec Frida (bypass_root_basic.js)
- [x] Script natif libc créé (bypass_native.js)
- [x] frida-trace diagnostic exécuté
- [x] Bypass complet avec Objection (android root disable + hooking)
- [x] Bypass complet avec Medusa (2 modules combinés)
- [x] Magisk inspecté (limites émulateur documentées)
