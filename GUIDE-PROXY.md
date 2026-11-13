# Guide : proxy et pare-feu anti-récursion pour Kasm

Objectif : les sessions Chromium peuvent aller partout, sauf sur votre portail,
votre serveur Kasm, votre réseau interne et votre liste noire, et elles ne
peuvent pas contourner le proxy.

Deux couches :

1. **Squid** (proxy) applique la liste noire.
2. **Pare-feu** (iptables) interdit aux sessions toute sortie qui ne passe pas par Squid.

Sans la couche 2, un utilisateur peut contourner le proxy en changeant les
réglages du navigateur. Les commandes ci-dessous n'ont pas été testées sur
votre serveur : essayez-les d'abord sur un environnement de test.

## 0. Prérequis

- Kasm Workspaces installé, sur Linux avec Docker.
- Accès administrateur (sudo) sur le serveur.
- Le sous-réseau des sessions :

```bash
docker network inspect kasm_default_network | grep Subnet
```

Dans la suite : sous-réseau `172.18.0.0/16`, IP de Squid `172.18.0.250`.
Remplacez par vos valeurs.

## 1. Fichiers de configuration Squid

```bash
sudo mkdir -p /opt/squid && cd /opt/squid
```

Copiez `squid.conf` dans `/opt/squid/`, puis adaptez-y le sous-réseau et l'IP
publique de votre serveur. Créez ensuite les deux listes :

`/opt/squid/blocked_own.txt` : votre portail et votre serveur Kasm
(le point initial inclut les sous-domaines)

```
.portail.mondomaine.fr
.kasm.mondomaine.fr
```

`/opt/squid/blocked_domains.txt` : votre liste noire (un domaine par ligne)

```
.exemple-a-bloquer.com
```

## 2. Lancer Squid sur le réseau de Kasm

```bash
docker run -d --name squid --restart unless-stopped \
  --network kasm_default_network --ip 172.18.0.250 \
  -v /opt/squid/squid.conf:/etc/squid/squid.conf:ro \
  -v /opt/squid/blocked_own.txt:/etc/squid/blocked_own.txt:ro \
  -v /opt/squid/blocked_domains.txt:/etc/squid/blocked_domains.txt:ro \
  ubuntu/squid
```

Vérifiez : `docker logs squid` ne doit afficher aucune erreur.

Pour modifier la liste noire plus tard : éditez `blocked_domains.txt`, puis

```bash
docker exec squid squid -k reconfigure
```

## 3. Dire aux sessions d'utiliser le proxy

Dans l'interface d'administration de Kasm : **Workspaces**, choisissez votre
espace Chromium, puis **Edit**. Dans le champ **Docker Run Config Override
(JSON)**, mettez :

```json
{
  "environment": {
    "http_proxy": "http://172.18.0.250:3128",
    "https_proxy": "http://172.18.0.250:3128",
    "HTTP_PROXY": "http://172.18.0.250:3128",
    "HTTPS_PROXY": "http://172.18.0.250:3128",
    "no_proxy": "localhost,127.0.0.1",
    "NO_PROXY": "localhost,127.0.0.1"
  }
}
```

Le comportement de Chromium vis-à-vis de ces variables dépend de l'image Kasm
utilisée. Si le test (étape 5) montre que Chromium les ignore, la solution
robuste est une stratégie Chromium « ProxyMode: fixed_servers » montée dans le
conteneur. La documentation Kasm sur les politiques Chromium et les montages de
volumes explique comment faire.

## 4. Verrouiller la sortie (pare-feu)

Les règles autorisent Squid à sortir, et les sessions à ne parler qu'à leur
réseau et à Squid. Tout le reste est coupé.

```bash
SUBNET=172.18.0.0/16
SQUID=172.18.0.250

# 1. Squid peut sortir
sudo iptables -I DOCKER-USER -s $SQUID -j ACCEPT
# 2. Les réponses aux connexions déjà établies passent
sudo iptables -A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# 3. Les sessions peuvent joindre Squid
sudo iptables -A DOCKER-USER -s $SUBNET -d $SQUID -p tcp --dport 3128 -j ACCEPT
# 4. Tout autre trafic des sessions vers l'extérieur du réseau est refusé
sudo iptables -A DOCKER-USER -s $SUBNET ! -d $SUBNET -j DROP
```

IPv6 : désactivez-le sur le réseau Docker (c'est le cas par défaut), sinon
ajoutez des règles `ip6tables` équivalentes.

Pour conserver les règles après un redémarrage :

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

## 5. Tests à faire

Lancez une session Kasm, puis, depuis le serveur :

```bash
CID=$(docker ps --filter "name=kasm" --format '{{.Names}}' | head -1)   # un conteneur de session
docker exec $CID curl -sI -x http://172.18.0.250:3128 https://chatgpt.com   # doit réussir
docker exec $CID curl -sI -x http://172.18.0.250:3128 https://portail.mondomaine.fr   # doit être refusé (403)
docker exec $CID curl -sI -x http://172.18.0.250:3128 http://192.168.1.1   # doit être refusé
docker exec $CID curl -sI -x http://172.18.0.250:3128 http://169.254.169.254   # doit être refusé
docker exec $CID curl -sI --noproxy '*' --max-time 5 https://chatgpt.com   # sans proxy : doit échouer
```

Puis dans la session, depuis l'interface de Chromium :

- ouvrez `chatgpt.com` : doit s'afficher ;
- tapez l'adresse de votre portail : doit être bloquée ;
- tentez de changer les réglages de proxy du navigateur : l'accès à Internet doit
  de toute façon rester impossible sans Squid.

## 6. Ce qu'il faut savoir

- **Visioconférence et temps réel.** Le pare-feu coupe l'UDP direct, donc les
  appels vidéo (WebRTC) ne marcheront pas dans les sessions. Pour les autoriser,
  il faut un relais dédié.
- **Filtrage par domaine uniquement.** Squid voit le nom du site, pas l'adresse
  complète des pages, car le trafic HTTPS reste chiffré. Pour filtrer par page,
  il faudrait déchiffrer le trafic, ce qui pose des questions de vie privée
  sérieuses. Je ne le recommande pas sans cadre juridique clair.
- **Services internes de Kasm.** Si une fonction de Kasm (envoi de fichiers,
  audio) cesse de marcher après les règles de l'étape 4, regardez quels flux
  sont coupés (`sudo iptables -L DOCKER-USER -v -n`) et ajoutez une règle
  d'autorisation ciblée avant la règle DROP.
- **Liste noire du portail.** La liste de l'onglet Administration du portail
  reste locale au navigateur : elle ne remplace pas `blocked_domains.txt`.
  Recopiez-y vos domaines pour que le blocage s'applique vraiment.
- **Journaux.** Squid enregistre les sites visités : informez les utilisateurs et
  fixez une durée de conservation, conformément au RGPD.
