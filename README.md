# LAB-14-Contournement-de-la-d-tection-Root-Android-avec-Frida-Objection-et-Hooks-natifs
Ce lab va un cran plus loin : on ne se contente plus d'un bypass Java ou d'une commande Objection. On combine les deux, on ajoute des hooks natifs, et on trace les appels système pour comprendre ce que l'application fait vraiment sous le capot.
L'application reste la même — OWASP UnCrackable Level 1 — mais cette fois on l'attaque méthodiquement, couche par couche, pour voir exactement où ça bloque et comment le débloquer.
ÉlémentDétailSystèmeWindows PowerShellÉmulateurAndroid Emulator 5554Frida17.8.0Objection1.12.4Android11Package cibleowasp.mstg.uncrackable1  .

Structure du projet
LAB14-Bypass-Root-Frida/
│
├── README.md
├── hello.js               ← juste pour vérifier que l'injection marche
├── bypass_root_basic.js   ← neutralise les checks Java
├── bypass_native.js       ← hookle les fonctions système en C

Étape 1 — Est-ce que Frida arrive à s'injecter ?
Avant d'écrire quoi que ce soit de compliqué, on vérifie que la plomberie fonctionne. Un script minimal suffit :
bashfrida -U -f owasp.mstg.uncrackable1 -l hello.js
Si frida-server tourne, qu'ADB est connecté et que les versions sont alignées, le terminal affiche ça :
Connected to Android Emulator 5554
Spawned `owasp.mstg.uncrackable1`. Resuming main thread!
[+] Script injecté: Java.perform OK
<img width="1232" height="406" alt="Test injection Frida avec hello.js" src="https://github.com/user-attachments/assets/dbacaa53-12bb-4e63-9631-7b5a93fdbb51" />
Parfait. Frida est bien dans le processus. On peut passer à la suite.

Étape 2 — Ce qu'on voit avant de toucher à quoi que ce soit
Sans intervention, l'application se ferme toute seule dès le lancement avec ce message :
Root detected!
This is unacceptable. The app is now going to exit.
<img width="398" height="821" alt="Root détecté - application bloquée" src="https://github.com/user-attachments/assets/178acd47-8c49-4879-aac9-6ef72dbec20c" />
C'est le point de départ. L'objectif du lab est de faire disparaître ce message — et de comprendre pourquoi il apparaît.

Étape 3 — Premier bypass : neutraliser les vérifications Java
La majorité des apps Android détectent le root en Java : elles regardent la valeur de Build.TAGS, cherchent des fichiers comme /system/bin/su ou Superuser.apk, ou lancent des commandes via Runtime.exec. C'est la couche la plus courante, et c'est ce que bypass_root_basic.js s'occupe de neutraliser.
bashfrida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js
Le script confirme chaque hook au fur et à mesure :
[+] Build.TAGS -> release-keys
[*] RootBeer non présent
[+] Runtime.exec hooks installés
[+] Bypass Java installé
[+] File.exists bypass: /system/bin/su
[+] File.exists bypass: /system/xbin/su
[+] File.exists bypass: /system/app/Superuser.apk
<img width="1368" height="466" alt="Bypass Java actif" src="https://github.com/user-attachments/assets/27471f65-aad4-46ba-aff4-b6fe223a1ea7" />
Et côté application — elle se lance. Plus de popup, plus de fermeture forcée.
<img width="403" height="828" alt="Application accessible après bypass Java" src="https://github.com/user-attachments/assets/dedac173-bb0c-46d2-af2c-e5d6b3bb93c7" />

Étape 4 — Quand Java ne suffit pas : les hooks natifs
Certaines applications ne font pas confiance aux APIs Java pour leurs vérifications de sécurité. Elles passent directement par le code natif — C ou C++ via le NDK — et appellent des fonctions système comme open, stat, access ou readlink pour chercher des traces de root dans le système de fichiers. Aucun hook Java ne les atteint.
C'est pour ça qu'on ajoute un deuxième script :
bashfrida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js -l bypass_native.js
Cette fois, Frida intercepte aussi les fonctions au niveau du binaire :
[+] Hooked native open
[+] Hooked native openat
[+] Hooked native access
[+] Hooked native stat
[+] Hooked native lstat
[+] Hooked native fopen
[+] Hooked native readlink
[+] Native root bypass installed
<img width="1476" height="686" alt="Bypass Java + natif combinés" src="https://github.com/user-attachments/assets/fa906311-0596-4b7d-9069-a3127270fea8" />
Avec les deux scripts actifs en même temps, on couvre à la fois la couche Java et la couche native. C'est le bypass le plus complet qu'on puisse faire avec cette approche.

Étape 5 — La version rapide avec Objection
Pour voir la différence avec ce qu'on a fait à la main, on teste Objection :
bashobjection -g owasp.mstg.uncrackable1 explore --startup-command "android root disable"
Running a startup command... android root disable
(agent) Registering job. Name: root-detection-disable
owasp.mstg.uncrackable1 on Android: 11 [usb]
<img width="1465" height="441" alt="Bypass via Objection" src="https://github.com/user-attachments/assets/4e314f01-9b08-4a3a-ac19-57b9b535ea09" />
Une commande, c'est tout. Objection gère les cas standards sans avoir à écrire une seule ligne de JavaScript. C'est pratique pour tester rapidement — mais dès qu'une application sort des sentiers battus avec des checks custom, les scripts manuels redeviennent nécessaires.

Étape 6 — Observer avant d'agir avec frida-trace
Dernière technique du lab, et probablement la plus utile en pratique : avant d'écrire un bypass, on peut simplement observer ce que l'application appelle en temps réel.
bashfrida-trace -U -i open -i access -i stat -i openat -i fopen -i readlink Uncrackable1
Instrumenting...
open: Auto-generated handler
access: Auto-generated handler
stat: Auto-generated handler
openat: Auto-generated handler
fopen: Auto-generated handler
readlink: Auto-generated handler
Started tracing 6 functions.
<img width="1458" height="244" alt="Trace des appels natifs" src="https://github.com/user-attachments/assets/25ce916a-1242-433d-acd8-1e36191ea2a5" />
C'est comme mettre une caméra sur le processus. On voit exactement quels fichiers l'application essaie d'ouvrir, quelles commandes elle teste, quels chemins elle inspecte. À partir de là, écrire un bypass ciblé devient beaucoup plus simple — parce qu'on sait précisément ce qu'on doit bloquer.
