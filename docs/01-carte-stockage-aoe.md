# Projet 1 : carte de stockage réseau (ATA over Ethernet et alternatives)

**Statut : en pause depuis le 22/11/2026.** Le marché de la mémoire (DRAM) rend l'équation économique trop instable pour trancher maintenant. Ce document résume la phase exploratoire pour pouvoir la reprendre sans tout refaire.

Le questionnaire de cadrage, ses réponses et les analyses détaillées sont aussi publiés dans un artefact Claude privé : <https://claude.ai/artifact/YCDX9rPS3agD2841wkzzJS>. Les réponses y sont enregistrées dans la base de l'artefact (collection `cadrage`, document `reponses`).

Tous les prix sont en dollars, hors TVA et frais de port. Les prix LCSC marqués « relevé » ont été lus en direct sur lcsc.com le 22/11/2026. Les autres sont des ordres de grandeur, marqués « est. ».

---

## 1. Besoin initial

Concevoir une carte peu coûteuse qui expose des disques durs, et éventuellement des SSD SATA, sur le réseau, à l'origine en ATA over Ethernet (AoE).

Contraintes posées au départ :

- **Particulier :** approvisionnement en petites quantités, chez des fournisseurs ouverts au public (LCSC/JLCPCB, Mouser, DigiKey, etc.).
- **Alimentation pilotée par disque :** décaler les démarrages (*staggered spin-up*) et couper les disques au repos.
- **Densité :** 1 disque par carte de préférence, 4 au maximum.
- **Mesh :** si possible, pouvoir relier les modules entre eux.
- **Piste de départ :** un FPGA, avec ouverture à d'autres solutions.

## 2. Réponses au cadrage

### 2.1 Tour 1 : objectifs, disques, hôte, réseau

| Thème | Réponse |
|---|---|
| FPGA | un moyen, pas un but en soi |
| Priorités essentielles | coût, débit, intégrité des données |
| Priorités importantes | consommation au repos, facilité de maintenance |
| Priorités secondaires | temps de développement, apprentissage |
| Usages | archives et stockage froid, NAS multimédia, disques de VM et de conteneurs |
| Fréquence d'accès | variable selon les disques ; la politique de veille doit être réglable par disque |
| Diffusion | quelques exemplaires pour des proches |
| Nombre de disques | 5 au départ, environ 100 à terme (hypothèse, pas une contrainte forte) |
| Formats | HDD 3,5″ et SSD SATA 2,5″, modèles très variés |
| Taille de secteur | inconnue ; MTU 9000 sur toute la chaîne |
| Disques « shuckés » | pas prévu, mais possible |
| TRIM | souhaité ; modifier un pilote est acceptable |
| OS des hôtes | Linux, Windows, macOS ; développer un pilote est acceptable |
| Usage côté hôte | Ceph ; l'idée d'un OSD par nœud a beaucoup plu |
| Hôtes | plusieurs, chacun ses disques |
| Protocole | AoE de préférence, ouvert à mieux |
| Switch principal | 48 × 10G SFP+ dont 8 réservés aux uplinks, jumbo frames partout |
| Débit visé | le maximum de chaque disque ; deux types de cartes acceptés s'ils interopèrent |
| Réseau de stockage | physiquement dédié (VLAN ou VXLAN appréciés) |
| Mesh | but réel : économiser les ports du switch principal et les petits switchs câblés |

### 2.2 Tour 1 : alimentation, gestion, mécanique, fabrication

| Thème | Réponse |
|---|---|
| Alimentation | fond de pile alimenté centralement, deux alimentations Mean Well par pile |
| Partage d'alimentation | par groupes |
| Repos des disques | arrêt de rotation et coupure complète, selon la durée d'inactivité |
| Décision de mise au repos | la carte (minuterie d'inactivité) |
| Réveil | automatique à la première requête |
| Délai de réveil | pas de préférence |
| Consommation disque coupé | moins de 1 W par carte |
| Mesures | courant et tension par disque, température SMART, ventilateur PWM régulé en température |
| SMART | un résumé lu par la carte suffit |
| Gestion | petit protocole UDP ou HTTP, LED d'état |
| Adressage | un logiciel associe numéro de série du disque et adresse ; la carte vérifie le disque branché |
| Mises à jour | par le réseau, avec image de secours |
| Éléments physiques | LED d'activité et d'état, commande de ventilateur, capteur de température |
| Forme | un boîtier individuel par disque, imprimé en 3D et empilable |
| Échange à chaud | inutile |
| Budget | moins de 20 € par disque (« si un serveur fait pareil pour moins cher, on perd l'intérêt ») |
| Budget de développement | 200 à 500 € |
| Fabrication | entièrement assemblée par le fabricant, BGA compris |
| Compétences | HDL, routage rapide et alimentations : notions ; Linux embarqué : à l'aise. Claude conçoit sous KiCad |
| Outillage | oscilloscope 200 MHz, alimentation de labo |
| Outils FPGA | open source de préférence, Vivado accepté |
| Écritures | doivent être sur le plateau (FLUSH respecté) |
| Chiffrement | côté hôte ; dans la carte si c'est quasi gratuit |
| Temps disponible | plus de 10 h par semaine, sans échéance |

### 2.3 Tour 2 : relance

| Question | Réponse |
|---|---|
| Répartition des disques | surtout Ceph |
| Budget | nœuds de 2 à 3 disques pour viser 20 à 35 € par disque |
| Référence à battre | Supermicro 16 baies d'occasion |
| Prix du kWh | 0,19 € |
| Taille de pile | « la plus efficace » (proposé : 4 nœuds de 3 disques) |
| Liaisons dans la pile | connecteurs traversants |
| Switch de pile | du commerce, non administrable, un seul uplink |
| Postes Windows et macOS | via partages réseau (SMB, NFS) ou RBD |
| Alimentations de pile | d'abord « une 12 V et une 5 V », puis **bus 12 V seul** retenu |
| SSD | pool rapide Ceph |
| Première version | chip-down, sauf si les modules complets coûtent moins cher |
| Disques froids | NVMe/TCP si c'est plus simple qu'AoE |
| Ventilation | un ventilateur par module |
| Preuve de concept | non, conception directe |

### 2.4 Décisions prises ensuite en conversation

- Une pile de **4 nœuds × 3 disques = 12 disques**, avec **2 × 2,5G par nœud** et un switch 8 × 2,5G + 10G SFP+ dans la base.
- **Ethernet par câbles courts** ; les connecteurs traversants ne portent que l'alimentation et la gestion.
- **Bus 12 V seul** dans la pile, 5 V produit sur chaque nœud, deux alimentations 12 V en parallèle pour la redondance.
- Coût d'achat d'environ 40 à 60 € par disque **accepté**, justifié par les économies d'énergie (avant que l'envolée de la DRAM ne remette tout en cause).
- **Pas d'eMMC :** seulement U-Boot en flash SPI sur les cartes. Quelques nœuds « infra » démarrent sur leurs disques et hébergent l'OS, les moniteurs Ceph et le serveur de démarrage réseau ; les autres démarrent par le réseau.
- Sur les nœuds infra, **RAID** pour la partie système (proposé par l'utilisateur en RAID 5 ; Claude recommande un RAID 1 à 3 disques sur une petite partition, point resté ouvert).
- **Module contre chip-down :** explorer les deux et comparer les prix (fait, voir section 6).

## 3. Repères techniques

### 3.1 Trame AoE

| En-tête Ethernet | En-tête AoE | En-tête ATA | Données | FCS |
|---|---|---|---|---|
| 14 o | 10 o | 12 o | 2 × 512 o (MTU 1500) ou 17 × 512 o (MTU 9000) | 4 o |

En MTU 1500, une trame ne porte que 1 Kio : environ 115 000 commandes par seconde pour remplir un lien gigabit, contre environ 14 000 en MTU 9000. L'efficacité sur le fil reste proche (≈ 94 % contre ≈ 99 %) ; c'est le nombre de commandes qui coûte.

### 3.2 Débits utiles

| Maillon | Débit |
|---|---|
| HDD 3,5″ 7200 tr/min | 200–280 Mo/s en début de disque, 100–130 Mo/s en fin |
| SSD SATA | 500–560 Mo/s |
| SATA Gen1 / Gen2 / Gen3 | ≈ 140 / 280 / 560 Mo/s |
| 1 GbE (AoE, MTU 1500 / 9000) | ≈ 118 / 124 Mo/s |
| 2,5 GbE | ≈ 295 / 310 Mo/s |
| 10 GbE | ≈ 1180 / 1240 Mo/s |

Point d'équilibre : un HDD, le SATA Gen2 et le 2,5 GbE plafonnent tous vers 280–300 Mo/s.

### 3.3 Consommation des disques (ordres de grandeur)

| Disque | Démarrage | En rotation | STANDBY |
|---|---|---|---|
| HDD 3,5″ | 12 V ≈ 2–2,5 A + 5 V ≈ 0,5–1 A, soit ≈ 25–30 W quelques secondes | 5–9 W | 0,5–1 W |
| HDD 2,5″ | 5 V ≈ 1 A | 1,5–2,5 W | 0,1–0,3 W |
| SSD SATA | 5 V, sans pic marqué | 2–4 W | ≈ 0,05–0,5 W |

Une pile de 8 disques 3,5″ avec son switch : pic ≈ 120 W avec démarrages décalés, régime ≈ 100 W.

## 4. Constats clés

1. **Ceph et la mise au repos des disques s'excluent.** Ceph répartit chaque objet sur de nombreux OSD et vérifie régulièrement les données (scrub léger quotidien, profond hebdomadaire). Un OSD dont le disque est coupé est déclaré absent puis reconstruit ailleurs (10 minutes par défaut). D'où deux familles : disques Ceph toujours alimentés, disques froids exposés en mode bloc avec mise au repos.
2. **Un OSD coûte de la mémoire :** 4 Gio par défaut, plancher documenté de 2 Gio. Un nœud de 3 OSD demande 8 Go.
3. **La DRAM a explosé en 2025–2026.** LPDDR4X 8 Go à 139,94 $ chez LCSC (relevé), soit ≈ 17 $/Go et sept fois le prix d'un RK3568. TrendForce annonçait +70–75 % sur la LPDDR4X au 2T26 et des hausses jusqu'en 2027. La DDR3 neuve n'est pas une issue (35–58 $/Go).
4. **Le pilote Linux aoe ne transmet pas les FLUSH.** Vérifié dans le code du noyau : `drivers/block/aoe/aoeblk.c` ne déclare que `BLK_FEAT_ROTATIONAL` (pas de cache d'écriture) et le pilote ne traite que lectures et écritures. Les écritures que Ceph croit sur le plateau peuvent rester dans le cache du disque. Remèdes : désactiver le cache d'écriture des disques, ou patcher le pilote (FLUSH CACHE, et TRIM pour les SSD).
5. **AoE n'a d'initiateur maintenu que sous Linux.** Avec des OSD sur des hôtes Linux, ce n'est plus gênant : les postes Windows et macOS passent par Ceph (RBD, SMB, NFS).
6. **Le « mesh » devient une hiérarchie.** Une chaîne partage sa bande passante (8 HDD ≈ 2 Go/s contre 300 Mo/s sur un lien 2,5G). Une pile avec un switch d'agrégation 8 × 2,5G + 10G SFP+ (puces Realtek RTL8373 + RTL8224, 40–70 $ dans le commerce) économise les ports du switch principal.
7. **Conception sans moyens de mesure :** un oscilloscope 200 MHz ne vérifie ni des liaisons à 2,5–6 Gb/s ni une LPDDR4. Un module du commerce supprime le risque mémoire pour un prix voisin.

## 5. Architecture de référence au moment de la pause

- **Pile :** 4 nœuds de 3 disques plus une base (switch 8 × 2,5G + 10G SFP+, alimentations 12 V redondantes, ventilation). Environ 9 piles et 9 ports SFP+ pour 100 disques.
- **Nœud :** Rockchip RK3568 (3 SATA natifs sur les liaisons série partagées, PCIe 3.0 scindé en 2 × x1 pour 2 × RTL8125B en `balance-alb`), eFuses 12 V et 5 V par disque, mesure courant-tension (INA3221 ou INA226), petit MCU toujours alimenté, LED, ventilateur PWM.
- **Démarrage :** U-Boot en flash SPI ; nœuds infra sur disque (partition système en RAID, moniteurs Ceph, DHCP et TFTP/HTTP sur au moins 2 nœuds) ; autres nœuds sur image squashfs en RAM. Au moins 3 nœuds dès le départ (quorum des moniteurs, 3 copies sur 3 hôtes).
- **Démarrage décalé :** au démarrage à froid, délai fixé par la position dans la pile ; ensuite, une ligne commune « démarrage en cours » en collecteur ouvert, tenue pendant le pic de courant.
- **À vérifier :** U-Boot sur RK3568 sait-il lire un disque SATA (sinon, noyau et initramfs en flash SPI de 32 Mo) et démarrer par le réseau via le RTL8125 (sinon, port 1 GbE sur le GMAC, que le module Radxa CM3I possède déjà) ?

## 6. Options étudiées et chiffrage

### 6.1 Prix relevés chez LCSC le 22/11/2026

| Composant | À l'unité | Par quantité | Stock |
|---|---|---|---|
| Rockchip RK3568 | 20,57 $ | 19,58 $ (10+) | 181 |
| Rockchip RK3568B2 | 18,42 $ | 17,11 $ (10+) | 281 |
| LPDDR4X 8 Go (Samsung K4UCE3Q4AB) | 139,94 $ | 132,70 $ (30+) | 16 |
| LPDDR4X 4 Go (Samsung K4UBE3D4AB) | 80,15 $ | 75,81 $ (30+) | 357 |
| LPDDR4 1 Go (Samsung K4F8E3S4HD) | 30,71 $ | 29,12 $ (30+) | 279 |
| DDR3L 512 Mo (Micron MT41K256M16TW) | 19,48–21,80 $ | 18,47 $ (30+) | 1 831 |
| DDR3L 1 Go (Micron MT41K512M16VRP) | 58,25 $ | 55,19 $ (30+) | 281 |
| DDR3L 256 Mo (Micron MT41K128M16JT) | 17,72 $ | 15,64 $ (30+) | 2 184 |
| DDR3 128 Mo (Winbond W631GG6MB) | 6,37 $ | 5,89 $ (10+) | 1 394 |
| DDR4 1 Go (Samsung K4A8G165WC) | 24,40 $ | — | 1 |
| HyperRAM 8 Mo (Winbond W956D8MBYA) | 14,12 $ | 13,40 $ (10+) | 7 805 |
| RK809-5 (gestion d'alimentation) | 2,42 $ | 2,07 $ (10+) | 1 826 |
| RTL8125BG (2,5 GbE PCIe) | 4,42 $ | 3,76 $ (10+) | 6 445 |
| RTL8221B (PHY 2,5G) | 2,62 $ | 2,37 $ (10+) | 3 140 |
| Artix-7 XC7A35T-2FGG484C | 35,29 $ | 33,46 $ (30+) | 89 |
| Artix-7 XC7A50T-2FGG484C | 34,13 $ | 32,45 $ (30+) | 73 |
| Artix-7 XC7A75T-2FGG484C | 41,53 $ | 39,39 $ (30+) | 78 |
| Artix-7 XC7A100T-2FGG484C | 85,46 $ | 82,62 $ (30+) | 144 |
| Artix-7 XC7A200T-2FBG676I | 170,96 $ | 151,63 $ (40+) | 191 |
| Kintex-7 XC7K160T-2FFG676C | 84,13 $ | 79,90 $ (40+) | 31 |
| Kintex-7 XC7K325T-2FFG676C | 102,59 $ | 97,50 $ (40+) | 14 |
| ECP5 LFE5UM-45F-8BG381C | 35,28 $ | 34,12 $ (10+) | 2 |

Remarques : le XC7A50T coûte moins cher que le XC7A35T, alors que c'est la même puce, bridée sur le 35T. Les prix des Kintex-7 chez LCSC sont environ vingt fois plus bas que chez DigiKey (XC7K325T ≈ 2 000 €) : presque sûrement du marché parallèle ou de la récupération.

### 6.2 Synthèse des options

Coût par disque : nœud, part de pile (switch, alimentation, connectique) et, pour les options B, ≈ 9–15 $ de mémoire d'hôte pour l'OSD. Estimations pour des lots d'une dizaine de cartes.

| Option | Principe | Coût par disque | Au repos, par disque | Développement |
|---|---|---|---|---|
| **A** | RK3568 + 8 Go, OSD Ceph sur chaque nœud de 3 disques | ≈ 85–105 $ | ≈ 1–1,3 W + switch | faible (Linux) |
| **B1** | RK3568 + 1 Go, cible NVMe/TCP ou iSCSI, OSD sur les hôtes | ≈ 56–82 $ | ≈ 2 W avec switch | faible (Linux) |
| **B2** | Artix-7 XC7A50T sans mémoire externe, cible AoE, 2 disques | ≈ 63–94 $ | ≈ 0,75–1,25 W + switch | lourd (logique) |
| **B3** | Kintex-7 XC7K160T, 6 disques, 2 × 10G direct, cible AoE | ≈ 41–63 $ | ≈ 0,5–1 W | lourd (logique) |
| **C** | concevoir maintenant, acheter quand la DRAM baissera | — | — | — |
| **D** | une carte PC par pile (serveur d'occasion, NAS mini-ITX ou mini-PC), OSD sur place | ≈ 30–60 $ | ≈ 2–4,5 W | presque nul |

Référence : Supermicro 16 baies d'occasion ≈ 30–50 $ par baie, 80–180 W à vide. À 0,19 €/kWh, un watt permanent coûte ≈ 1,66 € par an, soit ≈ 8 € par disque et par watt sur 5 ans.

### 6.3 Option A : SoC avec OSD natif

| Poste | Chip-down | Module Radxa CM3I 8 Go |
|---|---|---|
| SoC, 8 Go, gestion d'alimentation, flash SPI | ≈ 165 $ | module 150–200 $ (est., prix non vérifié) |
| Réseau 2 × 2,5G | 12–16 $ | 12–16 $ |
| Alimentation des disques, mesures, MCU | 11–17 $ (est.) | 11–17 $ (est.) |
| Connecteurs, quartz, passifs | 8–14 $ (est.) | 10–17 $ (est.) |
| PCB | 8 couches : 10–20 $ (est.) | 4–6 couches : 4–8 $ (est.) |
| Assemblage (lot de 10) | 15–30 $ (est.) | 8–15 $ (est.) |
| **Par nœud / par disque** | **≈ 220–260 $ / ≈ 75–87 $** | **≈ 195–275 $ / ≈ 65–90 $** |

Égalité de prix entre module et chip-down : le module l'emporte par le risque. Le Radxa CM3I expose bien 3 SATA et un PCIe 3.0 configurable en 1 × 1 + 1 × 1, plus un port 1 GbE.

### 6.4 Options B1 et B2 : nœuds légers

| | B1 · SoC + LPDDR4 1 Go | B2 · FPGA + DDR3 128 Mo |
|---|---|---|
| Disques par nœud | 3 | 2 |
| Processeur ou FPGA | RK3568 19,58 $ + RK809 2,07 $ | XC7A50T 32,45–34,13 $ |
| Mémoire | 29,12–30,71 $ | 5,89 $ (facultative pour AoE) |
| Réseau 2 × 2,5G | 2 × RTL8125 : 7,52 $ | 2 × RTL8221B : 4,74 $ |
| Carte complète (est.) | ≈ 112–156 $ | ≈ 88–127 $ |
| Par disque, nœud seul | ≈ 37–52 $ | ≈ 44–64 $ |
| Débit réseau par disque | ≈ 200 Mo/s | ≈ 300 Mo/s |
| Protocoles | NVMe/TCP, iSCSI, AoE fournis par Linux | AoE à développer |
| Intégrité | FLUSH et TRIM, code éprouvé | patch du pilote aoe, logique neuve |

Verdict : à débit comparable, B1 et B2 se valent à 10 $ près par disque. B1 l'emporte sur les critères essentiels (coût, intégrité). B2 n'a d'intérêt que pour l'absence d'OS, une carte sans LPDDR4 et une faible exposition à la DRAM.

### 6.5 Option B3 : un FPGA plus gros

Kintex-7 en boîtier FFG676 : 8 transceivers jusqu'à 12,5 Gb/s, soit 6 SATA et 2 × 10GBASE-R vers le switch principal, sans switch de pile. Sa mémoire interne de 1,46 Mo suffit à une cible AoE pour 6 disques.

| Nœud B3 (6 disques, XC7K160T) | Prix |
|---|---|
| FPGA | 79,90–84,13 $ |
| Régulateurs, quartz, flash de configuration | ≈ 16–30 $ (est.) |
| 2 cages SFP+ | ≈ 2–4 $ (est.) |
| Alimentation pilotée des 6 disques, mesures | ≈ 18–30 $ (est.) |
| Connecteurs, passifs | ≈ 10–18 $ (est.) |
| PCB 8 couches et assemblage | ≈ 30–60 $ (est.) |
| **Total / par disque** | **≈ 156–226 $ / ≈ 26–38 $** |

Réserves : authenticité et disponibilité des Kintex-7 à ce prix (le vrai pari), panne qui arrête 6 disques, PCB 8 couches avec du 10G, logique à écrire (LiteSATA a été mis au point sur Kintex-7 ; il existe des MAC 10G open source), patch aoe. Outils : le XC7K160T est couvert par Vivado gratuit ; le 325T seulement dans les versions récentes, et AMD revoit ses licences à partir de la 2026.1. openXC7 vise ces puces, sans confirmation pour les transceivers.

### 6.6 Protocoles sur FPGA

- **AoE :** le plus simple (niveau 2, pas de TCP). La mémoire interne du FPGA suffit.
- **Fibre Channel :** pas de TCP (contrôle de flux par crédits), initiateurs matures partout, FLUSH et TRIM natifs. Mais 8GFC = 8,5 Gbauds, hors de portée de l'Artix-7 et de l'ECP5 : 4GFC (≈ 400 Mo/s) seulement sur FPGA bon marché. Plus lourd qu'AoE (connexion à la fabrique, FCP, traduction SCSI → ATA). Pas de cœur open source prêt (fejkon est un HBA expérimental). Variante sans FPGA : carte QLogic en mode cible (LIO, `tcm_qla2xxx`), non validée sur ARM64. L'utilisateur dispose d'un switch Brocade 48 × 8G (modèle, licences et consommation à préciser).
- **iSCSI et NVMe/TCP :** pile TCP complète. À 2,5G sur réseau local, les tampons de retransmission (≈ 64 Ko par connexion) tiennent dans la mémoire interne ; une DDR3 de 128 Mo à 5,89 $ les met à l'aise. Le vrai coût est le développement (TCP câblé pour les données, puis protocole et traduction vers ATA). Sur un nœud SoC, ces cibles sont fournies par Linux.
- **NVMe sur RDMA (RoCEv2) :** cartes RDMA sur les hôtes et moteur RDMA dans le FPGA ; écarté.

### 6.7 Mémoire : barrettes et DDR3

- **DDR3 neuve :** plus chère au Go que la LPDDR4X (voir le tableau 6.1).
- **SO-DIMM ou DIMM sur RK3568 :** contrôleur 32 bits, donc moitié de barrette utilisée ; les RDIMM ne fonctionnent pas. Aucun gain.
- **Mémoire d'occasion (2026) :**

| Type | 16 Go | Par Go |
|---|---|---|
| RDIMM DDR4 ECC | 47–60 $ | ≈ 3–3,75 $ |
| SO-DIMM DDR4, peu de relevés | ≈ 25–40 $ | ≈ 1,5–2,5 $ |
| UDIMM DDR4 de bureau, relevés contradictoires | 35–160 $ | ≈ 2–10 $ |
| UDIMM DDR4 ECC | 130–190 $ | ≈ 8–12 $ |

- **SoC conçus pour des barrettes :** NXP Layerscape (LS1046A ≈ 113 € la puce, 1 seul SATA ; Traverse Ten64 à 699 $ sans mémoire), trop chers.

### 6.8 Option D : carte PC par pile

| Plateforme | Mémoire | Disques | Réseau | Au repos | Prix (indicatif) |
|---|---|---|---|---|---|
| Atom C3758 (ASRock Rack C3758D4I-4L, Supermicro A2SDi) | 4 DIMM, RDIMM ECC | jusqu'à 13 SATA | 4 × 1GbE + carte 10G | ≈ 19–21 W | C3558 d'occasion ≈ 280 $ ; C3758 neuve ≈ 520 € |
| Xeon D-1541 (X10SDV, ASRock Rack D1541) | 4 DIMM, RDIMM ECC | 6 SATA + carte SATA | 2 × 10GbE | ≈ 24–36 W | ≈ 430 $ (une vente) |
| Carte NAS N5105 / N6005 | 2 SO-DIMM DDR4 | 6 SATA | 2,5G | ≈ 15–28 W (est.) | ≈ 130–200 $ |
| Carte NAS N100 / N150 | 1 SO-DIMM DDR5 | 6 SATA | 10G + 2 × 2,5G | ≈ 15–28 W (est.) | ≈ 125–280 $ |
| Mini-PC Lenovo M720q / M920q + carte SATA | 2 SO-DIMM DDR4 | 8 | 1GbE seulement | ≈ 18–22 W (est.) | ≈ 85–170 $ + riser ≈ 23 $ |

Estimations par disque, OSD compris : Atom C3758 avec 12 disques et 48 Go ≈ 38–60 $ ; N5105 avec 6 disques et 32 Go ≈ 38–60 $ ; M920q avec 8 disques et 32 Go ≈ 30–48 $.

Réserves : marché de l'occasion instable, cartes serveur en fin de série, panne qui arrête 6 à 13 disques, processeurs justes pour Ceph pendant une reconstruction, N5105 officiellement limité à 16 Go, contrôleur JMB585 qui empêche la mise en veille du bus PCIe, réseau à bricoler sur le mini-PC.

## 7. Recommandation au moment de la pause

- **La plus pragmatique : D.** Une carte PC par pile, avec de la mémoire d'occasion. Le projet se recentre alors sur ce qui n'existe pas dans le commerce : un **fond de pile** (alimentation pilotée par disque, démarrage décalé, mesures, ventilation, LED, relié au PC par USB) et la mécanique imprimée.
- **Si l'on veut concevoir le nœud :** B1 en module pour la v1 (protocoles standard, intégrité sans patch). B3 si des Kintex-7 authentiques restent disponibles vers 80 $ : c'est la variante la moins chère et la plus sobre par disque.
- **A** (OSD sur chaque nœud) ne se justifie qu'avec une forte baisse de la DRAM.

## 8. Questions restées ouvertes

Posées dans l'artefact, sans réponse au moment de la pause :

- **E1 :** option retenue (A, B1, B2, B3, C, D).
- **E2 :** mémoire libre sur les hôtes (type, capacité, emplacements ; RDIMM DDR3 ou DDR4 acceptées ?).
- **E3 :** module pour la v1, ou chip-down quand même.
- **E4 :** premier déploiement (3 nœuds d'emblée, ou 2 nœuds plus un moniteur sur un hôte).
- **E5 :** switch Fibre Channel (modèle, ports sous licence, consommation, cartes FC sur les hôtes).
- RAID 1 à 3 disques contre RAID 5 pour la partition système des nœuds infra.

Points techniques à vérifier avant toute conception :

- Combinaisons des liaisons série partagées du RK3568 dans la dernière révision de la datasheet.
- U-Boot sur RK3568 : démarrage SATA et démarrage réseau via le RTL8125.
- Prix réel des modules RK3568 (Radxa CM3I 1, 2 et 8 Go), boutique Radxa non consultée.
- Compatibilité broche à broche RTL8126 (5G) et RTL8125.
- Réveil par le réseau sur requête AoE (Wake-on-LAN par motif ou unicast).
- Prise en charge des disques 4Kn par AoE et par le pilote Linux.
- Authenticité des Kintex-7 vendus chez LCSC.
- Prix et consommation réels des cartes PC d'occasion (Atom C3000, Xeon D).

## 9. Conditions de reprise

À réexaminer quand l'un de ces points bouge :

- La LPDDR4X repasse sous ≈ 8 $/Go (rend A et B1 nettement plus intéressantes).
- Des Kintex-7 authentiques sont disponibles vers 80 $ (B3).
- Une carte PC à RDIMM basse consommation se trouve d'occasion sous ≈ 250 € (D).

## 10. Sources principales

- Recommandations matérielles Ceph : <https://docs.ceph.com/en/squid/start/hardware-recommendations/> et <https://docs.ceph.com/en/reef/start/hardware-recommendations>
- Datasheet abrégée RK3568 : <https://core-electronics.com.au/attachments/localcontent/RK3568_Brief_Datasheet_60487b51a9e.pdf>
- Documentation Radxa CM3I : <https://docs.radxa.com/en/som/cm/cm3i/getting-started/introduction>
- Prévisions TrendForce DRAM mobile 2T26 : <https://www.trendforce.com/presscenter/news/20260514-13044.html>
- DDR3 en offre tendue : <https://www.trendforce.com/news/2026/01/19/news-ddr4-reportedly-leads-legacy-memory-rally-with-q1-prices-up-to-50-ddr3-in-tight-supply/>
- Pilote Linux aoe : `drivers/block/aoe/` dans <https://github.com/torvalds/linux>
- Contrôle du cache d'écriture du noyau : <https://docs.kernel.org/6.8/block/writeback_cache_control.html>
- Wiki LiteX (LiteSATA, LiteEth) : <https://github.com/enjoy-digital/litex/wiki/Cores-Ecosystem>
- LiteSATA avec outils open source (Antmicro) : <https://antmicro.com/blog/2021/12/sata-with-open-source-fpga-tools/>
- Dépôt LiteEth : <https://github.com/enjoy-digital/liteeth>
- Switchs RTL8373 (ServeTheHome) : <https://www.servethehome.com/real-hd-8-port-2-5gbe-1-port-10g-switch-review-realtek/>
- Vivado, appareils pris en charge : <https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license/Supported-Devices>
- openXC7 : <https://github.com/openXC7>
- fejkon (HBA Fibre Channel sur FPGA) : <https://www.github.com/bluecmd/fejkon>
- LIO, cible Fibre Channel : <https://linux-iscsi.org/?p=405>
- Test Supermicro A2SDi-8C+-HLN4F (ServeTheHome) : <https://www.servethehome.com/supermicro-a2sdi-8c_-hln4f-review/4/>
- Mémoire serveur d'occasion : <https://www.amcompute.com/appraisals/ddr4-16gb-rdimm> et <https://pcserverandparts.com/news/tag/Server+Memory>
- Pages produit LCSC citées en section 6.1 : `https://www.lcsc.com/product-detail/<référence LCSC>.html` (C2938930, C3345225, C29073172, C5827966, C2920231, C253882, C2831669, C36513, C408825, C2834754, C2844948, C2939628, C3013605, C5155988, C1521748, C1550658, C1521773, C1521783, C1521831, C1521842, C1551239, C1550814)
