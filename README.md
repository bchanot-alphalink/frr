# FRR 10.5 Alphalink

Branche `stable/10.5.5-alphalink-minimal` : FRR 10.5.5 officiel (tag `frr-10.5.5`)
plus les 14 rétroportages Alphalink, SANS les 18 correctifs publiés ensuite
par FRR sur `stable/10.5`. Variante complète avec ces correctifs :
`stable/10.5.5-alphalink-fixes`, même contenu Alphalink, recommandée pour la prod.
Le README d'origine de FRR suit plus bas.

Règle de base : **la branche est la source de vérité**. Le paquet est
construit en appliquant tous les commits de la branche depuis le tag
`frr-10.5.5` sur le paquet source Debian de la même version. Rien n'est
ajouté ni retiré à la main au moment du build.

## Contenu de la branche (14 commits sur `frr-10.5.5`)

### Correctifs officiels `stable/10.5` : absents

Les 18 correctifs publiés par FRR après 10.5.5 (liste et rôle dans le README
de `stable/10.5.5-alphalink-fixes`) ne sont pas dans cette branche. Parmi eux, deux
plantages bgpd (agrégat absent, double libération en affichage EVPN), une
fuite dans zebra et un gain mémoire bgpd. Cette branche sert de variante
minimale : seulement ce qu'Alphalink ajoute à 10.5.5. Vérifié : l'écart
entre les deux branches (hors `alphalink/` et `README.md`) est exactement
`frr-10.5.5..stable/10.5`, et les 14 commits ont le même contenu des deux
côtés (`git patch-id`).

### 1 à 12 : métrique par route statique (PR FRR 21296 + suites)

Besoin Alphalink : pouvoir écrire `ip route P NH metric M` et faire choisir
zebra entre des chemins de même distance selon leur coût. Natif en FRR 10.7,
absent de 10.5.

| # | Commit | Origine amont | Rôle |
|---|---|---|---|
| 1 | `270270a865` | `bb126962e8` | identité du nexthop comme clé YANG. **Adapté** : le déplacement de `weight` vers path-list (10.7) n'est pas repris, `weight` reste par nexthop comme en 10.5 |
| 2-3 | `e5e5d71630` `ff8ee8dc53` | `b36ebad54e` `0a0ef87d53` | tests mgmt/grpc alignés sur le schéma |
| 4 | `58492e74e5` | `653affcb52` | la métrique devient une clé d'identité des routes statiques dans zebra. **Adapté** : 10.5 n'a pas le paramètre `replace` de `rib_compare_routes`, le test de métrique est étendu de KERNEL à STATIC |
| 5 | `16a6cf3b38` | `14a2414c59` | option `metric` de staticd (attribut modifiable, pas une clé). **Adapté** : contexte des chaînes d'aide 10.5 |
| 6 | `9dc0721d49` | `799ec77c7d` | documentation. **Adapté** : syntaxe 10.5 |
| 7-9 | `f40fbf6a89` `e31a5bfdea` `d9e5ae796d` | `c0c1f0839b` `6163ed7d21` `2f81a63d0f` | topotests distance, tag, métrique |
| 10 | `2a3885e8c2` | `f97435db48` | `ecmp_path_list_validate` sans tri XPath |
| 11-12 | `462454a7cb` `4e822e5b87` | `f6f6f318c6` `09f028eb40` (PR 23059) | pas de mise à jour parasite des statiques au rechargement, avec test |

Usage, mesuré en maquette : pour un même nexthop, une nouvelle métrique
remplace l'ancienne. Une bascule demande deux nexthops différents à la même
distance. La métrique sert au choix dans zebra, le noyau affiche toujours
`metric 20`.

### 13 : IPv4 via un voisin IPv6 sans objets nexthop (`11267be4d2`)

Repris tel quel de FRR 10.6 (PR 19498). Quand les objets nexthop noyau sont
désactivés (`no zebra nexthop kernel enable`, ou instances en `netns`, cas
des configurations Muli), FRR 10.5 installe une route IPv4 dont le voisin
est IPv6 via la fausse adresse `169.254.0.1`. Celle-ci ne marche que si les
RA IPv6 sont actifs des deux côtés du lien. Avec ce commit, zebra donne
directement le voisin IPv6 au noyau (`via inet6 fe80::…`), sans commande.

Il remplace la commande maison `zebra v4-via-v6 rta-via` (`8c333a6521`,
branche `stable/10.5-backport-with-via-inet6`), qui n'est **pas** reprise :
elle demandait une ligne dans chaque frr.conf et la réinstallation des
routes, et FRR 10.7 ne la connaît pas. Une conf qui la contient est
refusée au rechargement par ce paquet.

### 14 : départage à égalité, la route la plus récente gagne (`20af103935`)

Décision d'équipe du 13/10/2026 : on le garde.

Ce qu'il fait : quand deux routes ont la même distance et la même métrique,
FRR 10.5 d'origine garde la plus ancienne. Ce commit fait gagner la plus
récente (`<=` devient `<` dans `rib_choose_best` et
`rib_choose_best_type`). Son message annonce l'inverse ; le code fait
bien « la plus récente gagne », mesuré en maquette.

Pourquoi on le met : une route statique avec métrique sert de route par
défaut pour un préfixe, et une route BGP de même distance peut prendre sa
place. La priorité est pilotée en changeant la métrique de la route BGP
(table-map). À égalité de métrique, l'équipe veut que la route la plus
récente l'emporte.

À savoir avant de s'appuyer dessus :
- « récente » veut dire « dernière renvoyée à zebra », pas « dernière
  décision de l'opérateur ». Une réannonce BGP (changement de route-map,
  avec quelques secondes de délai, `clear`, session qui retombe) ou une
  statique renvoyée par staticd change le gagnant. Un écart strict de
  métrique (même d'une unité) donne un choix stable, avec ou sans ce commit.
- zebra n'installe jamais deux protocoles en même temps pour un préfixe :
  l'égalité ne fait pas d'ECMP statique + BGP, elle désigne un gagnant.
- l'effet est global : sur un concentrateur PPP, la route locale choisie
  pour l'adresse partagée par les interfaces ppp change à chaque nouvelle
  session au lieu de rester sur la plus ancienne.
- amont : absent de 10.5, 10.6 et 10.7 ; présent dans `stable/10.8` (pas
  encore publiée au 13/10/2026). Le comportement sera donc natif en 10.8.

### Ce qui n'est pas dans la branche

- `8c333a6521` commande `zebra v4-via-v6 rta-via` : remplacée par le 13.
- `f0de8f8892` (10.6) : corrige un défaut de zebra 10.5 avec deux routes
  noyau même préfixe, même métrique, interfaces différentes (la suppression
  de la plus récente fait oublier l'autre, `redistribute kernel` retire un
  préfixe vivant). Conflit avec le commit 4, à adapter avec son test.
- les autres correctifs 10.6/10.7 candidats : liste triée dans le rapport
  de revue du 12/10/2026 (`annexe-T6-candidats-amont.md`).

## Construire le paquet

Sur le builder Debian 12 (builder-12), dans un clone de ce dépôt :

    git switch stable/10.5.5-alphalink-minimal
    git fetch upstream --tags        # le tag frr-10.5.5 doit être présent
    export DEBFULLNAME="Prénom Nom" DEBEMAIL="prenom.nom@alphalink.fr"
    alphalink/build-deb.sh --rev 4            # => 10.5.5-0~deb12u1+alphalink4
    alphalink/build-deb.sh --rev 4 --rc 1     # => 10.5.5-0~deb12u1+alphalink4~rc1

Prérequis, une seule fois :
- source apt du dépôt FRR (`/etc/apt/sources.list.d/frr.list`) :
  `deb-src [signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr bookworm frr-10.5`
- dépendances de build : si le script s'arrête sur « build deps missing »,
  lancer la commande `sudo mk-build-deps -r -i debian/control` qu'il
  affiche, puis relancer.

Ce que fait le script, et où il s'arrête en cas d'écart :
1. vérifie que l'arbre est propre et descend du tag `frr-10.5.5` ;
2. génère un patch par commit depuis le tag (hors `alphalink/` et
   `README.md`), puis rejoue ces patches sur le tag dans un arbre
   temporaire et vérifie qu'on retombe exactement sur la branche ;
3. télécharge `frr=10.5.5-0~deb12u1` (version figée), vérifie le
   changelog et le format quilt, pose les patches dans `debian/patches` ;
4. fixe la version avec `dch` ;
5. lance `dpkg-buildpackage -us -uc -b`, journal dans `build.log` ;
6. contrôle dans le journal : nombre de patches appliqués = nombre généré,
   résumé pytest présent (`N passed`) sans échec, liste des `.deb`.

Les `.deb`, `build.log` et une copie des patches sont dans
`~/build-frr-<version>/`.

### Numéro de version

- `+alphalinkN` : N est obligatoire et doit être **supérieur** à tout
  `+alphalinkN` déjà publié dans `bookworm-lisos` ou installé quelque part,
  sinon apt ne mettra pas à jour. Des paquets `+alphalink2` et
  `+alphalink3` ont été construits par la CI à partir de
  `stable/10.5-backport` : partir de `--rev 4`, sauf si une version plus
  haute existe déjà. Vérifier avec `apt-cache policy frr` sur une machine
  qui voit le dépôt.
- `~rcK` : candidat à tester. `+alphalink4~rc1` < `+alphalink4~rc2` <
  `+alphalink4`, la version finale remplace donc les candidats.

### Pourquoi partir du tag et non de `stable/10.5`

Le paquet source Debian est 10.5.5 figé. Les anciens paquets généraient les
patches avec `git format-patch origin/stable/10.5..stable/10.5-backport` :
tout ce que `stable/10.5` contenait déjà (15 correctifs officiels) sortait
du calcul, alors que le paquet source ne l'avait pas. Résultat : des
paquets sans ces correctifs, sans aucune erreur au build (vérifié sur
`+alphalink3`). Partir du tag de la version du paquet source, et vérifier
que tag + patches = branche, ferme ce trou.

## Faire évoluer la branche

- Un correctif amont : `git cherry-pick -x <commit amont>` (garde la trace
  « cherry picked from »), un commit par correctif, puis une ligne dans la
  table ci-dessus.
- Le code FRR et l'outillage (`alphalink/`, `README.md`) ne se mélangent
  pas dans un même commit : le script refuse ce cas.
- Nouvelle version amont (10.5.6) : rebaser la branche sur le nouveau tag
  (elle recevra alors les 18 correctifs, inclus dans 10.5.6), puis mettre à jour
  `UPSTREAM_TAG` et `DEB_SRC_VERSION` en tête de `alphalink/build-deb.sh`.

## Tests réalisés

Cette branche n'a jamais été construite ni testée telle quelle. Ses 14
commits ont été testés au-dessus des 18 correctifs (branche
`stable/10.5.5-alphalink-fixes`, paquet `+alphalink4~rc2` pour les 13 premiers,
paquet de comparaison pour le 14). Ils s'appliquent sans conflit sur
`frr-10.5.5`, mais la compilation et la maquette restent à faire si cette
variante doit être livrée.


---

<p align="center">
<img src="http://docs.frrouting.org/en/latest/_static/frr-icon.svg" alt="Icon" width="20%"/>
</p>

FRRouting
=========

FRR is free software that implements and manages various IPv4 and IPv6 routing
protocols. It runs on nearly all distributions of Linux and BSD and
supports all modern CPU architectures.

FRR currently supports the following protocols:

* BGP
* OSPFv2
* OSPFv3
* RIPv1
* RIPv2
* RIPng
* IS-IS
* PIM-SM/MSDP
* LDP
* BFD
* Babel
* PBR
* OpenFabric
* VRRP
* EIGRP (alpha)
* NHRP (alpha)

Installation & Use
------------------

For source tarballs, see the
[releases page](https://github.com/FRRouting/frr/releases).

For Debian and its derivatives, use the APT repository at
[https://deb.frrouting.org/](https://deb.frrouting.org/).

Instructions on building and installing from source for supported platforms may
be found in the
[developer docs](http://docs.frrouting.org/projects/dev-guide/en/latest/building.html).

Once installed, please refer to the [user guide](http://docs.frrouting.org/)
for instructions on use.

Community
---------

The FRRouting email list server is located
[here](https://lists.frrouting.org/listinfo) and offers the following public
lists:

| Topic             | List                         |
|-------------------|------------------------------|
| Development       | dev@lists.frrouting.org      |
| Users & Operators | frog@lists.frrouting.org     |
| Announcements     | announce@lists.frrouting.org |

For chat, we currently use [Slack](https://frrouting.slack.com). You can join
by clicking the "Slack" link under the
[Participate](https://frrouting.org/community) section of our website.


Contributing
------------

FRR maintains [developer's documentation](http://docs.frrouting.org/projects/dev-guide/en/latest/index.html)
which contains the [project workflow](http://docs.frrouting.org/projects/dev-guide/en/latest/workflow.html)
and expectations for contributors. Some technical documentation on project
internals is also available.

We welcome and appreciate all contributions, no matter how small!


Security
--------

To report security issues, please use our security mailing list:

```
security [at] lists.frrouting.org
```
