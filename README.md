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

Pour changer un prix, un barbier ou un horaire : tout est en haut de
`worker.js` (dépôt `barber`), dans `SERVICES`, `BARBERS` et `HOURS`. Le site
lit ces valeurs via `/api/config` — il n'y a rien à modifier ici, sauf les
cartes tarifs de la section « Tarifs » qui sont écrites en dur pour le SEO.

## Créneaux

- Créneaux de 30 minutes
- Lundi → samedi 9h30–20h00, dimanche 10h00–20h00 (dernier créneau 19h30)
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
| `ADMIN_KEY`      | oui         | Code tapé dans l'onglet RDV de l'app de caisse    |
| `RESEND_API_KEY` | non         | Active les emails de confirmation                 |
| `SHOP_EMAIL`     | non         | Adresse qui reçoit les nouveaux RDV               |
| `MAIL_FROM`      | non         | Expéditeur, ex. `Luxury Barber <rdv@votredomaine.fr>` |
| `SITE_URL`       | non         | Base des liens d'annulation dans les emails       |

Sans `RESEND_API_KEY`, les réservations fonctionnent normalement — seuls les
emails sont désactivés.

## Annulation

Chaque réservation génère un jeton secret. Le client annule :

- depuis l'écran de confirmation, juste après avoir réservé ;
- ou via le lien reçu par email : `https://votredomaine.fr/?annuler=<id>&token=<jeton>`.

Le salon peut aussi annuler depuis l'onglet RDV de l'app de caisse.
