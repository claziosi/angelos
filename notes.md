{
  "openapi": "3.0.0",
  "info": {
    "title": "Mon API Gravitee",
    "version": "1.0.0",
    "description": "API avec authentification OAuth2"
  },
  "servers": [
    {
      "url": "https://api.example.com/v1"
    }
  ],
  "components": {
    "securitySchemes": {
      "bearerAuth": {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT",
        "description": "Entrez votre access token OAuth2. Pour obtenir un token, utilisez votre application Gravitee."
      }
    }
  },
  "security": [
    {
      "bearerAuth": []
    }
  ],
  "paths": {
    "/users": {
      "get": {
        "summary": "Liste des utilisateurs",
        "operationId": "getUsers",
        "tags": ["Users"],
        "responses": {
          "200": {
            "description": "Liste des utilisateurs",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "type": "object"
                  }
                }
              }
            }
          },
          "401": {
            "description": "Non autorisé"
          }
        }
      }
    }
  }
}
