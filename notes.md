"components": {
    "securitySchemes": {
      "bearerAuth": {
        "type": "apiKey",
        "name": "Authorization",
        "in": "header",
        "description": "Entrez votre token avec le préfixe 'Bearer '. Exemple: Bearer eyJhbGc..."
      }
    }
  },
  "security": [
    {
      "bearerAuth": []
    }
  ],
