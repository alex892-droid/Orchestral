# Orchestral

Hub d'automatisation perso : n8n auto-heberge en Docker derriere Caddy (HTTPS
automatique), pour : alerte nouveau mail + resume vocal du matin sur Alexa, puis
YouTube, etc.

Stack : VPS Docker + n8n + Caddy + DuckDNS + Notify Me (Alexa) + Gmail + Claude.

## 1. Prerequis VPS

- Un VPS (Hetzner / OVH, ~5 EUR/mois) sous Debian/Ubuntu, Docker + Docker Compose installes.
- Les ports **80** et **443** ouverts dans le firewall du VPS et chez l'hebergeur.

## 2. Sous-domaine gratuit (DuckDNS)

1. Va sur https://www.duckdns.org, connecte-toi, cree un sous-domaine (ex `alexishub`).
2. Mets l'IP publique de ton VPS dans le champ `current ip`, valide.
3. Ton domaine est `alexishub.duckdns.org`.

> Caddy obtiendra tout seul un vrai certificat Let's Encrypt pour ce domaine
> (indispensable pour l'OAuth Gmail et le webhook Notify Me).

## 3. Deploiement

Copie ce dossier sur le VPS (scp / git), puis :

```bash
cp .env.example .env
# Edite .env : N8N_HOST = ton domaine DuckDNS, et genere la cle :
openssl rand -hex 32   # colle le resultat dans N8N_ENCRYPTION_KEY
docker compose up -d
docker compose logs -f caddy   # verifie l'obtention du certificat
```

Ouvre `https://alexishub.duckdns.org` : n8n te demande de creer le compte proprietaire (1re connexion).

## 4. Notify Me (Alexa)

1. Active la skill **Notify Me** : https://www.amazon.fr (rechercher "Notify Me" dans les skills Alexa).
2. Dis a Alexa "Alexa, ouvre Notify Me". Elle t'envoie un mail avec ton **access code**.
3. Dans n8n : menu **Variables** -> cree `NOTIFYME_CODE` avec ce code.
   (Les variables n8n requierent un plan ; sinon, mets le code en dur dans le node
   "Annoncer sur Alexa" a la place de `$vars.NOTIFYME_CODE`.)

Test rapide depuis le VPS :

```bash
curl -X POST https://api.notifymyecho.com/v1/NotifyMe \
  -H "content-type: application/json" \
  -d '{"notification":"Test depuis n8n","accessCode":"TON_CODE"}'
```

## 5. Credentials dans n8n

- **Gmail** : Credentials -> New -> *Gmail OAuth2*. Suis l'assistant Google Cloud
  (cree un projet, active l'API Gmail, ecran de consentement, identifiants OAuth ;
  URL de redirection = celle affichee par n8n). Autorise ton compte.
- **Anthropic** : Credentials -> New -> *Header Auth*. Name = `x-api-key`,
  Value = ta cle API Anthropic (console.anthropic.com). Nomme-la `Anthropic x-api-key`.

## 6. Importer le workflow

- Workflows -> menu (...) -> **Import from File** -> `workflow-resume-matin.json`.
- Ouvre chaque node Gmail / Claude et selectionne tes credentials (les ids "REMPLACER").
- Active le workflow. Teste avec **Execute Workflow** sans attendre 7h.

### Cout / modele Claude

Le node "Resume via Claude" utilise `claude-opus-4-8` (qualite max). Pour reduire
le cout d'un resume quotidien, remplace dans le `jsonBody` du node :
`model: 'claude-opus-4-8'` par `model: 'claude-haiku-4-5'` (~5x moins cher,
largement suffisant pour resumer des mails).

## Prochaines etapes

- **Workflow A (nouveau mail)** : Gmail Trigger -> filtre expediteurs importants
  -> Notify Me. A mettre un filtre des le depart, sinon Alexa parle non-stop.
- **YouTube** : node RSS sur `https://www.youtube.com/feeds/videos.xml?channel_id=...`
  -> Notify Me. Pas d'auth, pas de quota.
- **LinkedIn** : pas d'API officielle pour la messagerie. A garder en dernier,
  et plutot en alerte legere qu'en lecture des messages (risque de blocage du compte).
