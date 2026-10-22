# 📞 Grandstream HT813 × Orange Livebox × FreePBX

[![Hardware](https://img.shields.io/badge/Hardware-Grandstream%20HT813-blue.svg)](https://www.grandstream.com/)
[![PBX](https://img.shields.io/badge/PBX-FreePBX%20%2F%20Asterisk-orange.svg)](https://www.freepbx.org/)
[![ISP](https://img.shields.io/badge/Op%C3%A9rateur-Orange%20Livebox%20(France)-ff7900.svg)](https://www.orange.fr/)
[![SIP](https://img.shields.io/badge/Protocole-PJSIP%20%2F%20SIP-green.svg)](#)
[![License](https://img.shields.io/badge/Licence-MIT-lightgrey.svg)](LICENSE)

Guide complet et paramètres validés pour intégrer la ligne téléphonique analogique d'une **Orange Livebox** (port TEL) dans **FreePBX** via la passerelle hybride **Grandstream HT813** (FXO/FXS), avec gestion des tonalités françaises, de la déconnexion automatique d'appel et de la présentation du numéro (*Caller ID*).

---

## 📑 Sommaire

1. [Vue d'ensemble et Topologie](#-vue-densemble-et-topologie)
2. [Câblage physique](#-c%C3%A2blage-physique)
3. [Référentiel des adresses et ports](#-r%C3%A9f%C3%A9rentiel-des-adresses-et-ports)
4. [Configuration du Grandstream HT813](#-configuration-du-grandstream-ht813)
   - [1. Paramètres de base (Basic Settings)](#1-param%C3%A8tres-de-base-basic-settings)
   - [2. Paramètres avancés & Tonalités françaises (Advanced Settings)](#2-param%C3%A8tres-avanc%C3%A9s--tonalit%C3%A9s-fran%C3%A7aises-advanced-settings)
   - [3. Port FXO - Compte SIP & Réseau (FXO Port)](#3-port-fxo---compte-sip--r%C3%A9seau-fxo-port)
   - [4. Port FXO - Caller ID, Terminaison & Ligne Livebox](#4-port-fxo---caller-id-terminaison--ligne-livebox)
   - [5. Port FXS - Poste analogique local (FXS Port)](#5-port-fxs---poste-analogique-local-fxs-port)
5. [Configuration de FreePBX](#-configuration-de-freepbx)
   - [1. Création du Trunk PJSIP (Port FXO)](#1-cr%C3%A9ation-du-trunk-pjsip-port-fxo)
   - [2. Route entrante (Inbound Route)](#2-route-entrante-inbound-route)
     - [Bonus : Filtrage anti-démarchage (ARCEP)](#-astuce-bonus--filtrage-anti-d%C3%A9marchage-t%C3%A9l%C3%A9phonique-pr%C3%A9fixes-arcep)
   - [3. Route sortante standard (Outbound Route)](#3-route-sortante-outbound-route)
   - [4. Route sortante d'urgence (Emergency Route)](#4-route-sortante-durgence-emergency-route)
   - [5. Extension PJSIP (Poste FXS "Salon")](#5-extension-pjsip-poste-fxs-salon)
6. [Résolution des problèmes fréquents (Troubleshooting)](#-r%C3%A9solution-des-probl%C3%A8mes-fr%C3%A9quents-troubleshooting)
7. [Fichiers de provisionnement XML](#-fichiers-de-provisionnement-xml)

---

## 🔭 Vue d'ensemble et Topologie

Le Grandstream **HT813** est une passerelle VoIP hybride équipée de :
- **1 port FXO (LINE)** : se branche sur la sortie téléphonique analogique de la Livebox pour injecter la ligne fixe Orange dans FreePBX (appels entrants et sortants).
- **1 port FXS (PHONE)** : permet de brancher un téléphone analogique filaire ou une base DECT classique pour l'utiliser comme poste interne FreePBX (ex: extension 4000 "Salon").

```mermaid
flowchart LR
    subgraph RTC / Internet
        Orange[Réseau Orange]
    end

    subgraph Domicile / Bureau
        LB[Orange Livebox<br/>Port TEL RJ11]
        HT813[Grandstream HT813]
        FPBX[Serveur FreePBX<br/>10.10.10.5]
        PhoneFXS[Téléphone analogique<br/>Poste 4000 Salon]
        IPPhone[Téléphones IP / Softphones]
    end

    Orange <--> LB
    LB -- Câble RJ11 -->|Port FXO / LINE| HT813
    HT813 -- Câble RJ11 -->|Port FXS / PHONE| PhoneFXS
    HT813 <== Réseau LAN (SIP PJSIP) ==> FPBX
    FPBX <== Réseau LAN ==> IPPhone
```

---

## 🔌 Câblage physique

| Port HT813 | Type | Connecté à | Description |
| :--- | :--- | :--- | :--- |
| **LINE (FXO)** | RJ11 | Port **TEL** de la Livebox Orange | Capture la ligne analogique entrante/sortante d'Orange. |
| **PHONE (FXS)** | RJ11 | Téléphone analogique classique | Poste interne (optionnel, ex: téléphone du salon). |
| **WAN** | RJ45 | Switch / Réseau local (LAN) | Connexion au réseau IP où se trouve FreePBX. |
| **LAN** | RJ45 | Non connecté ou PC d'administration | Port secondaire (mode pont ou routeur désactivé). |

> [!IMPORTANT]
> Ne branchez **JAMAIS** le port **PHONE (FXS)** sur la Livebox ! Seul le port **LINE (FXO)** supporte la tension de ligne analogique délivrée par la Livebox.

---

## 📋 Référentiel des adresses et ports

Ce tableau reprend les valeurs configurées dans ce projet (à adapter selon votre plan d'adressage IP) :

| Élément | Valeur dans cet exemple | Remarque |
| :--- | :--- | :--- |
| **IP FreePBX** | `10.10.10.5` | Serveur PBX |
| **IP HT813 (WAN)** | `10.10.10.6` (ou DHCP fixe) | Interface web et point de contact SIP |
| **Port SIP FXS (Account 1)** | `5060` | Poste téléphonique analogique interne |
| **Port SIP FXO (Account 2)** | `5062` | Trunk de la ligne Livebox |
| **Port SIP FreePBX (PJSIP)** | `5060` | Port d'écoute PJSIP de FreePBX |
| **Numéro Livebox (DID)** | `04XXXXXXXX` | Votre numéro de ligne fixe Orange |
| **Extension FXS** | `4000` | Poste "Salon" |

---

## ⚙️ Configuration du Grandstream HT813

Connectez-vous à l'interface d'administration web du HT813 (`http://<IP_HT813>`, mot de passe par défaut : `admin`).

---

### 1. Paramètres de base (Basic Settings)

Dans l'onglet **Basic Settings**, configurez l'accès au boîtier, le réseau et le routage des appels entrants vers FreePBX :

#### Réseau & Accès d'administration
| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Web Access Mode** | `HTTP` | Accès web à l'interface |
| **HTTP Web Port** | `80` | Port d'accès web (défaut 80) |
| **Disable Telnet** | `Yes` | Désactive le protocole non sécurisé Telnet |
| **Internet Protocol** | `IPv4 Only` | Protocole réseau |
| **IPv4 Address** | `dynamically assigned via DHCP` | Adresse IP assignée par DHCP (recommandé : bail DHCP fixe sur votre routeur) |
| **Time Zone** | `GMT+01:00 (Paris, ...)` ou `customize` | Fuseau horaire métropolitain |
| **Allow DHCP server to set Time Zone** | `Yes` | Ajustement automatique via DHCP |

#### Routage direct Livebox vers FreePBX (Unconditional Call Forward to VOIP)
> [!IMPORTANT]
> Ce paramètre situé tout en bas de la page n'est pas utilisé puisque l'on passe par FreePBX.

---

### 2. Paramètres avancés & Tonalités françaises (Advanced Settings)

Dans l'onglet **Advanced Settings**, ajustez la sécurité, le comportement de secours et les tonalités téléphoniques :

#### Sécurité & Système
| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Disable User Level Web Access** | `Yes` | Désactive le compte utilisateur non-admin |
| **Disable Viewer Level Web Access** | `Yes` | Désactive le compte observateur |
| **Disable Direct IP Call** | `Yes` | **Sécurité critique :** bloque les appels SIP directs anonymes pour n'autoriser que FreePBX |
| **Disable Voice Prompt** | `Yes` | Désactive le serveur vocal interactif en anglais du boîtier |
| **Life Line Mode** | `Auto` | **Secours matériel :** relie physiquement le téléphone (FXS) à la Livebox (FXO) en cas de panne de courant |
| **NTP Server** | `pool.ntp.org` | Serveur de synchronisation horaire |
| **Allow DHCP Option 42 to override NTP server** | `Yes` | Permet la synchronisation NTP distribuée par le routeur |

#### Cadence de sonnerie & Tonalités officielles françaises (Call Progress Tones)

- **System Ring Cadence** : `c=1500/3500;`  
  *(Cadence de sonnerie officielle française : 1,5s sonnerie / 3,5s silence)*

Remplacez ensuite les tonalités américaines par défaut par les fréquences et cadences françaises :

| Tonalité | Syntaxe Grandstream | Description |
| :--- | :--- | :--- |
| **Dial Tone** | `f1=440@-10,f2=0@-10,c=0/0;` | 440 Hz continu |
| **Ringback Tone** | `f1=440@-10,f2=0@-10,c=1500/3500;` | 1,5s ON / 3,5s OFF |
| **Busy Tone** | `f1=440@-10,f2=0@-10,c=500/500;` | 500ms ON / 500ms OFF |
| **Reorder Tone** | `f1=440@-10,f2=0@-10,c=50/50;` | Tonalité d'occupation rapide |
| **Confirmation Tone** | `f1=350@-11,f2=440@-11,c=100/100-100/100-100/100;` | Bips d'acquittement |
| **Call Waiting Tone** | `f1=440@-13,c=300/10000;` | Bip d'appel en attente |
| **Prompt Tone** | `f1=350@-13,f2=440@-13,c=0/0;` | Tonalité d'invite vocale |

---

### 3. Port FXO - Compte SIP & Réseau (FXO Port)

Dans l'onglet **FXO Port**, configurez la connexion SIP vers FreePBX :

| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Account Active** | `Yes` | Active le port FXO |
| **Primary SIP Server** | `10.10.10.5` | Adresse IP de votre serveur FreePBX |
| **Prefer Primary SIP Server** | `Yes` | Force l'utilisation du serveur principal |
| **Outbound Proxy** | `10.10.10.5` | Adresse IP de votre serveur FreePBX |
| **Prefer Primary Outbound Proxy** | `Yes` | Maintient le proxy principal |
| **SIP Transport** | `UDP` | Protocole de transport SIP standard |
| **NAT Traversal** | `No` | En réseau local direct avec FreePBX |
| **SIP User ID** | `04XXXXXXXX` | Numéro de la ligne fixe Livebox |
| **Authenticate ID** | `04XXXXXXXX` | Identifiant d'authentification SIP |
| **Authenticate Password** | `••••••••` | Mot de passe du trunk configuré dans FreePBX |
| **SIP Registration** | `Yes` | Enregistre le trunk auprès de FreePBX |
| **Outgoing Call without Registration** | `Yes` | Permet d'émettre des appels même si l'enregistrement expire |
| **Register Expiration** | `5` | Délai de réenregistrement (en minutes) |
| **Local SIP Port** | `5062` | **Indispensable :** distinct du port FXS (`5060`) |
| **Local RTP Port** | `5012` | Port de base pour les flux audio RTP |
| **Allow Incoming SIP Messages from SIP Proxy Only** | `Yes` | Sécurité : n'accepte que les paquets provenant de FreePBX |
| **Preferred DTMF method** | Priority 1: `RFC2833`<br>Priority 2: `SIP INFO`<br>Priority 3: `In-audio` | Ordre de négociation DTMF |
| **Dial Plan** | `{ x+ \| \+x+ \| *x+ \| *xx*x+ }` | Autorise tous les numéros et préfixes |
| **Preferred Vocoder** | Choice 1 à 7 : `PCMA` | **G.711 A-law** (standard téléphonique français/européen) |

---

### 4. Port FXO - Caller ID, Terminaison & Ligne Livebox

Cette section regroupe les paramètres matériels analogiques pour interfacer parfaitement le HT813 avec la prise téléphonique de la Livebox Orange :

#### Présentation du numéro (Caller ID) & Audio
| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Caller ID Scheme** | `ETSI-FSK during ringing` | **Standard Orange Livebox :** trame FSK émise pendant la sonnerie |
| **Caller ID Transport Type** | `Relay via SIP P-Asserted-Identity` | Transmet le numéro de l'appelant à FreePBX dans l'en-tête PAI |
| **FSK Caller ID Minimum RX Level (dB)** | `-40` | Seuil de sensibilité pour la détection du Caller ID |
| **FSK Caller ID Seizure Bits** | `70` | Bits de synchronisation du signal FSK |
| **FSK Caller ID Mark Bits** | `40` | Bits de marquage |
| **Gain TX** | `+2dB` | Légère amplification du volume de la voix émise vers la Livebox |
| **Gain RX** | `0dB` | Volume voix reçue depuis la Livebox |

#### Terminaison de ligne (FXO Termination)
| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Enable Current Disconnect** | `Yes` | Raccroche la ligne lors de la coupure de boucle de courant (*Loop Current Drop*) |
| **Current Disconnect Threshold (ms)** | `100` | Seuil de détection de coupure (100 ms) |
| **Enable PSTN Disconnect Tone Detection** | `Yes` | Raccroche à la détection de la tonalité d'occupation |
| **PSTN Disconnect Tone** | `f1=440@-30, f2=440@-30, c=500/500` | **Tonalité d'occupation Orange :** 440 Hz, 500ms ON / 500ms OFF à -30 dBm |
| **AC Termination Model** | `Auto-Detected` | Adaptation d'impédance automatique |
| **Country-based** | `FRANCE` | Profil électrique des lignes françaises |
| **Impedance-based** | `600R -- 600 ohms` | Impédance de ligne 600 ohms |
| **Number of Rings** | `2` | Attend 2 sonneries avant de décrocher (permet au HT813 de décoder le Caller ID) |
| **PSTN Ring Thru FXS** | `No` | N'envoie pas la sonnerie directement sur le port téléphone analogique local |
| **PSTN Ring Thru Delay (sec)** | `4` | Délai de bascule de sécurité |

#### Numérotation vers la Livebox (Channel Dialing)
| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Stage Method (1/2)** | `1` | **Numérotation 1 étape :** composition directe du numéro sans 2e tonalité |
| **Wait for Dial-Tone** | `Yes` | Attend la tonalité de la Livebox avant de composer les chiffres |
| **Min Delay Before Dial PSTN Number** | `500` ms | Pause de 500 ms après décrochage avant l'envoi des tonalités DTMF |
| **DTMF Digit Length (ms)** | `100` | Durée d'émission de chaque chiffre DTMF |
| **DTMF Dial Pause (ms)** | `100` | Pause entre deux chiffres DTMF |
| **First Digit Timeout (sec)** | `10` | Délai d'attente du premier chiffre |
| **Inter-Digit Timeout (sec)** | `4` | Délai d'attente entre deux chiffres |

---

### 5. Port FXS - Poste analogique local (FXS Port)

Dans l'onglet **FXS Port**, configurez le port pour le téléphone analogique branché au dos du boîtier (ex: téléphone fixe ou base DECT du salon) :

#### Compte SIP & Enregistrement FreePBX
| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Account Active** | `Yes` | Active le port FXS |
| **Primary SIP Server** | `10.10.10.5` | Adresse IP de votre serveur FreePBX |
| **Prefer Primary SIP Server** | `Yes` | Maintient la connexion sur le serveur principal |
| **SIP Transport** | `UDP` | Protocole UDP |
| **NAT Traversal** | `No` | Connexion locale directe avec FreePBX |
| **SIP User ID** | `4000` | Numéro d'extension interne FreePBX |
| **Authenticate ID** | `4000` | Identifiant SIP de l'extension |
| **Authenticate Password** | `••••••••` | Mot de passe de l'extension configuré dans FreePBX |
| **Name** | `Salon` | Nom affiché pour ce poste |
| **SIP Registration** | `Yes` | Enregistre le poste auprès de FreePBX |
| **Register Expiration** | `60` | Délai de réenregistrement (en minutes) |
| **Local SIP Port** | `5060` | **Indispensable :** distinct du port FXO (`5062`) |
| **Local RTP Port** | `5004` | Port de base pour le flux audio RTP |
| **Preferred DTMF method** | Priority 1: `RFC2833`<br>Priority 2: `SIP INFO`<br>Priority 3: `In-audio` | Ordre DTMF pour la numérotation et les serveurs vocaux |
| **Dial Plan** | `{ x+ \| \+x+ \| *x+ \| *xx*x+ }` | Permet les appels internes, externes et les codes Asterisk (*xx) |
| **Preferred Vocoder** | Choice 1: `PCMU`<br>Choice 2: `PCMA`<br>Choice 4: `G729` | Codecs audio autorisés |

#### Paramètres de ligne analogique (SLIC & Téléphonie)
| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **SLIC Setting** | `EUROPEAN CTR21` | **Profil électrique officiel européen/français** pour les postes analogiques |
| **Caller ID Scheme** | `Bellcore/Telcordia` | Protocole de présentation du numéro vers le téléphone analogique |
| **Enable Pulse Dialing** | `Yes` ou `No` | Active le décodage des impulsions pour téléphones anciens à cadran rotatif. **Mettre impérativement sur `No`** si vous utilisez un téléphone à touches classique (DTMF) ! |
| **Enable Hook Flash** | `Yes` | Prise en charge de la touche R (*Flash*) |
| **Hook Flash Timing** | min: `300 ms` / max: `1100 ms` | Plage de détection de la touche R |
| **On Hook Timing** | `400 ms` | Durée minimale de raccrochage |
| **Gain TX / RX** | TX: `0dB` / RX: `-6dB` | Niveaux audio d'émission et de réception |
| **Ring Frequency** | `20Hz` | Fréquence de sonnerie standard |

---

## 💻 Configuration de FreePBX

### 1. Trunk PJSIP (Ligne Livebox)

Dans **Connectivity** > **Trunks** > **Add Trunk** > **Add SIP (chan_pjsip) Trunk** :

#### Onglet General
| Paramètre | Valeur | Explication |
| :--- | :--- | :--- |
| **Nom de la jonction** | `04XXXXXXXX` | Nom du trunk (Doit être votre numéro de ligne fixe) |
| **Cacher le ID appelant** | `Non` | Transmet le numéro lors des appels sortants |
| **Outbound CallerID** | `<04XXXXXXXX>` | Numéro de présentation sortant |
| **Options CID** | `Autorise tout SDA` | Conserve les identifiants d'appelants autorisés |
| **Nombre maximal de canaux** | `1` | **Indispensable :** la prise TEL de la Livebox est une ligne analogique unique (1 appel simultané) |
| **Asterisk Trunk Dial Options** | `TR` (Remplacer) | Options de numérotation Asterisk pour le trunk |
| **Continue if Busy** | `Non` | Ne continue pas sur d'autres routes si la ligne est occupée |
| **Désactivation du Trunk** | `Non` | Maintient la jonction active |

#### Onglet Dialed Number Manipulation Rules
> [!NOTE]
> Laissez cet onglet **entièrement vide** (aucun préfixe ni règle de manipulation).
> Tous les modèles de numérotation (*Dial Patterns*) et les règles de routage sont configurés dans les **Routes sortantes (Outbound Routes)** et les **Routes entrantes (Inbound Routes)** de FreePBX.

#### Onglet pjsip Paramètres > General
| Paramètre | Valeur | Explication |
| :--- | :--- | :--- |
| **Nom d'utilisateur** | `04XXXXXXXX` | Identifiant utilisateur SIP (DID) |
| **Auth username** | `04XXXXXXXX` | Identifiant d'authentification SIP |
| **Secret** | `••••••••` | Mot de passe SIP (identique au mot de passe configuré sur le port FXO) |
| **Authentification** | `En sortie` | Authentifie les requêtes émises vers le HT813 |
| **Enregistrement** | `Receive` | **Clé de voûte :** FreePBX reçoit l'enregistrement du HT813 (pas besoin d'IP statique dans SIP Server) |
| **Language Code** | `Par défaut` | Langue du trunk |
| **Context** | `from-trunk` | Contexte Asterisk pour le traitement des appels entrants |
| **Transport** | `0.0.0.0-udp` | Protocole UDP |

#### Onglet pjsip Paramètres > Avancé
| Paramètre | Valeur | Explication |
| :--- | :--- | :--- |
| **Mode DTMF** | `RFC 4733` | Standard DTMF moderne (successeur de RFC 2833) |
| **Send Line in Registration** | `Oui` | Ajoute l'en-tête `line` lors de l'enregistrement |
| **Trust RPID/PAI** | `Oui` | **Crucial :** permet à FreePBX de lire l'en-tête PAI (*P-Asserted-Identity*) transmis par le HT813 pour afficher le numéro de l'appelant |
| **Send RPID/PAI** | `Non` | Ne transmet pas d'en-tête PAI sortant vers le HT813 |
| **Direct Media** | `Non` | Conserve le flux audio au sein du serveur FreePBX |
| **Rewrite Contact** | `Oui` | Réécrit l'en-tête Contact avec l'IP et port source |
| **RTP Symmetric** | `Oui` | Envoie le flux RTP vers l'IP/port source de réception |
| **Force rport** | `Oui` | Force la conformité RFC 3581 pour la signalisation SIP |
| **Qualify Frequency** | `60` | Teste la connectivité du trunk toutes les 60 secondes |

#### Onglet pjsip Paramètres > Codecs
| Paramètre | Valeur | Explication |
| :--- | :--- | :--- |
| **alaw** | ✅ **Coché** | **G.711 A-law (Europe/France)** : seul codec nécessaire pour la ligne Livebox |
| **Autres codecs** | ❌ Décochés | Désactivez g722, ulaw, g726, g723, speex... |

---

### 2. Route entrante (Inbound Route)

Dans **Connectivity** > **Inbound Routes** > **Add Inbound Route** :

#### Onglet General
| Paramètre | Valeur | Explication |
| :--- | :--- | :--- |
| **Description** | `Livebox` | Nom descriptif de la route entrante |
| **Numéro DID** | `0XXXXXXXXX` | Votre numéro de ligne fixe (doit être **strictement identique** au *SIP User ID* configuré sur le port FXO) |
| **Numéro de l'appelant** | `ANY` | Accepte tous les numéros entrants |
| **Priorité de la route** | `Oui` | Donne la priorité à cette route spécifique |
| **Alert Info** | `Aucun` | Aucune sonnerie distinctive imposée |
| **Ringer Volume Override** | `Aucun` | Volume de sonnerie par défaut des combinés |
| **Préfixe au CID** | `Livebox` | Préfixe affiché sur l'écran des téléphones (ex: `Livebox:0612345678`) pour repérer immédiatement un appel vers la ligne fixe |
| **Musique d'attente** | `Par défaut` | Musique d'attente standard |
| **Choix Destination** | *Votre destination* | Vers où faire sonner l'appel (ex : *Groupes de Sonnerie*, *Extension 4000 Salon*, *File d'attente*, *Standard SVI/IVR*) |

> [!NOTE]
> Les onglets **Avancé**, **Protection**, **Télécopieur** et **Autre** peuvent être laissés à leurs valeurs par défaut.

#### 💡 Astuce Bonus : Filtrage anti-démarchage téléphonique (Préfixes ARCEP)

Depuis le 1er janvier 2023, la réglementation française (ARCEP) impose aux démarcheurs et centres de prospection commerciale d'utiliser des plages de numéros strictement réservées. Vous pouvez créer des **Routes entrantes dédiées** dans FreePBX pour filtrer ces appels indésirables avant même qu'ils ne fassent sonner vos téléphones.

> [!IMPORTANT]
> **Une route distincte par préfixe :** Le champ *Numéro de l'appelant* (*CallerID Number*) de FreePBX n'accepte qu'un seul masque à la fois. Il n'est pas possible de regrouper deux préfixes (comme `0162` et `0163`) sur une même règle : il faut créer **12 routes entrantes individuelles** (une par préfixe ARCEP).

##### Exemple de configuration pas-à-pas pour une règle (ex: préfixe `0162`)

Dans **Connectivity** > **Inbound Routes** > **Add Inbound Route** :

| Paramètre | Valeur | Explication |
| :--- | :--- | :--- |
| **Description** | `Block 0162` | Nom clair identifiant le préfixe bloqué |
| **Numéro DID** | `0XXXXXXXXX` | Votre numéro fixe (identique au port FXO) |
| **Numéro de l'appelant** | `_0162XXXXXX` | **Format Asterisk :** commence obligatoirement par un underscore `_` suivi du préfixe et de 6 `X` majuscules |
| **Priorité de la route** | `Non` | Laissez sur `Non` (Asterisk évalue naturellement les masques CID spécifiques avant le masque générique `ANY` de la route principale) |
| **Choix Destination** | `Terminate Call` > `Mettre définitivement l'appelant sur occupation` | Rejette l'appel avec tonalité d'occupation *(ou redirigez vers une musique d'attente infinie, ex: musique d'installation de Windows XP `title.wma` !)* |

##### Liste complète des 12 routes à créer (calquée sur l'interface FreePBX)

| Numéro DID | Numéro de l'appelant | Description | Destination |
| :--- | :--- | :--- | :--- |
| `0XXXXXXXXX` | `_0162XXXXXX` | `Block 0162` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0163XXXXXX` | `Block 0163` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0270XXXXXX` | `Block 0270` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0271XXXXXX` | `Block 0271` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0377XXXXXX` | `Block 0377` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0378XXXXXX` | `Block 0378` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0424XXXXXX` | `Block 0424` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0425XXXXXX` | `Block 0425` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0568XXXXXX` | `Block 0568` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0569XXXXXX` | `Block 0569` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0948XXXXXX` | `Block 0948` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |
| `0XXXXXXXXX` | `_0949XXXXXX` | `Block 0949` | `Terminate Call: Mettre définitivement l'appelant sur occupation` |

---

### 3. Route sortante (Outbound Route)

Dans **Connectivity** > **Outbound Routes** > **Add Outbound Route** :

#### Onglet Route Settings
| Paramètre | Valeur | Explication |
| :--- | :--- | :--- |
| **Nom de la Route** | `Livebox` | Nom descriptif de la route sortante |
| **Route CID** | `04XXXXXXXX` | Numéro de présentation sortant (votre numéro de ligne fixe Livebox) |
| **Remplacer le poste** | `Oui` | Force la présentation du numéro du Trunk quel que soit le CallerID interne du combiné |
| **Mot de Passe de la Route** | *(vide)* | Pas de mot de passe requis pour passer un appel |
| **Route Type** | *(Standard)* | Laissez décoché (*Urgence* et *Intra-Company* non sélectionnés) |
| **Musique d'attente** | `default` | Musique d'attente Asterisk standard |
| **Time Match Time Zone** | `Use System Timezone` | Utilise le fuseau horaire du serveur |
| **Time Match Time Group** | `---Route permanente---` | Route disponible sans restriction horaire |
| **Trunk Sequence for Matched Routes** | `04XXXXXXXX` | Sélectionnez le trunk PJSIP associé au port FXO du HT813 |
| **Optional Destination on Congestion** | `Annonces` > `Busy` | Diffuse une annonce d'occupation si la ligne analogique de la Livebox est déjà occupée |

#### Onglet Dial Patterns (Modèles de numérotation)
L'interface FreePBX utilise 4 colonnes pour analyser et reformater les numéros composés :
* **( prepend )** : Chiffres ajoutés automatiquement au début du numéro avant transmission au HT813.
* **prefix |** : Chiffres composés par l'utilisateur qui seront retirés avant transmission.
* **[ match pattern ]** : Modèle de numéro autorisé (`X` = chiffre 0-9, `.` = n'importe quelle suite de chiffres, `[1-79]` = ensemble de chiffres autorisés).
* **/ [ ID appelant ]** : Filtre par extension (laisser vide pour autoriser tous les téléphones).

| ( prepend ) | prefix \| | [ match pattern ] | Destination & Explication technique |
| :--- | :--- | :--- | :--- |
| | | `0[1-79]XXXXXXXX` | **France (Fixes, Mobiles & Box IP) :** autorise `01` à `05` (fixes), `06` et `07` (mobiles) et `09` (box/VoIP). **Exclut volontairement le chiffre `8`** (`[1-79]`) afin de bloquer automatiquement tous les numéros spéciaux surtaxés (`081X`, `082X`, `089X`). |
| | | `080[0-9]XXXXXX` | **France (Numéros Verts gratuits) :** autorise spécifiquement les tranches gratuites `0800` à `0809` (10 chiffres), sans débloquer les numéros payants. |
| | | `3103` | **Messagerie Orange :** accès direct au répondeur vocal officiel de la Livebox. |
| `0` | `+33` | `[1-79]XXXXXXXX` | **Normalisation nationale (+33) :** retire `+33` et ajoute `0` au début (ex: `+33612345678` devient `0612345678`). |
| `0` | `0033` | `[1-79]XXXXXXXX` | **Normalisation nationale (0033) :** retire `0033` et ajoute `0` au début (ex: `0033123456789` devient `0123456789`). |
| | | `0044[12].` | **Royaume-Uni (Fixes uniquement) :** indicatif international `0044`. Les numéros fixes britanniques débutent par `1` (ex: `0161`, `0121`) ou `2` (ex: Londres `020`). Ce masque exclut strictement les mobiles britanniques (`07`) ainsi que les numéros surtaxés (`08`/`09`). |
| | | `00353[124-79].` | **Irlande (Fixes) :** indicatif international `00353` suivi des indicatifs géographiques ComReg (Dublin `1`, Cork `21`, Galway `91`, etc.). |
| | | `003538[35-9].` | **Irlande (Mobiles) :** indicatif international `00353` suivi des préfixes mobiles irlandais (`083`, `085`, `086`, `087`, `089`). Exclut les numéros surtaxés `15xx` et à coûts partagés `0818`. |

> [!NOTE]
> **Pourquoi convertir `+33` et `0033` en `0` national ?**  
> Le port analogique TEL de la Livebox n'accepte que le plan de numérotation national à 10 chiffres pour les appels en France. Lorsqu'un numéro français est composé au format international (ce qui arrive constamment avec les contacts enregistrés dans un smartphone ou un softphone synchronisé), la Livebox rejette l'appel en le considérant comme invalide. La réécriture automatique rétablit le format national attendu (`0X...`).

---

### 4. Route sortante d'urgence (Emergency Route)

Il est vivement recommandé de créer une **route sortante distincte dédiée aux services d'urgence** plutôt que de mélanger ces numéros dans la route générale.

#### Pourquoi séparer les urgences dans une route dédiée ?
1. **Secours & Tolérance de panne (Failover multi-trunks) :** Dans FreePBX, la liste *Trunk Sequence for Matched Routes* permet de définir un **second trunk de secours** (ex: passerelle GSM/4G ou ligne VoIP SIP tierce). En cas d'urgence, si la ligne Livebox est occupée ou en panne, l'appel bascule instantanément sur le trunk de secours.
2. **Priorité absolue & Dérogation (*Route Type: Urgence*) :** En marquant la route comme **Urgence**, FreePBX désactive les restrictions de postes : même un combiné bridé en interne sans accès aux appels externes pourra composer le 15, 17, 18 ou 112.
3. **Ordre d'évaluation :** Placez impérativement cette route **au sommet de la liste** des routes sortantes de FreePBX pour qu'elle soit évaluée en premier.

*(Si vous n'avez qu'un seul trunk et souhaitez simplifier, vous pouvez également inclure ces 4 numéros directement dans l'onglet Dial Patterns de votre route principale).*

#### Configuration de la route d'urgence
Dans **Connectivity** > **Outbound Routes** > **Add Outbound Route** :

* **Route Settings :**
  * **Nom de la Route** : `Urgences`
  * **Route CID** : `<04XXXXXXXX>`
  * **Route Type** : ✅ **Urgence** *(Coché)*
  * **Trunk Sequence** : `04XXXXXXXX` *(Trunk HT813 / Livebox)* + *(Optionnel) Votre second trunk de secours*

* **Dial Patterns :**

| ( prepend ) | prefix \| | [ match pattern ] | Service d'urgence |
| :--- | :--- | :--- | :--- |
| | | `112` | **Numéro d'urgence européen** (valable pour toute urgence) |
| | | `15` | **SAMU** (Urgences médicales) |
| | | `17` | **Police Secours / Gendarmerie** |
| | | `18` | **Sapeurs-Pompiers** (Secours et incendies) |

---

### 5. Extension PJSIP (Poste FXS "Salon")

Dans **Applications** > **Extensions** > **Add Extension** > **Add New chan_pjsip Extension** :

- **User Extension** : `4000`
- **Display Name** : `Salon`
- **Secret** : Le mot de passe configuré dans le port FXS du HT813.

---

## 🛠️ Résolution des problèmes fréquents (Troubleshooting)

### 🔴 1. La ligne Livebox reste occupée indéfiniment après un raccrochage
* **Cause** : Le HT813 n'a pas détecté que l'interlocuteur distant a raccroché et maintient le port FXO ouvert.
* **Solution** : 
  1. Assurez-vous que **Enable PSTN Disconnect Tone Detection** est sur `Yes`.
  2. Vérifiez la chaîne exacte du **PSTN Disconnect Tone** : `f1=440@-30, f2=440@-30, c=500/500`.
  3. Activez **Enable Current Disconnect** avec un seuil de `100 ms`.

### 🔴 2. L'appelant s'affiche en "Inconnu" (*Unknown*) dans FreePBX
* **Cause** : Le HT813 décroche avant d'avoir reçu ou transmis la trame FSK d'identification d'appel.
* **Solution** :
  1. Réglez **Number of Rings** sur `2` (pour laisser le temps à la Livebox de transmettre la trame FSK).
  2. Vérifiez que **Caller ID Scheme** est sur `ETSI-FSK during ringing`.
  3. Assurez-vous que **Caller ID Transport Type** est réglé sur `Relay via SIP P-Asserted-Identity`.

### 🔴 3. Double tonalité ou demande de code PIN lors d'un appel sortant
* **Cause** : Le port FXO est resté en numérotation 2 étapes (*2-stage dialing*).
* **Solution** : Passez **Stage Method** à `1` (*1-stage dialing*). Le HT813 prend la ligne et compose immédiatement les chiffres reçus de FreePBX.

### 🔴 4. Conflit SIP entre le port FXS et le port FXO
* **Cause** : Si les deux ports écoutent sur le port local `5060`, FreePBX ne peut pas distinguer le Trunk de l'Extension.
* **Solution** :
  - Port FXS (Account 1) : `5060`
  - Port FXO (Account 2) : `5062`

---

## 📦 Fichiers de provisionnement XML

Un fichier de configuration prêt à l'emploi est disponible dans ce dépôt :
- [`config_ht813.xml`](config_ht813.xml) : Sauvegarde complète issue d'un matériel fonctionnel.
- [`config/freepbx_pjsip_sample.conf`](config/freepbx_pjsip_sample.conf) : Exemple de configuration brute Asterisk PJSIP.

Pour restaurer directement ces paramètres sur votre appareil :
1. Modifiez les adresses IP et identifiants dans `config_ht813.xml`.
2. Déposez le fichier sur un serveur HTTP/TFTP local.
3. Renseignez l'URL dans le menu **Maintenance > Provisioning** de votre HT813 et redémarrez.

---

## 📄 Licence

Ce projet est distribué sous licence MIT. N'hésitez pas à ouvrir une *Issue* ou proposer une *Pull Request* pour enrichir la documentation !
