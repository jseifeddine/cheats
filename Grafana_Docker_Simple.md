Make a place to keep the docker-compose.yml file below, for instance
```bash
mkdir -p /etc/docker/grafana
```

Then make the file `/etc/docker/grafana/docker-compose.yml`

With content:
```yaml
version: "3.8"

x-grafana-common: &grafana-common
  image: grafana/grafana-enterprise
  restart: always

x-environment:
  grafana: &grafana-environment
    GF_SERVER_DOMAIN: grafana.example.com.au
    GF_SERVER_SERVE_FROM_SUB_PATH: true
    # ^^^ depending on your setup grafana.example.com.au/grafana (true) or just grafana.example.com.au (false)
    GF_FEATURE_TOGGLES_ENABLE: nestedFolders
    GF_SERVER_HTTP_PORT: 3000
    GF_SERVER_PROTOCOL: http
    GF_DEFAULT_INSTANCE_NAME: "whatever"
    GF_SECURITY_ADMIN_USER: AdminUserNameCHANGEME
    GF_SECURITY_ADMIN_PASSWORD: AdminUserPasswordCHANGEME
    GF_DATE_FORMATS_FULL_DATE: "DD-MM-YYYY HH:mm:ss"
    GF_DATE_FORMATS_INTERVAL_SECOND: "HH:mm:ss"
    GF_DATE_FORMATS_INTERVAL_MINUTE: "HH:mm"
    GF_DATE_FORMATS_INTERVAL_HOUR: "DD/MM HH:mm"
    GF_DATE_FORMATS_INTERVAL_DAY: "DD/MM"
    GF_DATE_FORMATS_INTERVAL_MONTH: "MM-YYYY"
    GF_DATE_FORMATS_INTERVAL_YEAR: "YYYY"
    TZ: "Australia/Sydney"
    GF_SMTP_ENABLED: true
    GF_SMTP_HOST: smtp.example.com.au:25
    GF_SMTP_SKIP_VERIFY: false
    GF_SMTP_FROM_NAME: "Grafana Alerts"
    GF_SMTP_FROM_ADDRESS: grafana@example.com.au


services:
  grafana-enterprise:
    restart: always 
    <<: *grafana-common
    environment:
      <<: *grafana-environment
    volumes:
      - grafana_data:/var/lib/grafana:rw

volumes:
  grafana_data: {}
```

Then `cd /etc/docker/grafana; docker compose up -d` 
