# Troubleshooting Raspberry Pi deployment (roomctl)

Questa guida aiuta a capire perché l'interfaccia non "parla" col backend su un Raspberry Pi appena deployato.

## 1) Verifica che il backend sia in esecuzione
- Controlla lo stato del servizio systemd (deploy_rpi5.sh lo installa già):
  ```bash
  sudo systemctl status roomctl.service
  ```
  Dovresti vedere `Active: active (running)` e il comando `uvicorn app.main:app --port 8080`.
- Se non parte, avvia manualmente per avere log a video:
  ```bash
  cd /opt/roomctl
  sudo -u roomctl .venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8080 --proxy-headers
  ```

## 2) Controlla che FastAPI e le dipendenze siano installate
- Lo script `deploy_rpi5.sh` installa già FastAPI, uvicorn, httpx, PyYAML, jinja2 e python-multipart dentro la virtualenv `.venv`.
- Puoi confermare così:
  ```bash
  /opt/roomctl/.venv/bin/pip show fastapi uvicorn httpx PyYAML jinja2 python-multipart
  ```
  Se un pacchetto manca, reinstallalo con:
  ```bash
  /opt/roomctl/.venv/bin/pip install fastapi "uvicorn[standard]" httpx PyYAML jinja2 python-multipart
  ```

## 3) Verifica la porta e la base URL usata dalla UI
- Per impostazione predefinita il servizio espone il backend su `http://127.0.0.1:8080` (variabile `ROOMCTL_BASE` nel file `/opt/roomctl/config/roomctl.service`).
- Se avvii uvicorn su un'altra porta, imposta `ROOMCTL_BASE` di conseguenza prima di far partire l'app:
  ```bash
  export ROOMCTL_BASE="http://127.0.0.1:8000"
  sudo -u roomctl .venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000 --proxy-headers
  ```
- Un rapido controllo lato UI:
  ```bash
  curl -s http://127.0.0.1:8080/api/state | jq
  ```
  Se restituisce JSON, la UI può raggiungere il backend; se va in timeout, c'è un problema di porta o bind.

## 4) Controlla i log dell'app
- Log in tempo reale:
  ```bash
  sudo journalctl -u roomctl.service -f
  ```
- Errori di rete verso dispositivi esterni (proiettore, DSP, relè) non impediscono di rispondere alla UI: se vedi HTTP 5xx o timeout verso `/api/...`, concentrati sulla rete locale tra Raspberry e dispositivi.

## 5) Conferma configurazioni copiate
- I file di default vengono copiati in `/opt/roomctl/config` alla prima esecuzione del deploy. Se la directory non esiste, ricrea i permessi e rilancia la copia:
  ```bash
  sudo install -d -o roomctl -g roomctl -m 0750 /opt/roomctl
  sudo rsync -a --delete --exclude ".venv" --exclude ".git" /percorso/sorgenti/ /opt/roomctl/
  sudo chown -R roomctl:roomctl /opt/roomctl
  ```
- Se cambi `config/devices.yaml` o `config/config.yaml`, riavvia il servizio per ricaricare:
  ```bash
  sudo systemctl restart roomctl.service
  ```

## 6) Disabilitare temporaneamente il servizio per test manuali
Per escludere problemi di systemd, puoi fermare il servizio e lanciare uvicorn a mano:
```bash
sudo systemctl stop roomctl.service
cd /opt/roomctl
sudo -u roomctl .venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload --proxy-headers
```
Se in questo modo la UI torna a funzionare, c'era un problema di configurazione del servizio (porta, variabile `ROOMCTL_BASE`, permessi).
