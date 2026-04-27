# e-commerce-sb-k8s-kafka

# TD Pratique Avancé : Plateforme E-commerce Cloud Native avec Spring Boot, Kafka et Kubernetes

## Objectif pédagogique

Dans ce TD, nous allons construire une **plateforme e-commerce distribuée**, inspirée d’une architecture réelle d’entreprise, en utilisant les principaux composants d’une architecture **cloud native** moderne.

Objectifs techniques : 

* les microservices avec Spring Boot ;
* la communication synchrone avec OpenFeign ;
* la communication asynchrone avec Kafka ;
* la résilience avec Resilience4j ;
* l’API Gateway ;
* le Service Discovery ;
* la centralisation de configuration ;
* la conteneurisation Docker ;
* le déploiement Kubernetes ;
* l’observabilité avec Prometheus et Grafana.

Ce TD reprend une architecture proche de celles utilisées dans les plateformes e-commerce.

---

# 1. Contexte métier

Nous allons construire une mini plateforme e-commerce avec les services suivants :

1. **product-service** : gestion des produits
2. **inventory-service** : gestion du stock
3. **order-service** : gestion des commandes
4. **payment-service** : gestion des paiements
5. **notification-service** : notifications
6. **api-gateway** : point d’entrée unique
7. **discovery-service** : enregistrement des services
8. **config-service** : centralisation des configurations

Architecture simplifiée :

```text
Client
  |
API Gateway
  |
  +--> product-service
  +--> order-service --> inventory-service
  |                  --> payment-service
  |
  +--> Kafka --> notification-service
```

---

# 2. Architecture technique

Nous allons introduire plusieurs concepts majeurs :

## Communication synchrone

Le `order-service` appelle :

* `inventory-service`
* `payment-service`

avec **OpenFeign**.

## Communication asynchrone

Après paiement, un événement est envoyé sur **Kafka** :

```text
ORDER_CONFIRMED
```

Le `notification-service` consomme cet événement.

## Résilience

Les appels vers `payment-service` seront protégés avec :

* retry
* circuit breaker
* fallback

via **Resilience4j**.

## Déploiement

Tous les services seront déployés sur **Kubernetes**.

---

# 3. Mise en place du discovery-service (Eureka)

Ce service permet aux microservices de s’enregistrer automatiquement.

## Dépendance Maven

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

## Classe principale

```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(DiscoveryServiceApplication.class, args);
    }
}
```

## Configuration

```yaml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

---

# 4. Création du product-service

Le service produit expose le catalogue.

## Entité Product

```java
@Entity
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private Double price;
}
```

## Controller

```java
@RestController
@RequestMapping("/products")
public class ProductController {

    private final ProductRepository repository;

    public ProductController(ProductRepository repository) {
        this.repository = repository;
    }

    @GetMapping
    public List<Product> findAll() {
        return repository.findAll();
    }
}
```

---

# 5. Création du inventory-service

Ce service vérifie la disponibilité du stock.

```java
@RestController
@RequestMapping("/inventory")
public class InventoryController {

    @GetMapping("/{productId}")
    public Boolean checkStock(@PathVariable Long productId) {
        return true;
    }
}
```

Dans un vrai projet, ce service consulterait une base de données de stock.

---

# 6. Création du payment-service

Ce service simule un paiement.

```java
@RestController
@RequestMapping("/payments")
public class PaymentController {

    @PostMapping
    public String pay() {
        return "PAYMENT_OK";
    }
}
```

---

# 7. Communication inter-services avec OpenFeign

Le `order-service` doit appeler les autres services.

## Dépendance

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

## Client Feign vers inventory-service

```java
@FeignClient(name = "inventory-service")
public interface InventoryClient {

    @GetMapping("/inventory/{productId}")
    Boolean checkStock(@PathVariable Long productId);
}
```

---

# 8. Résilience avec Resilience4j

Si `payment-service` tombe, le système doit rester stable.

## Dépendance

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

## Utilisation

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public String processPayment() {
    return paymentClient.pay();
}

public String paymentFallback(Exception ex) {
    return "PAYMENT_PENDING";
}
```

### Explication

Si le service paiement échoue :

* on évite l’erreur globale ;
* on bascule sur un fallback.

C’est un concept essentiel en production.

---

# 9. Communication asynchrone avec Kafka

Après validation de commande, le `order-service` publie un événement.

## Dépendance Kafka

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

## Producteur Kafka

```java
@Service
public class OrderProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public void sendOrderConfirmed() {
        kafkaTemplate.send("orders", "ORDER_CONFIRMED");
    }
}
```

---

# 10. Consommateur Kafka

Le `notification-service` écoute les événements.

```java
@KafkaListener(topics = "orders", groupId = "notifications")
public void listen(String message) {
    System.out.println("Notification envoyée : " + message);
}
```

### Explication

Dès qu’une commande est confirmée :

* le message est reçu ;
* une notification est déclenchée.

---

# 11. API Gateway

Toutes les requêtes passent par le gateway.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/products/**
```

Le `lb://` indique que la résolution se fait via Eureka.

---

# 12. Dockerisation

Chaque service possède :

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build :

```bash
docker build -t product-service:1.0 .
```

---

# 13. Déploiement Kubernetes

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
        - name: product-service
          image: product-service:1.0
          ports:
            - containerPort: 8080
```

---

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product-service
  ports:
    - port: 80
      targetPort: 8080
```

---

# 14. ConfigMap Kubernetes

Externaliser les paramètres :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-config
data:
  APP_ENV: production
```

Injecter dans le pod :

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: product-config
        key: APP_ENV
```

---

# 15. Autoscaling avec HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-hpa
spec:
  minReplicas: 2
  maxReplicas: 5
```

Le nombre de pods augmentera selon la charge.

---

# 16. Monitoring avec Prometheus

Exposer les métriques :

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Configurer :

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
```

Prometheus viendra scraper les métriques.

---

# 17. Visualisation avec Grafana

Grafana permettra de visualiser :

* le nombre de requêtes ;
* la latence ;
* les erreurs ;
* la consommation CPU.

C’est indispensable dans une architecture cloud.

---

# 18. Scénario fonctionnel complet

1. L’utilisateur crée une commande.
2. `order-service` vérifie le stock.
3. `order-service` appelle le paiement.
4. Si succès : événement Kafka.
5. `notification-service` envoie la notification.
6. Les métriques sont collectées.
7. Kubernetes scale automatiquement.

C’est une vraie chaîne cloud native.

---

# 19. Travaux demandés

## Partie 1

Créer les microservices.

## Partie 2

Configurer Eureka.

## Partie 3

Ajouter OpenFeign.

## Partie 4

Ajouter Resilience4j.

## Partie 5

Ajouter Kafka.

## Partie 6

Dockeriser les services.

## Partie 7

Déployer sur Kubernetes.

## Partie 8

Ajouter ConfigMap + HPA.

## Partie 9

Activer Actuator + Prometheus.


