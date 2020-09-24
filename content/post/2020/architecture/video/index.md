---
title: La video
date: 2020-09-01
hero: /post/2020/architecture/video/images/archi-video.svg
excerpt: Le fonctionnement de la plateforme vidéo de MYTF1
authors:
  - dlecorfec
---

La vidéo est un domaine assez large, avec pas mal d'acronymes et de formats exotiques. Nous allons y aller progressivement 😉

## Types de vidéos

Nous distinguons 2 types de vidéos:

- les flux live, c'est à dire le direct des chaines TF1, TMC, TFX, TF1 Séries Films, LCI et lives évenementiels. Ils proviennent d'un _encodage_ en temps réel d'un flux vidéo "broadcast" vers un format de diffusion vidéo "informatique". Nous appelerons cette partie "live"
- les replays, extraits, spots publicitaires et bonus digitaux que nous regrouperons ici sous l'appelation "replay", et qui subissent des _transcodages_ vers différents formats pour les différents écrans de diffusion (dont les capacités varient)

## Modes de diffusion

Les contenus MYTF1 sont mis à disposition des internautes de 2 manières différentes:

- en OTT (_over-the-top_, terme consacré pour la diffusion via Internet) via notre infrastructure ou des services tiers que nous payons (CDN - Content Delivery Networks). Ici notre enjeu est d'offrir la meilleure expérience au plus grand nombre, en terme de qualité visuelle, de latence et d'accessibilité, tout en minimisant nos coûts de diffusion (ce qui est primordial pour une activité rémunérée principalement par la publicité), sans oublier la protection des contenus des ayants droit.
- via des partenaires qui assurent la diffusion: c'est le cas pour le portail MYTF1 sur la plupart des box actuelles, ou de SALTO, nous leur envoyons la vidéo aux formats demandés.

## Description d'un fichier vidéo

Avant de parler de la manière dont nous diffusons sur le net, voyons quelques notions sur les fichiers vidéo et leurs formats.

### Propriétés d'une vidéo

Une vidéo a une dimension spatiale et une dimension temporelle.
Au niveau spatial, elle a une _résolution_ exprimée en pixels horizontaux et pixels verticaux (par exemple, 1280x720).
Au niveau temporel, elle a une durée et un nombre constant d'images par seconde (ips, ou en anglais, frames per second, fps), 25 dans notre cas. Au niveau audio, on parle d'échantillons par seconde, par exemple 48000, ou de fréquence d'échantillonage, soit ici 48 kHz.

Pour quantifier la capacité réseau nécessaire à la lecture d'une vidéo, le terme _bitrate_ est employé et désigne la quantité de données nécessaire pour diffuser 1 seconde de vidéo.
Ce terme est généralement en kilobits ou megabits par seconde (1000 kbps = 1 mbps, et 8 bits = 1 octet). Une vidéo a 500 kbps prendra 10 fois moins de débit qu'une vidéo à 5 mbps, mais aura une qualité et/ou une résolution inférieure.

Théoriquement, une image en 1280x720 pixels avec 3 octets par pixels prendrait 2.7 Mo. Ce qui donnerait une vidéo à 550 mbps (soit environ 70 Mo/s). De la compression est donc nécessaire.

### Compression

Les pistes audio et vidéo sont compressées avec perte, ce qui permet de réduire énormément leur taille, et de les diffuser sur des réseaux avec des capacités limitées.
Il y a donc un compromis à faire entre taille et qualité.

Pour rendre la perte la moins significative possible, les algorithmes de compression usuels se basent notamment sur des modèles psychoacoustiques et psychovisuels: certaines informations sont plus importantes que d'autres pour nos oreilles, nos yeux et notre cerveau, qui interprète ces données.
Par exemple, dans le domaine audio, les plus hautes fréquences sont inaudibles par l'oreille humaine,
tandis que dans le domaine visuel, l'oeil est plus réceptif aux changements de lumière qu'aux changements
de teinte, ou plus réceptif aux basses fréquences (l'aspect général d'une zone d'une image, plutôt que
les détails les plus fins).

Dans le cas de la vidéo, la compression interviendra au niveau d'une image, qui sera découpée en petits blocs, chaque bloc étant compressé individuellement. C'est pour ça qu'on peut parfois voir apparaître
des blocs dans une vidéo, si la compression est trop aggressive.

Souvent, il y a peu de différences entre 2 images successives. On va donc stocker une image et les différences avec les images suivantes plutôt qu'une série d'images. Mais on ne peut pas stocker des
différences indéfiniment, que ce soit lors d'un changement de plan ou pour sauter plus loin dans la vidéo.

On aura donc des groupes d'images, composés d'une image complète (image de référence, appelée I-frame) et des différences (prédictives - P-frame - ou bi-prédictives - B-frame) nécessaires pour
reconstruire les images suivantes. Généralement ces groupes ne constituent pas plus de 2 secondes de vidéo.
Ces groupes sont appelés des GOP (group of pictures).

La compression/décompression, aspect essentiel et très pointu de la diffusion vidéo, est implémentée par des _codecs_: il faut non seulement qu'ils soient connus par le système qui encode les vidéos, mais surtout par les sytèmes de ceux qui les lisent. Avec le développement de la mobilité, un codec n'est
utilisable en pratique que s'il dispose d'une version accélérée matériellement sur la plupart des smartphones, tablettes et ordinateurs, afin de garantir de bonnes performances et une consommation énergétique raisonnable. Mais ça explique pourquoi les codecs les plus répandus ont toujours au moins 10 ans d'existence, le temps que les implémentations matérielles se généralisent.

Parmi les codecs que nous utilisons actuellement, on peut citer H.264 pour la vidéo, et AAC pour l'audio.

### Formats de fichiers

Dans un fichier vidéo, on peut la plupart du temps distinguer le contenant du contenu: un fichier vidéo sera le plus souvent un conteneur (par exemple, au format MPEG-TS ou MPEG-4) organisant ses données (notamment celles produites par les codecs audio/vidéo), et permettant de les retrouver facilement.
Ce conteneur aura le plus souvent au moins une piste vidéo et une piste audio, parfois plusieurs, et parfois d'autres types de pistes comme les sous-titres.

Par exemple, on peut avoir un fichier MP4 qui contient une piste vidéo au format H.264, une piste audio au format AAC et une piste de sous-titres au format WebVTT.

## Formats de diffusion OTT

Au niveau des formats de diffusion OTT (c'est à dire, via Internet), nous supportons les formats suivants:

- HLS ("HTTP Live Streaming", sur apps iOS et Safari Mobile)
- DASH ("Dynamic Adaptive Streaming over HTTP", sur le reste)
- MP4 ("MPEG-4", vous voilà bien avancés, pour les courts spots de pub des replays)

HLS et DASH sont des formats de diffusion développés pour la diffusion sur Internet via HTTP: la vidéo est transcodée en différentes qualités et segmentée en petits fichiers vidéos indépendants (_chunks_) de quelques secondes, ce qui permet au player de s'adapter en cours de visionnage en téléchargeant la qualité la plus appropriée à sa capacité actuelle de téléchargement. De plus, la segmentation en petits fichiers permet de faciliter la mise en cache et la robustesse en cas d'erreur (il suffit de redemander le fichier posant problème)
Le player va d'abord récupérer un fichier contenant du texte (_playlist M3U8_ pour HLS ou _manifest XML_ pour DASH).
Dans le cas des playlists HLS, le premier fichier est la _main playlist_, qui va référencer d'autres playlists, les _sub playlists_, mais considérons ces playlists comme un seul fichier pour l'explication.
Ce fichier textuel contient des métadonnées sur la durée des chunks, les bitrates disponibles,
les différentes pistes disponibles (vidéo, audio, sous-titres) et finalement décrit le moyen de récupérer
les pistes, soit en listant explicitement les fichiers en HLS, soit par un mécanisme de modèle (_template_) en fonction du timing ou du numéro de séquence du chunk en DASH.

Pour la protection contre la copie, nous utilisons sur les replays en DASH les DRM Widevine (DRM Google: players sous Chrome, Firefox, Android ...) et Playready (DRM Microsoft, donc players sous Edge) et sur les replays en HLS la DRM Fairplay (Apple)

Les manifests et chunks pour les différents formats possibles pour une vidéo ne sont pas stockés de manière permanente, ils sont générés à la demande et mis en cache.

## Plusieurs sources pour les vidéos sur MYTF1

Les vidéos peuvent provenir de différentes sources:

- le MAM TF1 (media assets manager), notre bibliothèque de programmes prêts à être diffusés (séries, émissions enregistrées ...), avec éventuellement plusieurs pistes audio (VO et/ou audiodescription) et sous-titres (pour VOST et/ou pour sourds et malentendants). Les contenus sont receptionnés quelques heures avant la diffusion, ce qui généralement nous permet une mise en ligne dès la fin de la diffusion antenne.
- le DVR (digital video recorder): nous enregistrons en HLS le direct des chaines et le gardons quelques jours. C'est à partir de là que nous livrons les replays des émissions en direct (JT notamment). Nous commandons une vidéo d'un intervalle de temps (à l'image près) d'une chaine. Ce système va identifier les chunks vidéos nécessaires, puis les recoller (facile, le format MPEG-TS permet une concaténation de fichiers) et rogner les bouts superflus afin de livrer un MP4 correspondant parfaitement à l'intervalle demandé.
- le FTP: on peut livrer une vidéo par simple transfert de fichier. C'est utilisé pour ce qui ne passe pas à l'antenne (contenus AVOD TFOUMAX et MYTF1, bonus digitaux ...)

## Plusieurs activités dans la gestion de la vidéo

La vidéo chez MYTF1 peut se décomposer en 2 grandes parties:

### Gestion des métadonnées live et replay (titre, résumé, dates de diffusion antenne et/ou de disponibilité sur MYTF1, ...) et des mises en ligne

- un backoffice éditorial de commande de "replay" (développé en interne)
- mise à disposition de ces informations aux autres services de MYTF1 et aux partenaires via différentes API et files de messages
- services pour les players (récupération des métadonnées et de l'URL de diffusion, protection de certains contenus via DRM - Digital Rights Management)

### Gestion des données vidéo live et replay

- ingestion (encodage/transcodage, gestion des sous-titres éventuels, packaging - préparation à la diffusion OTT)
- envoi aux partenaires, pour les vidéos (le live IPTV est géré par TF1)
- diffusion OTT (génération des formats HLS/DASH/MP4 éventuellement DRM-isés, caches, transit entre notre datacenter et les FAI, CDN)

La partie cache et transit est primordiale pour nos maîtrise des coûts de diffusion, afin d'utiliser le moins possible les services de CDN.
C'est pour cela qu'il doit être rapide de basculer la diffusion vidéo d'un point vers un autre, en fonction des besoins.

## Technos utilisées dans la vidéo

Une grande partie de nos services est développée en interne grâce à des projets OpenSource, mais nous avons recours à des systèmes propriétaires pour certains aspects très techniques (encodage/transcodage, packaging et génération à la volée des différents formats)

### Dans la partie métadonnées

Le service MOVE ("Outil Vidéo Multi-Ecrans"), qui est notre backoffice de commande de replays, de découpe d'extraits et de livraison aux partenaires, est écrit en PHP/Symfony avec du MySQL derrière.
Le service de réferentiel vidéo, qui regroupe toutes les métadonnées des vidéos, a une API écrite en NodeJS et une autre en Go. Son stockage primaire est une base Postgresql (avec utilisation de champs JSON)
Le système de notifications de changement de métadonnées est architecturé autour de RabbitMQ.
Les services de mises à jour des métadonnées vidéo coté publicité sont écrits en Go.
Le service de métadonnées vidéo (mediainfo) appelé par les players est écrit en Go.
Au niveau DRM, nous avons le service Widevine et le service Fairplay qui sont écrits en Go, et le service Playready qui est écrit en C# (car SDK .NET)

### Dans la partie vidéo proprement dite

Le pilotage des transcodages est effectué par un outil (videoworkflow), écrit en Go et s'appuyant sur RabbitMQ pour la communication entre ses différentes étapes, et ffmpeg pour certaines opérations (récupération des métadonnées de la vidéo, génération de la petite vidéo muette de preview, conversion de MPEG TS en MPEG PS pour certains opérateurs).
Les transcodeurs sont des Elemental Server. Ce sont des serveurs propriétaires avec des GPU pour accélérer les traitements. Ils disposent d'un backoffice web et d'une API REST, par lesquels on peut créer des profils d'encodage et soumettre des jobs.
Le système de génération à la demande des différents formats vidéo, avec gestion des DRM et des sous-titres, est également propriétaire, de chez Unified Streaming.
Nos caches sont basés sur l'excellent serveur Web nginx, avec des serveurs physiques gavés de RAM et de disque.
Le sytème de commande DVR s'appuie sur ffmpeg et est écrit en Go.
