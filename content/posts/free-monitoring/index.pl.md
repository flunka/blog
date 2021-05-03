---
title: "Darmowy monitoring z Graphana Labs"
date: 2021-05-03T09:37:41+02:00
draft: false
author: "Flunka.pl"
tags: ["Monitoring", "Ansible", "DevOps"]
categories: ["DevOps"]
theme: wide
toc:
  enable: true
summaryStyle:
  hiddenImage: false
  hiddenDescription: false
  hiddenTitle: true
  tags:
    theme: "image"
    color: "white"
    background: "black"
    transparency: 0.8
resources:
  - name: featured-image
    src: grafana-labs.png
---

## Darmowy plan Grafana Cloud

Zazwyczaj rozpoczynając mały ptojekt IT nie mamy wystarczająco zasobów, aby uruchomić monitoring dla naszego projetku. Na szczęście na początku tego roku Grafana Labs wprowadziła darmowy plan dla ich rozwiązania cloud-owego.
Ten plan zawiera:

- 10,000 serii dla metryk Prometheus albo Graphite
- 50 GB logów
- 14 dni retencji dla metry i logów
- Dostęp dla 3 członków zespołu

Dla mnie jest to wystarczające, więc postanowiłem wypróbować to rozwiązanie do monitorowania mojego serwera VPS (Centos 7).
Po pierwsze, należy założyć darmowe konto na [Grafana](https://grafana.com/auth/sign-up/create-user?pg=prod-cloud-pricing&plcmt=free).
Następnie możemy skonfigurować nasze narzędzia do zbierania logów, metryk i wysyłania alertów.

## Logi

### Promtail

Do wysyłania logów z VPS do usługi Loki będziemy używać Promtail.
Według ich dokumentacji najprostszym sposobem, aby uruchomić Promtail jest zrobienie tego przy użyciu dockera. Jako, że nie miałem wcześniej zainstalowanego dockera na swoim serwerze, stworzyłem Ansible playbooka, który instaluje dockera.

```yaml
---
- name: Add repository
  yum_repository:
    name: docker-ce
    description: Docker repo
    baseurl: https://download.docker.com/linux/centos/7/$basearch/stable
    gpgkey: https://download.docker.com/linux/centos/gpg
    state: present
  become: yes
- name: Install a packages for docker
  yum:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present
  become: yes
- name: Ensure docker is running and enabled
  systemd:
    name: docker
    state: started
    enabled: yes
  become: yes
```

Pierwszy task dodaje repo dockera. Dokumentacja dockera tutaj niestety nie pomogła. Według tej dokumentacji należy dodać repo `https://download.docker.com/linux/centos/docker-ce.repo`. Po dodaniu takiego adresu przy instalacji dostawałem błąd:

`https://download.docker.com/linux/centos/docker-ce.repo/repodata/repomd.xml: [Errno 14] HTTPS Error 404 - Not Found`

Po krótkim szukaniu rozwiązania problemu, znalazłem odpowiedź na [forum dockera](https://forums.docker.com/t/docker-ce-stable-x86-64-repo-not-available-https-error-404-not-found-https-download-docker-com-linux-centos-7server-x86-64-stable-repodata-repomd-xml/98965/6). Wystarczyło zmienić base url i gotowe.
Drugi task instaluje wymagane paczki, a trzeci uruchamia dockera i ustawia uruchamianie dockera przy starcie systemu.
Docker zainstalowany i uruchomiony. Zanim wystartujemy kontener z Promtail, stwórzmy podstawową konfiguracje w `/etc/promtail/config.yaml`. Do tego będziemy potrzebować wygenerowany klucz API, aby Promtail mógł się uwierzytelnić. Klucz powinien mieć rolę **MetricPusher**.

{{< admonition type=danger open=true >}}
Pamiętaj, aby nie udostępniać Twojego klucza API!
{{< /admonition >}}

```yaml
server:
  http_listen_port: 0
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

client:
  url: https://39544:<Your Grafana.com API Key>@logs-prod-us-central1.grafana.net/api/prom/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

Powyższa konfiguracja powoduje monitorowanie wszystkich plików w katalogu `/var/log/` kończących się na `.log`.
Do uruchomienia kontenera z Promtail napisałem poniższy ansible playbook.

```yaml
---
- name: Create a directory for Promtail config
  file:
    path: /etc/promtail
    state: directory
    group: "{{ansible_user}}"
    mode: "0775"
  become: yes
- name: Copy Promtail config file
  copy:
    src: config.yaml
    dest: /etc/promtail/config.yaml
    mode: "0644"

- name: Ensure docker-py is installed
  pip:
    name: docker-py
    state: present
  become: yes

- name: Container present
  docker_container:
    name: promtail
    state: started
    container_default_behavior: compatibility
    image: grafana/promtail:master
    volumes:
      - /etc/promtail:/etc/promtail
      - /var/log:/var/log
    command: -config.file=/etc/promtail/config.yaml
```

W tym playbook-u na początku tworzymy katalog, w którym będziemy trzymać konfigurację Promtail. Następnie kopiujemy naszą konfigurację do stworzonego katalogu. Do uruchomienia modułu `docker_container` jest nam potrzebna paczka `docker` lub `docker-py`. Miałem problem z paczką `docker` i pythonem 2, więc zainstalowałem `docker-py`. Ostatnią rzeczą jest uruchomienie kontenera. Obraz, z którego korzystamy, to `grafana/promtail`. Ważne jest, aby dodać wolumeny z konfiguracją (/etc/promtail/) oraz z katalogiem z logami (/var/log/). Jako komendę do uruchomienia kontenera podajemy ścieżkę do pliku z konfiguracją.

### Loki

Po pomyślnym uruchomieniu kontenera, nasze logi powinny być wysyłane do usługi Loki.
Teraz powinniśmy być w stanie przeglądać nasze logi z poziomu Grafany. Wszysto, co musisz zrobić to:

1. Zalogować się do Grafany
2. Przejść do zakładki Explore
3. Wybrać źródło danych grafanacloud-xxx-logs, gdzie xxx to twoja nazwa (to źródło danych powinno być automatycznie skonfigurowane).
4. Wpisać proste zapytanie, aby otrzymać dane np. aby otrzymać logi z pliku `/var/log/yum.log` wystarczy wpisać `{filename="/var/log/yum.log"}`.

Więcej informacji na temat tworzenia zapytań można znaleźć w [dokumentacji Grafany](https://grafana.com/docs/grafana/latest/datasources/loki/#querying-logs)

{{< figure src="loki-logs.png" title="Przykładowy widok logów" >}}

## Metryki

### Node exporter

Aby udostępić metryki z naszego serwera musimy mieć zainstalowanego node exporter-a. W tym celu możemy użyć roli `cloudalchemy.node_exporter`. Aby móc użyć tę role należy ją pobrać przy użyciu polecenia `ansible-galaxy install cloudalchemy.node_exporter`. Domyślnie node exporter będzie nasłuchiwał na porcie 9100.

### Prometheus

Następnie możemy uruchomić lokalną instancję Prometheus-a, która będzie wysyłać zebrane metryki do Grafana Cloud.
Konfiguracja Prometheusha wygląda następująco:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ["172.17.0.1:9100"]
    relabel_configs:
      - source_labels: [__address__]
        regex: ".*"
        target_label: instance
        replacement: "vps790031.ovh.net"

remote_write:
  - url: https://prometheus-blocks-prod-us-central1.grafana.net/api/prom/push
    basic_auth:
      username: <USERNAME>
      password: <PASSWORD>
```

`scrape_interval` definiuje co ile metryki będą pobierane. W sekcji `scrape_configs` określamy skąd metryki mają być pobierane. Jako, że Prometheus-a uruchomimy w kontenerze, metryki są pobierane przez wewnętrzny interfejs dockerowy. Aby w naszym monitoringu było wiadomo skąd pochodzą metryki, warto zmienić etykiety dla tego hosta, ponieważ w przeciwnym przypadku będziemy widzieć, że metryki pochodzą z adresu `172.17.0.1`, a nie z adresu naszego serwera. Dlatego w sekcji `relable_conifgs` podajemy pod jaką nazwą nasze metryki mają być widoczne. W ostatniej sekcji konfigurujemy gdzie metryki mają być wysyłane.

Konfig mamy gotowy, więc zostało storzyć ansiblową rolę, która będzie wrzucać konfig na serwer i uruchamiać kontener z Prometheus-em.

```yaml
---
- name: Create a directory for Prometheus config
  file:
    path: /etc/prometheus
    state: directory
    group: "{{ansible_user}}"
    mode: "0775"
  become: yes
- name: Copy Prometheus config file
  copy:
    src: prometheus.yaml
    dest: /etc/prometheus/prometheus.yaml
    mode: "0644"
  register: prom_config

- name: Ensure docker-py is installed
  pip:
    name: docker-py
    state: present
  become: yes

- name: Container present
  docker_container:
    name: prometheus
    state: started
    container_default_behavior: compatibility
    restart: "{{prom_config.changed}}"
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - /etc/prometheus/prometheus.yaml:/etc/prometheus/prometheus.yml
```

### Grafana dashboard

Jeżeli wszystko poszło zgodnie z planem, metryki powinny już być widoczne na Grafanie.
Aby zwizualizować metryki skorzystajmy z gotowego dashboardu dla node exportera. W tym celu klikamy w `Create->Import`, jak na na screenie poniżej.

{{< figure src="import-dashboard.png" title="Import dashboard" >}}

Następnie podajemy id `1860` i klikamy load. Po załadowaniu wybieramy odpowiednie źródło danych i klikamy import.
Po zaimportowaniu tego dashbordu powinien ukazać się nam widok podobny do tego zamieszczonego poniżej. Oczywiście możemy dostosować ten dashboard do naszych preferencji lub stworzyć własny, dopasowany do naszych potrzeb. Moim zdaniem ten sprawdza się całkiem nieźle.

{{< figure src="prometheus-metrics.png" title="Node exporter dashboard" >}}

## Alerty

{{< figure src="grafana-cloud-alerting.png" title="Grafana Cloud Alerting" >}}

Mamy już skonfigurowane zbieranie logów i metryk z naszego serwara, więc możemy teraz ustawić alarmy, które będą wywoływane na ich podstawie.
Dzięki Grafana Cloud Aletring możemy zdefiniować reguły dotyczące alarmów. Reguły możemy tworzyć na podstawie logów, jak i metryk.
Poniżej przykład dla reguły odnośnie wysokiego użycia CPU.

```yaml
alert: HighUsage
expr: irate(node_cpu_seconds_total{mode="idle"}[1m]) * 100 < 30
for: 60s
annotations:
  description: "{{ $labels.instance }} has a CPU idle (current value: {{
    $value }}s)"
  summary: High usage on {{ $labels.instance }}
```

Powyższy alert zostanie uruchomiony jeżeli przez ostatnią minutę idle CPU było poniżej 30%.
Jeżeli mamy już skonfigurowane alarmy, wystarczy podać adres mailowy, na który mają być wysyłane i gotowe. Domyślnie maile wysyłane są z serwerów Grafany, lecz nic nie stoi na przeszkodzie, żeby były wysyłane z twojego serwera smtp.

{{< figure src="example-alert.png" title="Przykładowy alert" >}}

## Podsumowanie

Moim zdaniem Grafana Labs dostarcza całkiem przydatny zestaw narzędzi do monitoringu. Ich darmowy plan wystarcza w zupełności dla małych projektów, więc jeżeli takie macie, to polecam wypróbować. Cały kod możecie znaleźć na moim [githubie](https://github.com/flunka/vps_monitoring)
