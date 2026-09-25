# Istruzioni per il Deployment Automatico sul NAS (nas.edoardosarri.com)

Questo progetto include un flusso di Continuous Deployment tramite **GitHub Actions** (`.github/workflows/deploy.yml`).
A ogni `git push` sul branch `main`, GitHub si collegherà automaticamente via SSH al tuo NAS, scaricherà le novità ed eseguirà `docker compose up -d`.

---

## 1. Configurazione della chiave SSH

Per consentire a GitHub di collegarsi in modo sicuro senza password:

### A. Genera la chiave SSH (dal tuo Mac o dal NAS)
Esegui sul terminale del tuo computer:
```bash
ssh-keygen -t ed25519 -C "github-actions-nas" -f ~/.ssh/nas_deploy_key
```
Questo comando genererà due file nella cartella `~/.ssh/`:
1. `nas_deploy_key` (Chiave **privata** -> andrà su GitHub)
2. `nas_deploy_key.pub` (Chiave **pubblica** -> andrà sul NAS)

### B. Installa la chiave pubblica sul NAS
Copia la chiave pubblica sul tuo NAS con il comando:
```bash
ssh-copy-id -i ~/.ssh/nas_deploy_key.pub -p <PORTA_SSH> <UTENTE_NAS>@nas.edoardosarri.com
```
*(In alternativa, apri il file `nas_deploy_key.pub`, copia la riga di testo e incollala in fondo a `~/.ssh/authorized_keys` del tuo utente sul NAS).*

### C. Verifica la connessione
Testa l'accesso da terminale:
```bash
ssh -i ~/.ssh/nas_deploy_key -p <PORTA_SSH> <UTENTE_NAS>@nas.edoardosarri.com
```
Se entri senza richiesta di password, l'autenticazione è pronta!

---

## 2. Inserimento dei Secret su GitHub

Vai sulla pagina del tuo repository su GitHub:
1. Clicca su **Settings** (ingranaggio in alto a destra).
2. Nel menu a sinistra, seleziona **Secrets and variables** -> **Actions**.
3. Clicca sul pulsante verde **New repository secret** e inserisci le seguenti 5 variabili:

| Nome Secret | Valore Esempio | Descrizione |
| :--- | :--- | :--- |
| `NAS_HOST` | `nas.edoardosarri.com` | Il dominio o IP per raggiungere il NAS |
| `NAS_PORT` | `22` (o porta custom) | La porta SSH aperta sul router per il NAS |
| `NAS_USER` | `edoardo` (o il tuo utente) | Il tuo utente SSH sul NAS (con permessi Docker) |
| `NAS_SSH_KEY` | *(Contenuto di `nas_deploy_key`)* | L'intera chiave privata (incluso `-----BEGIN...` e `-----END...`) |
| `NAS_PROJECT_PATH` | `/volume1/docker/face-recognition-allarm-system` | Il percorso esatto in cui hai clonato la cartella sul NAS |

> [!NOTE]
> Per trovare il valore di `NAS_PROJECT_PATH`, collegati in SSH al NAS, entra nella cartella del progetto ed esegui `pwd`.

---

## 3. Primo Clone sul NAS

Se non hai ancora clonato il repository sul NAS:
```bash
cd /percorso/desiderato/sul/nas
git clone https://github.com/edoardosarri24/face-recognition-allarm-system.git
cd face-recognition-allarm-system
```

Assicurati che l'utente che esegue i comandi appartenga al gruppo `docker` sul NAS, in modo da poter eseguire `docker compose` senza richiedere `sudo`:
```bash
# Esempio su Linux/UGREEN (se necessario):
sudo usermod -aG docker $USER
```

---

## 4. Verifica e Funzionamento

D'ora in poi, ogni volta che farai:
```bash
git add .
git commit -m "Aggiornamento configurazione"
git push origin main
```
1. Nella scheda **Actions** del repository su GitHub vedrai avviarsi il workflow `Deploy to NAS`.
2. Il runner si collegherà a `nas.edoardosarri.com`.
3. Eseguirà `git pull origin main` per aggiornare i file (inclusi quelli in `configuration/`).
4. Eseguirà `docker compose up -d --remove-orphans`, che riavvierà automaticamente **soltanto i container le cui configurazioni o file sono stati modificati**.
