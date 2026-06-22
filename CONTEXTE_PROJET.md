# RenoAssistant — Contexte Complet Projet

> Document de référence unique destiné à alimenter un agent conversationnel.
> Contient l'ensemble des décisions, architecture, stack, données et contexte métier.

---

## 1. Vision & Concept

**RenoAssistant** est un projet IA en deux volets complémentaires pour les **clients PRO Leroy Merlin** (artisans, entreprises du bâtiment) :

- **US1** — Un agent SMS qui reçoit un prompt en langage naturel et retourne le top 10 des articles en promo correspondants avec des liens directs.
- **US2** — Un filtre agentique intégré au site et à l'app LM (widget bouton) qui traduit les questions en langage naturel en filtres catalogue existants, en temps réel.

**Dimension stratégique — GEO (Generative Engine Optimization) :**

Le travail de structuration des données produits nécessaire pour faire fonctionner US2 constitue un **investissement GEO**. En structurant le catalogue LM pour qu'un agent IA interne puisse le comprendre (descriptions riches, attributs typés, cas d'usage, tags sémantiques), Leroy Merlin prépare simultanément son contenu à être cité par les IA génératives externes (Perplexity, ChatGPT, Google AI Overviews).

```
Effort unique de structuration catalogue
        ↓
US2 : agent interne filtre mieux     +     GEO : IA externes citent LM
```

**Un seul chantier data → double bénéfice commercial.**

---

## 2. Contexte Métier

| Élément | Détail |
|---------|--------|
| **Porteur du projet** | Mehdi Chadli — en formation AI Intensive (Wild Code School) |
| **Employeur actuel** | Leroy Merlin (jusqu'à fin septembre 2026) |
| **Objectif** | Présenter un MVP fonctionnel aux dirigeants LM avant fin septembre 2026 |
| **Cible utilisateurs** | Clients PRO Leroy Merlin (artisans, entreprises bâtiment) |
| **Scope produits** | Gamme Clients PRO uniquement (~2000-5000 références) |
| **Infrastructure** | DGX Spark (NVIDIA GB10 Grace Blackwell, 128GB RAM unifiée) — hébergement local |
| **Données LM** | Accès DB interne via poste actuel — données anonymisées/transformées pour le MVP |

---

## 3. Les 2 Fonctionnalités Principales (User Stories)

### Feature 1 — Matching Promo via SMS

```
En tant que CLIENT PRO LEROY MERLIN,
Je veux ENVOYER UNE DESCRIPTION DE MON BESOIN PAR SMS,
Afin de RECEVOIR LE TOP 10 DES ARTICLES EN PROMO CORRESPONDANTS AVEC LES LIENS POUR FINALISER SUR LE SITE OU L'APP.
```

**Flux conversationnel :**
```
Client : "perceuse haut de gamme en promo"
Agent  : "Top 10 perceuses premium :
          1. Bosch GBH 18V -30% → 189€ [lien]
          2. Makita HR2631F -25% → 159€ [lien]
          ..."
Client : "plutôt pour du béton"
Agent  : "Top 10 perceuses béton :
          1. Bosch GBH 18V-26 -30% → 189€ [lien]
          ..."
```

**Règles :** 5 shots max par session — le client finalise sur le site ou l'app via les liens

**Acteurs :** Client PRO, SMS Gateway (Twilio), FastAPI, Gemma 4, PostgreSQL + pgvector (table operations_commerciales — 3500 articles promo), Redis (contexte + compteur shots)

---

### Feature 2 — Filtre Agentique (navigation site/app LM)

```
En tant que CLIENT PRO LEROY MERLIN,
Je veux POSER UNE QUESTION EN LANGAGE NATUREL PENDANT MA NAVIGATION,
Afin que L'AGENT APPLIQUE AUTOMATIQUEMENT LES BONS FILTRES ET M'AFFICHE LES PRODUITS QUI CORRESPONDENT À MON BESOIN.
```

**Concept :** Un bouton agent discret sur le site web et l'app LM permet au client de décrire son besoin en langage naturel pendant sa navigation. L'agent analyse le contexte de la page courante, traduit la demande en filtres existants du catalogue LM, et met à jour les résultats instantanément — sans que le client manipule un seul filtre manuellement.

**Canal :** Widget bouton intégré au site LM et à l'app mobile (pas SMS)

**Flux conversationnel :**
```
[Client navigue sur la catégorie "Perceuses"]
Client : "je cherche quelque chose pour percer du béton dur, budget 100€ max"
Agent  : applique → type=burin, usage=béton, prix≤100€
         Résultats mis à jour automatiquement

Client : "plutôt en sans fil"
Agent  : ajoute → alimentation=batterie
         Résultats affinés automatiquement
```

**Principes clés :**
- Utilise les filtres **existants** du site/app LM (aucun développement DB supplémentaire)
- Utilise la **base de données LM existante** (pas de DB custom)
- Conversation multi-tours — le client affine librement
- Zéro friction : le client parle, les résultats suivent

**Acteurs :** Client PRO, Widget agent (site/app LM), Gemma 4 (traduction prompt → filtres), Catalogue LM existant, Système de filtres LM existant

---

## 4. Architecture Technique

### Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────┐
│                        DGX SPARK (Local)                        │
│                                                                 │
│  SMS Client ──► Twilio Webhook ──► FastAPI                      │
│                                       │                         │
│                              ┌────────┴────────┐                │
│                              │   Redis          │               │
│                              │ (contexte conv.) │               │
│                              └────────┬────────┘                │
│                                       │                         │
│                              ┌────────▼────────┐                │
│                              │   Gemma 4 27B    │               │
│                              │   (Ollama)       │               │
│                              │   Tool Calling   │               │
│                              └────────┬────────┘                │
│                                       │                         │
│              ┌────────────────────────┼──────────────┐          │
│              │                        │              │          │
│     ┌────────▼────────┐    ┌─────────▼──────┐  ┌────▼───────┐  │
│     │  PostgreSQL      │    │  Playwright    │  │ Streamlit  │  │
│     │  + pgvector      │    │  (scraping     │  │ (dashboard │  │
│     │  (catalogue PRO) │    │   promos)      │  │  admin)    │  │
│     └─────────────────┘    └────────────────┘  └────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Flux de données détaillé

**Chaque SMS entrant suit ce pipeline :**

```
1. Client envoie SMS
      ↓
2. Twilio reçoit → POST webhook vers FastAPI /sms/incoming
      ↓
3. FastAPI vérifie whitelist (numéro autorisé ?)
      ↓ OUI
4. FastAPI récupère contexte conversation → Redis (clé = numéro téléphone)
      ↓
5. Gemma 4 reçoit : [historique conversation] + [SMS entrant] + [outils disponibles]
      ↓
6. Gemma 4 décide quel outil appeler (Tool Calling) :
   - search_products(query) → PostgreSQL + pgvector
   - get_promos()           → Playwright scrape LM
   - create_basket(items)   → PostgreSQL
   - validate_pin(pin)      → PostgreSQL
      ↓
7. Gemma 4 génère réponse SMS
      ↓
8. FastAPI met à jour contexte → Redis (TTL 1h)
      ↓
9. Twilio envoie SMS réponse au client
```

---

## 5. Stack Technique (13 décisions validées)

| Composant | Technologie | Justification |
|-----------|------------|---------------|
| **Language** | Python (exclusif) | Formation AI, écosystème scraping/ML |
| **Framework API** | FastAPI | Moderne, async, compatible SQLModel |
| **Dashboard admin** | Streamlit | Monitoring + gestion whitelist |
| **LLM** | Gemma 4 27B | Multimodal, excellent en français, local |
| **Serving LLM** | Ollama | Setup 2 min, API REST locale |
| **Flow** | Tool Calling | Gemma décide quoi faire (pas de règles hardcodées) |
| **Contexte conv.** | Redis (TTL 1h) | Rapide, session par numéro téléphone |
| **Base de données** | PostgreSQL | Robuste, standard industrie |
| **ORM** | SQLModel | Fait par créateur FastAPI, Pydantic natif |
| **Recherche** | pgvector (embeddings) | Recherche sémantique (langage naturel) |
| **Scraping** | Aucun | Toutes les données viennent de la DB |
| **SMS Gateway** | Twilio | SDK Python excellent, sandbox gratuit |
| **Config** | .env + python-dotenv | Standard MVP |
| **Hébergement** | DGX Spark (local) | €0 — 128GB RAM, GPU Blackwell |

**Coût opérationnel total : ~€0.08 / commande complète (Twilio SMS uniquement)**

---

## 6. Modèle de Données

### Whitelist Téléphones
```python
class Whitelist(SQLModel, table=True):
    id: int = Field(primary_key=True)
    phone: str          # "+33612345678"
    client_name: str    # "Dubois Renovation"
    client_type: str    # "PRO"
    active: bool = True
    created_at: datetime
```

### Produits PRO (avec vecteur sémantique)
```python
class Produit(SQLModel, table=True):
    id: int = Field(primary_key=True)
    reference: str      # "LM-PRO-12345"
    nom: str            # "Peinture Pro Standard 2.5L"
    categorie: str      # "Peintures & Revêtements"
    prix: float         # 42.50
    disponible: bool
    embedding: List[float]  # Vecteur pgvector (généré par Gemma 4)
```

### Commandes
```python
class Commande(SQLModel, table=True):
    id: int = Field(primary_key=True)
    phone: str
    items: str          # JSON list des produits
    total: float
    status: str         # "pending" | "waiting_pin" | "confirmed" | "cancelled"
    pin: str            # Hash du PIN (jamais en clair)
    validation_url: str
    created_at: datetime
    confirmed_at: Optional[datetime]
```

### Conversations (Redis — pas PostgreSQL)
```python
# Clé Redis : phone number
# Valeur : JSON list des messages
# TTL : 3600 secondes (1h)
{
  "+33612345678": [
    {"role": "user", "content": "Je veux de la peinture"},
    {"role": "assistant", "content": "Quelle quantité ?"},
    {"role": "user", "content": "3 pots"}
  ]
}
```

---

## 7. Tool Calling — Outils de Gemma 4

Gemma 4 dispose de 2 outils qu'il appelle de manière autonome selon le contexte :

```python
tools = [
    {
        "name": "match_promos",
        "description": "Cherche les meilleures offres dans la table operations_commerciales selon le prompt du client (US1)",
        "parameters": {
            "prompt": "str — description libre du besoin client",
            "top": "int — nombre de résultats (défaut: 10)",
            "shots_remaining": "int — shots restants dans la session"
        }
    },
    {
        "name": "translate_to_filters",
        "description": "Traduit un prompt en langage naturel en filtres du catalogue LM existant, selon le contexte de la page (US2)",
        "parameters": {
            "prompt": "str — question libre du client",
            "page_context": "dict — catégorie/page courante, filtres déjà actifs"
        }
    }
]
```

---

## 8. Sources de Données

Voir le document dédié **[SOURCES_DONNEES.md](./SOURCES_DONNEES.md)** — réponse complète au Livrable 3 du brief (liens, pages, API précis pour chaque source).

---

## 9. Authentification

**Mécanisme : Whitelist téléphonique**

Chaque SMS entrant est vérifié contre une liste blanche de numéros autorisés avant tout traitement.

```python
@app.post("/sms/incoming")
async def receive_sms(From: str, Body: str):
    if not is_whitelisted(From):
        return TwiML("❌ Numéro non autorisé.")
    # Continue traitement...
```

Pas de mot de passe, pas de token — l'autorisation = appeler depuis le bon numéro.

---

## 10. Recherche Sémantique avec pgvector

**Pourquoi pgvector ?**

Un client PRO peut envoyer "truc pour peindre mon salon" ou "colle pour carrelage" — des descriptions qui ne matchent pas mot à mot les noms de produits en DB. pgvector permet une recherche par **similarité sémantique**.

**Processus :**
1. À l'import des produits → Gemma 4 génère un embedding pour chaque produit
2. À chaque recherche → Gemma 4 génère un embedding de la requête
3. pgvector trouve les produits les plus proches (cosine similarity)

```sql
-- Cherche les 5 produits les plus proches sémantiquement
SELECT nom, prix, 1 - (embedding <=> $1) AS score
FROM produits
ORDER BY embedding <=> $1
LIMIT 5;
```

**Avantage DGX Spark :** Les embeddings sont générés gratuitement en local — zéro coût d'API.

---

## 11. Risques & Mitigations

| Risque | Impact | Mitigation |
|--------|--------|------------|
| **Légal — données LM** | CRITIQUE | Données anonymisées + validation contrat |
| **Tool Calling Gemma 4 peu fiable** | HAUT | Fallback règles simples (regex) si Gemma échoue |
| **Latence Gemma 4** | MOYEN | DGX Spark GPU → < 2 sec par réponse |
| **Scraping LM cassé** | MOYEN | Promos non critiques (feature secondaire) |
| **Twilio coût** | BAS | ~€0.08/commande, acceptable MVP |
| **Time constraint** | MOYEN | Focus MVP strict — 2 features uniquement |

---

## 12. Timeline MVP (fin septembre 2026)

| Phase | Semaines | Tâches clés |
|-------|----------|-------------|
| **Phase 1 — Setup** | S1-S2 | Ollama + Gemma 4, PostgreSQL + pgvector, Redis, Twilio webhook |
| **Phase 2 — Feature 1** | S3-S4 | Tool Calling, search_products, create_basket, validation PIN |
| **Phase 3 — Feature 2** | S5-S6 | Playwright scraping promos, get_promos tool |
| **Phase 4 — Polish** | S7-S8 | Tests end-to-end, gestion erreurs, Streamlit dashboard |
| **Phase 5 — Présentation** | S9-S12 | Slides dirigeants, démo live, business case |

---

## 13. Variables d'Environnement (.env)

```env
# Twilio
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_PHONE_NUMBER=+1XXXXXXXXXX

# Base de données
DATABASE_URL=postgresql://user:password@localhost:5432/renoassistant

# Redis
REDIS_URL=redis://localhost:6379/0

# Ollama (Gemma 4)
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=gemma4:27b

# App
APP_SECRET_KEY=your-secret-key
DEBUG=true
```

---

## 14. Dimension GEO — Bénéfice Stratégique

### Définitions

**AEO (Answer Engine Optimization)** — Optimiser son contenu pour être la réponse directe retournée par les moteurs de recherche (Google Featured Snippets, assistants vocaux).

**GEO (Generative Engine Optimization)** — Optimiser son contenu pour être **cité comme source** par les IA génératives (ChatGPT, Perplexity, Google AI Overviews, Claude).

### Le lien avec RenoAssistant

US2 (filtre agentique) nécessite que le catalogue LM soit **structuré pour qu'une IA puisse le comprendre** :
- Descriptions produits riches en cas d'usage ("idéal pour béton, parpaing, pierre dure")
- Attributs bien typés (matériau cible, niveau pro, puissance, compatibilité)
- Tags sémantiques cohérents et complets
- Embeddings vectoriels générés sur chaque fiche produit

Ce travail de structuration est exactement ce que le GEO requiert pour être cité par des IA externes.

### Double bénéfice

```
Structuration catalogue LM (effort unique)
              ↓
┌─────────────────────────┬──────────────────────────────┐
│     Bénéfice interne    │      Bénéfice externe        │
│                         │                              │
│  US2 filtre mieux       │  Perplexity cite LM          │
│  Agent comprend le      │  ChatGPT recommande LM       │
│  catalogue natif        │  Google AI Overviews         │
│                         │  positionne LM en source     │
└─────────────────────────┴──────────────────────────────┘
```

### Argument pour la présentation aux dirigeants LM

> "Ce projet ne se limite pas à deux fonctionnalités. Le travail préliminaire de structuration des données produits que RenoAssistant nécessite constitue un investissement GEO : il prépare le catalogue Leroy Merlin à être cité et recommandé par les IA génératives comme ChatGPT, Perplexity ou Google AI Overviews. Un seul chantier data, deux retours sur investissement."

---

## 15. Questions Ouvertes

- [ ] Valider la légalité de l'utilisation des données LM avec le service juridique
- [ ] Tester le Tool Calling de Gemma 4 via Ollama (support natif à confirmer)
- [ ] Choisir entre Gemma 4 27B et 12B selon la latence réelle sur DGX Spark
- [ ] Identifier les contacts dirigeants LM pour la présentation
- [ ] Évaluer l'état actuel du catalogue LM vis-à-vis des critères GEO

---

*Dernière mise à jour : 17 juin 2026 — Ajout dimension GEO / AEO*
