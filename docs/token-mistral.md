# 🚀 Récupérer un Token d'API Mistral AI

Voici comment récupérer ton token d'authentification ! 🔑

## 🔹 Étapes

1. **Accède à la console Mistral AI** 🌍
   - Rends-toi sur la console Mistral AI à l'adresse suivante : [https://console.mistral.ai/](https://console.mistral.ai/).

2. **Crée un compte et/ou connecte-toi** 🔐

3. **Génère un token d'API** 🎟️
   - Une fois connecté(e), accède à la section "API Keys" ou "Clés API".
   - Clique sur "Créer une nouvelle clé" _(ou similaire)_.
   - Copie et sauvegarde ton token en lieu sûr. 🛡️

## 🛠️ Utilisation du Token

Le token est indispensable pour authentifier tes requêtes API. Exemple avec cURL :

```sh
curl -X GET "https://api.mistral.ai/v1/endpoint" \
     -H "Authorization: Bearer TON_TOKEN"
```

Remplace `TON_TOKEN` par la clé obtenue précédemment. 🔄

## ⚠️ Sécurité

- 🔒 **Ne partage jamais ton token !**
- 📂 **Stocke-le dans un gestionnaire de secrets ou une variable d'environnement.**
- 🔄 **Renouvelle-le régulièrement si nécessaire.**

---
🎉 Voilà, tu es prêt(e) à utiliser l'API Mistral AI ! Amuse-toi bien ! 🚀
