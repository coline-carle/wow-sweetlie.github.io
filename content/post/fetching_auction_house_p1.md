+++
date = "2017-05-12T00:00:00Z"
tags = ["elixir", "warcraft"]
title = "Récupération des données de l'hôtel des ventes, Partie 1"
+++

Bon, j'en ai un peu marre de la limitation de l'API de trafic de TradeskillMaster sur son API, qui plus est j'ai envie de faire joujou et potentiellement publier mes résultats, pas sûr que la société apprécie mon usage de ses données.
Mais quitte à faire un truc pas forcement passionnant au départ, autant y rajouter un peu de fun : le langage de programmation Elixir.

Elixir Késako ?

De la programmation fonctionnelle, avec une syntaxe sexy fonctionnant sur la VM BEAM, du langage erlang accès programmation concurrentielle.

Hop mon premier morceau de programme Elixir :
{{< codeblock "battlenet.ex" >}}
~~~
require Logger

defmodule BattleNET do
  defmodule AuctionHouseSnapshot do
    defmodule Snapshot do
      defstruct  [:lastModified, :url]
    end

    def get(region, realm, params \\ []) do
      BattleNET.get(region, "wow/auction/data/" <> realm, params)
      |> Map.get(:body)
      |> Poison.decode!(as: %{"files" => [%Snapshot{}]})
    end
  end


  def get(region, path, params) do
    build_url(region, path, params)
    |> HTTPoison.get!
  end


  defp process_params(params) do
    params
    |> Keyword.merge([apikey: key()])
    |> URI.encode_query
  end

  defp build_url(region, path, params) do
    "https://" <> region <> ".api.battle.net/" <>
      path <> "?" <> process_params(params)
  end

  defp key do
    System.get_env("BNET_API_KEY")
  end
end
~~~
{{< /codeblock >}}

Bilan

* je me suis contenté de l'essentiel / récupérer l'url du snaphsot de l'hv. Un bon moyen pour arrêter ma folie d'overthink de l'OO
* Pas de gestion d'erreur ? Non. Tout le code est contenu dans un process indépendant s'il doit crasher, il crashera. Ce sera le boulot du superviseur de le relancer. Je ne suis pas sûr que la gestion d'erreur soit absente dans tous cas, mais se passer de quelque chose de chiant pour se poser la question comment s'en relever en un minimum d'effort c'est très appréciable.
* Pas de test ? Même idée que pour les erreurs il y a en effet un test à faire est-ce que dans la situation ou ça marche j'ai bien le résultat que j'attends. Pas d'edge case !

C'est très positif, le côté performance est actuellement bonus, mais je me dis que le bonus de pouvoir analyser de quantité importante de données en quelques secondes en lançant plein de serveurs virtuels à la demande et laisser un serveur cheap se charger de servir les données aux clients c'est définitivement un truc que j'aimerais tester !

Au programme de la suite le superviseur.
