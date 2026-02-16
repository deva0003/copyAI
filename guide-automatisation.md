# Guide d'automatisation - Pipeline LinkedIn Content

> De la veille a la publication, en pilote automatique.
> Seule etape manuelle : choisir 3-4 idees dans Notion.

---

## ARCHITECTURE GLOBALE

```
PHASE 1 (auto)          PHASE 2 (auto)           PHASE 3 (toi)         PHASE 4 (auto)
Veille 7 comptes   -->  20 idees de posts   -->  Tu valides 3-4   -->  Posts rediges
tous les lundis         generees par IA          idees dans Notion      envoyes sur Notion
```

```
┌──────────────┐     ┌──────────┐     ┌───────────┐     ┌────────┐
│ PhantomBuster │────>│ Make.com │────>│ Claude API│────>│ Notion │
│ (scraping)    │     │ (orches.)│     │ (redaction)│    │ (hub)  │
└──────────────┘     └──────────┘     └───────────┘     └────────┘
```

---

## STACK TECHNIQUE

| Outil | Role | Cout/mois | Alternative |
|-------|------|-----------|-------------|
| **PhantomBuster** | Scraper les posts LinkedIn des 7 comptes | ~56€ (plan Starter) | Apify (~25€) ou Browserbear |
| **Make.com** | Orchestrer tout le workflow automatiquement | ~9€ (plan Core - 10K ops) | n8n (self-hosted gratuit, + technique) |
| **API Claude** | Generer les idees + rediger les posts | ~5-10€ (estimation volume) | API OpenAI (~similaire) |
| **Notion** | Hub central : veille, idees, validation, posts finaux | Gratuit (plan free suffit) | - |

**Cout total estime : 70-80€/mois**

---

## PHASE 1 : Veille automatique (tous les 7 jours)

### Ce qui se passe
Chaque lundi a 8h, PhantomBuster scrape les 10 derniers posts de tes 7 comptes et les envoie dans Notion.

### Setup PhantomBuster

**Etape 1 : Creer un compte PhantomBuster**
- Aller sur phantombuster.com
- Plan Starter (56€/mois) = 20h d'execution + 500 credits IA
- Ca suffit largement pour 7 comptes / semaine

**Etape 2 : Utiliser le Phantom "LinkedIn Profile Post Extractor"**
- Dans la bibliotheque, chercher "LinkedIn Post Extractor"
- Configurer les 7 URLs de profils :
```
https://www.linkedin.com/in/nazarii-stefanyshyn/
https://www.linkedin.com/in/etienne-garcia/
https://www.linkedin.com/in/aymeric-bourgeois-social-ads/
https://www.linkedin.com/in/olly-hudson/
https://www.linkedin.com/in/alex-fedotoff/
https://www.linkedin.com/in/elio-besson/
https://www.linkedin.com/in/peter-quadrel/
```
- Nombre de posts a extraire : 10 par profil
- Frequence : 1x par semaine (lundi 8h)

**Etape 3 : Connecter ton cookie LinkedIn**
- PhantomBuster te guide pour extraire ton cookie de session LinkedIn
- IMPORTANT : utilise un compte LinkedIn secondaire ou un compte Sales Navigator pour eviter les restrictions

**Ce que tu recuperes (par post)** :
- Texte complet du post
- Nombre de likes / commentaires
- Date de publication
- URL du post
- Nom de l'auteur

### Setup Notion (base de donnees "Veille")

Creer une base de donnees Notion "Veille LinkedIn" avec ces proprietes :

| Propriete | Type | Description |
|-----------|------|-------------|
| Titre | Title | Premiere ligne du post (hook) |
| Auteur | Select | Nom du compte LinkedIn |
| Texte complet | Text | Contenu integral du post |
| Date post | Date | Date de publication originale |
| Likes | Number | Nombre de reactions |
| Commentaires | Number | Nombre de commentaires |
| URL | URL | Lien vers le post original |
| Format | Select | histoire / liste / mecanisme / controverse / temoignage |
| Sujet | Text | Sujet principal (rempli par l'IA) |
| Pourquoi ca marche | Text | Analyse en 1 phrase (rempli par l'IA) |
| Semaine | Number | Numero de semaine |

---

## PHASE 2 : Generation de 20 idees (automatique, apres Phase 1)

### Ce qui se passe
Des que la veille est terminee, Make.com envoie les posts scrapes a l'API Claude avec ton profil d'ecriture. Claude genere 20 idees de posts adaptees a ton activite. Les idees sont injectees dans une 2e base Notion.

### Setup Make.com - Scenario 1 : "Veille → Idees"

**Declencheur** : Webhook PhantomBuster (quand le scraping est termine)
OU : Schedule tous les lundis a 10h (2h apres le scraping)

**Etapes du scenario** :

```
1. [PhantomBuster] → Recuperer les resultats CSV du scraping
         |
2. [Iterator] → Boucler sur chaque post scrape
         |
3. [Notion] → Creer une entree dans la BDD "Veille LinkedIn"
         |
4. [Notion] → Lire les 70 derniers posts (7 comptes x 10 posts)
         |
5. [HTTP / Claude API] → Envoyer le prompt de generation d'idees
         |
6. [JSON Parser] → Extraire les 20 idees du retour Claude
         |
7. [Iterator] → Boucler sur chaque idee
         |
8. [Notion] → Creer 20 entrees dans la BDD "Idees de Posts"
```

**Le prompt envoye a Claude (etape 5)** :

```
SYSTEME :
Tu es un ghostwriter specialise LinkedIn pour Kelian, consultant Paid SEA/SMA.

Voici son profil d'ecriture : [INSERER LE CONTENU DE profil-ecriture-kelian.md]

Voici son positionnement :
- Activite : Consultant Paid SEA & SMA (strategiste + operationnel) + coaching freelances + apport d'affaires
- Cible : E-commerce & SaaS (CMO / Responsable marketing / CEO)
- Expertise : Paid, SEA, SMA, Growth, CRO, CRM

UTILISATEUR :
Voici les posts LinkedIn de cette semaine provenant de 7 comptes de reference dans la niche Paid/Growth :

[INSERER LES POSTS SCRAPES]

A partir de cette veille :
1. Identifie les 5 mecanismes/formats qui ont le mieux fonctionne cette semaine (engagement le + eleve)
2. Genere 20 idees de posts adaptees a l'activite de Kelian, inspirees de ces mecanismes

Pour chaque idee, donne :
- Le hook (1-2 lignes)
- Le format (histoire / liste / mecanisme / controverse / temoignage)
- Le sujet principal
- La categorie cible (CMO-CEO / E-commerce / Freelances)
- Le mecanisme inspire de (nom du compte + post original)

Reponds en JSON.
```

### Setup Notion (base de donnees "Idees de Posts")

| Propriete | Type | Description |
|-----------|------|-------------|
| Hook | Title | L'accroche proposee (1-2 lignes) |
| Format | Select | histoire / liste / mecanisme / controverse / temoignage |
| Sujet | Text | Le sujet principal |
| Cible | Select | CMO-CEO / E-commerce / Freelances |
| Inspire de | Text | Compte + post d'origine |
| Statut | Select | **A valider** / Valide / Rejete / Redige |
| Semaine | Number | Numero de semaine |

---

## PHASE 3 : Validation manuelle (toi, 5 minutes)

### Ce qui se passe
Tu ouvres Notion, tu scrolles les 20 idees, tu passes 3-4 en statut "Valide". C'est tout.

### Setup Notion - Vue "Validation"

Creer une vue filtree de la BDD "Idees de Posts" :
- Filtre : Statut = "A valider"
- Tri : par categorie cible (pour alterner CMO / E-com / Freelances)
- Vue : Gallery ou Board (plus visuel)

**Comment tu valides** :
1. Ouvre la vue "Validation" chaque lundi apres 10h30
2. Lis les 20 hooks
3. Change le statut de 3-4 idees en "Valide"
4. (Optionnel) Ajoute une note si tu veux orienter la redaction

**Temps estime : 5 minutes max.**

---

## PHASE 4 : Redaction automatique + envoi Notion (auto)

### Ce qui se passe
Des qu'une idee passe en "Valide" dans Notion, Make.com detecte le changement, envoie l'idee a Claude avec ton profil d'ecriture, et le post redige revient dans une 3e base Notion.

### Setup Make.com - Scenario 2 : "Validation → Redaction"

**Declencheur** : Notion - Watch Database Items (filtre : Statut passe a "Valide")

**Etapes du scenario** :

```
1. [Notion Watch] → Detecter les idees passees en "Valide"
         |
2. [Notion] → Lire le detail complet de l'idee
         |
3. [HTTP / Claude API] → Envoyer le prompt de redaction
         |
4. [Notion] → Creer le post dans la BDD "Posts Rediges"
         |
5. [Notion] → Mettre a jour le statut de l'idee en "Redige"
```

**Le prompt envoye a Claude (etape 3)** :

```
SYSTEME :
Tu es le ghostwriter de Kelian. Tu rediges des posts LinkedIn en respectant
SCRUPULEUSEMENT son profil d'ecriture ci-dessous.

[INSERER LE CONTENU COMPLET DE profil-ecriture-kelian.md]

UTILISATEUR :
Redige un post LinkedIn sur cette idee :

Hook : [hook de l'idee validee]
Format : [format]
Sujet : [sujet]
Cible : [CMO-CEO / E-commerce / Freelances]

Regles :
- Entre 400 et 500 mots
- Vouvoiement si cible CMO-CEO ; tutoiement si cible Freelances ; mix si E-commerce
- Structure : Hook → Contexte → Mecanisme/Insight → Conclusion/CTA
- Applique le rythme court-court-long
- Inclus au moins 1 preuve (chiffre, cas client, experience vecue)
- Termine par une question ouverte + dragon 🐉
- Si cible CMO-CEO ou E-commerce, ajoute la signature BOFU :
  "Si on ne se connait pas encore, je suis Kelian et j'ai accompagne +120 societes..."
```

### Setup Notion (base de donnees "Posts Rediges")

| Propriete | Type | Description |
|-----------|------|-------------|
| Titre | Title | Hook du post |
| Contenu | Text | Texte complet du post, pret a copier-coller |
| Format | Select | histoire / liste / mecanisme / controverse / temoignage |
| Cible | Select | CMO-CEO / E-commerce / Freelances |
| Nb mots | Number | Nombre de mots |
| Statut | Select | **Brouillon** / Relu / Publie |
| Date publication prevue | Date | Quand tu prevois de le publier |
| Semaine | Number | Numero de semaine |

---

## SETUP MAKE.COM PAS-A-PAS

### Prerequis
1. Compte Make.com (plan Core a 9€/mois)
2. Compte PhantomBuster (plan Starter a 56€/mois)
3. Cle API Anthropic (console.anthropic.com → API Keys)
4. Integration Notion (creer une integration sur notion.so/my-integrations)

### Scenario 1 : Veille + Generation d'idees

**1. Creer un nouveau scenario dans Make.com**

**2. Module 1 - Schedule**
- Frequence : Every week
- Jour : Monday
- Heure : 10:00 (laisser 2h apres le lancement PhantomBuster)

**3. Module 2 - HTTP (Get)**
- URL : `https://api.phantombuster.com/api/v2/agents/output`
- Headers : `X-Phantombuster-Key: [TA_CLE_API]`
- Query : `id=[ID_DE_TON_PHANTOM]`
- Ceci recupere le CSV des posts scrapes

**4. Module 3 - CSV Parse**
- Parser le CSV en objets exploitables

**5. Module 4 - Iterator**
- Boucler sur chaque ligne (= chaque post)

**6. Module 5 - Notion Create Database Item**
- Connexion : ton integration Notion
- Database : "Veille LinkedIn"
- Mapper chaque champ (titre, auteur, texte, date, likes, commentaires, URL)

**7. Module 6 - Notion Search**
- Recuperer les 70 posts de la semaine courante
- Filtre : Semaine = [numero semaine actuel]

**8. Module 7 - HTTP (Post) → Claude API**
- URL : `https://api.anthropic.com/v1/messages`
- Headers :
  ```
  x-api-key: [TA_CLE_API_ANTHROPIC]
  anthropic-version: 2023-06-01
  Content-Type: application/json
  ```
- Body :
  ```json
  {
    "model": "claude-sonnet-4-5-20250929",
    "max_tokens": 4096,
    "messages": [
      {
        "role": "user",
        "content": "[LE PROMPT DE GENERATION - voir Phase 2]"
      }
    ]
  }
  ```

**9. Module 8 - JSON Parse**
- Parser la reponse de Claude (les 20 idees en JSON)

**10. Module 9 - Iterator**
- Boucler sur chaque idee

**11. Module 10 - Notion Create Database Item**
- Database : "Idees de Posts"
- Mapper : Hook, Format, Sujet, Cible, Inspire de
- Statut par defaut : "A valider"

### Scenario 2 : Redaction des posts valides

**1. Module 1 - Notion Watch Database Items**
- Database : "Idees de Posts"
- Filtre : Statut = "Valide"
- Polling : Every 15 minutes

**2. Module 2 - Notion Get Database Item**
- Recuperer tous les details de l'idee validee

**3. Module 3 - HTTP (Post) → Claude API**
- URL : `https://api.anthropic.com/v1/messages`
- Body : le prompt de redaction (voir Phase 4) avec les infos de l'idee

**4. Module 4 - Notion Create Database Item**
- Database : "Posts Rediges"
- Contenu : texte genere par Claude
- Statut : "Brouillon"

**5. Module 5 - Notion Update Database Item**
- Database : "Idees de Posts"
- Mettre le statut de l'idee a "Redige"

---

## PLANNING HEBDOMADAIRE TYPE

| Jour | Heure | Action | Automatise ? |
|------|-------|--------|-------------|
| Lundi | 08:00 | PhantomBuster scrape les 7 comptes | Oui |
| Lundi | 10:00 | Make.com injecte la veille dans Notion + genere 20 idees | Oui |
| Lundi | 10:30 | **TOI : tu ouvres Notion, tu valides 3-4 idees (5 min)** | **Non** |
| Lundi | 10:45 | Make.com detecte les validations et lance la redaction | Oui |
| Lundi | 11:00 | 3-4 posts rediges apparaissent dans Notion "Posts Rediges" | Oui |
| Lundi-Vendredi | - | **TOI : tu relis, ajustes et publies selon ton calendrier** | **Non** |

**Temps actif par semaine : ~10-15 minutes** (5 min validation + 5-10 min relecture/ajustement)

---

## COUTS RECAPITULATIFS

| Poste | Cout mensuel | Notes |
|-------|-------------|-------|
| PhantomBuster Starter | 56€ | 7 profils x 4 semaines = 28 executions |
| Make.com Core | 9€ | ~2000 operations / mois (largement suffisant) |
| API Claude (Sonnet 4.5) | 5-10€ | ~80 appels/mois (20 idees + 4 posts x 4 semaines) |
| Notion | 0€ | Plan gratuit suffisant |
| **TOTAL** | **70-75€/mois** | |

---

## ALTERNATIVE LOW-COST (si budget serre)

Si tu veux reduire les couts a ~15-25€/mois :

| Composant | Alternative | Cout |
|-----------|-------------|------|
| PhantomBuster | **Scraping manuel 1x/semaine** : copier-coller les posts dans un Google Sheet en 15 min | 0€ |
| Make.com | **n8n self-hosted** sur un VPS a 5€/mois (Hetzner/OVH) | 5€ |
| API Claude | Idem | 5-10€ |
| Notion | Idem | 0€ |
| **TOTAL** | | **10-15€/mois + 15 min de scraping manuel** |

### Setup scraping manuel (alternative PhantomBuster)

Si tu choisis le scraping manuel :
1. Ouvre chaque profil LinkedIn (7 comptes)
2. Scrolle les 10 derniers posts
3. Copie le texte + likes + commentaires dans un Google Sheet
4. Make.com lit le Google Sheet et enchaine sur la generation d'idees

**Temps supplementaire** : ~15-20 min/semaine
**Economie** : 56€/mois

---

## FAQ

**Q : Est-ce que PhantomBuster peut me faire bannir de LinkedIn ?**
Utilise un compte secondaire pour le scraping et respecte les limites de PhantomBuster (pas plus de 80 profils/jour). En scrapant 7 profils 1x/semaine, tu es tres loin des limites.

**Q : Pourquoi Claude API plutot que ChatGPT ?**
Le profil d'ecriture est construit pour Claude. Tu peux utiliser l'API OpenAI (GPT-4o) comme alternative : le prompt fonctionnera, mais les resultats seront potentiellement moins fideles au ton. Claude respecte mieux les consignes de style longues.

**Q : Et si je veux publier directement depuis Notion ?**
Tu peux ajouter un scenario Make.com supplementaire :
- Declencheur : Statut du post passe a "Publie" dans Notion
- Action : Publier via l'API LinkedIn (necessite une app LinkedIn developer)
- OU : utiliser un outil comme Taplio / Buffer / Publer connecte a Make.com
Note : la publication automatique LinkedIn est moins fiable (API restrictive). Je recommande le copier-coller depuis Notion.

**Q : Comment ameliorer la qualite des posts generes au fil du temps ?**
1. Quand tu ajustes un post avant publication, sauvegarde ta version finale dans une 4e BDD Notion "Posts publies (version finale)"
2. Tous les mois, injecte tes 10-15 meilleurs posts publies dans le prompt Claude comme exemples supplementaires
3. Le modele s'ameliorera en s'alignant sur tes corrections reelles

**Q : Je peux ajouter / retirer des comptes a surveiller ?**
Oui. Modifie la liste d'URLs dans PhantomBuster. Le reste du pipeline s'adapte automatiquement.

---

*Guide cree le 2026-02-16 | Pipeline d'automatisation LinkedIn pour consultant Paid SEA/SMA*
