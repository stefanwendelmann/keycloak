# keycloak

## Keycloak Prometheus Grafana stack manual

Create Docker Network "kc"

```bash
docker network create kc
```

Start Keycloak with mount of the deployment path 

```bash
docker run -d --name keycloak -p 8080:8080 -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin -v /mnt/c/Users/swendelmann/keycloak_spi_test/deployments/:/opt/jboss/keycloak/standalone/deployments/ -e DB_USER=keycloak --network kc jboss/keycloak:11.0.3
```

Deploy Keycloak SPI Addon

```bash
cd deployments
wget https://github.com/aerogear/keycloak-metrics-spi/releases/download/2.2.0/keycloak-metrics-spi-2.2.0.jar
```

Enable metrics-listener event

To enable the event listener via the GUI interface, go to Manage -> Events -> Config. The Event Listeners configuration should have an entry named `metrics-listener`.


Write: 
prometeus.yml Config

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'keycloak'
    metrics_path: /auth/realms/master/metrics
    static_configs:
    - targets: ['keycloak:8080']

```

Start Prometheus with mount for the config yml

```bash
docker run -d --name prometheus -p 9090:9090 -v /mnt/c/Users/swendelmann/keycloak_spi_test/prometheus/:/etc/prometheus/ --network kc prom/prometheus:v2.25.2
```

Start Grafana

```bash
docker run --name grafana -d -p 3000:3000 --network kc grafana/grafana:7.5.2
```

Install Dashboard and plugins / reconfigure

see https://grafana.com/grafana/dashboards/10441


### Test Login

Generate Token and save to ENV 

```bash
TOKEN=$(curl -s --location --request POST 'http://localhost:8080/auth/realms/master/protocol/openid-connect/token' --header 'Content-Type: application/x-www-form-urlencoded' --data-urlencode 'client_id=admin-cli' --data-urlencode 'username=admin' --data-urlencode 'password=admin' --data-urlencode 'grant_type=password' | jq .'access_token' | sed -e 's/^"//' -e 's/"$//')
```

Use Token 

```bash
curl -H "Authorization: bearer $TOKEN" "http://localhost:8080/auth/admin/realms/master"
```