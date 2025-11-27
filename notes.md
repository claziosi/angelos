{
  "name": "Custom Health Check API",
  "version": "1.0.0",
  "description": "Mock API with customizable text response - No backend needed",
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
            "name": "dummy-endpoint",
            "target": "https://httpbin.org/status/200",
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
      "pre": [
        {
          "name": "Mock Response - Skip Backend",
          "description": "Return mock response without calling backend",
          "enabled": true,
          "policy": "mock",
          "configuration": {
            "status": "200",
            "headers": [
              {
                "name": "Content-Type",
                "value": "application/json"
              },
              {
                "name": "Cache-Control",
                "value": "no-cache"
              }
            ],
            "content": "{\"status\":\"UP\",\"message\":\"Service is healthy\",\"timestamp\":\"{#now()}\",\"service\":\"gravitee-gateway\",\"custom_text\":\"Votre texte personnalise ici\",\"details\":{\"environment\":\"production\",\"checks\":{\"gateway\":\"UP\",\"backend\":\"UP\"}}}"
          }
        }
      ],
      "post": [],
      "enabled": true,
      "response": []
    },
    {
      "name": "Status Text Flow",
      "path-operator": {
        "path": "/status",
        "operator": "EQUALS"
      },
      "condition": "",
      "consumers": [],
      "methods": ["GET"],
      "pre": [
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
      ],
      "post": [],
      "enabled": true,
      "response": []
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
      "pre": [
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
            "content": "{\"application\":\"Gravitee API Gateway\",\"description\":\"Informations systeme personnalisees\",\"version\":\"1.0.0\",\"build_date\":\"{#now()}\",\"endpoints\":{\"/\":\"Health check principal\",\"/status\":\"Status texte simple\",\"/info\":\"Informations detaillees\"},\"maintenance\":false,\"custom_message\":\"Modifiez ce texte selon vos besoins\"}"
          }
        }
      ],
      "post": [],
      "enabled": true,
      "response": []
    }
  ],
  "response_templates": {}
}
