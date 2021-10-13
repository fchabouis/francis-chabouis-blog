---
title: "Stream data from an API to your database with Elixir"
date: 2021-10-08T08:21:16+02:00
draft: false
tags: ["elixir", "stream"]
---

## Make your Elixir streams come true

Last time we talked about simply [streaming a paginated API](/posts/stream-api-with-elixir/) with Elixir. It was fun, but somehow pointless, because we ended up writing this:

```elixir
datasets = stream_api.("https://data.gouv.fr/api/1/datasets/") |> Stream.take(50) |> Enum.to_list()
```

Streams in Elixir are lazy, meaning that no work will be done by the stream until necessary. If you just write:

```elixir
datasets = stream_api.("https://data.gouv.fr/api/1/datasets/") |> Stream.take(50)
```

You create a stream, say that you are only interested by the first 50 elements. But if you don't *do* anything that actually needs the streamed values, nothing happens.

An easy way to say that you need the values is to put them in a list by calling `Enum.to_list()`. The `Enum` module is *eager* (as opposed to the lazy stream) : you want the value in a list, no matter if you use them later or not.

Let's try to do something more concrete, where streaming is actually useful : store the incoming data in a database.

## The streaming benefits

Of course there is an easy way to fetch some data from an API and to save it in a database:
1. Fetch all the data
2. Save it in the database

But problems will arise when "all the data" become too big. Too big to fit into your RAM for example. If you try to play with the Twitter API this way, you will soon be in trouble.

Luckily we already know how to stream the incoming API data, that's what we did [last time](/posts/stream-api-with-elixir/). And the good news is, streaming **into** a database is simple with Ecto!

## Ecto streaming
There is not much to say about Ecto streaming, except that, as explained in the [Ecto doc](https://hexdocs.pm/ecto/Ecto.Repo-callback-stream.html), Ecto streaming happens inside a **transaction**, via the `Repo.transaction` callback.

So if you stream data from the API to the database, and in the middle of the process an exception is raised, you've got a rollback. That means that nothing is actually commited in the database.

Apart from that, there is nothing complicated about it. Let say we have a database to store datasets information coming from the API. 

We have a  `datasets` table and a corresponding `Dataset` schema. Our table has 2 columns : one to store the dataset id (called `external_id`) and one for its title (called `title`).

Streaming into the database requires only a couple of lines of code:

```elixir
# create the incoming API stream
datasets = stream_api.("https://data.gouv.fr/api/1/datasets/") 

# the Ecto transaction wraps the insertion logic
Repo.transaction(fn ->
  # our incoming API stream
  datasets
  # we only keep the dataset information we need : id and title
  |> Stream.map(fn dataset ->
    %{
      external_id: dataset["id"],
      title: dataset["title"]
    }
  end)
  # in this example, we limit ourselfs to 200 datasets only
  |> Stream.take(200)
  # make batches of 100 datasets
  |> Stream.chunk_every(100)
  # insert them in the table
  |> Stream.each(fn changesets -> Repo.insert_all(Dataset, changesets) end)
  # we trigger the stream!
  |> Stream.run()
end)
```

Let's think a minute about what is supposed to happen. By default the API sends 20 results per page. We want to insert information in the database by batches of 100 records. So we expect to have:
- 5 calls to the API (5 calls * 20 results = 100 records)
- followed by 1 insert operation
- followed by 5 more calls to the API (5 calls * 20 results = 100 records)
- followed by 1 insert operation

And reach the end of the program, as 200 records have been inserted, which is what we asked for.

Lets's look at the logs:

```

11:14:12.271 [debug] QUERY OK db=0.5ms queue=0.1ms idle=1151.6ms
begin []

11:14:13.522 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/

11:14:14.061 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=2

11:14:14.461 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=3

11:14:14.989 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=4

11:14:15.296 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=5

11:14:15.305 [debug] QUERY OK db=3.3ms
INSERT INTO "datasets" ("external_id","title") VALUES ($1,$2),($3,$4),($5,$6),($7,$8),($9,$10),($11,$12),($13,$14),($15,$16),($17,$18),($19,$20),($21,$22),($23,$24),($25,$26),($27,$28),($29,$30),($31,$32),($33,$34),($35,$36),($37,$38),($39,$40),($41,$42),($43,$44),($45,$46),($47,$48),($49,$50),($51,$52),($53,$54),($55,$56),($57,$58),($59,$60),($61,$62),($63,$64),($65,$66),($67,$68),($69,$70),($71,$72),($73,$74),($75,$76),($77,$78),($79,$80),($81,$82),($83,$84),($85,$86),($87,$88),($89,$90),($91,$92),($93,$94),($95,$96),($97,$98),($99,$100),($101,$102),($103,$104),($105,$106),($107,$108),($109,$110),($111,$112),($113,$114),($115,$116),($117,$118),($119,$120),($121,$122),($123,$124),($125,$126),($127,$128),($129,$130),($131,$132),($133,$134),($135,$136),($137,$138),($139,$140),($141,$142),($143,$144),($145,$146),($147,$148),($149,$150),($151,$152),($153,$154),($155,$156),($157,$158),($159,$160),($161,$162),($163,$164),($165,$166),($167,$168),($169,$170),($171,$172),($173,$174),($175,$176),($177,$178),($179,$180),($181,$182),($183,$184),($185,$186),($187,$188),($189,$190),($191,$192),($193,$194),($195,$196),($197,$198),($199,$200) ["5c4ae55a634f4117716d5656", "Demandes de valeurs foncières", "57457f7cc751df61688cc4b3", "Liste des départements avec arrêtés préfectoraux sur la pénurie d'essence au 21 mai 2016", "5de8f397634f4164071119c5", "Fichier des personnes décédées", "5e7e104ace2080d9162b61d8", "Données hospitalières relatives à l'épidémie de COVID-19", "53699fe4a3a729239d206227", "Service-public.fr - Annuaire de l’administration - Base de données locales", "53699569a3a729239d2046eb", "FINESS Extraction du Fichier des établissements", "549355bbc751df357a04805a", "Communauté universitaire Enseignement Supérieur et recherche - Vie Etudiante  2010- 2014", "58e53811c751df03df38f42d", "Répertoire National des Associations", "53699668a3a729239d2049e8", "Indicateur Avancé Sanitaire IAS® - SYNDROME GRIPPAL", "5cc1b94a634f4165e96436c1", "Demandes de valeurs foncières géolocalisées", "586a824588ee3835ec3f4e61", "Fichier des prénoms - Edition 2016 (voir Fichier des prénoms de 1900 à 2019)", "55f281abc751df532e1f92b1", "Ecole de formation aux professions sociales en Bretagne", "561bcab288ee3833e0628efc", "DONNEES - Fuseaux de mobilité - SAGE Ill Nappe Rhin - 2013", "57e92dbc88ee3804385ff490", "SRCE Limousin - Milieux supports de la sous trame milieux aquatiques", "5b0438be88ee3816f5915238", "Entité surfacique à l'origine du risque du PPRT d'EFR France (ex Delek et BP)", "5acb4db988ee384506ec53c2", "Localisation des points de visio-conférence du Département des Hautes-Alpes", "5899a105c751df0846ae0a66", "Point de rejet collectivité", "615650423f9c95de7fb0681a", "Zones de stationnement pour les campings-cars", "61565042c4f1aae6b1b0681c", "Point d'apport volontaire", "6156504298b9f37249b0681b", "Horodateurs", "612ec4ec17b0bac701ef6a5a", "Fourrière", "615fc30b20717ce904b0681a", "Réussite aux examens de la voie professionnelle", "57fbbb6988ee3817235ff490", "Tracé des lignes du réseau de transport 'Auray-Bus' ( ligne jaune)", "5729a08388ee3818a96242c9", "Enveloppe de travaux miniers en Pays de la Loire", "5ae9b88588ee387a6772378d", "ROUTES : Itinéraires cyclables - 01/01/2017 - département du Bas-Rhin (67)", ...]

11:14:16.450 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=6

11:14:16.978 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=7

11:14:17.372 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=8

11:14:18.109 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=9

11:14:18.474 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=10

11:14:18.477 [debug] QUERY OK db=1.6ms
INSERT INTO "datasets" ("external_id","title") VALUES ($1,$2),($3,$4),($5,$6),($7,$8),($9,$10),($11,$12),($13,$14),($15,$16),($17,$18),($19,$20),($21,$22),($23,$24),($25,$26),($27,$28),($29,$30),($31,$32),($33,$34),($35,$36),($37,$38),($39,$40),($41,$42),($43,$44),($45,$46),($47,$48),($49,$50),($51,$52),($53,$54),($55,$56),($57,$58),($59,$60),($61,$62),($63,$64),($65,$66),($67,$68),($69,$70),($71,$72),($73,$74),($75,$76),($77,$78),($79,$80),($81,$82),($83,$84),($85,$86),($87,$88),($89,$90),($91,$92),($93,$94),($95,$96),($97,$98),($99,$100),($101,$102),($103,$104),($105,$106),($107,$108),($109,$110),($111,$112),($113,$114),($115,$116),($117,$118),($119,$120),($121,$122),($123,$124),($125,$126),($127,$128),($129,$130),($131,$132),($133,$134),($135,$136),($137,$138),($139,$140),($141,$142),($143,$144),($145,$146),($147,$148),($149,$150),($151,$152),($153,$154),($155,$156),($157,$158),($159,$160),($161,$162),($163,$164),($165,$166),($167,$168),($169,$170),($171,$172),($173,$174),($175,$176),($177,$178),($179,$180),($181,$182),($183,$184),($185,$186),($187,$188),($189,$190),($191,$192),($193,$194),($195,$196),($197,$198),($199,$200) ["615fc30d20717ce904b0681b", "Taux d'emploi des diplômes de la voie professionnelle", "615fc30d32f02e8407b0681b", "Attractivité dans la voie professionnelle", "545b55e1c751df52de9b6045", "Base officielle des codes postaux", "53699509a3a729239d2045c4", "Etude sur la consommation de traitements contre la dysfonction érectile", "5890b81488ee38211f9b81a4", "Gender Scan", "58c98b1888ee38770950152b", "Mon Réseau Mobile", "53699233a3a729239d203e69", "Découpage administratif communal français issu d'OpenStreetMap", "56eee78dc751df55fad6e93b", "Zones réglementées du plan de prévention des risques naturels de la commune de BENAGUES", "55f1b14fc751df226d1f92ec", "Document d'urbanisme de Charmont sous Barbuise", "53f0a959a3a72905a3504b1d", "Indemnités des élus de la Communauté d'Agglomération de la Vallée de la Marne (CAVM) (2014)", "55f18f3488ee380c7fa46ed5", "Aquitaine : TRI de Libourne (aléa débordement) - Objets décrivant l’emprise et les caractéristiques utiles du territoire à risque d’inondation, Directive inondation", "55886ebb88ee3856924a22f3", "Points de repère remarquables du réseau des véloroutes et voies vertes dans le département des Vosges", "56eee7e788ee381d7d908574", "Zones réglementées du plan de prévention des risques naturels de la commune de BEZAC", "56eee7c0c751df561cd6e93c", "Périmètres du plan de prévention des risques naturels (PPRN) de la commune de PECH", "55ae44ce88ee381da83ca28b", "Plan de prévention des risques naturels - commune Montpouillan - Lot-et-garonne : Périmètres surfaciques", "558ea3e888ee3856774a22fa", "PPR NOUSTY (64DDTM19970011) - Zone d'aléa du Plan de Prévention du Risque Inondation de Nousty  (64419), département des Pyrénées-Atlantiques.", "558ae04088ee38680b4a2300", "Plan de prévention des risques naturels - commune Fauguerolles - Lot-et-garonne : Périmètres surfaciques", "558ae02588ee3806d04a22f8", "Plan de prévention des risques naturels - Risque INONDATION - Commune Nérac - Lot-et-garonne : Zones réglementées surfaciques", "56eee32c88ee381538908575", "Périmètre du plan de prévention des risques naturels de Taninges - Giffre (Haute-Savoie) – approuvé le 28/06/2004", "5852bf15c751df55b1c0bb81", "PPR LOUVIE-SOUBIRON (64DDTM20120005) - Document PPRN sur LOUVIE-SOUBIRON (64354), département des Pyrénées-Atlantiques.", "5852bee588ee382918c65bb8", "PPR SAINT-PIERRE-D'IRUBE (64DDTM20060003) - Périmètre du Plan de Prévention du Risque Inondation de SAINT-PIERRE-D'IRUBE (64496), département des Pyrénées-Atlantiques.", "5a09782cc751df6035c1ff0d", "Plan de prévention des risques naturels de Chutes de blocs - Commune de Caylus - Département de Tarn-et-Garonne", "5883cea488ee386def9b81f4", "Zones de bruit liées au grandes infrastructures ferroviaires - Carte de type a - indice Ln (niveau sonore sur une période nocturne de 22h à 6h) - Départemant de Tarn-et-Garonne", "5a099825c751df0dba6dd184", "Assiettes surfaciques liées aux servitudes de la catégorie PM1 en Côte-d'Or", "58aec2a0c751df0e9c1cc550", "Périmètre d'exposition au risque du PPRN de la commune de Treclun en Côte-d'Or", ...]

```

Using this method, we can very finely control how much dataset records are stored in the RAM before being inserted. As a result, we can insert an arbitrary big number of records in our database, without blowing up our available memory :heart:.

## Just for fun: make it crash
Now I insert a bug in my streaming function, raising an exception on the 7th page call, with the following lines of code in my streaming function:

```elixir
  page = content |> Map.fetch!("page")

  if page == 7 do
    raise "oooops I do not like this one"
  end
```

Let's look at the logs:

```

18:02:57.644 [debug] QUERY OK db=0.4ms queue=0.1ms idle=1492.1ms
begin []

18:02:58.304 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/

18:02:58.618 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=2

18:02:58.861 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=3

18:02:59.454 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=4

18:02:59.737 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=5

18:02:59.741 [debug] QUERY OK db=1.5ms
INSERT INTO "datasets" ("external_id","title") VALUES ($1,$2),($3,$4),($5,$6),($7,$8),($9,$10),($11,$12),($13,$14),($15,$16),($17,$18),($19,$20),($21,$22),($23,$24),($25,$26),($27,$28),($29,$30),($31,$32),($33,$34),($35,$36),($37,$38),($39,$40),($41,$42),($43,$44),($45,$46),($47,$48),($49,$50),($51,$52),($53,$54),($55,$56),($57,$58),($59,$60),($61,$62),($63,$64),($65,$66),($67,$68),($69,$70),($71,$72),($73,$74),($75,$76),($77,$78),($79,$80),($81,$82),($83,$84),($85,$86),($87,$88),($89,$90),($91,$92),($93,$94),($95,$96),($97,$98),($99,$100),($101,$102),($103,$104),($105,$106),($107,$108),($109,$110),($111,$112),($113,$114),($115,$116),($117,$118),($119,$120),($121,$122),($123,$124),($125,$126),($127,$128),($129,$130),($131,$132),($133,$134),($135,$136),($137,$138),($139,$140),($141,$142),($143,$144),($145,$146),($147,$148),($149,$150),($151,$152),($153,$154),($155,$156),($157,$158),($159,$160),($161,$162),($163,$164),($165,$166),($167,$168),($169,$170),($171,$172),($173,$174),($175,$176),($177,$178),($179,$180),($181,$182),($183,$184),($185,$186),($187,$188),($189,$190),($191,$192),($193,$194),($195,$196),($197,$198),($199,$200) ["5c4ae55a634f4117716d5656", "Demandes de valeurs foncières", "5de8f397634f4164071119c5", "Fichier des personnes décédées", "5e7e104ace2080d9162b61d8", "Données hospitalières relatives à l'épidémie de COVID-19", "53699fe4a3a729239d206227", "Service-public.fr - Annuaire de l’administration - Base de données locales", "53699569a3a729239d2046eb", "FINESS Extraction du Fichier des établissements", "549355bbc751df357a04805a", "Communauté universitaire Enseignement Supérieur et recherche - Vie Etudiante  2010- 2014", "58e53811c751df03df38f42d", "Répertoire National des Associations", "53699668a3a729239d2049e8", "Indicateur Avancé Sanitaire IAS® - SYNDROME GRIPPAL", "5cc1b94a634f4165e96436c1", "Demandes de valeurs foncières géolocalisées", "586a824588ee3835ec3f4e61", "Fichier des prénoms - Edition 2016 (voir Fichier des prénoms de 1900 à 2019)", "55f281abc751df532e1f92b1", "Ecole de formation aux professions sociales en Bretagne", "561bcab288ee3833e0628efc", "DONNEES - Fuseaux de mobilité - SAGE Ill Nappe Rhin - 2013", "57e92dbc88ee3804385ff490", "SRCE Limousin - Milieux supports de la sous trame milieux aquatiques", "5b0438be88ee3816f5915238", "Entité surfacique à l'origine du risque du PPRT d'EFR France (ex Delek et BP)", "5acb4db988ee384506ec53c2", "Localisation des points de visio-conférence du Département des Hautes-Alpes", "5899a105c751df0846ae0a66", "Point de rejet collectivité", "615650423f9c95de7fb0681a", "Zones de stationnement pour les campings-cars", "61565042c4f1aae6b1b0681c", "Point d'apport volontaire", "6156504298b9f37249b0681b", "Horodateurs", "612ec4ec17b0bac701ef6a5a", "Fourrière", "615fc30b20717ce904b0681a", "Réussite aux examens de la voie professionnelle", "57fbbb6988ee3817235ff490", "Tracé des lignes du réseau de transport 'Auray-Bus' ( ligne jaune)", "5729a08388ee3818a96242c9", "Enveloppe de travaux miniers en Pays de la Loire", "5ae9b88588ee387a6772378d", "ROUTES : Itinéraires cyclables - 01/01/2017 - département du Bas-Rhin (67)", "5629ee3c88ee3828e912613c", "Compte Administratif 2013 dépenses du BUDGET TRANSPORTS URBAIN", ...]

18:03:00.397 [info]  fetch 20 items from the API coming from https://www.data.gouv.fr/api/1/datasets/?page=6

** (RuntimeError) oooops I do not like this one
    (stdlib 3.15.1) erl_eval.erl:685: :erl_eval.do_apply/6
    (stdlib 3.15.1) erl_eval.erl:123: :erl_eval.exprs/5
    (elixir 1.12.2) lib/stream.ex:1531: Stream.do_resource/5
    (elixir 1.12.2) lib/stream.ex:1719: Enumerable.Stream.do_each/4
    (elixir 1.12.2) lib/stream.ex:649: Stream.run/1
    (ecto_sql 3.7.0) lib/ecto/adapters/sql.ex:1013: anonymous fn/3 in Ecto.Adapters.SQL.checkout_or_transaction/4
    (db_connection 2.4.0) lib/db_connection.ex:1512: DBConnection.run_transaction/4


18:03:01.342 [debug] QUERY OK db=0.4ms
rollback []

```

We have 5 API calls, 1 database insertion, followed by a crash !

The final line of logs confirms that everything has been cleaned nicely with a rollback. Nothing has been persisted in the database :thumbsup:.

## Try it yourself 🎮
With the magic of the `mix_install` macro, all this code can be run in a single file. It will create the database, create the table, the Ecto schema, stream some data from the data.gouv.fr API and stream it into the database :rocket:. The only prerequesite is to have a **running postgres service** and that you fill your **credentials** to access it (line 13).

The best way to play with the code is probably to copy paste it in a [Livebook](https://github.com/livebook-dev/livebook/) and launch it.

Have fun!

<script src="https://gist.github.com/fchabouis/928d9d64230dd7e952e90ba3e8c8628d.js"></script>
