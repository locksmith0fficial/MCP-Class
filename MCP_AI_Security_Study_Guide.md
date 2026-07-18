# Guide d’étude — MCP et sécurité des agents IA

> **Leçons 3 à 5 : architecture, flux de confiance et Lethal Trifecta**  
> Fiche de synthèse, aide-mémoire et cheatsheet technique.

> [!IMPORTANT]
> **Le modèle propose. La politique autorise. Le backend vérifie.**

## Table des matières

- [1. Résumé compréhensible](#1-résumé-compréhensible)
- [2. Les trois ères d’architecture](#2-les-trois-ères-darchitecture)
- [3. Anatomie d’une architecture MCP](#3-anatomie-dune-architecture-mcp)
- [4. Transports MCP](#4-transports-mcp)
- [5. Parcours d’une requête MCP](#5-parcours-dune-requête-mcp)
- [6. Frontières de confiance](#6-frontières-de-confiance)
- [7. Perte d’identité et confused deputy](#7-perte-didentité-et-confused-deputy)
- [8. L’agent : nouveau consommateur des API](#8-lagent--nouveau-consommateur-des-api)
- [9. Lethal Trifecta](#9-lethal-trifecta)
- [10. Exemples présentés dans le cours](#10-exemples-présentés-dans-le-cours)
- [11. Pourquoi les prompts et filtres ne suffisent pas](#11-pourquoi-les-prompts-et-filtres-ne-suffisent-pas)
- [12. Comment casser la Lethal Trifecta](#12-comment-casser-la-lethal-trifecta)
- [13. Contrôles de sécurité prioritaires](#13-contrôles-de-sécurité-prioritaires)
- [14. Scorecard opérationnelle](#14-scorecard-opérationnelle)
- [15. Red flags techniques](#15-red-flags-techniques)
- [16. Nuances et corrections techniques](#16-nuances-et-corrections-techniques)
- [17. Mots-clés à étudier](#17-mots-clés-à-étudier)
- [18. Aides-mémoire](#18-aides-mémoire)
- [19. Cheatsheet technique](#19-cheatsheet-technique)
- [20. Questions de révision](#20-questions-de-révision)

---

## 1. Résumé compréhensible

Avant MCP, un modèle d’IA pouvait raisonner et produire du texte, mais il ne pouvait pas facilement agir dans des systèmes externes. Chaque connexion à Jira, GitHub, Gmail, une base de données ou Stripe devait être développée séparément.

> [!TIP]
> **Modèle mental : le LLM est le cerveau; MCP lui donne des mains et des outils.**

MCP, ou Model Context Protocol, standardise la façon dont une application d’IA découvre et utilise des capacités externes. Il ne télécharge pas une nouvelle connaissance dans le modèle : il lui présente des outils, leurs descriptions, leurs paramètres et la façon de les appeler.

> [!TIP]
> **À mémoriser : MCP donne des capacités au modèle, pas nécessairement du jugement.**

## 2. Les trois ères d’architecture

### 2.1 Application traditionnelle

```text
Utilisateur → Application → Backend/API
```

L’utilisateur clique sur des boutons; l’application transforme ces actions en appels API structurés. Les actions possibles sont limitées par l’interface et par la logique codée.

### 2.2 Application avec agent IA

```text
Utilisateur → Application/Agent IA → Backend/API
```

Le modèle interprète une demande en langage naturel et génère une action structurée. Une couche interprétative apparaît : elle peut mal comprendre l’intention, sélectionner le mauvais outil, halluciner un paramètre ou suivre une instruction malveillante.

### 2.3 Agent externe avec MCP

```text
Utilisateur → Hôte IA/LLM → Client MCP → Serveur MCP → Backend/API
```

L’utilisateur peut maintenant demander à un agent généraliste d’agir dans un système externe. L’architecture contient davantage de composants, de propriétaires, de credentials et de frontières de confiance.

## 3. Anatomie d’une architecture MCP

| Composant | Rôle | Risque principal |
| --- | --- | --- |
| Host / Hôte | Application utilisée par l’utilisateur; orchestre le modèle et les connexions. | Mauvaise gestion des permissions, du contexte ou des serveurs installés. |
| LLM / Agent | Interprète la demande, choisit les outils et construit les arguments. | Décision probabiliste, hallucination, prompt injection, mauvais enchaînement. |
| MCP client | Gère la connexion à un serveur MCP et transporte les appels. | Fait confiance à la décision de l’agent; validation ou authentification insuffisante. |
| MCP server | Expose les outils et traduit les appels MCP vers les API réelles. | Accès à des secrets, fichiers, bases de données et fonctions privilégiées. |
| Backend | Système final qui lit ou modifie les données. | Accepte un credential valide sans connaître l’identité ou l’intention d’origine. |

> [!TIP]
> **Le Host choisit, le Client transmet, le Serveur traduit, le Backend exécute.**

### MCP ne remplace pas les API

```text
Agent → MCP → API existante → Backend
```

Le changement réel est l’ajout d’un nouveau consommateur des API : l’agent IA. Le backend existait déjà, mais son exposition et les comportements possibles changent.

## 4. Transports MCP

### 4.1 STDIO — serveur local

```text
Host local <-> Processus MCP local
```

- Fréquent dans les applications de bureau et les IDE.
- Le serveur s’exécute comme un sous-processus local.
- Risques : paquet malveillant, supply chain, accès aux fichiers, variables d’environnement, secrets et exécution de code local.

### 4.2 Streamable HTTP — serveur distant

```text
Host → Réseau/Internet → Serveur MCP distant
```

- Fréquent pour les services SaaS et les environnements multi-tenant.
- Risques : absence d’authentification, fuite inter-tenant, vol de jetons, mauvaise configuration TLS et confiance envers le fournisseur.

> [!TIP]
> **Question essentielle : qui contrôle le serveur, où s’exécute-t-il et avec quels credentials?**

## 5. Parcours d’une requête MCP

> [!TIP]
> **Ask → Decide → Request → Execute → Respond**

1. Demander : l’utilisateur exprime son intention en langage naturel.
2. Décider : le modèle sélectionne l’outil et construit les paramètres.
3. Requêter : le client MCP encapsule la décision dans une requête structurée, souvent JSON-RPC.
4. Exécuter : le serveur MCP appelle l’API ou le système cible.
5. Répondre : le backend retourne le résultat jusqu’à l’utilisateur.

### Exemple JSON-RPC simplifié

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "jira.search",
    "arguments": {
      "query": "status = New"
    }
  }
}
```

## 6. Frontières de confiance

```text
Utilisateur → Agent → Client MCP → Serveur MCP → Backend
```

| Frontière | Hypothèse de confiance | Échec possible |
| --- | --- | --- |
| Utilisateur → Agent | La demande représente l’intention réelle. | Ambiguïté, ingénierie sociale, contenu indirect malveillant. |
| Agent → Client | L’agent a choisi le bon outil et les bons arguments. | Mauvais outil, montant erroné, paramètres hallucinés. |
| Client → Serveur | La requête est légitime et autorisée. | Serveur sans authentification, rejeu, usurpation, validation faible. |
| Serveur → Backend | Le credential technique représente une action légitime. | Le backend ignore l’utilisateur initial, son intention ou le contexte métier. |

> [!TIP]
> **Une identité technique valide ne prouve pas une intention humaine valide.**

## 7. Perte d’identité et confused deputy

```text
Utilisateur → Agent → MCP → Backend voit seulement « service-mcp »
```

Dans une architecture faible, le backend ne sait plus quel utilisateur humain a initié l’action. Un composant privilégié peut alors être manipulé pour utiliser ses permissions au bénéfice d’un acteur moins privilégié : c’est le problème du confused deputy.

### Contrôles recommandés

- Propager une identité déléguée par utilisateur lorsque possible.
- Utiliser des jetons à courte durée et des scopes minimaux.
- Faire revérifier les droits par le backend pour chaque objet et chaque tenant.
- Exiger une confirmation explicite pour les actions sensibles.
- Éviter les comptes de service « god mode » pour des tâches qui lisent du contenu non fiable.

## 8. L’agent : nouveau consommateur des API

| Consommateur | Comportement typique | Profil de risque |
| --- | --- | --- |
| Humain | Faible volume, un clic à la fois, limité par l’interface. | Erreurs humaines et abus interactifs. |
| Automatisation traditionnelle | Volume élevé, logique codée, endpoints connus. | Prévisible; dérives généralement mesurables. |
| Agent IA | Décisions dynamiques, enchaînement d’outils, langage naturel. | Comportements émergents, prompt injection, boucles, sur-permission. |

> [!TIP]
> **Une API est un outil. Un agent choisit comment utiliser les outils.**

Une décision produite par un modèle ne doit jamais servir seule de mécanisme d’autorisation pour une action sensible.

## 9. Lethal Trifecta

| Capacité | Définition | Exemples |
| --- | --- | --- |
| Private Data | Accès à des données sensibles. | Courriels, dépôts privés, dossiers clients, secrets, bases de données. |
| Untrusted Content | Lecture de contenu contrôlé par une source externe. | Courriel, ticket, issue publique, page Web, document téléversé. |
| External Communication | Capacité d’envoyer ou rendre des données visibles à l’extérieur. | Courriel, webhook, PR publique, réponse client, image Markdown distante. |

```text
Private Data + Untrusted Content + External Communication = chemin structurel d’exfiltration
```

Chaque capacité seule peut être acceptable. Lorsque le même agent ou le même workflow possède les trois, un contenu non fiable peut influencer l’agent, accéder à des données privées et les transmettre vers une destination externe.

> [!TIP]
> **L’attaquant a besoin des trois jambes; le défenseur doit en casser au moins une.**

## 10. Exemples présentés dans le cours

### Echo Leak

- Données privées : contenu de la boîte courriel.
- Contenu non fiable : courriel contrôlé par l’attaquant.
- Sortie externe : chargement d’une image Markdown ou d’une URL distante.

### Supabase MCP

- Données privées : base de données accessible avec un credential très privilégié.
- Contenu non fiable : ticket de soutien soumis par un client.
- Sortie externe : réponse publiée dans le ticket visible par l’attaquant.

### GitHub MCP

- Données privées : dépôts privés.
- Contenu non fiable : issue publique.
- Sortie externe : pull request ou contenu visible publiquement.

> [!TIP]
> **Dans ces scénarios, le système n’a pas nécessairement « planté » : il a utilisé ses outils selon un contexte mal séparé.**

## 11. Pourquoi les prompts et filtres ne suffisent pas

Un system prompt, un classificateur ou un filtre peut réduire le risque, mais ne doit pas constituer la frontière principale de sécurité. Le problème vient souvent de la structure des accès, des permissions et des canaux de sortie.

> [!TIP]
> **On ne corrige pas uniquement avec un prompt un problème créé par l’architecture.**

- Séparer les instructions de confiance des données non fiables.
- Ne pas donner à un même workflow un accès inutile aux trois jambes de la trifecta.
- Faire appliquer l’autorisation par du code déterministe et par le backend.
- Utiliser des contrôles techniques même si le modèle affirme que l’action est sûre.

## 12. Comment casser la Lethal Trifecta

| Jambe à retirer ou limiter | Contrôles possibles |
| --- | --- |
| Données privées | Moindre privilège, vues limitées, row-level security, séparation des tenants, masquage des secrets, token à portée réduite. |
| Contenu non fiable | Isolation, provenance, conversion vers formats structurés, sandbox, agent de lecture séparé, aucune action privilégiée dans le même workflow. |
| Communication externe | Allowlist de destinations, blocage des URL arbitraires, DLP, approbation humaine, outils d’écriture séparés, ressources distantes désactivées. |

## 13. Contrôles de sécurité prioritaires

### Moindre privilège

Chaque outil doit recevoir uniquement les permissions nécessaires. Un agent de soutien ne devrait pas avoir un accès administrateur à toute la base de production.

### Identité et autorisation

- Identifier l’utilisateur humain, l’agent, le serveur MCP et le credential utilisé.
- Autoriser chaque action selon l’objet, le tenant, le montant et le contexte métier.
- Préférer les tokens délégués aux comptes de service partagés.

### Approbation humaine

Afficher l’action exacte avant confirmation : type d’action, cible, montant, destination, identité de l’initiateur et données concernées.

### Limites de comportement

- Rate limits, budgets par tâche, nombre maximal d’appels et timeouts.
- Limites de montant, restrictions de destination et détection de boucles.
- Idempotence et contrôle des chaînes d’outils.

### Journalisation

```text
Utilisateur → Session → Modèle → Outil → Arguments → MCP → Credential → Backend → Résultat → Destination
```

## 14. Scorecard opérationnelle

| Outil | Données privées? | Contenu non fiable? | Sortie externe? | Credential | Approbation? |
| --- | --- | --- | --- | --- | --- |
| repos.read | Oui | Non | Non | OAuth utilisateur | Non |
| issues.read | Non | Oui | Non | OAuth utilisateur | Non |
| pull_request.create | Non | Non | Oui | OAuth organisation | Oui |
| database.query | Oui | Possible | Non | Service role | Oui |
| ticket.reply | Non | Oui | Oui | Compte support | Oui |

Ne pas analyser seulement chaque outil isolément. Plusieurs outils combinés dans une même tâche peuvent compléter les trois jambes de la trifecta.

### Questions de revue par outil

1. Qui développe et maintient le serveur?
2. STDIO local ou HTTP distant?
3. Comment le client et l’utilisateur sont-ils authentifiés?
4. L’identité humaine est-elle propagée jusqu’au backend?
5. Quels tenants, objets et données le credential peut-il atteindre?
6. L’outil ingère-t-il du contenu externe?
7. Peut-il écrire ou envoyer vers l’extérieur?
8. Quelles actions nécessitent une approbation?
9. Quelles limites de volume et de fréquence existent?
10. Peut-on reconstruire toute la chaîne dans les logs?

## 15. Red flags techniques

- Serveur MCP public sans authentification.
- Paquet MCP installé depuis une source inconnue.
- Credential administrateur ou service role partagé.
- Accès simultané à des données privées et à du contenu externe.
- Capacité d’effectuer des requêtes HTTP, SQL ou commandes arbitraires.
- Agent pouvant publier ou envoyer des données sans approbation.
- Identité humaine non transmise au backend.
- Aucune limite de fréquence ou séparation multi-tenant.
- Secrets accessibles dans les variables d’environnement.
- Logs incapables d’identifier l’utilisateur initial et la destination finale.

## 16. Nuances et corrections techniques

| Affirmation simplifiée | Nuance à retenir |
| --- | --- |
| « MCP télécharge une compétence » | Il expose des outils; il ne modifie pas nécessairement les poids ou connaissances du modèle. |
| « Plus besoin d’API ni de documentation » | Le serveur MCP doit toujours intégrer les API, gérer l’authentification, les permissions, les schémas et la maintenance. |
| « Le serveur MCP est le seul point d’attaque » | Le host, le contenu, les descriptions d’outils, le client, le transport et le backend font aussi partie du threat model. |
| « Le backend suit simplement les règles » | Il doit revérifier l’identité, l’autorisation, le tenant, la cible et le contexte métier. |
| « L’IA est aléatoire » | Elle peut être non déterministe; cela ne signifie pas qu’elle est entièrement chaotique. |

## 17. Mots-clés à étudier

| Terme | Définition courte |
| --- | --- |
| MCP | Protocole standardisé permettant à une application IA d’utiliser des capacités externes. |
| Host | Application qui héberge l’expérience IA et les connexions. |
| MCP client | Composant qui gère une connexion à un serveur MCP. |
| MCP server | Composant qui expose les outils et appelle les systèmes cibles. |
| Tool | Fonction structurée que le modèle peut demander d’exécuter. |
| JSON-RPC | Format structuré de requêtes et de réponses. |
| Indirect prompt injection | Instruction malveillante cachée dans un contenu que l’agent lit. |
| Confused deputy | Composant privilégié manipulé pour agir au bénéfice d’un acteur non autorisé. |
| Identity propagation | Conservation de l’identité d’origine dans toute la chaîne. |
| Tool chaining | Utilisation successive de plusieurs outils pour une même tâche. |
| Tenant isolation | Séparation des données et actions entre clients ou organisations. |
| Human-in-the-loop | Approbation humaine avant une action sensible. |

## 18. Aides-mémoire

> [!TIP]
> **Composants : H → C → S → B — Host, Client, Server, Backend**

> [!TIP]
> **Flux : Ask → Decide → Request → Execute → Respond**

> [!TIP]
> **Trifecta : P + U + E — Private, Untrusted, External**

- MCP donne des mains, pas de la sagesse.
- Le badge prouve qui entre, pas pourquoi l’action est correcte.
- Le modèle propose; la politique autorise; le backend vérifie.
- Une API est un outil; un agent choisit comment utiliser les outils.

## 19. Cheatsheet technique

```text
ARCHITECTURE
User → Host/LLM → MCP Client → MCP Server → Backend/API

FLOW
Ask → Decide → Request → Execute → Respond

TRANSPORTS
STDIO: local subprocess; supply-chain, secrets, filesystem, code execution
HTTP: remote server; auth, TLS, tenant isolation, provider trust

LETHAL TRIFECTA
Private Data + Untrusted Content + External Communication

PRIORITY CONTROLS
Least privilege | Per-user identity | Narrow scopes | Tenant isolation
Destination allowlists | Human approval | Rate limits | Tool-call logging
Backend authorization | Separation of read/write capabilities
```

> [!TIP]
> **Règle finale : ne jamais utiliser le jugement probabiliste d’un LLM comme seule frontière d’autorisation pour une action sensible.**

## 20. Questions de révision

1. MCP remplace-t-il les API? — Non, il s’appuie généralement sur les API existantes.
2. Quel composant choisit l’outil? — Le modèle ou la logique de l’agent dans le host.
3. Quel composant appelle réellement Jira ou Stripe? — Le serveur MCP.
4. Pourquoi un credential valide ne suffit-il pas? — Il ne prouve pas l’identité ni l’intention de l’utilisateur initial.
5. Quelles sont les trois jambes de la Lethal Trifecta? — Données privées, contenu non fiable, communication externe.
6. Un seul outil doit-il posséder les trois capacités? — Non, leur combinaison dans un workflow suffit.
7. Pourquoi un meilleur prompt n’est-il pas suffisant? — Le risque vient aussi des permissions et de l’architecture.
8. Quelle défense a le plus fort levier? — Supprimer ou isoler au moins une jambe de la trifecta.
9. Quelle règle doit rester dans le backend? — Vérifier l’identité, l’autorisation et le contexte métier.
10. Quel principe résume la sécurité des agents? — Le modèle propose; le système déterministe décide et contrôle.

---

## Utilisation recommandée

- Utiliser la **scorecard opérationnelle** avant d’ajouter un nouvel outil ou serveur MCP.
- Vérifier les trois jambes de la **Lethal Trifecta** au niveau du workflow complet, pas seulement outil par outil.
- Conserver l’autorisation sensible dans du code déterministe et dans le backend.

## Licence et attribution

Ce fichier est une fiche d’étude dérivée des notes de cours fournies par l’utilisateur. Adapter la section de licence selon le dépôt cible.
