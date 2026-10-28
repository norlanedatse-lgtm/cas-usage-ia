# Formulaire « Proposer un cas d'usage IA »

Formulaire statique en un seul fichier (`index.html`). Il ne nécessite aucun serveur et peut être publié sur GitHub Pages.

## 1. Régler l'adresse de réception

En haut du script de `index.html`, modifier :

```js
const CONFIG = {
  emailDestinataire: "equipe-ia@exemple.fr",   // ← votre adresse (idéalement une boîte partagée)
  relayUrl: ""                                 // ← laisser vide pour le mode e-mail
};
```

## 2. Publier sur GitHub Pages

1. Sur github.com : **New repository**, par exemple `cas-usage-ia`. Laisser *Public*, car Pages est gratuit pour les dépôts publics.
2. **Add file → Upload files**, déposer `index.html` (et ce README), puis **Commit changes**.
3. **Settings → Pages → Build and deployment** : Source *Deploy from a branch*, branche `main`, dossier `/ (root)`, puis **Save**.
4. Une à deux minutes plus tard, le site est en ligne à l'adresse `https://<votre-identifiant>.github.io/cas-usage-ia/`.

Le site est public, mais il ne contient aucune réponse. Les réponses partent par e-mail ou vers Grist et ne sont jamais stockées sur GitHub.

## 3. Mode e-mail (par défaut)

À l'envoi, la messagerie du demandeur s'ouvre avec un message prêt :
- objet : `[Cas d'usage IA] <nom> (CU-AAAAMMJJ-XXXX)` ;
- corps : la fiche de spécification complète et les points d'attention.

Si la fiche est trop longue pour un lien e-mail (certaines messageries tronquent au-delà d'environ 2 000 caractères), elle est copiée automatiquement et le demandeur n'a plus qu'à la coller. Un bouton « Copier la fiche » sert aussi de solution de secours.

Ce qu'il faut savoir : l'e-mail part de la messagerie du demandeur, donc rien ne garantit qu'il clique bien sur « Envoyer ». La référence CU-… permet de rapprocher les demandes reçues.

## 4. Mode Grist (optionnel, avec un relais)

Grist refuse les appels d'API depuis un navigateur. Mettre la clé d'API dans la page la rendrait de toute façon lisible par n'importe qui, qui pourrait alors lire ou effacer le document. Il faut donc un petit relais côté serveur qui détient la clé.

### Table Grist à créer (`Demandes`)

Identifiants de colonnes : `reference`, `date`, `statut`, `nom`, `service`, `demandeur`, `urgence`, `frequence`, `types`, `donnees`, `pointsAttention`, `fiche`.
Colonnes à ajouter pour le suivi : `outilPressenti`, `faisabilite`.

### Relais (exemple Cloudflare Worker, offre gratuite)

Secrets à définir dans le Worker : `GRIST_KEY` (clé d'API Grist), `GRIST_DOC` (identifiant du document), `ALLOWED_ORIGIN` (`https://<votre-identifiant>.github.io`).

```js
const COLS = ["reference","date","statut","nom","service","demandeur","urgence",
              "frequence","types","donnees","pointsAttention","fiche"];
export default {
  async fetch(req, env) {
    const cors = { "Access-Control-Allow-Origin": env.ALLOWED_ORIGIN,
                   "Access-Control-Allow-Methods": "POST",
                   "Access-Control-Allow-Headers": "Content-Type" };
    if (req.method === "OPTIONS") return new Response(null, { headers: cors });
    if (req.method !== "POST") return new Response("", { status: 405, headers: cors });
    const body = await req.json();
    const fields = {};
    for (const c of COLS) if (body[c] != null) fields[c] = String(body[c]).slice(0, 20000);
    const r = await fetch(`https://grist.numerique.gouv.fr/api/docs/${env.GRIST_DOC}/tables/Demandes/records`, {
      method: "POST",
      headers: { "Authorization": `Bearer ${env.GRIST_KEY}`, "Content-Type": "application/json" },
      body: JSON.stringify({ records: [{ fields }] })
    });
    return new Response(r.ok ? "ok" : "erreur", { status: r.ok ? 200 : 502, headers: cors });
  }
};
```

Renseigner ensuite l'URL du Worker dans `relayUrl`. Les demandes arrivent alors directement dans Grist.

**À valider avec la DSI ou le DPO avant la mise en place :** la clé d'API donne accès à votre compte Grist, et le relais est hébergé hors de l'administration. Si possible, utiliser un compte ou un document dédié au formulaire, et un hébergeur validé par votre structure.
