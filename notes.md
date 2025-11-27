{
  "name": "Custom Health Check API",
  "version": "1.0.0",
  "description": "Mock API with customizable text response",
  "visibility": "PUBLIC",
  "gravitee": "2.0.0",
  "flow_mode": "DEFAULT",
  "resources": [],
  "properties": [],
  "members": [],
  "pages": [],
  "plans": [
    {
      "name": "Free Plan",
      "description": "Free plan for health check",
      "validation": "AUTO",
      "security": "KEY_LESS",
      "type": "API",
      "status": "PUBLISHED",
      "order": 1,
      "paths": {},
      "flows": []
    }
  ],
  "metadata": [],
  "path_mappings": [],
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
        },
        "http": {
          "connectTimeout": 5000,
          "idleTimeout": 60000,
          "keepAlive": true,
          "readTimeout": 10000,
          "pipelining": false,
          "maxConcurrentConnections": 100,
          "useCompression": true,
          "followRedirects": false
        }
      }
    ]
  },
  "flows": [
    {
      "name": "Health Check Flow",
      "path-operator": {
        "path": "/",
        "operator": "STARTS_WITH"
      },
      "condition": "",
      "consumers": [],
      "methods": ["GET"],
      "pre": [],
      "post": [],
      "enabled": true,
      "response": [
        {
          "name": "Mock Response",
          "description": "Return custom health status",
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
            "content": "{\"status\":\"UP\",\"message\":\"Gravitee Gateway is healthy\",\"timestamp\":\"{#now()}\",\"service\":\"gravitee-gateway\",\"custom_text\":\"Votre texte personnalise ici\"}"
          }
        }
      ]
    },
    {
      "name": "Status Flow",
      "path-operator": {
        "path": "/status",
        "operator": "EQUALS"
      },
      "condition": "",
      "consumers": [],
      "methods": ["GET"],
      "pre": [],
      "post": [],
      "enabled": true,
      "response": [
        {
          "name": "Status Mock",
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
            "content": "Service operationnel - Tous les systemes fonctionnent normalement"
          }
        }
      ]
    },
    {
      "name": "Info Flow",
      "path-operator": {
        "path": "/info",
        "operator": "EQUALS"
      },
      "condition": "",
      "consumers": [],
      "methods": ["GET"],
      "pre": [],
      "post": [],
      "enabled": true,
      "response": [
        {
          "name": "Info Mock",
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
            "content": "{\"application\":\"Gravitee API Gateway\",\"description\":\"Texte personnalise pour les informations\",\"build_date\":\"{#now()}\",\"endpoints\":{\"/\":\"Health check principal\",\"/status\":\"Status texte simple\",\"/info\":\"Informations detaillees\"},\"maintenance\":false}"
          }
        }
      ]
    }
  ],
  "response_templates": {}
}
