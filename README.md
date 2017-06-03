# traefik

Træfik, a modern reverse proxy

## Prefazione
Uno degli argomenti più discussi in ambito docker è la strategia da utilizzare per l'automatizzazione del  deploy delle proprie applicazioni, tra i vari step di un **deploy di un'applicazione web** vi è anche la necessità di esporla su internet mediante web server con l'ausilio all'occorrenza di un bilanciatore di nodi.

Per far fronte a questo problema incominciai ad usare [nginx-proxy](https://github.com/jwilder/nginx-proxy) che in combo al container [docker-letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion) permettevano l'uno di rimappare i virtual servers di nginx per esporre in automatico il sito internet col giusto dominio mentre l'altro richiedeva e rinnovava i certificati per l'https associandoli automaticamente.
Entrambi facevano ciò intercettando le chiamate di avvio e spegnimento dei container direttamente dal socket di docker che gli veniva opportunamente esposto dal sistema.

Di fatto questa soluzione si è rivelata ottima per diverso tempo ma negli ultimi mesi ho riscontrato diveri problemi o noie che mi hanno fatto propendere per cercare altre soluzioni.

A differenza di nginx-proxy [traefik](https://github.com/containous/traefik) svolge entrambe le funzioni di web server e di erogatore di certificati per l'https tramite [Let's Encrypt](https://letsencrypt.org/) in un unico container e ciò a vantaggio di velocità e stabilità, inoltre fornisce anche una comoda pagina web in grado di fornirci dettagli sempre aggiornati sulle connessioni in entrata e le route impostate con i container rilevati.
Inoltre per quanto possa sembrare più complicato da gestire traefik è molto semplice, imposteremo tutte le configurazioni di default nel file traefik.toml e dal docker-compose.yml ci limiteremo a definire ecceioni e parametri aggiuntivi della configurazione caso per caso.

## Configurazione

Nella [documentazione](https://docs.traefik.io/toml/) ufficiale di traefik è possibile trovare molteplici esempi adeguati alle casistiche più diffuse, qui sotto quello che ho scelto io.

Il mio obiettivo era quello di fargli ricevere sia le connessioni http (porta 80) che le connessioni https (porta 443) per poi forzare sempre a rispondere con l'https e sfruttare il certificato messomi a dispositione gratuitamente da Let's Encrypt. Inoltre mi interessava che gli eventuali container esposti senza un appropriato dominio venissero automaticamente mappati come sottodomini di quello generico del server.


```toml
logLevel = "DEBUG"
defaultEntryPoints = ["http", "https"]

# WEB interface of Traefik - it will show web page with overview of frontend and backend configurations 
[web]
address = ":8080"

# Connection to docker host system (docker.sock)
[docker]
domain = "domain.org"
watch = true
# This will hide all docker containers that don't have explicitly  
# set label to "enable"
exposedbydefault = false

# Force HTTPS
[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
  
# Let's encrypt configuration
[acme]
  email="your-email@gmail.com"
  storage="/etc/traefik/acme.json"
  entryPoint="https"
  acmeLogging=true
  onDemand=true
  OnHostRule=true
```

A questo punto basterà modificare la parte riguardante il dominio e l'email per la registrazione dei certificati.

### Il file toml nel dettaglio 

La prima parte definisce che tipologia di log che vogliamo stampato, in questo caso ho scelto **debug** per avere la maggior quantità di log possibile per vdere se tutto procede bene dall'output del processo quando lo richiamo con `docker-compose up`, mentre nel caso in cui avessi avviato il container con il comando docker-compose up -d, ovvevo in background, posso comunque accedere ai log mediante il comando `docker-compose logs`.
Subito dopo abbiamo **defaultEntryPoints** che definisce a quali tipologie di protoccli rispondere, in questo caso **http** e **https**.

```toml
logLevel = "DEBUG"
defaultEntryPoints = ["http", "https"]
```

Il seguente serve semplicemente ad abilitare l'**interfaccia web** per le statistiche, spesso è utile ma non indispensabile ed esporta verso l'esterno non è mai una scelta saggia.

```toml
# WEB interface of Traefik - it will show web page with overview of frontend and backend configurations 
[web]
address = ":8080"
```

A questo punto ci troviamo a configurare il dominio sotto la voce **domain** che verrà utilizzato come base per tutti i servizi web esposti.
Questo campo potrebbe anche essere lasciato vuoto in quanto è possibile definire i domini container per container nel docker-compose sotto forma di **label**.
Inoltre tramite il parametro **watch** diremo a traefik che dovrà controllare, tramite il socket di docker, quali container vengono eseguiti e quali spenti.
Per qual che riguarda il valore **exposebydefault** personalmente lo imposto su **false** questo perché diversamente rischierei una esposizione incontrollata di qualsiasi container inizializzato e francamente questa cosa non è molto conveniente. Preferisco di gran lunga definire caso per caso cosa esporre e cosa no.

Tanto per fare un esempio se il campo **domain** fosse popolato e avessimo instanziato un semplice progetto rails composto da due container, uno con rails che ovviamente andrebbe esposto ed un'altro con il database che ovviamente non va esposto, traefik genererebbe ben due route verso l'esterno una con il nome per esempio di app.nomeCartellaProgetto.domain.org e l'altra come db.nomeCartellaProgetto.domain.org.

```toml
# Connection to docker host system (docker.sock)
[docker]
domain = "domain.org"
watch = true
# This will hide all docker containers that don't have explicitly  
# set label to "enable"
exposedbydefault = false
```

La parte seguiente dice a traefik di ridirezionare forzatamente il traffico proveniente dalla porta 80 sulla 443 ovvero https, ovviamente la generazione del certificato come il suo rinnovo avviene in maniera automatica.

```toml
# Force HTTPS
[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
```

Ultima ma non meno importante la configurazione di **let's encrypt**.
Qui definiamo l'**email** per la quale saranno rilasciati i certificati, la posizione del file **acme.json** all'interno del quale traefik conserverà informazioni agli stessi, il punto di accesso consentito come **https**, logging ecc... 

```toml
# Let's encrypt configuration
[acme]
  email="your-email@gmail.com"
  storage="/etc/traefik/acme.json"
  entryPoint="https"
  acmeLogging=true
  onDemand=true
  OnHostRule=true
```

## Il file docker-compose.yml

Qui dibbiamo orchestrare il tutto in maniera tale che funzioni a puntino.
Prima di tutto impostiamo la versione due nel campo **version** all'inizio ed incominciamo e scrivere il servizio.

Il servizio proxy è basato sull'immagine di traefik che sicuramente utilizzarà come **entrypoint** l'eseguibile di traefik al quale passeremo direttamente i parametri tramite il paramentro ***command***. In questo caso stiamo rimandando alcuni comandi già prsenti nel file toml, ma quello che ci interessa veramente è **--docker**.

A questo punto bisogna **definire un network** che servirà a traefik per scoprire tutti i servizi istanziati ed esporli, in questo caso sto usando il network **webgateway** in fondo alla pagina con driver **bridge**.
*Tutti i container che faranno parte della rete webgateway saranno raggiunti da traefik e quindi esposti.*

Per quel che riguarda l'apertura delle porte abbiamo impostato traefik affinché accettasse le chiamate in **http** (80) rimandandole sull'**https** (443) quindi dobbiamo esporre entrambe le porte.
Poi considerando che abbiamo scelto di avviare l'**interfaccia web* dobbiamo aprire anche la porta 8080.

In ultimo bisogna fornire a traefik i file e le cartelle che gli servono per poter lavorare. Prima di tutto gli diamo **accesso al socket docker**,
 consecutivamente al **file di configurazione toml** creato poco fa ed in ultimo il file **acme.json** che *anche se vuoto* deve essere presente all'avvio del container, pena causare un errore che non farà funzionare traefik finché non risolto. 

 Infine la parte relativa allo stanziamento del network.

```yaml
version: '2'

services:
  proxy:
    image: traefik
    command: --web --docker --docker.domain=domain.org --logLevel=DEBUG
    networks:
      - webgateway
    ports:
      - "80:80"
      - "8080:8080"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/traefik.toml
      - ./acme.json:/etc/traefik/acme.json

networks:
  webgateway:
    driver: bridge
```

## Utilizzo
Lutilizzo è molto semplice in pratica basterà aggiungere in un qualsiasi docker-compose.yml le ultime tre righe viste prima corrispondenti alla definizione del network, in questo caso chiamato **webgateway** che ospita traefik. 
```yaml
networks:
  web:
    external:
      name: traefik_webgateway
```
Tutti i container che verranno aggiunti a questa rete saranno individuati da traefik e quindi mappati se necessario. In questo caso la proprietà da aggiungere è la seguente.
```yaml
  networks:
    - web
```
Ecco un esempio completo:
```yaml
version: "2"
services:
  whoami:
    image: emilevauge/whoami
    networks:
      - web
    labels:
      - "traefik.port=80"
      - "traefik.frontend.rule=Host:whoami.domain.org"

    
networks:
  web:
    external:
      name: traefik_webgateway
```

Una cosa che si nota fin da subito è la presenza della sezione label.
Questa sezione viene utilizzata da traefik per mappare quelle che possiamo definire impostazioni dinamiche sulle impostazioni di default sovrascrivendole per il container in uso.

In questo modo con **traefik.port=80** diciamo a traefik che la porta da utilizzare del container è la 80.

Allo stesso tempo definiamo mediante **traefik.frontend.rule=Host:whoami.domain.org** l'indirizzo internet al quale verrà mappato automaticamente il servizio e per il quale verrà generato e successivamente rinnovato nel tempo il certificato https per le connessioni sicure.

Se non avessimo impostato **traefik.frontend.rule=Host:whoami.domain.org** traefik avrebbe generato l'indirizzo automaticamente utilizzando il nome della cartella ed il nome del servizio. In questo caso se la cartella dove è posto il file docker-compose.yml si chiama traefik il dominio sarà traefik.whoami0.domain.com.

Se al contempo non avessimo voluto pubblicare il servizio su internet ma semplicemente esporlo nella stessa sottorete di altri servizi si sarebbe potuta utilizzare la label **traefik.enable=false**. In questo modo avremmo sovrascritto la regola di default che dice di mappare in automatico tutti i container avviati facenti parte del network  **traefik_webgateway**.



Una lista esaustiva delle label disponibili può essere trovata [qui](https://docs.traefik.io/toml/).

