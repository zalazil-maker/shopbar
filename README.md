# Luxury Barber — site & réservation en ligne

Site vitrine du salon **Luxury Barber** (11 Rue Dumeril, 80000 Amiens) avec un
système de réservation intégré : le client choisit sa prestation, son barbier et
son créneau sans quitter la page.

Tout tient dans `index.html` — pas de build, pas de dépendances.

## Comment ça marche

```
index.html  ──►  Cloudflare Worker  ──►  Neon Postgres
(le site)        (barbershop.ezalazil        (bookings +
                  .workers.dev)               blocked_slots)
                        ▲
                        │
                  app LX Barbershop
                  (onglet RDV)
```

Le Worker et la base sont **déjà en place** — ce sont ceux de l'app de caisse
(dépôt `barber`). Le site de réservation ne fait qu'ajouter des routes `/api/*`
au même Worker.

Le créneau ne peut pas être pris deux fois : un index unique en base
(`bookings_slot_uniq`) rejette la deuxième réservation même si deux clients
valident à la même seconde.

## Tarifs

Les prix affichés sont les **tarifs RDV en ligne**, réservés aux rendez-vous
pris sur ce site. Le tarif Planity / sans rendez-vous est affiché barré à côté.

| Prestation      | Sur ce site | Sur Planity | Économie |
|-----------------|-------------|-------------|----------|
| Cheveux         | 18,00 €     | 20,00 €     | 2,00 €   |
| Barbe           | 9,99 €      | 12,00 €     | 2,01 €   |
| Cheveux + Barbe | 23,99 €     | 26,00 €     | 2,01 €   |
| Enfant          | 12,00 €     | —           | —        |

Ces tarifs sont ceux d'origine : ils s'éditent désormais depuis
[`/admin`](#panneau-dadministration), sans toucher au code.

Les cartes de la section « Tarifs » sont écrites en dur dans `index.html` pour
que les moteurs de recherche les voient, puis reconstruites depuis
`/api/config` au chargement — une modification faite dans `/admin` apparaît donc
immédiatement, y compris pour une prestation ajoutée après coup.

## Créneaux

- Créneaux de 30 minutes
- Lundi → samedi 9h30–20h00, dimanche 10h00–20h00 (dernier créneau 19h30)
- Email obligatoire à la réservation (confirmation + lien d'annulation)
- Réservation jusqu'à 60 jours à l'avance
- Maximum 3 rendez-vous à venir par numéro de téléphone

## Déploiement

Le site est un fichier statique : n'importe quel hébergeur convient.
Recommandation : **Cloudflare Pages** (voir `HOSTING.md`).

## Configuration du Worker

Dans le dashboard Cloudflare → Workers → `barbershop` → Settings → Variables :

| Nom              | Obligatoire | Rôle                                              |
|------------------|-------------|---------------------------------------------------|
| `DATABASE_URL`   | oui         | Connexion Neon (déjà configurée)                  |
| `ADMIN_KEY`      | oui         | Code du panneau `/admin` et de l'onglet RDV       |
| `RESEND_API_KEY` | non         | Active les emails de confirmation                 |
| `SHOP_EMAIL`     | non         | Adresse qui reçoit les nouveaux RDV               |
| `MAIL_FROM`      | non         | Expéditeur, ex. `Luxury Barber <rdv@luxurybarber80.fr>` |
| `SITE_URL`       | non         | Base des liens d'annulation dans les emails       |
| `VAPID_*`        | non         | Notifications push (voir dépôt `barber`)          |

Plus une liaison R2 nommée **`MEDIA`** pour les photos et vidéos.

Chaque brique est facultative et se dégrade proprement : sans `RESEND_API_KEY`
les réservations marchent mais n'envoient pas d'email ; sans `MEDIA` tout
fonctionne sauf la galerie.

## Panneau d'administration

`https://luxurybarber80.fr/admin` — déverrouillé avec le code `ADMIN_KEY`.

- **Prestations** : prix ici, prix Planity, description, ordre, ajout et retrait
- **Barbiers** : nom, ordre, photo de profil, ajout et retrait
- **Horaires** : ouverture/fermeture par jour, jours fermés, durée des créneaux
- **Galerie** : photos et vidéos du travail du salon

Retirer une prestation ou un barbier les masque du site sans toucher à
l'historique des rendez-vous ni aux recettes.

Les prix, barbiers et horaires vivent en base — les constantes en haut de
`worker.js` ne servent qu'à initialiser une base vide.

## Emails de confirmation (Resend)

1. Créez un compte sur [resend.com](https://resend.com) (offre gratuite).
2. **Domains → Add domain** → `luxurybarber80.fr`.
3. Resend affiche des enregistrements DNS à créer dans Cloudflare
   (`luxurybarber80.fr` → DNS → Records) : une clé **DKIM** et souvent un
   **MX** de retour sur un sous-domaine `send`.
4. **Attention au SPF.** Le domaine a déjà un enregistrement TXT pour la
   messagerie OVH :

   ```
   v=spf1 include:mx.ovh.com -all
   ```

   Il ne doit **jamais** y en avoir deux — le SPF casse et les mails partent en
   spam. Modifiez celui qui existe au lieu d'en ajouter un :

   ```
   v=spf1 include:mx.ovh.com include:_spf.resend.com -all
   ```

5. **API Keys → Create** → copiez la clé dans le secret `RESEND_API_KEY`.
6. Renseignez `SHOP_EMAIL` (qui reçoit les RDV) et `MAIL_FROM` (l'expéditeur,
   sur le domaine vérifié).

Tant que le domaine n'est pas vérifié, l'expéditeur par défaut
`onboarding@resend.dev` fonctionne pour tester mais finit souvent en spam.

## Photos et vidéos (R2)

1. Cloudflare → **R2** → **Create bucket** → nom `luxurybarber-media`.
2. Worker `barbershop` → **Settings → Bindings → Add → R2 bucket**.
3. Nom de la variable : **`MEDIA`** (exactement), bucket : celui créé.
4. Déployez.

Les fichiers sont servis par le Worker via `/api/media/<clé>`, avec un cache
d'un an. Formats acceptés : JPEG, PNG, WebP, MP4, MOV, WebM — 25 Mo maximum.

La section « Nos réalisations » reste masquée tant qu'aucun média n'est envoyé.

## Annulation

Chaque réservation génère un jeton secret. Le client annule :

- depuis l'écran de confirmation, juste après avoir réservé ;
- ou via le lien reçu par email : `https://votredomaine.fr/?annuler=<id>&token=<jeton>`.

Le salon peut aussi annuler depuis l'onglet RDV de l'app de caisse.
