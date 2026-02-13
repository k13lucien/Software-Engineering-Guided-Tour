# JWT in Action

## Introduction

Ce projet est une **démonstration pratique du fonctionnement interne de JWT (JSON Web Token)**

Il montre comment :
1. Générer un JWT
2. Lire et vérifier un JWT (vérification + validation).  
3. Comprendre concrètement les concepts de **signature**, **claims**, et **expiration**.

> Ce projet ne met pas en œuvre un système d’authentification complet — il sert uniquement à **observer comment un JWT est créé, signé et vérifié**.
> Référer vous à la version de jjwt pour adapter les implémentation.

---

## 1. JWT with Java (jjwt librarie)

### Installation des dépendances

Dans ton fichier **`pom.xml`**, ajoute les dépendances suivantes :

```xml
<dependencies>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

### Variables de configuration

```java
@Value(${jwt.secret-key})
private String secretKey;
@Value(${jwt.token-validity})
private long tokenValidity;
```

- Générer la clée ```secretKey``` et la placée dans ```application.properties```
```bash
openssl rand -base64 64
```

- Convertir la chaine ```secretKey``` encodée en BASE64 en un objet ```Key``` utilisatble.
```java
public Key getKey() {
  byte[] keyBytes = Decoders.BASE64.decode(secretKey);
  return Keys.hmacShaKeyFor(keyBytes);
}
```

### Généner un token
```java
public String generateToken(String username) {
  return Jwts.builder()
              .subject(username)
              .issuedAt(new Date(System.currentTimeMillis())) // Date de création du token
              .expiration(new Date(System.currentTimeMillis() + tokenValidity)) // Période de validité du token
              .signWith(getKey()) // Signée en utilisant la clée
              .compact();
}
```

### Décodage et vérification du token
```java
public Claims extractAllClaims(String token) {
  return Jwts
          .parser()
          .verifyWith((SecretKey) getKey())
          .build()
          .parseSignedClaims(token)
          .getPayload();
}

public String extractUsername(String token) {
    return extractAllClaims(token).getSubject();
}

private Date extractExpiration(String token) {
    return extractAllClaims(token).getExpiration();
}

public boolean validateToken(String token, UserDetails userDetails) {
    final String username = extractUsername(token);
    return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
}

public boolean isTokenExpired(String token) {
    return extractAllClaims(token).getExpiration().before(new Date());
}
```

## Sources

- [Java JWT Documentation](https://github.com/jwtk/jjwt)
- [Build and Parse JWTs in Java](https://www.youtube.com/watch?v=O-sTJbeUagE)