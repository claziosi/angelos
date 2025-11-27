{
  "name": "Custom Health Check API",
  "version": "1.0.0",
  "description": "Mock API with customizable text response",
  "visibility": "PUBLIC",
  "gravitee": "2.0.0",
  "flow_mode": "DEFAULT",
  "paths": {
    "/": {
      "get": [
        {
          "name": "Custom Text Response",
          "enabled": true,
          "request": [],
          "response": [
            {
              "name": "Mock Response",
              "description": "Return custom text",
              "enabled": true,
              "policy": "mock",
              "configuration": {
                "status": "200",
                "headers": [
                  {
                    "name": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "content": "{\"status\":\"UP\",\"message\":\"Gravitee Gateway is healthy\",\"timestamp\":\"{{now}}\",\"version\":\"1.0.0\",\"service\":\"gravitee-gateway\",\"custom_text\":\"🚀 Votre texte personnalisé ici\",\"details\":{\"environment\":\"production\",\"region\":\"eu-west-1\",\"uptime\":\"99.9%\"}}"
              }
            }
          ]
        }
      ]
    },
    "/status": {
      "get": [
        {
          "name": "Simple Status",
          "enabled": true,
          "request": [],
          "response": [
            {
              "name": "Status Response",
              "enabled": true,
              "policy": "mock",
              "configuration": {
                "status": "200",
                "headers": [
                  {
                    "name": "Content-Type",
                    "value": "text/plain"
                  }
                ],
                "content": "✅ Service opérationnel - Tous les systèmes fonctionnent normalement"
              }
            }
          ]
        }
      ]
    },
    "/info": {
      "get": [
        {
          "name": "Detailed Info",
          "enabled": true,
          "request": [],
          "response": [
            {
              "name": "Info Response",
              "enabled": true,
              "policy": "mock",
              "configuration": {
                "status": "200",
                "headers": [
                  {
                    "name": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "content": "{\"application\":\"Gravitee API Gateway\",\"description\":\"Voici votre texte personnalisé pour les informations système\",\"build_date\":\"{{now}}\",\"endpoints\":{\"/\":\"Health check principal\",\"/status\":\"Status texte simple\",\"/info\":\"Informations détaillées\",\"/custom\":\"Réponse personnalisée avec paramètres\"},\"maintenance\":false,\"contact\":\"support@votre-domaine.com\"}"
              }
            }
          ]
        }
      ]
    },
    "/custom": {
      "get": [
        {
          "name": "Custom with Query Params",
          "enabled": true,
          "request": [],
          "response": [
            {
              "name": "Dynamic Response",
              "enabled": true,
              "policy": "mock",
              "configuration": {
                "status": "200",
                "headers": [
                  {
                    "name": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "content": "{\"status\":\"success\",\"message\":\"Texte personnalisé via paramètre\",\"query_param\":\"{{request.params['text']}}\",\"default_text\":\"Aucun texte fourni - utilisez ?text=votre_message\",\"timestamp\":\"{{now}}\",\"request_id\":\"{{request.id}}\"}"
              }
            }
          ]
        }
      ]
    }
  },
  "proxy": {
    "virtual_hosts": [
      {
        "path": "/health-check"
      }
    ],
    "strip_context_path": false,
    "preserve_host": false,
    "groups": [
      {
        "name": "default-group",
        "endpoints": [
          {
            "name": "mock-endpoint",
            "target": "https://api.gravitee.io/echo",
            "weight": 1,
            "backup": false,
            "type": "http",
            "inherit": true
          }
        ],
        "load_balancing": {
          "type": "ROUND_ROBIN"
        }
      }
    ]
  },
  "response_templates": {}
}
