# Débrider une trottinette Todimart/Vicont 039ZQ (25 → ~31 km/h) par le diamètre de roue

> **English abstract** — The 25 km/h cap on Todimart/Vicont `039ZQ` controllers is enforced by
> firmware and cannot be raised: the `0x3C` "max speed" command is silently clamped to the
> ceiling the dashboard reports (25). The workaround documented here does not touch the limit at
> all: it under-declares the wheel diameter (EEPROM parameter **P14**, in tenths of an inch) so
> that the controller computes a lower speed than reality. The motor then spins faster while
> still believing it is at 25 km/h. Measured on hardware: 625 rpm → 781 rpm, i.e. **+25 % real
> speed**, with the speedometer still reading 25.

---

## ⚠️ Avertissements

- Cette procédure **fausse volontairement la métrologie** de l'engin. Après application :
  - le **compteur de vitesse ment** (il affiche 25 quand vous roulez à ~31),
  - l'**odomètre et l'autonomie restante sont sous-estimés d'environ 19 %**.
- Un engin ainsi modifié **n'est plus conforme à son homologation** (25 km/h, catégorie
  EDPM en France). Assurance, garantie et responsabilité en cas d'accident sont impactées.
- La distance de freinage, la tenue de route et la charge batterie/moteur ne sont plus
  couvertes par le constructeur (36 V, 350 W, 15 A).
- Toutes les écritures décrites sont **réversibles** et se font en EEPROM, sans flash.
  Aucune n'écrit dans le firmware du contrôleur.
- Les valeurs sont données pour **un modèle précis**. Sur un autre modèle, commencez par
  mesurer (voir §4) au lieu de recopier.

---

## 1. Matériel concerné

| Élément | Valeur relevée |
|---|---|
| Marque (cloud) | Todimart (`todimart.com`) |
| Modèle | `039ZQ-34` |
| Nom libre (cloud) | `YY-XMPRO` (clone Xiaomi Pro) |
| Moteur | `YY-039ZQ34-D36V6G17A-S-6528` |
| Batterie | 36 V, 350 W, 15 A max |
| Tableau de bord (AD) | hardware **104**, firmware **58**, board **0528** |
| Contrôleur moteur (AE) | hardware **10**, firmware **50**, board **0000** |
| BLE | nom `vA-<MAC>`, service `FEE0`, écriture/notify `FEE2` |
| Application d'origine | Vicont `com.vicontout.benben` |

---

## 2. Pourquoi le réglage de vitesse de l'app ne marche pas

L'application Vicont envoie la commande `0x3C` (« vitesse maximum », 1..99, `99` = *不限速*
« sans limite »). La trame est correcte — vérifié dans le journal BLE de l'app :

```
FA AF A5 5A 3C 01 63 FA      →  code 0x3C, valeur 0x63 = 99, checksum valide
```

Le contrôleur **écrête silencieusement** et répond la valeur réellement appliquée accompagnée
de son plafond :

```
5A 3C 02 <valeur> <plafond> <checksum> <status>
         ↑ appliqué  ↑ plafond firmware
```

L'app teste `valeur == 0` pour valider ; recevant `25`, elle affiche « 设置失败 » et **remet
l'ancienne valeur**. C'est exactement le clignotement observé : ce n'est pas l'app qui bride,
c'est le firmware qui refuse.

**L'écriture directe en EEPROM le confirme** : en écrivant P10 = 20, la trottinette roule
réellement à 20 km/h. En écrivant P10 = 30 ou 35, elle revient à 25. Le plafond de 25 est donc
une valeur dure du firmware, pas un défaut de configuration.

Aucun firmware n'est par ailleurs publié pour ce modèle : `/api/upgrade/last/info` répond
`升级类型信息不存在` pour les deux types (`AD` afficheur, `AE` contrôleur), et aucun binaire
n'existe dans l'APK.

---

## 3. Le contournement : sous-déclarer le diamètre de roue

Le contrôleur calcule la vitesse affichée **et** le point de limitation à partir du régime
moteur et d'un **diamètre de roue déclaré en EEPROM** :

```
vitesse_kmh = régime_tr_min × π × D_mètres × 60 / 1000
```

Si on déclare une roue **plus petite** que la vraie, alors pour un même régime le contrôleur
calcule une vitesse plus faible. Il autorise donc un régime plus élevé avant d'atteindre sa
limite de 25 — mais la roue, elle, tourne vraiment plus vite.

Le paramètre est **P14** (« Wheel diameter »), en **dixièmes de pouce** :

```
D_mètres = P14 × 0.00254        (P14 = 83  →  8,3 pouces  →  210,8 mm)
```

Formule d'ajustement :

```
P14_nouveau = P14_origine × (vitesse_affichée / vitesse_réelle_voulue)
```

Pour passer de 25 à 31 km/h : `83 × 25/31 ≈ 67`.

---

## 4. Mesurer le diamètre déclaré sans le lire

Les paramètres d'EEPROM **ne sont pas lisibles** par le protocole sur ce firmware : la commande
`0xC9` (« afficher le paramètre sur l'écran du tableau de bord ») n'est pas implémentée, et il
n'existe aucune trame de lecture. La seule méthode fiable est de **déduire** la valeur d'une
mesure physique : régime moteur et vitesse affichée sont émis en permanence dans la trame de
télémétrie `0x10`.

Protocole de mesure :

1. Roue arrière **levée** (l'écrêtage s'applique aussi à vide, ce qui rend la mesure reproductible).
2. Plein gaz, **régime stabilisé** 15-20 s.
3. Relever simultanément le **régime** (tr/min) et la **vitesse affichée**.

Le régime est la grandeur honnête : il ne peut pas être écrêté comme le compteur. Calcul :

```
D_mètres = vitesse_affichée × 1000 / (60 × π × régime)
P14      = D_mètres / 0.00254
```

Exemple réel (valeurs d'usine) : 24,9 km/h à 625 tr/min → D = 211,4 mm → **P14 = 83**.

---

## 5. Procédure complète

### Matériel / logiciel

L'application d'origine **ne sait pas** écrire les paramètres P. Il faut un client qui expose la
commande `0xD2` (écriture de paramètre EEPROM). Utilisé ici :

- **[AovoControl](https://github.com/YouRooni/AovoControl)** (open source, Android, support ViCont)
- activer **Settings → Expert mode**, puis **Settings → Engineering menu**

### Étapes

1. **Relever les valeurs d'origine** (à noter avant toute écriture) :
   - menu d'ingénierie → **P14** (affiche `10` par défaut : le logiciel ne lit rien, c'est une
     valeur de remplissage, ne pas la confondre avec la valeur réelle) ;
   - **P10** (limite Sport) — lisible, c'est la valeur renvoyée par `0x3C` ;
   - **P15** (source tachymètre) ;
   - couple de démarrage : écran *Controller parameters* → *Throttle response* (1..10).
2. **Mesurer le diamètre réel** comme au §4 (roue levée, plein gaz, régime + vitesse).
3. **Calculer** `P14_nouveau = P14_origine × 25 / vitesse_voulue`.
4. **Écrire** : menu d'ingénierie → P14 → curseur → *Apply*.
   Trame émise : `FA AF A5 5A D2 02 0D <valeur> <somme>`
5. **Vérifier** : refaire la mesure du §4. Le régime doit **augmenter** dans le rapport prévu ;
   la vitesse affichée doit rester **25**. C'est la signature du succès.
6. Contrôle final **en roulant**, sur route fermée, avec un GPS : la vitesse réelle doit
   correspondre au calcul.

### Valeurs mesurées sur cet engin (référence)

| P14 | Régime max (roue levée) | Vitesse réelle |
|---|---|---|
| **83** (usine) | 625 tr/min | **24,9 km/h** |
| 80 | 660 tr/min | 26,3 km/h |
| **67** (retenu) | 768–781 tr/min | **≈31 km/h** |

---

## 6. Couple (optionnel, indépendant de la vitesse)

Les deux réglages de couple ne sont pas soumis à l'écrêtage de vitesse. Commandes directes
`0x3D` (couple de démarrage) et `0x3E` (couple max en roulage), valeurs brutes **0..200**
(200 = 100 %) :

```
FA AF A5 5A 3D 01 C8 60     → couple de démarrage = 200/200
FA AF A5 5A 3E 01 C8 61     → couple de roulage    = 200/200
```

Vérification de `0x3D` : écran *Controller parameters* → *Throttle response* passe de `8` à `10`
(échelle 1..10 reconstruite depuis 0..200). `0x3E` n'a pas d'affichage dédié sur cet écran.

Effet : accélération plus franche, courant d'appel plus élevé, usure et échauffement accrus.

---

## 7. Retour à l'état d'usine

| Paramètre | Valeur d'origine |
|---|---|
| **P14** (diamètre) | **83** |
| **0x3D** (couple démarrage) | **160** (0xA0, soit 8/10) |
| **P15** (source tachymètre) | **0** (défaut documenté) |
| P10 (limite) | 25 (inchangée par la procédure) |

Trames correspondantes :

```
FA AF A5 5A D2 02 0D 53 <somme>     → P14 = 83
FA AF A5 5A 3D 01 A0 <somme>        → couple démarrage = 160
FA AF A5 5A D2 02 0E 00 <somme>     → P15 = 0
```

Après remise à 83, le compteur, l'odomètre et la limite retrouvent leur cohérence d'origine.

---

## 8. Référence du protocole (relevée sur l'engin)

### Trame

```
FA AF A5 | zt(0x5A) | code | len | data… | checksum
```

`checksum = (zt + code + len + Σdata) & 0xFF`

Les réponses **n'ont pas** le préambule :

```
zt | code | len | payload… | checksum | status
```

### Commandes utiles

| Code | Sens | Payload |
|---|---|---|
| `0x01` | démarrer la télémétrie | — |
| `0x10` | **RX** télémétrie | vitesse `u16/10` km/h, rpm `u16`, tension `u16/100`, courant |
| `0x11` | **RX** état | rapport, éclairage, défauts |
| `0x12` | **RX** versions | hw/sw afficheur et contrôleur |
| `0x3C` | vitesse max | 1..99 (**écrêté à 25**) · réponse `[valeur, plafond]` |
| `0x3D` | couple de démarrage | 0..200 |
| `0x3E` | couple / courant max | 0..200 |
| `0x3F` | frein électromagnétique | 0..200 |
| `0x42` | rapport | 1..5 |
| `0xAB` | versions | `AD` = afficheur, `AE` = contrôleur |
| `0xC9` | afficher un paramètre à l'écran | index (**non implémenté ici**) |
| `0xD2` | **écrire un paramètre EEPROM** | `index, valeur` |

`index = numéro_P - 1` (P10 → 9, P14 → 13, P15 → 14).

### Paramètres P utiles

| P | Index | Rôle | Plage | Unité |
|---|---|---|---|---|
| P8 | 7 | limite Eco | 1..99 | km/h |
| P9 | 8 | limite Drive | 1..99 | km/h |
| **P10** | **9** | **limite Sport** (= `0x3C`) | 1..99 | km/h |
| P11 | 10 | limite rapport 4 | 1..99 | km/h |
| P12 | 11 | limite rapport 5 | 1..99 | km/h |
| P13 | 12 | paires de pôles | 1..99 | — |
| **P14** | **13** | **diamètre de roue** | 1..99 | 1/10 pouce |
| P15 | 14 | source du tachymètre | 0..9 | — |

---

## 9. Pistes écartées (pour ne pas les refaire)

| Piste | Résultat |
|---|---|
| P10 > 25 par `0x3C` | écrêté à 25 par le firmware |
| P10 > 25 par `0xD2` (écriture directe) | écrêté à 25 également |
| Firmware OTA Vicont (`AD`/`AE`) | `升级类型信息不存在` — rien de publié |
| Firmware dans l'APK | aucun `.bin`, `demo1.8` n'est qu'un nom de fichier par défaut |
| Firmware public (GitHub, scooterhacking) | inexistant pour `039ZQ` / board `0528` |
| Firmware Xiaomi (M365, Pro, 1S…) | matériel et protocole différents — incompatible, brick |
| SHU / DownG | parlent `55 AA` / `5A A5`, services `FFE0`/`FFE1` → « appareil inconnu » |
| P15 = 6 (« vitesse réelle ») | sans effet : la même constante de diamètre est utilisée partout |
| P13 (paires de pôles) | même famille d'effet que P14, non testé sur cet engin |

---

## 10. Pourquoi l'affichage ne peut pas être corrigé

Tous les calculs — vitesse affichée, odomètre, point de limitation — dérivent du **même**
diamètre déclaré. Il n'existe pas de constante « vraie » séparée. Dès que P14 ment, tout ment
de façon cohérente. Le compromis est donc **structurel**, pas un réglage oublié :

- vitesse réelle maximale ; ou
- affichage et odomètre justes.

---

## Crédits

- **[YouRooni/AovoControl](https://github.com/YouRooni/AovoControl)** — client open source qui
  documente le menu d'ingénierie ViCont et la commande `0xD2`, sans lequel ce contournement
  aurait été difficile à identifier. Les descriptions des P-paramètres citées ici viennent de
  son code (`EngineeringParams.kt`).
- La cartographie `0x3C`/`0x3D`/`0x3E`/`0x3F` et le format des trames ont été confirmés par
  rétro-ingénierie du bundle uni-app de l'app Vicont `com.vicontout.benben` v2.0.4.
