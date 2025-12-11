# Deploy su Raspberry Pi 5 (Bookworm)

Questi file automatizzano l'installazione e l'avvio dell'applicazione su un Raspberry Pi 5 con Raspberry Pi OS Bookworm.

## deploy_rpi5.sh

Script di deploy principale che va eseguito come `root` o con `sudo` dalla cartella del progetto clonata sul Raspberry Pi.

Cosa fa:
1. Installa i pacchetti di sistema necessari (Python, git, rsync, curl, ecc.).
2. Crea l'utente di servizio `roomctl` se manca.
3. Crea la directory di deploy `/opt/roomctl` con permessi ristretti (0750) e proprietà `roomctl:roomctl`, poi sincronizza i sorgenti ignorando `.git`, `.venv` e i file YAML di configurazione esistenti.
4. Crea l'ambiente virtuale Python in `/opt/roomctl/.venv` e installa le dipendenze principali (FastAPI, Uvicorn, ecc.).
5. Copia i file di configurazione di default in `/opt/roomctl/config` senza sovrascrivere quelli già presenti.
6. Installa e abilita il servizio systemd `roomctl.service` e il power scheduler.

Esecuzione tipica:
```bash
sudo ./deploy_rpi5.sh
```
Lo script ricarica systemd e abilita/avvia i servizi; al termine l'applicazione sarà raggiungibile sulla porta 8080.

Se devi preparare la directory manualmente (ad esempio prima di lanciare lo script), usa:

```bash
sudo install -d -o roomctl -g roomctl -m 0750 /opt/roomctl
```
In questo modo solo l'utente `roomctl` e il gruppo omonimo potranno scrivere nella cartella di deploy.

## config/roomctl.service

Unità systemd che avvia l'applicazione FastAPI con Uvicorn come utente `roomctl`.

Principali impostazioni:
- `WorkingDirectory`: `/opt/roomctl` (dove lo script di deploy copia il progetto).
- `ExecStart`: avvia Uvicorn dall'ambiente virtuale `/opt/roomctl/.venv` esponendo l'app su tutte le interfacce alla porta 8080.
- `Environment`: imposta `ROOMCTL_BASE` a `http://127.0.0.1:8080` (puoi modificarlo in base alle esigenze di rete/reverse proxy).

Lo script di deploy copia automaticamente il file in `/etc/systemd/system/roomctl.service` e lo abilita. Se modifichi l'unità manualmente, ricorda di eseguire `sudo systemctl daemon-reload` e poi `sudo systemctl restart roomctl.service`.

## Relazione tecnica
Per una panoramica dell'architettura applicativa, dei flussi UI/backend e delle funzioni implementate consulta [RELAZIONE_TECNICA.md](RELAZIONE_TECNICA.md).

## Troubleshooting rapido
Se, dopo il deploy, la UI non intercetta le azioni del backend su un altro Raspberry Pi:
- Verifica che `roomctl.service` sia `active (running)` e ascolti sulla porta prevista (default 8080).
- Conferma che le dipendenze siano nella virtualenv con `/.venv/bin/pip show fastapi uvicorn`.
- Se avvii Uvicorn su una porta diversa, aggiorna `ROOMCTL_BASE` nel file dell'unità systemd oppure esportala prima di lanciare il server.
Per maggiori dettagli consulta [TROUBLESHOOTING.md](TROUBLESHOOTING.md).
