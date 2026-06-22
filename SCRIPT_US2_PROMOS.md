# Filtre Agentique — Navigation LM intelligente

## En une phrase
Pendant sa navigation sur le site ou l'app LM, le client pose une question en langage naturel. L'agent traduit sa demande en filtres et met à jour les résultats instantanément.

---

## Le concept

Un bouton discret sur le site et l'app LM ouvre un agent conversationnel.
Le client décrit son besoin librement, comme il parlerait à un vendeur.
L'agent comprend la demande et applique les filtres existants du catalogue LM en temps réel — sans que le client touche un seul filtre manuellement.

---

## Comment ça marche

1. **Le client clique sur le bouton agent** → pendant sa navigation sur une page de résultats
2. **Il pose sa question librement** → en langage naturel
3. **L'agent analyse le contexte** → page actuelle + filtres disponibles + demande client
4. **L'agent traduit en filtres** → catégorie, prix, marque, usage, matériau…
5. **Les résultats se mettent à jour** → en utilisant les filtres et la base LM existants
6. **Le client affine si besoin** → la conversation continue

---

## Exemples de conversations

**Navigation sur "Perceuses"**
> Client : "je cherche quelque chose pour percer du béton dur, budget 100€ max"
> Agent  : applique → type=burin, usage=béton, prix≤100€
> Résultats : mis à jour automatiquement

**Navigation sur "Peintures"**
> Client : "une peinture lessivable pour une cuisine avec des enfants"
> Agent  : applique → type=lessivable, pièce=cuisine, usage=intensif
> Résultats : mis à jour automatiquement

**Affinage**
> Client : "plutôt en blanc cassé"
> Agent  : ajoute → couleur=blanc cassé
> Résultats : affinés

---

## Chiffres clés

- 1 bouton agent sur site + app LM
- Filtres existants LM utilisés (aucun développement DB supplémentaire)
- Réponse en temps réel
- Conversation multi-tours possible
- Zéro friction — le client parle, les résultats suivent

---

## Bénéfices

**Pour le client** → Trouve ce qu'il cherche sans manipuler les filtres manuellement, navigation guidée comme avec un vendeur expert

**Pour Leroy Merlin** → Augmentation du taux de conversion, réduction des abandons de navigation, données sur les besoins réels des clients PRO

---

## User Story

> En tant que **client PRO Leroy Merlin**,
> je veux **poser une question en langage naturel pendant ma navigation**,
> afin que **l'agent applique automatiquement les bons filtres et m'affiche les produits qui correspondent à mon besoin**.
