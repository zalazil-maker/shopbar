# Mettre le site en ligne

Le site est un seul fichier statique (`index.html`). Il n'a besoin d'aucun
serveur : toute la logique de réservation vit dans le Cloudflare Worker qui
tourne déjà.

## Recommandation : Cloudflare Pages

C'est le meilleur choix **parce que le Worker de réservation est déjà chez
Cloudflare**. Même compte, même dashboard, et surtout la possibilité de servir
le site et l'API sur le même domaine.

Pourquoi :

- Gratuit pour ce type de site, sans limite de trafic pratique
- Déploiement automatique à chaque `git push`
- HTTPS et CDN inclus, serveurs en France → site rapide à Amiens
- Domaine personnalisé gratuit (vous ne payez que le nom de domaine)

### Étapes

1. Achetez un domaine, par ex. `luxurybarber-amiens.fr` (~10 €/an chez
   Cloudflare Registrar, OVH ou Gandi).
2. Sur https://dash.cloudflare.com → **Workers & Pages** → **Create** →
   **Pages** → **Connect to Git**.
3. Choisissez le dépôt `shopbar`, branche `main`.
4. Build command : **laissez vide**. Output directory : **`/`**.
   (Il n'y a rien à compiler.)
5. **Save and Deploy**. Le site est en ligne sur
   `shopbar.pages.dev` en une minute.
6. Onglet **Custom domains** → **Set up a domain** → entrez votre domaine et
   suivez les instructions DNS.

### Bonus : tout sur le même domaine

Une fois le domaine branché, vous pouvez router l'API sous le même nom pour
éviter le CORS et les blocages de navigateurs :

Workers & Pages → `barbershop` → **Settings** → **Domains & Routes** →
**Add route** : `votredomaine.fr/api/*`

Puis dans `index.html`, remplacez :

```js
var WORKER = "https://barbershop.ezalazil.workers.dev";
```

par :

```js
var WORKER = "";
```

Les appels deviennent `/api/...` sur votre propre domaine.

## Alternatives

| Hébergeur | Gratuit | Avantage | Inconvénient |
|-----------|---------|----------|--------------|
| **Cloudflare Pages** | oui | Même compte que le Worker, tout au même endroit | — |
| **Netlify** | oui | Interface très simple, glisser-déposer possible | Un fournisseur de plus à gérer |
| **Vercel** | oui | Déploiement Git impeccable | Orienté apps React, surdimensionné ici |
| **GitHub Pages** | oui | Déjà sur GitHub, zéro inscription | Pas de logs, domaine perso un peu plus manuel |
| **OVH / Hostinger** | non (~3–5 €/mois) | Hébergeur français, support téléphone | Payant et inutile pour un fichier statique |

**Le plus simple sans compte supplémentaire :** GitHub Pages — dépôt `shopbar`
→ Settings → Pages → Source `main` / `/root`. En ligne sur
`zalazil-maker.github.io/shopbar`.

**Le plus adapté à votre installation :** Cloudflare Pages.

## Avant d'ouvrir au public

- [ ] Déployer la nouvelle version de `worker.js` (dépôt `barber`)
- [ ] Définir le secret `ADMIN_KEY` sur le Worker
- [ ] Ouvrir l'onglet RDV de l'app de caisse et entrer ce code sur le téléphone
      de chaque barbier
- [ ] Faire une réservation de test, vérifier qu'elle apparaît dans l'onglet RDV
- [ ] Annuler cette réservation de test
- [ ] (optionnel) Configurer `RESEND_API_KEY` + `SHOP_EMAIL` pour les emails
- [ ] Mettre le lien du site sur la fiche Google Business et la bio Instagram
