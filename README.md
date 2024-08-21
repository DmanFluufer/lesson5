## Установка и запуск VictoriaMetrics

1. Скачиваем архив

   ```shell
   wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.102.1/victoria-metrics-linux-amd64-v1.102.1.tar.gz
   ```

2. Создаем пользователя и нужные каталоги, настраиваем для них владельцев 

   ```shell
   useradd --no-create-home --shell /bin/false victoriametrics
   mkdir /var/lib/victoriametrics
   chown victoriametrics:victoriametrics /var/lib/victoriametrics
   ```
   
3. Распаковываем архив  и копируем бинарник в /usr/local/bin 

   ```shell
   tar -xvzf victoria-metrics-linux-amd64-v1.102.1.tar.gz
   mv victoria-metrics-prod /usr/local/bin/
   ```
   
4. Меняем владельцев у бинарника

   ```shell
   chown victoriametrics:victoriametrics /usr/local/bin/victoria-metrics-prod
   ```
   
5. Создаём systemd сервис с периодом хранения метрик в 2 недели и запускаем:

   ```shell
   [Unit]
   Description=VictoriaMetrics time-series database
   After=network.target
   
   [Service]
   User=victoriametrics
   Group=victoriametrics
   ExecStart=/usr/local/bin/victoria-metrics-prod \
       -storageDataPath=/var/lib/victoriametrics \
       -retentionPeriod=2w
   Restart=always
   RestartSec=5
   LimitNOFILE=65536
   
   [Install]
   WantedBy=multi-user.target
   
   $ systemctl daemon-reload
   $ systemctl start victoriametrics
   ```

   

## Конфигурация Prometheus

1. Добавим лейбл site: prod в наш конфиг, а также добавим VictoriaMetrics в качестве хранилища

   ```yaml
   global:
    scrape_interval: 10s
   
   scrape_configs:
    - job_name: 'prometheus_master'
      scrape_interval: 5s
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'pcrf_node_exporter_centos'
      scrape_interval: 5s
      static_configs:
        - targets: ['192.168.110.120:9100']
          labels:
            site: prod
    - job_name: 'pcrf_postgres_exporter'
      static_configs:
        - targets: ['192.168.110.120:9187']
          labels:
            site: prod
    - job_name: 'pcrf_jmx_eporter'
      static_configs:
        - targets: ['192.168.110.120:9011']
          labels:
            site: prod
    - job_name: 'blackbox_http'
      metrics_path: /probe
      params:
        module: [http_2xx]
      static_configs:
        - targets:
          - http://192.168.110.120:8080/pcrf.admin
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: 192.168.110.120:9115
   
   remote_write:
    - url: http://localhost:8428/api/v1/write
   
   remote_read:
    - url: http://localhost:8428/api/v1/read
   ```

2. Проверяем, что метрики поступают в VictoriaMetrics:

   ```shell
   $ curl "192.168.90.5:8428/metrics" | head -n 10
   
   4process_memory_limit_bytes 33620287488
   8vm_active_force_merges 0
   6vm_cache_chars_current{type="promql/regexp"} 0
   8vm_cache_chars_max{type="promql/regexp"} 1000000
   9vm_cache_entries{type="promql/parse"} 0
    vm_cache_entries{type="promql/regexp"} 0
    vm_cache_entries{type="promql/rollupResult"} 0
    vm_cache_misses_total{type="promql/parse"} 0
    vm_cache_misses_total{type="promql/regexp"} 0
   ```
