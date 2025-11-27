{
  "name": "Health Check API",
  "version": "1.0.0",
  "description": "Simple health check endpoint",
  "visibility": "PUBLIC",
  "gravitee": "2.0.0",
  "flow_mode": "DEFAULT",
  "paths": {
    "/": {
      "get": [
        {
          "name": "Health Check Flow",
          "enabled": true,
          "request": [],
          "response": [
            {
              "name": "Mock Response",
              "description": "Return healthy status",
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
                "content": "{\"status\":\"UP\",\"timestamp\":\"{{now}}\",\"service\":\"gravitee-gateway\",\"checks\":{\"gateway\":{\"status\":\"UP\"},\"backend\":{\"status\":\"UP\"}}}"
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
        "path": "/health"
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
