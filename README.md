# REST in peace

# Disclaimer

Ce document est une sorte de recette pour produire une API "REST" selon les bonnes pratiques en cours.

Il n'est pas exhaustif, ne recensent pas toutes les pratiques.

Des choix sont fait pour rester pragmatique quand il n'y a pas vraiment de standard.

D'ailleurs on parle içi d'API REST au sens "marketing" du terme puisqu'il ne vise pas à atteindre le level3 du 
[modèle de maturité de Richardson](http://blog.xebia.fr/2010/06/25/rest-richardson-maturity-model/) mais pluôt d'une API HTTP++ (définit très justement par [William Durant](https://youtu.be/u_jDzcXCimM?list=PL9zDdgiGjkIc_1wnKTdU68dmVZ77ayPwW)).

# Base

Tous les appels doivent être fait via SSL.

Tout est encodé en UTF-8: la réponse et la requète (header, body, querystring).

L'api doit être versionnée via l'URL: https://api.domain.com/v2

> Pas plus de 2 versions en même temps sinon c'est ingérable

# Type de données

Type | Description
------------ | -------------
String | Encodée en UTF-8
Integer | Entier signé en 32-bit ou 64-bit
Float | Nombre flottant signé en 32-bit ou 64-bit
Boolean | true ou false
Date | Toujours UTC et au format [ISO8601](https://en.wikipedia.org/wiki/ISO_8601): `2015-07-20T20:10:55Z`

# Ressource

Le but de L'API est d'exposer des ressources qui pourront être consommées par un client (frontend et/ou backend selon les cas).
Elles ne reflètent pas **forcément** votre modèle de donnée.

Quelques règles pour définir une resource
* on utilise toujours le pluriel
* nommé avec des - ou des _
* les ids sont des UUID

Pour interragir avec les ressources, on s'appuie sur HTTP:

URL | Action
------------ | -------------
GET /items | Liste d'item
GET /items/1782 | Item 1782
POST /items | Creation d'un nouvel item
PUT /items | Mise à jour de plusieurs items
PUT /items/1782 | Mise à jour de l'item 1782
PATCH /items/1782 | Mise à jour partielle de l'item 1782 (1 ou plusieurs propriétés)
DELETE /items/1782 | Suppression de l'item 1782

On a parfois des relations entre nos ressources. On utilisera la même mécanique via HTTP:

URL | Action
------------ | -------------
GET /items/1782/comments | Liste de commentaire de l'item 1782
GET /items/1782/comments/56 | Commentaire 56 de l'item #1782
POST /items/1782/comments | Création d'un commentaire pour l'item 1782
PUT /items/1782/comments/56 | Mise à jour du commentaire 56 pour l'item 1782
PATCH /items/1782/comments/56 | Mise à jour partielle du commentaire #56 pour l'item #1782
DELETE /items/1782/comments/56 | Suppression du commentaire 56 pour l'item 1782

Il nous faut parfois effectuer des actions sur nos ressources, la pratique veut qu'on utilisera systématiquement POST:

URL | Action
------------ | -------------
POST /items/1782/translate | Traduit l'item 1782
POST /items/1782/enable | Active l'item 1782
POST /items/1782/comments/56/star | Met en favori le commentaire 56 de l'item 1782

# Authentification/Authorization

## HTTP basic authentification

```HTTP
Authorization: Basic cGhwOm1lZXR1cA==
```

* username:password encodé en base64
* toujours utilisé avec SSL
* maîtrise le client et le serveur (le user:password est côté client)

> Rapide à mettre en place, mais pas très secure, on doit avoir les credentials sur le client

## Query Authentification

```HTTP
GET /items?param=X&timestamp=1261496500&apiKey=MeetupPHPBbx&signature=3051d3c053e291b723f16944893df966ccea2a34ee162d5ccda1276a47e796e7
```

* le client signe la requête ```signature=SHA256( items?param=X&timestamp=1261496500&apiKey=MeetupPHPBbx&param=X )```
* le serveur valide la signature et vérifie si le timestamp < X secondes (X étant définit sur le serveur)

> Il faut que le client et le serveur soit configuré de la même façon niveau date pour avoir des timestamps comparables

## OAuth2

Voir la [doc](http://oauth.net/2) 

> Une grande majorité des géants du web l'utilise

# Requête

On ne supporte que le format **JSON** pour la réponse.

La requète doit donc avoir un header `Accept: application/json`

Retourner [`406 Not acceptable`](http://httpstatus.es/406) si on demande autre chose.

> [Plus personne n'utilise XML](http://www.google.com/trends/explore?q=xml+api#q=xml%20api%2C%20json%20api&cmpt=q) sauf dans un contexte grand compte / DSI

> Dans ce cas on gère `Accept: application/json; application/xml` mais on garde JSON en choix n°1

Ajouter pour chaque requète un header `X-Request-UUID: 454684315618613`, ceci aidera le client dans sont logging, debuging... en identifiant de manière unique chaque requète. 

On accèpte du JSON pour le body des requètes dans le cas d'un POST, PUT ou PATCH. Ca nous permet d'avoir la même sérialisation entre le body de la requète et le body de la réponse.
On pourra facilement passer des structures complètes ou partielles de ressources et avoir le typage JSON: Array, String, Number, Object, Boolean, Null.

La requête doit donc comporter un header Content-Type: `Content-Type: application/json;charset=utf-8`.

Retourner [`415 Unsupported media type`](http://httpstatus.es/415) si Content-type n'est pas supporté par le serveur.

> On peut supporter form-encoded data en parallèle, à voir en fonction des clients qui consommeront l'API. Mais ça obligera côter serveur à typer les valeurs manuelement et on n'aura pas de structure de ressource out of box.

Il ne faudra pas oublier d'indiquer qu'on veut la réponse gzippée via `Accept-Encoding: gzip`.

> La pluspart des clients supportent out of box gzip il ne faut pas s'en priver!

```bash
$ curl -X POST https://api.domain.com/v2/items \
    -H "Content-Type: application/json;charset=utf-8" \
    -H "Accept: application/json" \
    -H "Accept-Encoding: gzip" \
    -H "X-Request-UUID: 454684315618613" \
    -d '{"name": "Jo", "age": 55, "isGeek": true}'

{
  "id": 7856,
  "name": "Jo",
  "age": 18,
  "isGeek": true
  ...
}
```

# Réponse

On retourne toujours un JSON pretty print. C'est plus human-friendly et ce n'est pas trop un problème avec la compression gzip.

On n'enveloppe pas les réponses avec une propriété data ou item içi, ça n'a pas d'intérêt:

```json
{
  "id": 7856,
  "name": "Jo",
  "age": 18,
  "isGeek": true
}
```

Si on a des ressources imbriquées, on retourne:

```json
{
  "id": 7856,
  "name": "Jo",
  "age": 18,
  "isGeek": true,
  "country": {
    "id": 569
  }
}
```

Plutôt que:

```json
{
  "id": 7856,
  "name": "Jo",
  "age": 18,
  "isGeek": true,
  "country_id": 569
}
```

Ce qui nous permettra éventuelement de retourner la resource imbriquée inline et ainsi de garder la même structure de réponse et d'éviter des requètes supplémentaires. On pourra utiliser un header `X-Resource-Nested: true` pour indiquer au serveur que l'on veut aussi les resources imbriquées.

```json
{
  "id": 7856,
  "name": "Jo",
  "age": 18,
  "isGeek": true,
  "country": {
    "id": 569,
    "name": "France",
    "code_iso": "FR"
  }
}
```

Il faudra aussi retourner le bon code HTTP:

HTTP status code | Information
------------ | -------------
[`200 Ok`](http://httpstatus.es/200) | GET, PUT, PATCH et DELETE ainsi que pour POST lors d'une "action"
[`201 Created`](http://httpstatus.es/201) | POST
[`202 Accepted`](http://httpstatus.es/204) | La requête est ok, mais on la traitera plus tard
[`204 No Content`](http://httpstatus.es/204) | DELETE sans body
[`206 Partial content`](http://httpstatus.es/206) | Si la réponse ne renvoie pas l'ensemble de la resource (une liste par ex) 

Lors d'un [`200 Ok`](http://httpstatus.es/200) **on doit retourner la ressource**.

Lors d'un [`201 Ok`](http://httpstatus.es/201):
* on doit retourner la ressource
* on doit indiquer l'URI de la nouvelle resource dans le header `Location: https://api.domain.com/v2/items/1783`

# Error

Lorsqu'une erreur survient, il faut que le client puisse comprendre ce qui se passe et éventuelement agir.
Il faut s'appuyer sur les status HTTP 40x et 50x qui répondent à tous les cas, même si dans la pratique une dizaine suffit. Il faut aussi retourner une réponse avec une structure qui sera toujours la même quelque soit l'erreur:

```json
{
  "code": "error_code",
  "description": "More details about the error here",
  "url": "https://doc.domain.com/error/error_code"
}
```

Exemples:

[`400 Bad Request`](http://httpstatus.es/400): on a mal formaté la requète, par exemple un body JSON non parsable.

```json
{
  "code": "invalid_request",
  "message": "Can't parse the request body, JSON not valid.",
  "url": "https://doc.domain.com/error/invalid_request"
}
```

[`422 Unprocessable Entity`](http://httpstatus.es/400): la requète est ok, mais les données envoyées ne sont pas valide.

```json
{
  "code": "invalid_item",
  "message": "Name is required, isGeek must be a boolean.",
  "url": "https://doc.domain.com/error/invalid_item"
}
```

Dans le cas de la validation, on peut avoir besoin de gérer un retour d'erreur plus fin et donc renvoyer un tableau d'erreur.

```json
[
    {
        "code": "invalid_item_name",
        "message": "Name is required",
        "url": "https://doc.domain.com/error/invalid_item_name"
    },
    {
        "code": "invalid_item_geek",
        "message": "isGeek must be a boolean.",
        "url": "https://doc.domain.com/error/invalid_item_geek"
    },
]
```

Les principaux statut HTTP pour gérer les erreurs:

HTTP status code | Information
------------ | -------------
[`400 Bad Request`](http://httpstatus.es/400) | Requête mal formée (body non parsable etc...)
[`401 Unauthorized`](http://httpstatus.es/401) | Authentification invalide
[`403 Forbidden`](http://httpstatus.es/403) | Authentication ok, mais on a pas les droits
[`404 Not Found`](http://httpstatus.es/404) | Resource pas trouvée (inexistante ou suite à un `DELETE`)
[`405 Method Not Allowed`](http://httpstatus.es/405) | Méthode HTTP non autorisée (utilisation d'un `POST` alors qu'on attend un `DELETE`)
[`406 Not acceptable`](http://httpstatus.es/405) | Format de retour non disponible (la requête demande du XML alors qu'on ne gère que du JSON)
[`415 Unsupported Media Type`](http://httpstatus.es/415) | Content type pas supporté (on envoie du XML alors qu'on ne suppporte que JSON)
[`422 Unprocessable Entity`](http://httpstatus.es/422) | Tout ce qui touche à la validation
[`429 Too Many Requests`](http://httpstatus.es/429) | Trop de requêtes (on a dépassé le rate limit)
[`500 Internal Server Error`](http://httpstatus.es/500) | Certainement une coquille dans le code ^^
[`503 Service Unvailable`](http://httpstatus.es/429) | Lors d'une maintenance ou si l'on veut couper l'API

# Cache via timestamp

On envoie le header ```If-Modified-Since``` pour validé que le ressource n'a pas été modifiée. Dans ce cas on retourne un [`304 Not Modified`](http://httpstatus.es/304).
Sinon on retourne la ressource avec le header ```Last-Modified```.

> On pourrait utiliser Etag, mais ça nécéssite de maintenir un hash de la ressource alors qu'on aura toujours un timestamp de modification.

# Pagination

## Requête

On utilise la querystring:

```bash
$ curl -X POST https://api.domain.com/v2/item?page=2&per_page=100 \
    -H "Content-Type: application/json"
```

> On pourrait utiliser le header `Range` mais par affordance et pour le côté pratique il vaut mieux utiliser la querystring.

## Réponse

Le serveur doit retourner [`206 Partial content`](http://httpstatus.es/206) si on a pas toutes les ressrouces, si elles sont toutes retournée [`201 OK`](http://httpstatus.es/200)

Utiliser le header `Link` pour transmettre la pagination:
```HTTP
Link: <https://api.domain.com/v2/item?page=3&per_page=100>; rel="next", <https://api.domain.com/v2/item?page=1&per_page=100>; rel="prev"
```
> Le client n'aura pas à construire la pagination.

> On peut aussi ajouter la première et la dernière page `rel=first` et `rel=last`.

Ajouter un header custom pour indiquer le nombre totale de ressource disponible:
```HTTP
X-Total-Count: 456
X-Page-Max-Range: 100
```
Le serveur doit retourner [`400 Bad request`](http://httpstatus.es/400) si on dépasse les capacités de l'API.

# Filtering, sort & search

On utilise la querystring:

```bash
$ curl -X POST https://api.domain.com/v2/item?q=toto&isGeek=false&age=18,19&sort=name,id \
    -H "Content-Type: application/json"
```

> q pour une recherche fulltext. On peut aussi se servir des filtres pour faire une recherche sur un champs particulier, exemple name=Marado*

# Rate limiting

Pour garder un niveau de qualité et éviter les abus, il faut mettre en place un système de limitation des appels vers l'API. Classiquement on définit une période (1h) et un nombre de requête maximum pour cette période.

Header | Description
------------ | -------------
X-Rate-Limit-Limit | Le nomber de requête possible pendant la mériode 
X-Rate-Limit-Remaining | Le nombre de requête qu'il reste pour la période
X-Rate-Limit-Reset | Le nombre de seconde qu'il reste avant de remettre les compteurs à 0

Si on dépasse la limite [`429 Too many requests`](http://httpstatus.es/429)

# CORS

Permet à une API et un client type Web App d'être sur des domaines différents sans que ça pose problème pour XMLHttpRequest.
Le client enverra une requête `OPTION` (preflighted request) avant chaque requète pour vérifier ce qui est autorisé.

En retour le serveur indiquera ce qui est permis, exemple:

```HTTP
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: X-Rate-Limit-Limit, X-Rate-Limit-Remaining, X-Rate-Limit-Reset
```

> IE<10 ne supporte pas correctement CORS, dans ce cas il faudra se trourner vers JSONP

# Documentation

C'est un point clé pour que l'API soit populaire. Il faut qu'elle soit maintenue et **facile à maintenir**!
Le mieux c'est que la documentation parte du code.

http://apidocjs.com permet, via des annotations dans le code, de générer la documentation complète de votre API.

Il faut mettre des exemples cURL dès que possibles.