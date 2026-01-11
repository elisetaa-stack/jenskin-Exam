
├── cast-service/
│   ├── app/
│   │   ├── api/
│   │   │   └── casts.py           # Cast endpoints
│   │   ├── main.py                # FastAPI application
│   │   ├── db.py                  # Database connection
│   │   ├── db_manager.py          # Database operations
│   │   └── models.py              # Data models
│   ├── tests/
│   │   ├── helpers.py
│   │   └── NOTES.txt
│   ├── Dockerfile                 # Container image
│   └── requirements.txt           # Python dependencies
│
├── movie-service/
│   ├── app/
│   │   ├── api/
│   │   │   └── main.py            # Movie endpoints
│   │   ├── main.py                # FastAPI application
│   │   ├── db.py                  # Database connection
│   │   ├── db_manager.py          # Database operations
│   │   └── models.py              # Data models
│   ├── tests/
│   ├── Dockerfile                 # Container image
│   └── requirements.txt           # Python dependencies
│
├── helm-charts/
│   ├── cast-service/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       └── _helpers.tpl
│   └── movie-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── _helpers.tpl
│
├── kubernetes-manifests/
│   ├── all-services.yaml          # Complete K8s config
│   └── nginx-gateway.yaml         # API Gateway
│
├── Jenkinsfile                    # CI/CD Pipeline
├── docker-compose.yml             # Local development
├── nginx_config.conf              # Nginx configuration
└── README.md                      #