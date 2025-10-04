# 🧠 DeepSeek - Architecture Gateway & Microservices Parallèle
## 👨‍💻 Auteurs

- **Angham Regaieg**  
- **Aness Ben Slama**  
_3IDL02_

---
## 📘 Présentation générale

Ce document décrit deux approches centrales utilisées dans le système **DeepSeek** :

- **Architecture Gateway** : point d’entrée unique qui gère le routage, la sécurité et la résilience (fallback).  
- **Architecture Microservices Parallèle** : plusieurs microservices traitent les mêmes requêtes en parallèle (ex. Chat + Training) et s’agrègent via un message bus pour améliorer la qualité et la rapidité des réponses.

---

# 🚪 1. Architecture Gateway

## 🧩 Description

L’**API Gateway** centralise l’accès client vers les microservices internes (Auth, Chat, Model, Training, File, ...). Elle implémente :  
- Authentification / quotas / vérification de token  
- Routage primaire / fallback (DeepSeek-V3 → DeepSeek-Lite)  
- Monitoring, logging et métriques  
- Logique de retry / timeout / agrégation de réponse

## ⚙️ Fonctionnement simplifié

1. Le client envoie une requête.  
2. La Gateway appelle le service principal (DeepSeek-V3).  
3. Si timeout/erreur → fallback vers DeepSeek-Lite.  
4. La Gateway uniformise la réponse, ajoute des métadonnées et la renvoie.

## ✅ Avantages

- Centralisation de la sécurité et du contrôle d’accès  
- Supervision et métriques sur un point unique  
- Fallback automatique pour améliorer la disponibilité côté client

---

## ⚠️ **Limites de l’Architecture Gateway et solution**

1. **Point unique de défaillance (SPOF)**  
   - *Problème* : si la Gateway tombe, tout le système devient indisponible.  
   - *Solution* : déployer plusieurs instances de la Gateway derrière un load-balancer, utiliser health-checks, probes, et autoscaling (K8s). Mettre en place des déploiements multi-région si nécessaire.

2. **Goulot d’étranglement (bande passante / CPU)**  
   - *Problème* : tout le trafic traverse la Gateway → risque de saturation.  
   - *Solution* : scale horizontal, cache côté Gateway (réponses LRU pour requêtes fréquentes), mettre en place du backpressure et limiter le débit (rate limiting, quotas par API key).

3. **Complexité opérationnelle et configuration**  
   - *Problème* : règles de routage, timeouts, retries deviennent lourds à maintenir.  
   - *Solution* : stocker la configuration dans un store central (ex. Consul), utiliser feature flags, tests d’intégration automatisés et observabilité (OpenTelemetry).

4. **Sécurité concentrée (attaque ciblée)**  
   - *Problème* : Gateway devient cible privilégiée pour attaques DDoS / abuse.  
   - *Solution* : protections WAF, limits par IP/API key, CDN en frontal, authentification forte et rotation des clés.

---

# ⚙️ 2. Architecture Microservices Parallèle

## 🧩 Description

Plusieurs microservices (ex. Chat Service, Training Service, Model Service) traitent **en parallèle** une même requête complexe. Un **message bus** (ex. Kafka, RabbitMQ) orchestre la distribution, la synchronisation et l’agrégation des résultats.

## 🔄 Fonctionnement simplifié

1. Le client envoie une requête (via Gateway).  
2. Le message est publié sur le message bus; plusieurs consommateurs/services la traitent en parallèle.  
3. Chaque service renvoie son verdict/partie de réponse.  
4. Un agrégateur combine les sorties et renvoie la réponse finale.

## ✅ Avantages

- Meilleure latence pour tâches parallélisables  
- Réponses plus riches en combinant expertises (training + chat + retrieval)  
- Scalabilité horizontale (ajout de consommateurs)  
- Possibilité de tolérance via redondance fonctionnelle

---

## ⚠️ **Limites de l’Architecture Parallèle et Solution**

1. **Synchronisation et orchestration complexe**  
   - *Problème* : attendre, fusionner et ordonnancer des réponses asynchrones est délicat (latence variable).  
   - *Solution* : définir des timeouts globaux, utiliser des patterns d’agrégation (scatter-gather), esquisser priorités, et supporter réponses partielles/progressives (streaming).

2. **Surcharge de communication / charge sur le bus**  
   - *Problème* : forte consommation du bus → latence ou perte de messages.  
   - *Solution* : partitionner topics, mettre en place des quotas, consommateurs en mode batch, compression des messages, monitoring de lag, et scalabilité du cluster de message bus.

3. **Gestion d’erreurs et cohérence**  
   - *Problème* : si un service échoue, quelle version de la réponse utiliser ?  
   - *Solution* : patterns de compensation (saga), réponses probabalistes (best-effort), fallback local au service, DLC (dead-letter queue) pour analyser les erreurs.

4. **Consommation élevée de ressources (GPU/CPU)**  
   - *Problème* : exécuter plusieurs modèles lourds en parallèle coûte cher.  
   - *Solution* : planification intelligente (prioritization), pooling de GPU, batching d’inférence, tiers de modèles (lite / heavy) selon coût vs qualité.

---

# 🚧 3. Comparaison synthétique & limites communes

| Critère | Gateway | Microservices parallèle |
|---|---:|---:|
| Objectif | Contrôle & sécurité | Performance & qualité |
| Risque principal | SPOF, bottleneck | Orchestration & coûts |
| Scalabilité | Scale Gateway (horizontal) | Scale services horizontalement |
| Observabilité | Centralisée | Distribuée, plus complexe |
| Coût | Concentré (gateway infra) | Potentiellement élevé (GPU/Compute) |

**Limites communes** : gestion de la latence, nécessité d’une observabilité solide, coût d’infrastructure, et complexité opérationnelle. Les deux modèles exigent des tests de charge, des stratégies de déploiement progressives (canary / blue-green) et une surveillance des SLO.

---

# 🧾 Conclusion

- L’**API Gateway** apporte centralisation, sécurité et capacité de fallback — mais nécessite redondance et scalabilité pour éviter les SPOF et goulots.  
- L’**architecture parallèle** augmente la qualité et la rapidité des réponses, mais demande une orchestration robuste, de la résilience côté message bus et une gestion stricte des ressources.  
- **Combiner** les deux (Gateway + microservices parallèles) est souvent la meilleure approche : la Gateway orchestre, protège et normalise ; les microservices fournissent la puissance et la spécialisation.

