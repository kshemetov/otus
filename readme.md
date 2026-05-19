Установка экспортеров на lab-prom02
Node Exporter (метрики ОС)
Установка:
sudo apt install -y prometheus-node-exporter
Проверка:
root@lab-prom02:~# curl http://localhost:9100/metrics | head -n 5
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0# HELP go_gc_duration_seconds A summary of the wall-time pause (stop-the-world) duration in garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 3.2959e-05
go_gc_duration_seconds{quantile="0.25"} 4.1508e-05
go_gc_duration_seconds{quantile="0.5"} 4.5911e-05
100 12086    0 12086    0     0   658k      0 --:--:-- --:--:-- --:--:--  694k

MySQL Exporter (метрики базы данных)
Установка:

sudo apt install -y prometheus-mysqld-exporter
Создание пользователя в MySQL:
sudo mysql -u root -p

CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'password';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;
EXIT;

Настройка подключения:
nano /etc/default/prometheus-mysqld-exporter:
DATA_SOURCE_NAME="exporter:password@(localhost:3306)/"

sudo systemctl restart prometheus-mysqld-exporter
sudo systemctl enable prometheus-mysqld-exporter

Проверка:
curl http://localhost:9104/metrics | head -n 5

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 2.9868e-05
go_gc_duration_seconds{quantile="0.25"} 7.793e-05
go_gc_duration_seconds{quantile="0.5"} 0.000110311
100 12108    0 12108    0     0   419k      0 --:--:-- --:--:-- --:--:--  422k

Blackbox Exporter
Установка:
sudo apt install -y prometheus-blackbox-exporter


Установка и настройка Prometheus на lab-prom01

sudo apt update
sudo apt install -y prometheus

Конфигурация Prometheus
nano /etc/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Мониторинг самого Prometheus
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Node Exporter (метрики ОС)
  - job_name: "node"
    scrape_interval: 5s
    static_configs:
      - targets: ["10.18.7.125:9100"]   # IP lab-prom02

  # MySQL Exporter (метрики БД)
  - job_name: "mysql"
    scrape_interval: 5s
    static_configs:
      - targets: ["10.18.7.125:9104"]

  # Blackbox Exporter (проверка доступности CMS)
  - job_name: "blackbox"
    scrape_interval: 5s
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://10.18.7.125   # CMS на lab-prom02
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: "10.18.7.125:9115"   # Blackbox на lab-prom02


sudo systemctl restart prometheus
sudo systemctl enable prometheus

Проверка:
```
root@lab-prom01:~# curl http://localhost:9090/api/v1/targets
{"status":"success","data":{"activeTargets":[{"discoveredLabels":{"__address__":"http://10.18.7.125","__metrics_path__":"/probe","__param_module":"http_2xx","__scheme__":"http","__scrape_interval__":"5s","__scrape_timeout__":"5s","job":"blackbox"},"labels":{"instance":"http://10.18.7.125","job":"blackbox"},"scrapePool":"blackbox","scrapeUrl":"http://10.18.7.125:9115/probe?module=http_2xx\u0026target=http%3A%2F%2F10.18.7.125","globalUrl":"http://10.18.7.125:9115/probe?module=http_2xx\u0026target=http%3A%2F%2F10.18.7.125","lastError":"","lastScrape":"2026-05-19T17:20:40.068070513+03:00","lastScrapeDuration":0.003099981,"health":"up","scrapeInterval":"5s","scrapeTimeout":"5s"},{"discoveredLabels":{"__address__":"10.18.7.125:9104","__metrics_path__":"/metrics","__scheme__":"http","__scrape_interval__":"5s","__scrape_timeout__":"5s","job":"mysql"},"labels":{"instance":"10.18.7.125:9104","job":"mysql"},"scrapePool":"mysql","scrapeUrl":"http://10.18.7.125:9104/metrics","globalUrl":"http://10.18.7.125:9104/metrics","lastError":"","lastScrape":"2026-05-19T17:20:39.623587718+03:00","lastScrapeDuration":0.039203116,"health":"up","scrapeInterval":"5s","scrapeTimeout":"5s"},{"discoveredLabels":{"__address__":"10.18.7.125:9100","__metrics_path__":"/metrics","__scheme__":"http","__scrape_interval__":"5s","__scrape_timeout__":"5s","job":"node_exporter_clients"},"labels":{"instance":"10.18.7.125:9100","job":"node_exporter_clients"},"scrapePool":"node_exporter_clients","scrapeUrl":"http://10.18.7.125:9100/metrics","globalUrl":"http://10.18.7.125:9100/metrics","lastError":"","lastScrape":"2026-05-19T17:20:40.776971212+03:00","lastScrapeDuration":0.020189386,"health":"up","scrapeInterval":"5s","scrapeTimeout":"5s"},{"discoveredLabels":{"__address__":"localhost:9090","__metrics_path__":"/metrics","__scheme__":"http","__scrape_interval__":"15s","__scrape_timeout__":"10s","app":"prometheus","job":"prometheus"},"labels":{"app":"prometheus","instance":"localhost:9090","job":"prometheus"},"scrapePool":"prometheus","scrapeUrl":"http://localhost:9090/metrics","globalUrl":"http://lab-prom01:9090/metrics","lastError":"","lastScrape":"2026-05-19T17:20:35.821718511+03:00","lastScrapeDuration":0.01372594,"health":"up","scrapeInterval":"15s","scrapeTimeout":"10s"}],"droppedTargets":[],"droppedTargetCounts":{"blackbox":0,"mysql":0,"node_exporter_clients":0,"prometheus":0}}}root@lab-prom01:~#
```
