# Deep Dive into JWT (JSON Web Token)

## Introduction

**JWT (JSON Web Token)** est un standard ouvert (**RFC 7519**) utilisé pour **transmettre des informations sécurisées** entre deux parties (par exemple un client et un serveur).  
Ces informations sont :
- **encodées** en JSON,  
- **signées** numériquement,  
- et **autoportantes**, c’est-à-dire qu’elles contiennent tout ce qu’il faut pour être vérifiées sans base de données.

JWT est largement utilisé dans les systèmes distribués, les microservices et les API modernes.

---

## Structure du JWT

Un token JWT est composé de **trois parties** séparées par des points :
```
HEADER.PAYLOAD.SIGNATURE
```

### 1. Header

Le **header** indique le type de jeton et l’algorithme de signature utilisé (HMAC SHA256 ou RSA) :

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Il est ensuite encodé en Base64URL.

### 2. Payload

Le payload contient les données (claims) que l’on veut transmettre, par exemple l’identifiant de l’utilisateur, son rôle, ou la date d’expiration :

```json
{
  "id": 1,
  "email": "exemple@mail.com",
  "role": "admin",
  "exp": 1739114400
}
```

Il existe **trois types de claims** :

**Registered claims**
- Claims **pré-définis** par le standard RFC 7519.  
- **Non obligatoires**, mais recommandés pour l’interopérabilité.  
- Exemples :
  - `iss` → issuer (émetteur du token)  
  - `sub` → subject (le sujet, souvent l’ID de l’utilisateur)  
  - `aud` → audience (destinataire prévu du token)  
  - `exp` → expiration time (date d’expiration du token)  
  - `nbf` → not before (le token n’est valide qu’après cette date)

**Public claims**
- Claims **définis librement** par l’utilisateur ou l’application.  
- Pour éviter les collisions de noms, ils doivent :
  - soit être enregistrés dans le **IANA JSON Web Token Registry**,  
  - soit être définis comme une **URI unique**.

**Private claims**
- Claims personnalisés, utilisés entre deux parties qui s’entendent sur leur signification.
- Ni enregistrés ni publics, ils servent à partager des infos spécifiques à une application.

*Remarque* : Le payload est encodé mais pas chiffré → toute personne ayant le token peut le décoder et lire son contenu.
Ne jamais y inclure des informations sensibles (mot de passe, numéro de carte, données médicales, etc.).

### 3. Signature

La signature garantit l’intégrité du token (qu’il n’a pas été modifié).
Elle est calculée ainsi :

```makefile
signature = HMACSHA256(
  base64urlEncode(header) + "." + base64urlEncode(payload),
  secret_key
)
```

Cette signature dépend :
- du contenu du token,
- et de la clé secrète du serveur.

Toute modification du token rendra la signature invalide.

---

## Vérification et Validation

### Vérification (Integrity Check)

Le serveur recalcule la signature avec sa clé secrète et la compare à celle reçue dans le token.
- Si elles correspondent → le token n’a pas été altéré.
- Si elles diffèrent → le token est invalide ou falsifié.

Cela garantit **l’intégrité** du message.

### Validation (Claim Check)

Après la vérification, le serveur vérifie la validité logique du token :

- Le champ exp (expiration) n’est pas dépassé.
- Les champs (iss, aud, sub, etc.) correspondent aux valeurs attendues.
- L’utilisateur ou le service associé existe encore, si nécessaire.

Cela garantit la validité temporelle et contextuelle du token.

---

## Chiffrement : le rôle de JWE

JWT assure l’intégrité (il garantit que le contenu n’a pas été modifié).
Mais il n’assure pas la confidentialité : le contenu reste lisible.

Si tu veux que personne ne puisse voir ce qu’il y a dans le token,
il faut utiliser **JWE (JSON Web Encryption)**, qui est la version chiffrée du JWT.

## Références et sources

- RFC 7519 – [JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- RFC 7516 – [JSON Web Encryption (JWE)](https://datatracker.ietf.org/doc/html/rfc7516)
- Auth0 Blog – [JWT Introduction](https://auth0.com/learn/json-web-tokens)