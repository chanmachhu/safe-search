# safe-search

Un skill Claude qui répond aux questions factuelles et vérifie les affirmations douteuses (rumeurs, citations, chiffres, articles) **uniquement à partir de sources ouvertes, datées et indépendantes**, avec la liste des sources à la fin.

## Ce que fait le skill

Deux modes :
- **Standard** : `/safe-search <question ou affirmation>`
- **Strict** : `/safe-search strict <question ou affirmation>`, qui ajoute une double vérification indépendante par de petits agents.

Pour chaque demande, le skill :
1. découpe l'entrée en faits individuels ;
2. cherche d'abord si l'affirmation a déjà été vérifiée par un fact-checker reconnu (membre EFCSN ou IFCN) ;
3. remonte à l'origine de l'information (étude, texte officiel, citation d'origine) ;
4. juge chaque source par lecture latérale (qui en parle, quelle date, quel type) ;
5. note, pour chaque fait, une citation mot pour mot de la page ouverte ;
6. recoupe avec des sources indépendantes et donne un niveau de confiance (Élevée / Moyenne / Faible) ;
7. relit les pages citées avant de répondre, et retire tout ce qu'elles ne disent pas ;
8. (strict) fait revérifier les points à risque par des agents séparés.

Règles : 8 faits clés au plus, un fait par ligne, 1 à 2 sources par fait, source primaire d'abord, formulation fidèle à la source, introduction qui ne fait que résumer. Pour une affirmation, la réponse donne un verdict (Vrai, Plutôt vrai, Trompeur, Hors contexte, Faux, Invérifiable pour l'instant), une section « Ce que je n'ai pas pu vérifier » et une liste de sources au format fixe. La réponse est dans la langue de l'utilisateur.

Ce que le skill ne fait pas : vérifier qu'une image a été retouchée ou réutilisée, ni tenir une liste de sites douteux.

## Installation (plugin)

Le dépôt est un plugin : il contient le skill (`skills/`), l'agent (`agents/`) et les fichiers `.claude-plugin/plugin.json` et `.claude-plugin/marketplace.json`. Il n'utilise aucun connecteur : seulement les outils intégrés WebSearch et WebFetch.

```
/plugin marketplace add chanmachhu/safe-search
/plugin install safe-search@safe-search
```

Utilisation : `/safe-search <question ou affirmation>` ou `/safe-search strict <question ou affirmation>`.

À tester : le nom exact de l'agent une fois le plugin installé (`verifier` ou `safe-search:verifier`). Le skill accepte les deux ; si aucun n'existe, la vérification est faite par l'assistant lui-même et signalée comme non indépendante.

## Contenu du dépôt

```
skills/safe-search/SKILL.md   le skill
.claude-plugin/plugin.json     description du plugin
.claude-plugin/marketplace.json  permet d'installer le plugin depuis GitHub
LICENSE                       licence MIT
agents/verifier.md            l'agent vérificateur utilisé par le mode strict
docs/RAPPORT-test.md          rapport du test d'évaluation
docs/REPONSES.md              les 15 réponses du test et les erreurs relevées (lisible sur GitHub)
```

## Pourquoi l'agent `verifier`

Le mode strict lance un petit agent par source citée pour revérifier les affirmations à risque (chiffres, citations, auteurs, formulations). Un agent lancé avec les réglages par défaut reçoit tous les outils et tout le contexte de la session : dans nos mesures, **un agent qui ne fait rien consomme 72 564 tokens** juste pour démarrer. Avec 4 à 5 vérificateurs par réponse, c'est ce démarrage, et non la vérification, qui rend le mode strict cher.

L'agent `verifier` est limité à **WebSearch et WebFetch** (les deux seuls outils dont un vérificateur a besoin : trouver la page, l'ouvrir) et tourne sur Haiku. Mesure : **3 487 tokens** pour le même test à vide, soit environ 21 fois moins. Les consignes sont dans le message de lancement et le verdict revient dans la réponse de l'agent, sans fichiers.

Installation : en plugin (voir ci-dessous), l'agent est installé avec le skill. Sans plugin, copier `agents/verifier.md` dans `.claude/agents/` (projet) ou `~/.claude/agents/` (global), **avant de démarrer la session** ; un agent ajouté en cours de session n'est pas reconnu. Sans lui, le skill ne lance aucun autre agent : la vérification est faite par l'assistant lui-même et signalée comme non indépendante.

Coût mesuré : un vérificateur `verifier` consomme environ 9 500 tokens (contre environ 72 000 pour un agent par défaut) ; le mode strict coûte environ 117 000 tokens par réponse contre environ 84 000 en standard.

---

# safe-search — rapport du test final (avec l'agent `verifier`)

**Protocole.** 5 questions (une citation, une statistique officielle, un mythe scientifique, un fait vérifiable, un événement récent avec enquête en cours : Voltaire, population de la France, « 10 % du cerveau », Grande Muraille, vol du Louvre), chacune dans 3 configurations : sans skill, `/safe-search` (standard), `/safe-search strict` (un agent `verifier` par source citée). Un seul passage par cas, soit 15 réponses. Chaque réponse a été anonymisée (A/B/C, ligne « Vérification indépendante » retirée) puis fact-checkée à l'aveugle par un agent indépendant qui ne savait pas quelle configuration l'avait produite. Les réponses exactes et les erreurs relevées sont dans [docs/REPONSES.md](docs/REPONSES.md).

**Lecture des indicateurs.** Taux de confiance = (affirmations correctes + ½ × partiellement vraies) ÷ affirmations vérifiables. Les réponses n'ont pas toutes le même nombre d'affirmations (de 5 à 12) : un seul point change vite un pourcentage.

## Résultats (moyenne sur 5 cas)

| Indicateur | Sans skill | Standard | Strict |
|---|---|---|---|
| Taux de confiance | 89,7 % | 94,3 % | **96,6 %** |
| Taux de confiance strict (seulement « correct ») | 83,2 % | 90,8 % | **93,1 %** |
| Affirmations fausses (total) | 2 | 1 | **0** |
| Affirmations partiellement vraies (total) | 6 | 3 | 3 |
| Réponses avec confiance ≥ 90 % | 2/5 | 3/5 | **5/5** |
| Sources citées à problème | **6 sur 10** (3 incorrectes, 3 invérifiables : sans URL ou page illisible) | 0 sur 17 | 0 sur 15 |
| Réponses dont les sources sont toutes correctes | 0/4 (la 5e ne cite aucune source) | 5/5 | 5/5 |
| Signalements de sur-confiance | 4 | 4 | **1** |
| Verdict final correct | 5/5 | 5/5 | 5/5 |
| **Tokens par réponse** | ≈ 72 000 | ≈ 84 000 | **≈ 117 000** |
| Durée moyenne | ≈ 21 s | ≈ 60 s | ≈ 98 s (rédaction 60 s + vérificateur le plus long 23 s + correction 14 s, en moyenne ; hors délai d'orchestration) |

### Cas par cas (taux de confiance)

| Question | Sans skill | Standard | Strict |
|---|---|---|---|
| Voltaire | 85 % | 100 % | 100 % |
| Population de la France | 94 % | 100 % | 100 % |
| « 10 % du cerveau » | 83 % | 89 % | 94 % |
| Grande Muraille | 86 % | 89 % | 93 % |
| Vol au Louvre | **100 %** | 94 % | 95 % |

## Coût du mode strict (mesuré)

- Un vérificateur `verifier` (WebSearch et WebFetch seulement) consomme **≈ 9 500 tokens** (de 8 965 à 10 361 sur 15 mesures), contre ≈ 72 000 pour un agent par défaut. À vide : 3 487 tokens (`verifier`) contre 72 564 (par défaut).
- 15 vérificateurs pour 5 réponses (3 par réponse), soit ≈ 28 600 tokens de vérification par réponse. Avec l'agent par défaut, le même nombre de vérificateurs aurait coûté ≈ 215 000 par réponse.
- Le strict coûte ≈ 117 000 tokens contre ≈ 84 000 pour le standard (×1,4) et ≈ 72 000 sans skill (×1,6). Dans le test précédent, avec des agents par défaut, il coûtait ≈ 445 000 par réponse (estimation d'après les 29 vérificateurs mesurés), soit environ 5 fois le standard.

## Modèles utilisés (lus dans les journaux des agents)

| Rôle | Dernier test (5 cas) | Test précédent (10 cas) |
|---|---|---|
| Sans skill, standard, rédaction et correction du strict | **claude-sonnet-5-5** | **claude-opus-5-5** |
| Vérificateurs `verifier` (strict) | **claude-haiku-5-5** pour 3 groupes sur 15 (défaut de `agents/verifier.md`) ; **claude-sonnet-5-5** pour les 12 autres (imposé au lancement pour les points qui demandent du jugement) | agents par défaut, Haiku 5.5 ou Sonnet 5.5 selon le groupe |
| Fact-check aveugle | **claude-sonnet-5-5** | **claude-sonnet-5-5** |
| Agent à vide (mesure du coût) | `verifier` : **claude-haiku-5-5** ; agent par défaut forcé en Haiku : **claude-haiku-5-5** | n/a |

Les agents « sans skill », « standard » et le rédacteur strict héritent du modèle de la session, qui est passé d'Opus 5.5 à Sonnet 5.5 entre les deux tests : **les deux tests ne sont pas comparables entre eux**. Dans un même test, les trois configurations ont tourné sur le même modèle. Pour le dernier test, le modèle a été vérifié dans le journal de chacun des 37 agents (15 qui ont produit les réponses, dont les 5 rédacteurs du strict qui ont aussi appliqué les corrections, 15 vérificateurs, 5 fact-checkers aveugles, 2 agents à vide). Pour le test précédent, seulement sur un échantillon de chaque rôle.

## Conclusion

**Ce que le test établit**
- **Le skill rend les sources fiables.** Sans skill, 6 sources citées sur 10 posent problème : 3 sont incorrectes (un titre d'article qui n'est pas le vrai, une affirmation attribuée au mauvais ouvrage, un chiffre dépassé) et 3 sont invérifiables (pas d'URL, ou page jamais ouverte). Avec le skill, 0 sur 32, et chaque réponse a ses sources ouvertes et datées. Trois des cinq réponses sans skill (Voltaire, « 10 % du cerveau », Grande Muraille) disent elles-mêmes avoir été écrites de mémoire, sans recherche en ligne, et ne donnent aucune URL ; celle de la Grande Muraille ne cite aucune source.
- **La confiance globale monte** : 89,7 % sans skill, 94,3 % en standard, 96,6 % en strict, avec moins de formulations approximatives (6 affirmations partiellement vraies sans skill, 3 avec).
- **Les vérificateurs corrigent de vrais défauts.** Dans les réponses strictes finales, ils ont fait retirer une auteure que la page ne confirmait pas (Grande Muraille), nommer la propriétaire d'une couronne (Louvre), enlever un « environ » absent de la source (population) et abaisser une confiance de Élevée à Moyenne (10 %).
- **Le coût du strict est maîtrisé** avec l'agent `verifier` : environ 1,4 fois le standard.

**Ce que le test n'établit pas**
- **L'écart sur les erreurs de fond est mince.** Il y a 2 affirmations fausses sans skill, 1 en standard, 0 en strict, sur une quarantaine d'affirmations par configuration. L'écart repose sur 1 ou 2 affirmations ; ce peut être du bruit.
- **Le strict ne se démarque pas nettement du standard** : +2,3 points de confiance, au moins égal dans les 5 cas mais meilleur dans 3 seulement.
- **Le skill ne gagne pas partout.** Pour le Louvre, le standard (94 %) fait moins bien que l'absence de skill (100 %). Pour les questions très connues (Voltaire, population), l'absence de skill est déjà correcte sur le fond : le plafond est vite atteint.
- Le verdict final est correct dans les 15 réponses : sur ces questions, le skill n'a pas changé la conclusion donnée à l'utilisateur.

En résumé : sur des questions simples, le skill sert surtout à **prouver** (sources ouvertes, datées, exactes) plus qu'à **corriger**. Son bénéfice sur le fond reste à démontrer sur des questions où l'erreur est plus probable.

## Limites

- 5 cas, un seul passage, un fact-checker qui est lui-même un modèle et a pu se tromper.
- Questions peu exigeantes : mythes classiques et chiffres officiels faciles à trouver, sur lesquels un modèle sans outil est souvent déjà bon.
- Pas de réponse de référence écrite à l'avance : le fact-checker juge d'après ses propres recherches.
- Le coût d'un vérificateur dépend des outils et du contexte chargés dans l'environnement ; les chiffres sont ceux de l'environnement de test.
- Test précédent (10 cas, agents par défaut, modèle Opus 5.5), pour mémoire : confiance 93,7 % sans skill, 97,7 % en standard, 96,7 % en strict ; coût du strict ≈ 445 000 tokens. Non comparable au dernier test.

## Suite proposée : un test sur des questions à enjeu

Les questions de ce test étaient volontairement simples. Un second test devrait viser ce pour quoi le skill est fait : les cas où se tromper coûte cher et où un modèle sans recherche a plus de risques de se tromper.

- **Sujets à enjeu** : santé (interactions, seuils, rappels de produits), droit et démarches (délais, montants en vigueur), finance personnelle (taux, plafonds), actualité récente et contestée, rumeurs virales sur des événements en cours.
- **Cas pièges** : chiffre récemment mis à jour (l'ancienne valeur circule encore), citation attribuée à tort et déjà démentie, étude retirée ou contredite, deux sources officielles qui divergent, page inaccessible, affirmation vraie mais sortie de son contexte.
- **Plus de cas, plus de passages** : au moins 20 questions et 3 passages chacune, pour que 1 ou 2 erreurs ne décident pas du résultat.
- **Réponse de référence** écrite avant les tests, plutôt qu'un seul fact-checker : deux fact-checkers indépendants ou un modèle plus puissant, et les désaccords tranchés à la main.
- **Même modèle partout**, et des mesures séparées pour les erreurs de fond, les erreurs de source et les formulations trop fortes.
- **Question de coût** : vérifier si le strict (≈ 1,4 fois le standard) se justifie spécifiquement sur ces cas à enjeu.
