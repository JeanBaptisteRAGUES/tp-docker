# TP — Docker avancé

Ce TP permet d'approfondir le fonctionnement interne de Docker, les mécanismes de sécurité et les outils de diagnostic.

---
## Exercice 1 — Comprendre le cycle de vie des processus dans un conteneur

**Objectif :** comprendre comment Docker gère les processus et pourquoi certains conteneurs s'arrêtent immédiatement.

### Étape 1 : Lancer un conteneur avec un processus actif

```bash
docker run -d --name processus-test alpine:3.22 sleep 300
```

### Étape 2 : Identifier les processus

```bash
docker top processus-test
docker inspect --format='PID hôte : {{.State.Pid}}' processus-test
```

Observez le processus exécuté dans le conteneur et son processus correspondant sur l'hôte.

### Étape 3 : Arrêter le conteneur

```bash
docker stop processus-test
```

Par défaut, Docker envoie un signal `SIGTERM` au processus principal, attend le délai prévu, puis peut envoyer `SIGKILL` si le processus ne s'arrête pas.

Vérifiez le résultat :

```bash
docker ps -a
docker inspect --format='État : {{.State.Status}}' processus-test
```

### Étape 4 : Comparer avec un processus de longue durée

Lancez un nouveau conteneur :

```bash
docker run -d --name processus-test-2 alpine:3.22 sleep 300
```

Puis envoyez un signal d'arrêt au processus principal :

```bash
docker kill --signal=TERM processus-test-2
```

Observez l'état du conteneur :

```bash
docker inspect --format='État : {{.State.Status}}' processus-test-2
```

### Étape 5 : Nettoyer

```bash
docker rm processus-test processus-test-2
```

### Questions

1. Quel est le rôle du processus principal dans un conteneur ?  
Le rôle du processus principal dans un conteneur est d'en être le point d'entrée de son exécution et de déterminer 
son cycle de vie :  
- Tant qu'il est actif, le conteneur reste en fonctionnement  
- Lorsqu'il se termine, le conteneur s'arrête  
Il est également le destinataire des signaux envoyés par Docker lors des opérations d'arrêt.  

2. Quelle différence existe-t-il entre `docker stop` et `docker kill` ?  
- docker stop demande un arrêt progressif du conteneur. Il envoie d'abord SIGTERM au processus principal, lui laissant le temps de terminer proprement. Si le processus ne s'arrête pas dans le délai imparti, Docker envoie automatiquement SIGKILL.  
- docker kill envoie directement un signal au processus principal. Par défaut, ce signal est SIGKILL, mais il est possible d'en choisir un autre avec l'option --signal.  

3. Pourquoi un conteneur s'arrête-t-il lorsque son processus principal se termine ?  
Docker utilise le processus principal (PID 1) pour déterminer l'état du conteneur.  
Lorsque le processus principal se termine, Docker considère que l'exécution du conteneur est terminée.  

4. Quelle différence existe-t-il entre `SIGTERM` et `SIGKILL` ?  
SIGTERM : demande au processus de se terminer. Il peut intercepter ce signal et effectuer des opérations avant de s'arrêter, comme sauvegarder des données ou fermer des connexions. Il peut également ignorer ce signal.  

SIGKILL : force immédiatement la terminaison du processus. Il ne peut être ni intercepté, ni ignoré, ni bloqué.  

5. Pourquoi est-il important qu'une application gère correctement les signaux d'arrêt ?  
Il est important qu'une application gère correctement les signaux d'arrêt afin de pouvoir terminer son exécution proprement.  
Sans cette gestion, une application risque d'être arrêtée brutalement par SIGKILL, ce qui peut entraîner une perte de données, des opérations incomplètes ou des ressources qui n'ont pas été correctement libérées.  

## Exercice 2 — Comprendre les couches d'une image et le système de fichiers

**Objectif :** comprendre comment Docker stocke les images et les modifications effectuées dans un conteneur.

### Étape 1 : Examiner les couches d'une image

Téléchargez une image Alpine :

```bash
docker pull alpine:3.22
```

Affichez l'historique de construction :

```bash
docker history alpine:3.22
```

Examinez ensuite les métadonnées de l'image :

```bash
docker image inspect alpine:3.22
```

Observez notamment les informations relatives aux couches de l'image.

### Étape 2 : Créer un conteneur de test

```bash
docker run -d \
  --name couches-test \
  alpine:3.22 \
  sh -c "sleep 600"
```

Le conteneur reste actif grâce à la commande `sleep`.

### Étape 3 : Modifier le système de fichiers

Créez un fichier et un répertoire dans le conteneur :

```bash
docker exec couches-test sh -c \
  "echo 'Modification Docker' > /tmp/test.txt"

docker exec couches-test mkdir /opt/test
```

Modifiez également un fichier existant :

```bash
docker exec couches-test sh -c \
  "echo '# modification locale' >> /etc/alpine-release"
```

### Étape 4 : Identifier les modifications

Exécutez :

```bash
docker diff couches-test
```

Les lignes affichées utilisent généralement les indicateurs suivants :

- `A` : élément ajouté.
- `C` : élément modifié.
- `D` : élément supprimé.

Vérifiez le contenu des fichiers :

```bash
docker exec couches-test cat /tmp/test.txt
docker exec couches-test cat /etc/alpine-release
```

### Étape 5 : Supprimer le conteneur

```bash
docker rm -f couches-test
```

Relancez ensuite un nouveau conteneur à partir de la même image :

```bash
docker run --rm alpine:3.22 cat /etc/alpine-release
```

La modification précédente n'est pas présente dans ce nouveau conteneur.

### Questions

1. Quelle différence existe-t-il entre les couches d'une image et la couche inscriptible d'un conteneur ?  
Une image Docker est constituée de plusieurs couches (en lecture seule) de fichiers empilées les unes sur les autres.  
Lorsqu'un conteneur est créé à partir d'une image, Docker ajoute une couche inscriptible au-dessus de ces couches. Cette couche permet d'enregistrer les modifications effectuées pendant l'exécution du conteneur, sans modifier les couches originales de l'image.  

2. À quoi sert le mécanisme *copy-on-write* ?  
Le mécanisme copy-on-write permet de partager les couches en lecture seule d'une image entre plusieurs conteneurs, sans avoir besoin de dupliquer l'intégralité de leur système de fichiers.  
Ce mécanisme permet donc d'économiser de l'espace disque et d'éviter de dupliquer inutilement les données.  

3. Pourquoi les modifications effectuées dans un conteneur ne modifient-elles pas directement l'image d'origine ?  
Les couches constituant une image Docker sont en lecture seule. Un conteneur ne peut donc pas les modifier directement.  
Toutes les modifications effectuées pendant son exécution sont enregistrées dans sa propre couche inscriptible, située au-dessus des couches de l'image.  
Cela permet de conserver l'image d'origine intacte et de créer plusieurs conteneurs indépendants à partir d'une même image.  

4. À quoi sert la commande `docker diff` ?  
La commande docker diff permet d'identifier les modifications apportées au système de fichiers d'un conteneur depuis sa création.  
Elle affiche les fichiers et répertoires ajoutés, modifiés ou supprimés par rapport à l'image d'origine.  

5. Pourquoi les modifications apportées à la couche inscriptible sont-elles perdues lorsqu'on supprime le conteneur ?  
La couche inscriptible appartient exclusivement au conteneur. Lorsqu'on le supprime, Docker supprime également cette couche et les modifications qu'elle contient. 
Les couches originales de l'image ne sont pas affectées puisqu'elles sont indépendantes du conteneur.  

---

## Exercice 3 — Construire une image avec plusieurs étapes (Multi-stage build)

**Objectif :** séparer l'environnement de compilation de l'environnement d'exécution pour produire une image finale plus légère.

Dans cet exercice, vous allez créer une petite application HTTP en Go et utiliser un build multi-stage.

### Étape 1 : Créer le projet

Créez le dossier suivant :

```text
tp-multistage/
├── main.go
└── Dockerfile
```

### Étape 2 : Écrire l'application

Dans `main.go`, ajoutez :

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Application Go executee dans Docker !")
	})

	log.Println("Serveur demarre sur le port 8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Étape 3 : Créer le Dockerfile multi-stage

Dans `Dockerfile`, écrivez :

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /src

COPY main.go .

RUN CGO_ENABLED=0 GOOS=linux go build -o /app main.go

FROM scratch

COPY --from=builder /app /app

EXPOSE 8080

ENTRYPOINT ["/app"]
```

**Explication :**

- `FROM golang:1.24-alpine AS builder` définit l'environnement de compilation.
- `RUN` compile le programme en un exécutable autonome.
- `FROM scratch` démarre une image finale vide.
- `COPY --from=builder` récupère uniquement l'exécutable produit pendant la première étape.
- `ENTRYPOINT` définit le programme exécuté au démarrage.

L'image `scratch` ne contient ni shell ni outils système classiques. L'application doit donc être compilée de manière compatible avec cet environnement.

### Étape 4 : Construire l'image

Depuis le dossier `tp-multistage`, exécutez :

```bash
docker build -t application-go:v1 .
```

Affichez la taille de l'image :

```bash
docker image ls application-go
```

### Étape 5 : Exécuter l'application

```bash
docker run -d \
  --name application-go-test \
  -p 8086:8080 \
  application-go:v1
```

Testez le serveur :

```bash
curl http://localhost:8086
```

**Résultat attendu :**

```text
Application Go executee dans Docker !
```

### Étape 6 : Examiner les processus et nettoyer

```bash
docker top application-go-test
docker logs application-go-test
docker rm -f application-go-test
```

### Questions

1. Pourquoi utiliser plusieurs instructions `FROM` ?  
L'utilisation de plusieurs instructions FROM permet de créer plusieurs étapes de construction indépendantes au sein d'un même Dockerfile.  
Ça permet notamment de séparer l'environnement nécessaire à la compilation d'une application de celui nécessaire à son exécution.  
Ça permet donc de réduire la taille de l'image finale et de ne pas embarquer les outils de compilation qui ne sont plus nécessaires à l'exécution.  

2. Quelle différence existe-t-il entre l'étape `builder` et l'étape finale ?  
L'étape builder correspond à l'environnement de compilation.  
L'étape finale correspond à l'environnement d'exécution.  


3. Pourquoi l'image finale ne contient-elle pas le compilateur Go ?  
Le compilateur Go est uniquement nécessaire pendant la phase de construction de l'application.  
Le compilateur Go et les autres éléments de l'environnement de compilation ne sont donc pas copiés dans l'image finale.  
Ça permet de réduire considérablement la taille de l'image.  

4. Quels sont les avantages d'une image finale minimale ?  
Une image finale minimale présente plusieurs avantages :  
- Réduction de la taille : elle nécessite moins d'espace disque et consomme moins de bande passante lors de son transfert.  
- Déploiement plus rapide : son téléchargement et son déploiement sont généralement plus rapides.  
- Sécurité renforcée : elle contient moins de composants susceptibles de présenter des vulnérabilités.  
- Surface d'attaque réduite : l'absence d'outils système classiques et de shell limite les possibilités d'exploitation en cas de compromission du conteneur.  
- Maintenance simplifiée : moins de composants signifie également moins d'éléments à mettre à jour.  

5. Quelles précautions faut-il prendre avant d'utiliser une image `scratch` en production ?  
L'image scratch étant totalement vide, il faut s'assurer que l'application peut fonctionner sans système d'exploitation complet ni outils supplémentaires, notamment avec des tests.  

---

## Exercice 4 — Renforcer la sécurité d'un conteneur

**Objectif :** limiter les privilèges et les possibilités de modification du système de fichiers d'un conteneur.

### Étape 1 : Exécuter un conteneur avec un utilisateur non privilégié

```bash
docker run --rm \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'id'
```

Observez l'identifiant de l'utilisateur et son groupe.

### Étape 2 : Tester les permissions

Exécutez :

```bash
docker run --rm \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'touch /root/test.txt'
```

**Résultat attendu :** l'opération échoue normalement, car l'utilisateur n'a pas les permissions nécessaires.

### Étape 3 : Exécuter un conteneur en lecture seule

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'id; touch /tmp/test.txt; echo "Fichier temporaire cree"'
```

Les options utilisées permettent de :

- `--read-only` : rendre le système de fichiers racine du conteneur non modifiable.
- `--tmpfs` : fournir un espace temporaire en mémoire pour les fichiers qui doivent être écrits.
- `--cap-drop=ALL` : supprimer les capacités Linux supplémentaires du processus.
- `--security-opt=no-new-privileges` : empêcher l'acquisition de privilèges supplémentaires via les mécanismes concernés.
- `--user` : exécuter le processus avec un identifiant non nul.

### Étape 4 : Vérifier les limites d'écriture

Essayez de créer un fichier dans le répertoire racine :

```bash
docker run --rm \
  --read-only \
  --user 10001:10001 \
  alpine:3.22 \
  sh -c 'touch /test.txt'
```

**Résultat attendu :** l'écriture échoue.

### Étape 5 : Comparer les configurations

Lancez un conteneur avec les options de sécurité précédentes, puis examinez sa configuration avec :

```bash
docker inspect --format='Utilisateur : {{.Config.User}}' NOM_DU_CONTENEUR
docker inspect --format='Lecture seule : {{.HostConfig.ReadonlyRootfs}}' NOM_DU_CONTENEUR
```

Remplacez `NOM_DU_CONTENEUR` par le nom d'un conteneur actif créé pour cette vérification.

### Questions

1. Pourquoi est-il préférable d'exécuter une application avec un utilisateur non privilégié ?  
Il est préférable d'exécuter une application avec un utilisateur non privilégié afin de limiter les conséquences d'une éventuelle faille de sécurité.  

2. Que sont les capacités Linux ?  
Les capacités Linux (Linux capabilities) sont un mécanisme permettant de diviser les privilèges traditionnellement associés à l'utilisateur root en plusieurs permissions indépendantes.  
Au lieu d'accorder tous les privilèges administrateur à un processus, on peut lui attribuer uniquement certaines capacités.  

3. Quelle différence existe-t-il entre `--read-only` et `--tmpfs` ?  
Ces deux options ont des rôles complémentaires :  
- --read-only rend le système de fichiers racine du conteneur non modifiable. Les processus ne peuvent donc plus écrire librement dans les répertoires du système. 
- --tmpfs crée un espace temporaire inscriptible en mémoire, qui permet aux applications de stocker temporairement des fichiers.  

4. Pourquoi un conteneur peut-il avoir besoin de zones inscriptibles malgré un système de fichiers racine en lecture seule ?  
Certaines applications ont besoin d'écrire des fichiers pendant leur fonctionnement.  
Rendre l'ensemble du système de fichiers inscriptible n'est pas nécessaire pour répondre à ces besoins.  
On peut donc conserver une racine en lecture seule tout en autorisant l'écriture dans des emplacements précis, par exemple avec --tmpfs ou des volumes Docker.  
Cela permet de limiter les modifications possibles tout en conservant le fonctionnement de l'application.  

5. Ces options suffisent-elles à garantir une isolation parfaite ? Justifiez votre réponse.  
Non, ces options renforcent la sécurité du conteneur, mais ne garantissent pas une isolation parfaite.  
Cependant, les conteneurs Docker partagent le noyau Linux de la machine hôte. Une vulnérabilité du noyau ou du moteur Docker pourrait donc compromettre l'isolation entre le conteneur et l'hôte.  

---

## Exercice 5 — Diagnostiquer un conteneur qui se termine anormalement

**Objectif :** analyser le code de sortie d'un processus et exploiter les journaux pour identifier un problème.

### Étape 1 : Simuler une application en erreur

Lancez un conteneur dont le processus retourne volontairement un code d'erreur :

```bash
docker run \
  --name diagnostic-test \
  alpine:3.22 \
  sh -c 'echo "Demarrage de l application"; echo "Erreur simulee" >&2; exit 42'
```

Le conteneur doit se terminer avec le code de sortie `42`.

### Étape 2 : Examiner l'état du conteneur

```bash
docker ps -a
```

Affichez son code de sortie :

```bash
docker inspect \
  --format='Code de sortie : {{.State.ExitCode}}' \
  diagnostic-test
```

### Étape 3 : Consulter les journaux

```bash
docker logs diagnostic-test
```

Les journaux doivent permettre de retrouver les messages émis par le processus, notamment celui envoyé sur la sortie d'erreur standard.

### Étape 4 : Inspecter les métadonnées

```bash
docker inspect diagnostic-test
```

Recherchez notamment les champs :

- `State.Status`
- `State.ExitCode`
- `State.Error`
- `Config.Cmd`
- `Config.Image`

### Étape 5 : Nettoyer

```bash
docker rm diagnostic-test
```

### Questions

1. Quelle différence existe-t-il entre un conteneur arrêté et un conteneur supprimé ?  
Un conteneur arrêté existe toujours dans Docker, mais son processus principal ne s'exécute plus.  
Un conteneur supprimé, en revanche, n'existe plus dans Docker. Ses métadonnées et sa couche inscriptible sont supprimées. Il n'est donc plus possible de consulter ses journaux ou de le redémarrer.  

2. Que signifie un code de sortie différent de zéro ?  
Un code de sortie différent de zéro indique généralement que le processus s'est terminé avec une erreur ou qu'une situation particulière s'est produite.  

3. Quelle différence existe-t-il entre les sorties standard et d'erreur ?  
stdout (sortie standard) : utilisée pour afficher les résultats normaux d'un programme.  
stderr (sortie d'erreur) : utilisée pour afficher les messages d'erreur, les avertissements ou les informations de diagnostic.  

4. Pourquoi les journaux sont-ils utiles lorsqu'un conteneur se termine trop rapidement ?  
Lorsqu'un conteneur se termine rapidement, il peut être difficile d'observer directement ce qui s'est passé pendant son exécution.  
Les journaux permettent de consulter les messages envoyés sur stdout et stderr afin de comprendre les dernières opérations effectuées et d'identifier une éventuelle erreur.  

5. Pourquoi faut-il examiner le code de sortie au lieu de se fier uniquement au message d'erreur ?  
Le message d'erreur fournit des informations sur le problème rencontré, mais il ne permet pas toujours de déterminer précisément comment le processus s'est terminé.  
Le code de sortie fournit une information supplémentaire et exploitable par les outils de supervision et les scripts.  

---

## Exercice 6 — Configurer la rotation des journaux

**Objectif :** comprendre comment éviter une consommation excessive de l'espace disque par les journaux Docker.

### Étape 1 : Créer un conteneur générant des journaux

```bash
docker run -d \
  --name logs-test \
  --log-driver local \
  --log-opt max-size=1m \
  --log-opt max-file=2 \
  alpine:3.22 \
  sh -c 'i=0; while [ "$i" -lt 10000 ]; do echo "Journal numero $i"; i=$((i+1)); done; sleep 600'
```

Le conteneur génère des messages puis reste actif.

### Étape 2 : Consulter les journaux

```bash
docker logs --tail 20 logs-test
```

Affichez la configuration du pilote de logs :

```bash
docker inspect \
  --format='Pilote : {{.HostConfig.LogConfig.Type}}' \
  logs-test
```

Puis affichez les options configurées :

```bash
docker inspect \
  --format='{{json .HostConfig.LogConfig.Config}}' \
  logs-test
```

### Étape 3 : Comprendre la rotation

Dans cette configuration :

- `--log-driver local` utilise le pilote de journaux local de Docker.
- `max-size=1m` fixe une taille cible de rotation par fichier.
- `max-file=2` limite le nombre de fichiers conservés par la rotation.

Ces paramètres évitent une croissance illimitée des fichiers de logs gérés par ce pilote. Ils ne constituent pas une limite globale de stockage pour tout le conteneur.

### Étape 4 : Comparer avec une configuration sans rotation explicite

Inspectez les paramètres de journalisation d'un autre conteneur :

```bash
docker inspect \
  --format='Pilote : {{.HostConfig.LogConfig.Type}} ; Options : {{json .HostConfig.LogConfig.Config}}' \
  NOM_DU_CONTENEUR
```

Remplacez le nom par celui d'un conteneur existant.

Comparez les résultats avec la configuration de `logs-test`.

### Étape 5 : Nettoyer

```bash
docker rm -f logs-test
```

### Questions

1. Pourquoi des journaux trop volumineux peuvent-ils poser problème ?  
Des journaux trop volumineux peuvent consommer une quantité importante d'espace disque sur la machine hôte.  
Si leur taille augmente continuellement, ils peuvent finir par saturer le stockage disponible.  

2. À quoi sert un pilote de journaux Docker ?  
Un pilote de journaux (logging driver) détermine la manière dont Docker collecte, stocke et éventuellement transmet les journaux produits par les conteneurs.  

3. Quelle différence existe-t-il entre la consultation des journaux avec `docker logs` et leur stockage sur le long terme ?  
La commande docker logs permet de consulter les journaux associés à un conteneur, notamment les messages envoyés sur stdout et stderr.  
Le stockage sur le long terme répond à un autre objectif : conserver les journaux pendant une période prolongée afin de pouvoir effectuer des analyses historiques, des audits ou rechercher des incidents passés.  

4. Pourquoi la rotation des journaux est-elle importante sur un serveur ?  
La rotation des journaux permet de limiter l'espace disque utilisé par les fichiers de journalisation.  
Lorsqu'un fichier atteint une certaine taille, il est remplacé ou archivé selon la stratégie de rotation configurée.  

5. Pourquoi faut-il choisir une stratégie de journalisation adaptée à l'environnement de production ?  
Une stratégie adaptée permet de maîtriser l'utilisation du stockage tout en conservant les informations nécessaires à la maintenance, à la sécurité et au diagnostic des applications.  

---

## Exercice 7 — Explorer les processus et les modifications d'un conteneur en fonctionnement

**Objectif :** utiliser les outils de diagnostic Docker pour comprendre le comportement d'un conteneur actif.

### Étape 1 : Lancer un conteneur

```bash
docker run -d \
  --name inspection-test \
  alpine:3.22 \
  sh -c 'while true; do sleep 30; done'
```

### Étape 2 : Examiner les processus

```bash
docker top inspection-test
```

Observez les processus associés au conteneur.

### Étape 3 : Exécuter des commandes dans le conteneur

```bash
docker exec inspection-test ps
docker exec inspection-test sh -c 'echo "Test de diagnostic" > /tmp/diagnostic.txt'
docker exec inspection-test cat /tmp/diagnostic.txt
```

`docker exec` lance un nouveau processus dans un conteneur déjà actif. Il ne remplace pas le processus principal.

### Étape 4 : Comparer le système de fichiers

```bash
docker diff inspection-test
```

Repérez le fichier `/tmp/diagnostic.txt` dans la liste des modifications.

### Étape 5 : Inspecter les ressources

```bash
docker stats --no-stream inspection-test
```

Observez les informations sur le CPU, la mémoire et les entrées/sorties.

### Étape 6 : Nettoyer

```bash
docker rm -f inspection-test
```

### Questions

1. Quelle différence existe-t-il entre `docker exec` et `docker run` ?  
docker run crée et démarre un conteneur, tandis que docker exec permet d'interagir avec un conteneur déjà en fonctionnement.  

2. À quoi sert `docker top` ?  
La commande docker top permet d'afficher la liste des processus actuellement exécutés dans un conteneur.  

3. Pourquoi `docker diff` peut-il être utile pendant un diagnostic ?  
La commande docker diff permet d'identifier les modifications apportées au système de fichiers d'un conteneur depuis sa création, par comparaison avec son image initiale.  
Elle epeut être utile pour repérer des modifications inattendues, vérifier les fichiers créés par une application ou identifier des changements manuels effectués dans un conteneur.  

4. Quelles informations fournit `docker stats` ?  
La commande docker stats fournit des statistiques sur l'utilisation des ressources d'un conteneur en fonctionnement.  
Elle affiche notamment :  
- CPU % : pourcentage d'utilisation du processeur.  
- MEM USAGE / LIMIT : mémoire utilisée et limite mémoire.  
- MEM % : pourcentage de mémoire utilisée par rapport à la limite.  
- NET I/O : quantité de données réseau reçues et envoyées.  
- BLOCK I/O : quantité de données lues et écrites sur les périphériques de stockage.  
- PIDS : nombre de processus présents dans le conteneur.  

5. Pourquoi faut-il éviter de modifier manuellement un conteneur en production sans documenter les changements ?  
Un conteneur en production doit idéalement être reproductible à partir d'une configuration connue, plutôt que dépendre de modifications manuelles réalisées pendant son fonctionnement.  

---

### Livrables attendus

1. Les réponses argumentées aux questions.
2. Une courte conclusion sur les améliorations apportées à l'image et à la sécurité du conteneur.  
Ce TP a permis d'améliorer l'image Docker en utilisant une construction multi-stage. Cette approche permet de séparer l'environnement de compilation de l'environnement d'exécution et de ne conserver dans l'image finale que les éléments nécessaires, notamment l'exécutable. L'image est ainsi plus légère, ce qui réduit l'espace de stockage et la surface d'attaque.

La sécurité du conteneur a également été renforcée grâce à plusieurs mesures : exécution avec un utilisateur non privilégié, système de fichiers en lecture seule, utilisation d'un espace temporaire limité avec tmpfs, suppression des capacités Linux inutiles et activation de no-new-privileges. Ces protections limitent les possibilités d'action d'un processus compromis et réduisent les risques liés à l'exécution du conteneur.
---

**Fin du TP Docker avancé**
