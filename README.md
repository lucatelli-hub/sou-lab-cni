# Progetto SOU-LAB-CNI: Monitoraggio con Podman e Ansible

Questo repository contiene l'automazione per il deploy di un'infrastruttura di monitoraggio locale su macOS, utilizzando Ansible per l'orchestrazione e Podman per la gestione dei container.

L'obiettivo del laboratorio è esporre Grafana e Prometheus attraverso un unico punto di accesso sicuro (HAProxy in HTTPS) con gestione dei subpath.

## Requisiti e Setup Iniziale

1. Installazione Strumenti
E' necessario avere Homebrew installato per gestire i pacchetti del sistema:
- brew install ansible podman git openssl

2. Configurazione Podman
I passaggi per l'avvio sono:
- podman machine init
- podman machine start

Nota: Se i container necessitano di permessi root per gestire porte privilegiate, eseguire: podman machine set --rootful.

## Struttura del Progetto

Il progetto utilizza un ruolo Ansible chiamato sou_podman per mantenere il codice pulito e modulare. I file principali sono:

- deploy.yml: Playbook principale che richiama il ruolo sou_podman.
- inventory: Inventario che punta a localhost con connessione locale.
- .gitignore: Configurato per escludere dati sensibili e file generati.
- roles/sou_podman/tasks/main.yml: Logica di deploy per directory, certificati e container.
- roles/sou_podman/templates/haproxy.cfg.j2: Template per la configurazione di HAProxy.
- roles/sou_podman/templates/prometheus.yml.j2: Template per la configurazione di Prometheus.
- containers_vols: Cartella generata automaticamente per i volumi persistenti dei container.

## Implementazione HTTPS e Certificati

Per garantire la sicurezza, il traffico verso il proxy è criptato. Il playbook gestisce automaticamente i seguenti passaggi:

- Generazione chiavi: Creazione di una chiave privata (server.key) e un certificato pubblico (server.crt) auto-firmato tramite OpenSSL.
- Formato PEM: Unione dei due file in haproxy.pem, formato richiesto da HAProxy per gestire il protocollo SSL.
- SSL Termination: HAProxy riceve le connessioni sulla porta 8443 (HTTPS), decifra il traffico e lo smista ai backend in HTTP sulla rete interna.

## Configurazione Reverse Proxy (HAProxy)

Il sistema utilizza un'unica porta (8443) e distingue i servizi tramite l'uso dei subpath:

- Grafana: raggiungibile su https://localhost:8443/grafana
- Prometheus: raggiungibile su https://localhost:8443/prometheus

Dettaglio configurazione (Jinja2):
All'interno di HAProxy viene gestito il rewriting dei path per garantire che i backend ricevano le richieste correttamente. Esempio per Grafana:
backend grafana_back
    http-request set-path %[path,regsub(^/grafana/,/)]
    server grafana host.containers.internal:3000

## Servizi Monitorati

1. Prometheus
- Raccoglie le metriche dai target configurati.
- Supporta il subpath tramite l'argomento: --web.external-url=/prometheus/.
- Comunica con l'host tramite l'indirizzo speciale: host.containers.internal.

2. Grafana
- Piattaforma per dashboard professionali.
- Configurato tramite variabili d'ambiente (GF_SERVER_ROOT_URL) per gestire il path /grafana.
- Data Source puntato all'indirizzo: http://host.containers.internal:9090.

## Modalita di Esecuzione

1. Assicurarsi che la macchina Podman sia avviata correttamente.
2. Eseguire il playbook dalla cartella principale:
   ansible-playbook -i inventory deploy.yml --ask-become-pass

Verrà richiesta la password di sistema per permettere ad Ansible di gestire la creazione dei file nelle directory locali.

3. Verifica dello stato dei container:
   podman ps

## Note Tecniche

- Persistenza: Tutti i dati (database, dashboard e certificati) risiedono nella cartella containers_vols. Questa cartella è esclusa dal controllo di versione via .gitignore.
- Portabilita: L'uso della variabile {{ ansible_env.HOME }} nei percorsi dei file assicura che il laboratorio funzioni su qualsiasi postazione macOS senza modifiche manuali al codice.

Federico Lucatelli
Aggiornamento: 21 Aprile 2026
