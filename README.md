# Pourquoi ?

À l'occasion de la refonte du collecteur Instagram, suite à la mise à jour de l'API, l'approche actuelle du processus de capture, d'agrégation et d'indexation des métriques utilisées comme indicateurs de performance est remise en question. Objectif : permettre une plus grande souplesse dans l'utilisation des métriques de façon à pouvoir proposer en front des modalités étendues de visualisation de ces métriques.

3 métriques sont utilisées comme indicateurs de performance :
* le nombre de followers
* le nombre de publications
* le nombre moyen d'engagements

__Approche actuelle__

Pour une métrique donnée, chaque jour :
* Capture du cumul
* Agrégation sur la semaine, le mois, les trois derniers mois, les six derniers mois, la dernière année
* Indexation de ces agrégations

__Approche proposée__

Pour chaque métrique, chaque jour :
* Capture du cumul ou de la valeur pour la journée considérée (selon les métriques).
* Dans le cas où le cumul est capturé, calcul du différentiel par rapport au cumul de la veille, de façon a obtenir la valeur pour la journée considérée.
* Indexation de la valeur pour la journée considérée
* Agrégations effectuées à la demande par Elasticsearch

Ce projet valide l'approche proposée en :
* Reproduisant les capacités d'agrégations actuelles
* Proposant une capacité d'agrégation étendue, du point de vue fonctionnel


# Comment ?

Deux scripts, `create_users` et `create_posts`, permettent, respectivement, de créer un index d'utilisateurs et un index de posts sur une instance Elasticsearch identifiée par un host et un port définis dans `conf/es.cfg`. Ces deux index sont réduits au strict minimum pour les besoins de l'expérimentation :

* users (les champs num_followers et num_posts comportent le nombre d'événements survenus le jour indiqué par le champ date) :
  * id
  * date (yyyy-MM-dd)
  * num_followers
  * num_posts

* posts (les chammps num_likes et num_comments comportent le nombre d'événements survenus le jour indiqué par le champ date) :
  * id
  * user_id
  * date (yyyy-MM-dd)
  * num_likes
  * num_comments  

Deux autres scripts, `populate_users`et `populate_posts`, permettent, respectivement, d'alimenter l'index des utilisateurs et celui des posts, suivant un paramétrage défini dans `conf/populate_users.cfg` (nombre de jours dans l'index ; nombre d'utilisateurs ; nombre maximum de posts et de followers par jour) et `conf/populate_posts.cfg` (nombre de jours dans l'index ; nombre maximum de posts par user par jour ; nombre maximum de likes et de commentaires par post). 

### Cumul du nombre de followers et de posts
Ce cumul pour un user donné est obtenu par deux requêtes :
  * celle contenue dans le script `cumulative_followers_posts` pour le calcul du cumul  du nombre de followers et de posts sur la période antérieure à la période désirée ; il prend en paramètres un user_id et une date max qui doit être égale à la date min du script suivant.
  * celle contenue dans le script `cumulative_followers_posts_timeframe` pour le calcul du nombre d followers et de posts sur une période de temps donnée ; il prend en paramètres un user_id, un intervalle de temps et une période de segmentation temporelle. 

Par exemple, pour obtenir le cumul du nombre de followers et de posts du compte dont le user_id est 1, par jour, sur le dernier mois :
```
./cumulative_followers_posts 1 now-1M | jsonlint -p
```
qui, après pretty printing, donne :
```
"aggregations": {
  "followers": {
    "value": 9608
  },
  "posts": {
    "value": 956
  }
} 
```
et
```
./cumulative_followers_posts_timeframe 1 now-1M now day
```
qui, après pretty printing, donne (extrait) :
```
"buckets": [
  {
    "key_as_string": "2018-03-31T00:00:00.000Z",
    "key": 1522454400000,
    "doc_count": 1,
    "followers": {
      "value": 47
    },
    "posts": {
      "value": 4
    },
    "cumulative_followers": {
      "value": 47
    },
    "cumulative_posts": {
      "value": 4
    }
  },
  {
    "key_as_string": "2018-04-01T00:00:00.000Z",
    "key": 1522540800000,
    "doc_count": 1,
    "followers": {
      "value": 17
    },
    "posts": {
      "value": 5
    },
    "cumulative_followers": {
      "value": 64
    },
    "cumulative_posts": {
      "value": 9
    }
  },
  {
    "key_as_string": "2018-04-02T00:00:00.000Z",
    "key": 1522627200000,
    "doc_count": 1,
    "followers": {
      "value": 39
    },
    "posts": {
      "value": 3
    },
    "cumulative_followers": {
      "value": 103
    },
    "cumulative_posts": {
      "value": 12
    }
  },

  ...
```

Dans ce scénario, il est nécessaire d'ajouter le cumul obtenu avec la première requête à ceux obtenus avec la seconde. Étudier la possibilité de tout calculer en une seule requête.

### Nombre moyen d'engagements sur une fenêtre glissante
Il est donné par la requête contenue dans le script `avg_engagement_sliding`, lequel prend en paramètres un user_id et un intervalle de temps. Par exemple, pour obtenir le nombre moyen d'engagements sur les posts du compte dont le user_id est 1 sur le dernier mois :
```
./avg_engagement_sliding 1 now-1M now
```
qui, après pretty printing, donne :
```
"hits": {
  "total": 39,
  "max_score": 0,
  "hits": []
},
"aggregations": {
  "comments": {
    "value": 141
  },
  "avg_engagement": {
    "value": 44.717948717948715
  },
  "likes": {
    "value": 1603
  }
}
```

### Nombre moyen d'engagements segmenté sur une période temporelle
Il est donné par la requête contenue dans le script `avg_engagement_timeframe` qui prend en paramètres un user_id, un intervalle de temps et une période de segmentation temporelle. Par exemple, le nombre moyen d'engagements sur les posts du compte dont le user_id est 1, par jour, sur le mois dernier, est obtenue par :
```
./avg_engagement_timeframe 1 now-1M now day
```
qui, après pretty printing, donne (extrait) :
```
"buckets": [
  {
    "key_as_string": "2018-03-31",
    "key": 1522454400000,
    "doc_count": 4,
    "comments": {
      "value": 15
    },
    "avg_engagement": {
      "value": 37.75
    },
    "likes": {
      "value": 136
    }
  },
  {
    "key_as_string": "2018-04-01",
    "key": 1522540800000,
    "doc_count": 1,
    "comments": {
      "value": 3
    },
    "avg_engagement": {
      "value": 15
    },
    "likes": {
      "value": 12
    }
  },
  {
    "key_as_string": "2018-04-02",
    "key": 1522627200000,
    "doc_count": 0,
    "comments": {
      "value": 0
    },
    "avg_engagement": {
      "value": null
    },
    "likes": {
      "value": 0
    }
  },

  ...
```

