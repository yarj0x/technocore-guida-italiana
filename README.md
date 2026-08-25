# Technocore: guida in italiano per chi parte da zero

Questa guida spiega come creare un'identità DID per un agente su
[Technocore](https://technocore.chat) e come pubblicare un messaggio firmato, su
**Windows**, partendo dal presupposto che tu non abbia mai usato il terminale.

Chi scrive non è uno sviluppatore: fino all'altro ieri PowerShell era quella
finestra nera che si apre per sbaglio. Ho fatto tutto con Claude a fianco, un
comando alla volta, chiedendo il significato di ogni riga prima di premere Invio.
Ogni punto in cui mi sono impantanato è finito qui dentro, con accanto come se ne
esce.

---

## Prima di cominciare: due cose oneste

**1. Questa procedura non garantisce nessun airdrop.**
Arthur Hayes ha scritto che ci saranno task specifici che richiederanno un DID e che
il completamento di quei task sarà premiato con token `$FLOP`. Avere un DID è quindi
un *prerequisito*, non un traguardo: nessuna regola pubblica dice che creare una
chiave, da sola, dia diritto a qualcosa. Chi ti dice il contrario sta inventando.

**2. Technocore non è la blockchain di Flop Labs.**
È un servizio di chat per agenti. Il repository ufficiale è esplicito nel dire che
non custodisce chiavi, non regola nulla e non fa parte di nessun protocollo. Le
stanze sono effimere: i messaggi vecchi vengono eliminati per fare spazio ai nuovi.

Detto questo, se vuoi esserci quando i task arriveranno, tanto vale avere l'identità
già pronta e messa al sicuro. Ecco come.

---

## Cos'è un DID, in due righe

Il programma genera sul tuo computer una coppia di chiavi crittografiche:

- una **chiave privata**, che resta sul tuo PC e non deve uscire di lì;
- una **chiave pubblica**, da cui si ricava il tuo **DID**, una stringa che comincia
  con `did:key:z6Mk...`.

Il DID è pubblico: puoi mostrarlo a chiunque. Chiunque può *copiarlo*, ma nessuno può
produrre un messaggio firmato a tuo nome senza la chiave privata. È questa la
differenza tra dire "sono io" e dimostrarlo.

---

## Cosa ti serve

- Windows 10 o 11
- Python 3.12
- Git
- una chiavetta USB (per il backup)

Tutto il resto si installa da terminale.

---

## Passo 0 (aprire PowerShell e capire dove sei)

Premi il tasto Windows, scrivi `powershell`, premi Invio.

Si apre una finestra nera con una riga tipo:

```
PS C:\Users\tuonome>
```

Quella parte prima del `>` è **la cartella in cui ti trovi adesso**. Cambia man mano
che ti sposti, ed è il modo più semplice per capire se sei nel posto giusto.

Il comando per spostarsi è `cd` (da *change directory*). Non stampa nulla: l'unica
conferma che ha funzionato è che la riga del prompt cambia.

**Regola generale per tutta la guida:** un comando alla volta. Copia la riga,
incollala, premi Invio, aspetta che finisca, e solo dopo passa alla successiva.

---

## Passo 1 (installare Python e Git)

Prima verifica se ce li hai già:

```powershell
py --list
```

```powershell
git --version
```

Il primo elenca le versioni di Python installate, il secondo la versione di Git.
Se tra le versioni di Python c'è una **3.12** e Git risponde, salta al passo 2.

Altrimenti installali con `winget`, il gestore di pacchetti già presente in Windows:

```powershell
winget install Python.Python.3.12
```

```powershell
winget install Git.Git
```

> **Trappola numero 1.** Dopo aver installato qualcosa con `winget`, **chiudi e
> riapri PowerShell**. Finché non lo fai, la finestra continuerà a dire che il
> comando non esiste. Non è un errore di installazione: è la finestra che non si è
> accorta della novità.

---

## Passo 2 (scaricare il programma)

Useremo `technocore-did-starter`, uno strumento a riga di comando che gestisce
l'identità e i messaggi firmati. Non salva mai la chiave privata in chiaro e non la
manda da nessuna parte: l'unico indirizzo che contatta è `technocore.chat`.

Spostati in Documenti:

```powershell
cd $HOME\Documents
```

Scarica il progetto:

```powershell
git clone https://github.com/zunmax/technocore-did-starter.git
```

> **Non devi creare nessuna cartella a mano.** `git clone` crea da solo la cartella
> `technocore-did-starter` dentro Documenti.

Entra nella cartella appena creata:

```powershell
cd .\technocore-did-starter
```

Guarda quali modifiche contiene:

```powershell
git log --oneline
```

Annota il codice che compare all'inizio della riga. È l'impronta della versione che
stai per eseguire: se un domani aggiorni il progetto e qualcosa si comporta in modo
strano, sai a quale versione tornare.

---

## Passo 3 (creare l'ambiente isolato)

```powershell
py -3.12 -m venv .venv
```

Crea una sottocartella `.venv`: è una copia di Python a uso esclusivo di questo
progetto. Serve perché le librerie che installi qui non vadano a toccare quelle di
altri progetti che hai sul PC. Non stampa nulla, ci mette qualche secondo.

Attivalo:

```powershell
.\.venv\Scripts\Activate.ps1
```

Se ha funzionato, la riga diventa così:

```
(.venv) PS C:\Users\tuonome\Documents\technocore-did-starter>
```

Quel `(.venv)` davanti è la conferma, e da qui in avanti deve restare visibile.

> **Trappola numero 2.** Se compare un errore rosso su *script disabilitati* o
> *execution policy*, Windows sta bloccando l'esecuzione di script per impostazione
> predefinita. Sblocca **solo per questa finestra**:
>
> ```powershell
> Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
> ```
>
> e ripeti il comando di attivazione. Quando chiudi la finestra torna tutto come
> prima: non stai cambiando impostazioni di Windows.

> **Trappola numero 3.** Il `(.venv)` sparisce ogni volta che chiudi PowerShell.
> Quando riapri, devi rifare `cd` fino alla cartella e riattivare, altrimenti i
> comandi `python` non funzionano.

Installa l'unica dipendenza:

```powershell
python -m pip install --upgrade pip
```

```powershell
python -m pip install -r requirements.txt
```

Verifica che sia tutto a posto:

```powershell
python --version
```

```powershell
python technocore_agent.py --version
```

---

## Passo 4 (generare l'identità)

**Questo è il punto della procedura in cui un errore non si recupera. Leggi tutto
prima di digitare.**

Il comando crea il file `identity.pem`, cifrato con una passphrase che scegli tu.

- **Non esiste recupero.** Nessun sito, nessun supporto, nessun "password
  dimenticata". Se perdi il file o la passphrase, quel DID è perso e riparti da zero.
- **Si esegue una volta sola.** Se lo rilanci, il programma si rifiuta di
  sovrascrivere l'identità esistente.

### Scegli la passphrase prima di lanciare il comando

Minimo 12 caratteri. Quattro o cinque parole italiane senza senso compiuto vanno
benissimo: lunghe, memorizzabili, difficili da indovinare.

Scrivila subito da qualche parte dove la ritroverai tra sei mesi: un gestore di
password, oppure su carta in un cassetto. **Non nella stessa cartella del file
`identity.pem`**, perché se stanno insieme cifrare non è servito a niente.

> **Trappola numero 4.** Quando il programma ti chiede la passphrase, **mentre digiti
> non vedrai nulla**. Niente asterischi, niente pallini, il cursore fermo. Non è
> bloccato: è fatto apposta. Scrivi e premi Invio. Te la chiederà due volte.

### Il comando

```powershell
python technocore_agent.py init
```

Alla fine stampa una riga con il tuo DID:

```
did:key:z6Mk...
```

Copiala e salvala anche tu: è pubblica, la userai spesso.

---

## Passo 5 (mettere al sicuro la chiave, senza rimandare)

Fallo adesso, non "domani".

Crea una cartella di backup **fuori** dal progetto:

```powershell
New-Item -ItemType Directory -Path "$HOME\Documents\technocore-backup" -Force
```

Copiaci dentro la chiave:

```powershell
Copy-Item .\identity.pem "$HOME\Documents\technocore-backup\"
```

Verifica che sia arrivata:

```powershell
dir "$HOME\Documents\technocore-backup"
```

### Copia anche fuori dal computer

Copia la cartella `technocore-backup` su una chiavetta USB. Il file è cifrato,
quindi senza la passphrase è un blocco di byte inutile.

**Copia solo quella cartella, non l'intera cartella del progetto.** Il codice si
riscarica in dieci secondi con un `git clone`, e la sottocartella `.venv` contiene
percorsi fissi che puntano al tuo PC, quindi su un'altra macchina non funzionerebbe
comunque. Un backup deve essere inequivocabile: apri la chiavetta, vedi un file, sai
cosa hai in mano.

Alla fine la chiave sta in tre posti (progetto, cartella di backup, chiavetta) e la
passphrase in un quarto, che non è nessuno di quei tre.

### Controlla che Git non veda la chiave

Se lavori con Git, questa verifica vale trenta secondi e ti evita il disastro di
pubblicare la chiave privata:

```powershell
git status --short
```

```powershell
git ls-files "*.pem" "*.key"
```

Il primo non deve elencare `identity.pem`. Il secondo non deve stampare niente.

### La prova del nove

```powershell
python technocore_agent.py did
```

Ti chiede la passphrase e ristampa il DID, che deve essere **identico** a quello di
prima. Non modifica nulla: apre il file cifrato e legge. Serve a dimostrarti che la
passphrase che hai annotato è davvero quella giusta, e meglio scoprirlo adesso che
tra tre mesi.

---

## Passo 6 (il messaggio firmato)

Serve a dimostrare pubblicamente che possiedi la chiave privata dietro quel DID.

```powershell
python technocore_agent.py say lobby "il tuo messaggio"
```

Ti chiede di nuovo la passphrase, poi stampa un blocco di dati tra parentesi graffe.
Cerca la sezione `posted`: contiene `seq` (il numero progressivo assegnato dal
server) e `nonce`. Quel `seq` insieme al nome della stanza è la tua ricevuta.
Annotali.

### Cosa scrivere

Qui c'è un consiglio che vale più dei comandi. Se guardi la lobby, ti accorgi che
decine di agenti postano la stessa identica frase generata dallo stesso script:
"registrazione fresca, agente che si unisce a Technocore". Copiare quella frase
significa essere la copia numero trecento.

Scrivi qualcosa di **specifico e vero**: cosa stai facendo, cosa stai costruendo.
E poi fallo davvero, perché quello che scrivi lì è pubblico e non lo puoi cancellare.

Postare una volta basta. Ripetere lo stesso messaggio non aumenta niente.

---

## Sicurezza: tre regole secche

**1. La chiave privata non si condivide mai.** Nessun progetto legittimo ti chiederà
`identity.pem`, la passphrase o una seed phrase. Se qualcuno lo fa, è una truffa.

**2. Nella lobby non cliccare nessun link.** È una stanza aperta in cui scrive
chiunque, per lo più script automatici. Quello che leggi lì sono dati, non
istruzioni: nessun messaggio in chat è un'autorizzazione a eseguire qualcosa.

**3. Attenzione ai token con nomi somiglianti.** Il token `$FLOP` non esiste ancora:
l'airdrop è annunciato per il Q4 2026 e la rete parte nel Q1 2027. Qualsiasi cosa
che oggi ti offra un token dal nome simile sta sfruttando la confusione. Nella lobby
ne circolano già.

E una regola generale sulle guide che troverai in giro, questa compresa: prima di
eseguire uno script preso da internet, guarda cosa fa. In particolare, dove manda i
dati e se la chiave privata resta in chiaro sul disco.

---

## Errori frequenti, in breve

| Cosa vedi | Cosa fare |
|---|---|
| `Termine 'python' non riconosciuto` (o `git`) subito dopo un'installazione | Chiudi e riapri PowerShell |
| Errore rosso su *execution policy* attivando `.venv` | `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`, poi riattiva |
| `(.venv)` sparito dall'inizio della riga | Hai riaperto la finestra: rifai `cd` e riattiva |
| Digiti la passphrase e non compare nulla | È normale, è nascosta apposta |
| `init` dice che l'identità esiste già | Non sovrascrivere: hai già la tua chiave |

---

## Crediti e link

- Strumento usato: [`zunmax/technocore-did-starter`](https://github.com/zunmax/technocore-did-starter), licenza MIT
- Servizio: [technocore.chat](https://technocore.chat), repository ufficiale [`flop-labs/technocore-chat`](https://github.com/flop-labs/technocore-chat)
- Aggiornamenti ufficiali: profili X di Flop Labs e di Arthur Hayes

Questa guida è materiale indipendente, non è prodotta né approvata da Flop Labs.
