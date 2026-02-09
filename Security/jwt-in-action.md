# JWT in Action

## Introduction

Ce projet est une **démonstration pratique du fonctionnement interne de JWT (JSON Web Token)**

Il montre comment :
1. Générer un JWT
2. Lire et vérifier un JWT (vérification + validation).  
3. Comprendre concrètement les concepts de **signature**, **claims**, et **expiration**.

> Ce projet ne met pas en œuvre un système d’authentification complet — il sert uniquement à **observer comment un JWT est créé, signé et vérifié**.

---

## 1. JWT with Java (jjwt librarie)

### Installation des dépendances

Dans ton fichier **`pom.xml`**, ajoute les dépendances suivantes :

```xml
<dependencies>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

### Variables de configuration

```java
private String secretKey;
private long jwtExpiration;
```

### Généner un token
```java
public String generateToken(String email) {
    return Jwts.builder()
            .setSubject("Authentication")
            .claim("email", email)
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration)) // Définit la validité du token
            .signWith(secretKey)
            .compact(); // Assemble Header.Payload.Signature
}
```

### Décodage et vérification du token
```java
public Claims decodeToken(String token) {
    return Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .build()
            .parseClaimsJws(token) // Vérifie la signature et extrait les claims
            .getBody();
}
```


## Sources

[Java JWT Documentation](https://github.com/jwtk/jjwt)
[Build and Parse JWTs in Java](https://www.youtube.com/watch?v=O-sTJbeUagE)