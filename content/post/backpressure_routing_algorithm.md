+++
date = "2017-05-27T00:00:00Z"
tags = ["elixir", "warcraft"]
title = "Réglage de la pression arrière"
+++

Dans mes applications très dépendantes de l'API Battle.net, j'ai une problématique importante: La limitation du nombre de requêtes, qu'elle soit imposée par la société, la congestion du réseau où la non disponibilité du service. Grosso merdo un processus qui dit non, repasse plus tard, où attends un peu. Mine de rien le problème n'est pas si simple que ça j'ai beaucoup entendu le terme *backpressure*, certainement en référence à l'algorithme de routage *[backpressure]*, j'y pige pas grand-chose, mais la page Wikipédia est bien fournie, probablement un abus de langage, mais ça peut donner une approche de l'idée et surtout du concept qui va nous intéresser : les queues.

Voici un schéma sommaire de l'architecture, on reste dans le contexte erlang, avec des processus indépendants, mais un peu de multi threading revient à la même idée.

{{< image src="/images/basic_queue_workflow.png" title="Bassic Queue Workflow" >}}

ok ! C'est bien beau d'avoir un mécanisme de régulation, encore faut il lui donner des règles. On a droit a 3600 req/h et un *burst* de 100 req/s. Mettons de côté le *burst* pour le moment et restons à 1 req/s. Je n'ai pas trouvé de références intéressantes, mais on appelle ça un *ticker*, référence à notre cher organe le cœur. Dans notre exemple, on prends 1 req/s mais on peut se baser sur des algorithmes bien plus complexes comme celui de la librairie [ex_rated], qui essaye de coller au mieux aux algorithmes de limitation de débits des api sur la base de l'algorithme *[Token Bucket]* ( Il existe une francisation horrible de ce terme « Seau à jetons »).

Bref ! Admettons: j'ai un ticker qui choisis la vitesse a laquelle consommer la queue et des processus qui vont tranquillement la remplir, à quels problème puis-je être confronté.

1. Problème réseau : la requête n'arrive pas à destination
2. Requête mal formée : J'essaye de récupérer par exemple l'armurerie d'un level 6; en vain
3. L'API Battle.net n'est pas disponible, c'est le cas pendant les maintenances souvent.
4. J'ai un résultat, mais pas ce à quoi je m'attendais
5. J'ai atteint mon quota maximal de requêtes
6. la queue se remplit plus vite qu'elle ne se vide.

Le point 6 révèle probablement une inconsistance dans les algorithmes, ou tout du moins un fonctionnement incohérent avec les quotas. Néanmoins, je préfère considérer le cas du processus fou qui viendrait monopoliser la queue. Changeons un peu de design.


{{< image src="/images/workflow_with_subscribe.png" title="Workflow with subscribe" >}}


Yep ! Plutôt d'avoir des programmes qui vont empiler les requêtes sans avoir aucune connaissance de la pression qu'ils exercent sur le réseau ou le quota, on insère un *scheduler*, qui va s'adapter et demander X events max pour chaque processus. Ça parait un peu plus compliqué du coup, mais si à la place on avait un *event fuck off* la queue est plaine, renvoyé à X processus qu'ils devront gérer chacun de leur côté: c'est bien pire !

Point numéro 2:  ce n'est pas vraiment une erreur, [l'erreur 404] doit être attendue est traité convenablement, ça veut probablement dire que beaucoup de code d'erreurs auxquels je ne m'attends pas vont passer à la trappe, ce qui implique que j'ai un bonne gestion des rapports d'erreurs le temps que j'ai fait le tour de toutes les réponses possibles


Point numéro 4: C'est que je ne gère pas correctement les réponses de l'API, un petit rapport d'erreur avant de terminer le processus proprement.

Reste le point numéro 1. auquel ont peut associer certaines erreurs du type erreur 500 (*[Internal Server Error]*), OK la requête n'a pas marché on la retente. Disons 3 fois. Problème : si il y a beaucoup d'erreurs les erreurs vont monopoliser 75% des appels ! Là encore, on va faire appel à un algorithme réseau bien connu *[Exponential Backoff Algorithm]*, qui va tout simplement de plus en plus espacer chaque erreur.

Ça, c'est une erreur ponctuelle. Et pour le blackout total de l'API ou du réseau ? Pas vraiment la peine de spammer le réseau de requêtes qui n'arriverons pas ! Je vais faire appel à un design pattern tout simple un *[Circuit Breaker]*, un petit switch en amont des requêtes qui lorsque le réseau est hs va tout simplement court-circuiter les appels par un bon taggle dans les règles de l'art. Comment déterminer si le circuit est hs ? Si pour une *Time Frame* le pourcentage de paquet retournant une erreur est trop élevé, l'API est hs. On met tout en pause, on met en place un processus de ping qui va contrôler l'état de l'API et fermer (oui le pattern vient de l'électricité hein !) le circuit.

Derrière on met en place un système de ping qui va contrôler l'état de l'API jusqu'à ce qu'elle retourne un état positif et le système reprends son fonctionnement normal.

Un Schéma complet du bordel:

{{< image src="/images/complete_workflow.png" title="Complete Workflow" >}}



J'ai envisagé la possibilité d'avoir plusieurs queues, même si ce n'est pas vraiment à l'ordre du jour.


OK prochaine étape une implémentation solide de tout ça.

[backpressure]: https://en.wikipedia.org/wiki/Backpressure_routing
[ex_rated]: https://github.com/grempe/ex_rated
[Token Bucket]: https://en.wikipedia.org/wiki/Token_bucket
[Exponential Backoff Algorithm]: https://en.wikipedia.org/wiki/Exponential_backoff
[Circuit Breaker]: https://en.wikipedia.org/wiki/Circuit_breaker_design_pattern
[l'erreur 404]: https://fr.wikipedia.org/wiki/Erreur_HTTP_404
[Internal Serveur Error]: https://en.wikipedia.org/wiki/List_of_HTTP_status_codes#5xx_Server_error
