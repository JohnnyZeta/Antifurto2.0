## La configurazione in ```apps.yaml```

Grazie alle [caratteristiche di AppDaemon](https://appdaemon.readthedocs.io/en/latest/APPGUIDE.html#passing-arguments-to-apps) ho potuto separare il codice vero e proprio dalla sua configurazione, rendendo l'App in questione sostanzialmente "indipendente" dal sistema dovre verrà eseguita. Gli argomenti passati al codice saranno quindi (da inserire nel mio caso nel file ```apps.yaml```): [^1]

```python3
Allarme:
  module: Allarme
  class: pyAllarme
  log: log_allarme
  indirizzo_ip: 192.168.1.2
  token: !secret token_appdaemon
  notifiche_ios: True
  sensore_sirena: binary_sensor.allarme_antifurto
  switch_antifurto: switch.antifurto_template
  rfid:
    lettore_rfid: input_text.rfid
    tag_abilitati:
      "Alberto": 11-11-11-11
      "Alessandra": 22-22-22-22
  gruppo_abitanti_casa: group.famiglia
  servizio_notify_push:
    - mobile_app_iphone_di_alberto
    - mobile_app_iphone_di_alessandra
  accendi_luci:
    elenco_luci:
      - light.bagno
      ...
  avviso_finestre:
    elenco_finestre:
      - binary_sensor.finestra_bagno
      ...
  allarme_dimenticato:
    minuti_attesa: 15
    avviso_notturno:
      ora_inizio: "22:30:00"
```

[^1]: Vi ricordo che i moduli ```rfid:```, ```accendi_luci:```, ```avviso_finestre:```, ```allarme_dimenticato:``` e ```avviso_notturno:```, il cui funzionamento vi spiegherò in seguito, andranno configurati con ```False``` nel caso si decida di non utilizzarli.

## Il file pyAllarme.py

Iniziamo dal file ```Allarme.py```, che contiene il codice vero e proprio interpretato da AppDaemon: vediamolo e commentiamolo un pezzo alla volta, così da chiarirne bene il funzionamento.

```python3
import hassapi as hass
import requests as req

class pyAllarme(hass.Hass):

    def initialize(self):
        self.log("Inizializzazione pyAllarme!")
        self.log("Inizializzazione pyAllarme!", log="main_log")
```

Partendo dalle prime righe notiamo l'aggiunta delle librerie ```hassapi``` e ```requests```: la prima è indispensabile per il funzionamento di AppDaemon, mentre la seconda la utilizzeremo per la generazione delle notifiche di tipo *actionable* e *critical* che approfondiremo in seguito.
Subito dopo troviamo la fase in cui dichiariamo la classe che sfrutteremo per la nostra App ```Allarme``` e definiamo la funzione di inizializzazione, indispensabile in ogni classe di AppDaemon. Nelle prime righe del file di configurazione dell'App ```Allarme``` ritroviamo i riferimenti che abbiamo appena citato:
- ```Allarme:``` indica il nome dell'App che siamo configurando;
- ```module: Allarme``` fa riferimento al file ```Allarme.py``` che contiene il codice della classe che utilizzeremo;
- ```class: pyAllarme``` specifica che utilizzeremo proprio al classe in questione, scritta nel file (e quindi nel modulo) appena citato.

Proseguiamo scrivendo nei log che è in corso l'inizializzazione della classe pyAllarme: sfruttiamo così un file di log specifico per questa App (```log_allarme.log``` appunto, inserito nella configurazione generale dell'App alla riga ```log: log_allarme```, che va creato prima dell'uso) dove si possano controllare tutti gli eventi collegati all'Antifurto 2.0. In questo caso il messaggio "Inizializzazione pyAllarme!" verrà scritto anche nel log dell'addon di AppDaemon su Home Assistant, che viene indicato di default come ```main_log```.


## Callbacks

### La fase di registrazione

Subito dopo la scrittura dei log inizia la fase di inizializzazione vera e propria, composta dalla registrazione delle funzioni di callback.

```python3
##### 1
self.listen_state(self.toggle_antifurto, self.args["switch_antifurto"])

##### 2
self.listen_state(self.sta_suonando, self.args["sensore_sirena"], new="on")

###### 3
if self.args["notifiche_ios"]:
    evento_notifica = "ios.notification_action_fired"
else:
    evento_notifica = "mobile_app_notification_action"
self.listen_event(self.gestisci_eventi, event=evento_notifica)
```

La funzione [```listen_state()```](https://appdaemon.readthedocs.io/en/stable/AD_API_REFERENCE.html#appdaemon.adapi.ADAPI.listen_state) è una delle più comunemente sfruttate in AppDaemon: permette infatti di registrare una funzione di [**callback**](https://it.wikipedia.org/wiki/Callback) (o di *richiamo*, in italiano) legandola al cambio di stato di un entità specificata.

La funzione di callback non è nient'altro che una funzione, appunto, che viene chiamata nel caso che l'entità osservata da ```listen_state``` cambi stato.
Prendiamo come esempio la parte di codice indicata con il numero 1: il callback è ```toggle_antifurto```, legato ai cambiamenti di stato dell'entità inserita in configurazione sotto ```switch_antifurto``` (nel mio caso ```switch.antifurto_template```, come abbiamo visto prima) e richiamata tramite ```self.args[]``` (che non fa nient'altro che andare a *pescare* nel file di configurazione dell'App il nome dell'entità che noi avremo specificato).

!!! info ""
    Come avremo modo di descrivere meglio [in seguito](App_Allarme.md#funzione-toggle_antifurto), nel momento in cui il callback viene *richiamato* gli vengono anche forniti una serie di informazioni su ciò che ha causato il suo richiamo: le potremo utilizzare per costruire la nostra logica con cui far lavorare AppDaemon.

Alla stessa maniera registriamo un callback ```sta_suonando``` relativamente all'entità specificata in configurazione sotto ```sensore_sirena``` (nel mio caso ```binary_sensor.allarme_antifurto```). In questo caso specifichiamo che il callback verrà chiamato *solo se* lo stato in cui il sensore sirena cambia sarà **on**: questa ulteriore specifica ci permette di procedere solo quando la sirena inizi a suonare, e non nel caso in cui abbia appena smesso (nuovo stato **off**).

La terza registrazione di questo blocco di codice permette di specificare un callback che utilizzeremo per gestire le notifiche Push provenienti dalle App di Home Assistant sui nostri cellulari. ```notifiche_ios``` è un parametro booleano presente in configurazione, che serve a specificare se intendiamo utilizzare per le notifiche telefoni iOS (```True```) o Android (```False```).

Incontriamo anche la funzione di AppDaemon ```listen_event```: permette di registrare un callback che verrà chiamato quando Home Assistant genererà un evento specifico. Nel nostro caso si tratta di un evento di tipo ```ios.notification_action_fired``` o ```mobile_app_notification_action``` nel caso in cui lo generi un'App ufficiale di HA installata rispettivamente su un dispositivo iOS o Android.


```python3
##### 4
if self.args["rfid"]:
    self.listen_state(self.gestisci_rfid, self.args["rfid"]["lettore_rfid"])

##### 5
if self.args["avviso_finestre"]:
    for finestra in self.args["avviso_finestre"]["elenco_finestre"]:
        self.listen_state(self.finestre, finestra, new="on")

##### 6
if self.args["allarme_dimenticato"]:
    durata_secondi = self.args["allarme_dimenticato"]["minuti_attesa"] * 60
    self.listen_state(self.dimenticanza, self.args["gruppo_abitanti_casa"], new="not_home", duration=durata_secondi)

    if self.args["allarme_dimenticato"]["avviso_notturno"]:
        orario = self.args["allarme_dimenticato"]["avviso_notturno"]["ora_inizio"]
        self.run_at(self.dimenticanzanotturna, start=orario)
```
Il secondo ed ultimo blocco della fase di ```initialize``` del codice contiene le ultime tre registrazioni che ci interessano.

La numero 4 verrà utilizzata per inserire e disinserire l'antifurto mediante chip RFID o NFC, passando dal callback ```gestisci_rfid```.[^2]

[^2]: Per la parte hardware ho utilizzato un [lettore PN532](https://cdn-shop.adafruit.com/datasheets/pn532ds.pdf) collegato ad una [Wemos D1 Mini](https://wiki.wemos.cc/products:d1:d1_mini) con installato un firmware generato da  [ESPHome](https://esphome.io/guides/getting_started_hassio.html), come vedremo nella descrizione della funzione appena citata.

La registrazione successiva ci permette di mostrare la possibilità di leggere un elenco di entità presenti nel file di configurazione, mediante un [ciclo for](https://appdaemon.readthedocs.io/en/3.0.5/APPGUIDE.html#passing-arguments-to-apps): essendo ```elenco_finestre``` annidato dentro le preferenze di ```allarme_dimenticato```, dobbiamo fornire a ```self.args[]``` il percorso corretto: ```self.args["avviso_finestre"]["elenco_finestre"]```. Per ogni ```finestra``` viene registrato un callback (```finestre```) che verrà chiamato ogni volta che il nuovo stato dell'entità sarà pari a **on**.

L'ultima registrazione viene usata per implementare gli avvisi che dovrebbero scattare nel caso in cui si dimentichi di inserire l'antifurto uscendo di casa, o la sera prima di andare a dormire. Nel primo caso è necessario inserire in configurazione il numero di minuti da attendere a *casa vuota* prima di chiamare il callback conseguente, mentre nel secondo l'orario al quale far generare la notifica.

!!! info "Casa vuota"
    Per l'informazione della *casa vuota* utilizziamo il [gruppo](https://www.home-assistant.io/integrations/group/#group-behavior) ```gruppo abitanti casa``` che contiene all'interno tutte le entità ```person``` o ```device_tracker``` di Home Assistant riferite alle persone che vivono con noi: questo tipo di gruppo assume lo stato di ```home``` se *almeno una* di queste risulta a casa, e ```not_home``` se *nessuna* di queste risulta a casa.

Le [specifiche](https://appdaemon.readthedocs.io/en/stable/AD_API_REFERENCE.html#appdaemon.adapi.ADAPI.listen_state) della funzione ```listen_state``` permettono l'uso di un parametro ```duration```, che indica la durata (in secondi) per cui deve essere mantenuto il nuovo stato che assume l'entità monitorata: nel nostro caso la funzione di callback ```dimenticanza``` verrà chiamata solo dopo un periodo di attesa pari a ```durata_secondi```, ottenuta moltiplicando ```minuti_attesa``` per 60.
La funzione di avviso notturno funziona invece impostando in configurazione un orario nel formato ```HH:MM:SS``` e sfruttando la [funzione di pianificazione](https://appdaemon.readthedocs.io/en/stable/AD_API_REFERENCE.html#appdaemon.adapi.ADAPI.run_at) AppDaemon ```run_at```, che chiamerà il callback ```dimenticanzanotturna``` ogni giorno all'orario specificato.

## Descrizione del funzionamento

I callbacks sono la parte di codice *operativa* che indica ad Home Assistant una serie di operazioni di effettuare, quali servizi chiamare e cosa notificare, per esempio.
La nostra attuale versione di pyAllarme ne sfrutta otto: iniziamo a vederli in dettaglio.

```python3
#####
def toggle_antifurto(self, entity, attribute, old, new, kwargs):
    if new == "on":
        self.log("Antifurto Inserito")
    elif new == "off":
        self.log("Antifurto Disinserito")

#####
def notifica_mobile(self, servizio_notify, tipo, titolo, messaggio):
    url = "http://" + str(self.args["indirizzo_ip"]) + ":8123/api/services/notify/" + servizio_notify
    headers = { "Authorization": "Bearer " + str(self.args["token"]), "content-type": "application/json", }
    if tipo == "actionable":
        data_json = '{"message":"' + messaggio + '","title":"' + titolo + '","data":{"push":{"category":"allarme"}}}'
    elif tipo == "critical":
        data_json = '{"message":"' + messaggio + '","title":"' + titolo + '","data":{"push":{"sound":{"name":"default","critical":"1","volume":"1.0"}}}}'
    elif tipo =="mix":
        data_json = '{"message":"' + messaggio + '","title":"' + titolo + '","data":{"push":{"category":"allarme","sound":{"name":"default","critical":"1","volume":"1.0"}}}}'
    req.post(url, headers=headers, data=data_json)

#####
def gestisci_eventi(self, event_name, data, kwargs):

    # iOS
    if self.args["notifiche_ios"]:
        if data["actionName"] == "ALLARME_ON":
            self.log("Attivo allarme da " + str(data["sourceDeviceName"]))
            if self.get_state(self.args["switch_antifurto"]) == "off":
                self.call_service("switch/turn_on", entity_id=self.args["switch_antifurto"])
        elif data["actionName"] == "ALLARME_OFF":
            self.log("Disattivo allarme da " + str(data["sourceDeviceName"]))
            if self.get_state(self.args["switch_antifurto"]) == "on":
                self.call_service("switch/turn_off", entity_id=self.args["switch_antifurto"])
            
    # Android
    else:
        if data["action"] == "ALLARME_ON":
            self.log("Attivo allarme")
            if self.get_state(self.args["switch_antifurto"]) == "off":
                self.call_service("switch/turn_on", entity_id=self.args["switch_antifurto"])
        elif data["action"] == "ALLARME_OFF":
            self.log("Disattivo allarme")
            if self.get_state(self.args["switch_antifurto"]) == "on":
                self.call_service("switch/turn_off", entity_id=self.args["switch_antifurto"])
```

### Funzione ```toggle_antifurto```

Il primo che incontriamo è un callback molto semplice: lo scopo di ```toggle_antifurto``` è quello di scrivere nel log della App quando l'antifurto viene inserito o disinserito, ai fini di debug.
Come anticipato [in precedenza](App_Allarme.md#la-fase-di-registrazione), un callback che viene chiamato da una funzione ```listen_state``` (come specificato nella [documentazione](https://appdaemon.readthedocs.io/en/latest/APPGUIDE.html#state-callbacks)) ha a disposizione una serie di argomenti (```entity, attribute, old, new```) da utilizzare nel suo codice. In questo caso abbiamo sfruttato il parametro ```new```, che ci fornisce il nuovo stato che assume l'entità dopo il cambio: essendoci *abbonati* all'entità specificata in ```toggle_antifurto``` il ciclo if ci permette di loggare due messaggi differenti, a seconda che il nuovo stato sia **on** o **off**.

### Funzione ```notifica_mobile```

La seconda funzione che incontriamo sfrutta le caratteristiche della libreria Python [Request](https://requests.readthedocs.io/projects/it/it/latest/#), che permette di semplificare molto l'uso di chiamate [RESTful](https://it.wikipedia.org/wiki/Representational_State_Transfer) (e per questo motivo è tra le librerie più utilizzate in assoluto).

??? tip "Addon"
    Se abbiamo installato AppDaemon dall'addon della Community di Home Assistant ricordiamoci di inserire ```Requests``` nella configurazione dell'addon, sotto ```python_packages``` per permetterne l'installazione.

Non si tratta di un callback in questo caso, ma di una funzione che scriviamo per facilitarci la successiva gestione del meccanismo di notifica sulle App di Home Assistant su iOS ed Android. ```notifica_mobile```, infatti, accetta ```servizio_notify``` e ```tipo``` come argomenti: il primo ci serve per specificare su quale dispositivo inviare la notifica (basandoci sul servizio ```notify``` che crea Home Assistant per ogni modalità di notifica aggiunta), mentre il secondo è usato per scegliere che tipo di notifica inviare. Per quanto riguarda iOS, Apple permette di utilizzarne due tipi *speciali*:

Tipo          | Caratteristiche
--------------|----------------
**Actionable** | permettono di sfruttare fino a quattro *pulsanti* che appaiono sotto la notifica per compiere una serie di azioni da noi specificate, o di inserire del testo in una maschera;
**Critical** | introdotte con iOS 12, sono notifiche considerate prioritarie: per questo motivo appaiono sempre in cime alle altre, e riproducono il loro suono anche se il telefono è in modalità sileziosa.

!!! info ""
    Sfruttare una notifica *actionable* implica una configurazione ulteriore che più essere fatta sia a livello del file ```configuration.yaml``` sia attraverso l'App ufficiale di Home Assistant: in ogni caso vi consiglio una lettura della [pagina della documentazione di HA](https://companion.home-assistant.io/docs/notifications/actionable-notifications) per i passaggi esatti con cui procedere.

Nel mio caso sono passato dal file di configurazione:
```yaml
ios:
  push:
    categories:
      - name: Allarme
        identifier: 'allarme'
        actions:
          - identifier: 'ALLARME_ON'
            title: 'Allarme ON'
            activationMode: 'background'
            authenticationRequired: true
            destructive: false
            behavior: 'default'
          - identifier: 'ALLARME_OFF'
            title: 'Allarme OFF'
            activationMode: 'background'
            authenticationRequired: true
            destructive: true
            behavior: 'default'
```

Lo scopo del codice era originariamente quello di poter sfruttare le notifiche *actionable* per inserire e disinserire l'antifurto, il suonare della sirena (e l'apertura finestre ad antifurto inserito) segnalato invece con notifiche *critical*.

La funzione ```notifica_mobile``` va oltre, e permette di utilizzare anche una notifica di tipo *mix*: una notifica che unisce le caratteristiche di entrambe (*actionable* + *critical*). Per sfruttare questa funzione sarà necessario conoscere l'indirizzo ip dell'istanza di Home Assistant, e generare un *token* (Profilo -> Token di accesso a lunga vita) che dovrà essere scritto nel file di configurazione sotto ```token```, o nel file ```secret.yaml``` di Home Assistant e richiamato con ```!secret token_appdaemon``` come nel caso indicato sopra.

### Funzione ```gestisci_eventi```

Allo schiacciare dei bottoni che appiono sotto delle notifiche di tipo *actionable* viene generato un evento specifico nel Bus eventi di Home Assistant, del tipo [già citato](App_Allarme.md#la-fase-di-registrazione). Lo scopo della funzione ```gestisci_eventi``` è quello di interpretare l'evento in arrivo, ed eseguire istruzioni di conseguenza.

La funzione è divisa in due parti: una è relativa ai dispositivi iOS, e l'altra a quelli Android. Il ```payload``` proveniente dall'evento cambia infatti a seconda dell'App che l'ha generato, sia in struttura che in contenuto: la decodifica di questi dati avviene tramite un paio di cicli if, e sfrutta [la funzione di AppDaemon](https://appdaemon.readthedocs.io/en/latest/AD_API_REFERENCE.html#appdaemon.adapi.ADAPI.call_service) ```call_service```, che permette di chiamare il servizio ```turn_on``` o ```turn_off``` sullo switch che controlla l'antifurto.

```python3
#####
def sta_suonando(self, entity, attribute, old, new, kwargs):
    self.log("Allarme! La sirena sta suonando!", log="main_log", level="WARNING")
    self.log("Allarme! La sirena sta suonando!", level="WARNING")
    if self.sun_down():
            if self.args["accendi_luci"]:
                for luce in self.args["accendi_luci"]["elenco_luci"]:
                    self.turn_on(luce)
                self.log("Allarme Sirena! Accese tutte le luci")
                # accendi tutte le luci di casa se suona l'allarme di notte
    for servizio_notify in self.args["servizio_notify_push"]:
        self.notifica_mobile(servizio_notify, "mix", "Antifurto", "Sirena attivata!!")
        
#####
def finestre(self, entity, attribute, old, new, kwargs):
    allarme_inserito = self.get_state(self.args["switch_antifurto"], default="off")
    if allarme_inserito == "on":
        self.log("Avviso finestre!")
        nome = self.get_state(entity, attribute="friendly_name")
        messaggio = "Aperta " + str(nome) + " con Antifurto inserito!"
        for servizio_notify in self.args["servizio_notify_push"]:
            self.notifica_mobile(servizio_notify, "mix", "Antifurto", messaggio)
    
#####
def dimenticanza(self, entity, attribute, old, new, kwargs):
    if self.get_state(self.args["switch_antifurto"]) == "off":
        messaggio = "Sono passati " + str(self.args["allarme_dimenticato"]["minuti_attesa"]) + " minuti con nessuno a casa. Vuoi attivare l'antifurto?"
        self.notifica_mobile(next(iter(self.args["servizio_notify_push"])), "actionable", "Antifurto", messaggio)
        # manda la notifica solo al primo del dizionario
        
#####
def dimenticanzanotturna(self, kwargs):
    if self.get_state(self.args["switch_antifurto"]) == "off":
        self.log("Avviso dimenticanza notturna antifurto")
        messaggio2 = "Sono le " + str(self.args["allarme_dimenticato"]["avviso_notturno"]["ora_inizio"]) + ". Vuoi attivare l'antifurto?"
        self.notifica_mobile(next(iter(self.args["servizio_notify_push"])), "actionable", "Antifurto", messaggio2)
        
#####
def gestisci_rfid(self, entity, attribute, old, new, kwargs):
    for utente, tag in self.args["rfid"]["tag_abilitati"].items():
        if self.get_state(self.args["rfid"]["lettore_rfid"]) == tag:
            if self.get_state(self.args["switch_antifurto"]) == "off":
                self.call_service("switch/turn_on", entity_id=self.args["switch_antifurto"])
                self.log("Attivato Antifurto da tag RFID di " + str(utente))
            elif self.get_state(self.args["switch_antifurto"]) == "on":
                self.call_service("switch/turn_off", entity_id=self.args["switch_antifurto"])
                self.log("Disattivato Antifurto da tag RFID di " + str(utente))
```

### Funzione ```sta_suonando```

Lo scopo di questo callback è semplice: nel caso che l'antifurto stia effettivamente suonando (informazione che arriva da ```sensore_sirena```, cioè ```binary_sensor.allarme_antifurto``` nel mio caso) la funzione:

- scrive l'evento in entrambi i log presenti (dell'addon e specifico dell'App Allarme);
- se il sole è tramontato accende tutte le luci elencate in ```elenco_luci```;
- causa l'invio di una notifica di tipo *mix* in ciascuno dei servizi di notifica elencati in ```servizio_notify_push```, con titolo "Antifurto" e messaggio "Sirena attivata!!".

Se usiamo abitualmente scene con luci ad intensità variabile, potrebbe essere comodo sfruttare [la possibilità](https://appdaemon.readthedocs.io/en/stable/HASS_API_REFERENCE.html#appdaemon.plugins.hass.hassapi.Hass.turn_on) della funzione ```turn_on``` di accettare parametri: potremmo ad esempio impostare un valore di ```brightness_pct``` per specificare con quale luminosità percentuale accendere le luci (```self.turn_on(luce, brightness_pct=100)```).

### Funzione ```finestre```

!!! failure ""
    La maggior parte dei (rari) falsi allarmi scattati in casa sono dovuti alla dimenticanza.

Questa funzione cerca di ridurre questa possibilità, notificando sul mio telefono e quello della mia compagna se si apre una finestra (o porta-finestra) ad antifurto inserito.[^3]

[^3]: Chiaramente questa *feature* ha senso solo se i sensori dell'antifurto sono montati all'esterno dell'abitazione, e cerca di sfruttare il tempo necessario ad aprire le persiane o le tapparelle montate all'esterno.

Il messaggio inviato da questa funzione contiene al suo interno il ```friendly_name``` della finestra che ne ha causato l'invio, sfruttando il parametro ```attribute``` della funzione [```get_state```](https://appdaemon.readthedocs.io/en/stable/AD_API_REFERENCE.html#appdaemon.adapi.ADAPI.get_state): è bene ricordare che il parametro ```entity``` passato al callback in questo caso equivale al classico ```entity_id``` utilizzato da Home Assistant.

### Funzioni ```dimenticanza``` e ```dimenticanzanotturna```

La prima funzione permette di unire la *presence detection* di Home Assistant al nostro impianto di antifurto. AppDaemon possiede [diverse funzioni](https://appdaemon.readthedocs.io/en/latest/HASS_API_REFERENCE.html#presence) che lavorano su questo argomento, permettendo un controllo molto sofisticato se fosse necessario. Nel nostro caso (come spiegato in precendenza) sfruttiamo un gruppo creato *ad hoc*, perché di rapida e semplice implementazione.

Una novità che incontriamo qui è l'uso di ```next(iter(self.args["servizio_notify_push"]))```. Questo codice consegnerà al callback solo il *primo* dei servizi di notifica elencati in ```self.args["servizio_notify_push"]```, consentendo di stabilire una sorta di *priorità* all'interno dell'elenco. Questa necessità nasce dal fatto che non è necessario che tutti coloro ai quali si dovrebbe notificare il suonare della sirena debbano anche poter intervenire nel caso ci fossimo dimenticati di inserirlo.

!!! info ""
    Nel mio caso essendo ```mobile_app_iphone_di_alberto``` il primo della lista, solo a me verranno notificate anche le dimenticanze (ed essendo una notifica *actionable* potrò inserire l'antifurto immediatamente), mentre a ```mobile_app_iphone_di_alessandra```  verrà notificato solo il resto.

Se la funzione ```dimenticanza``` si lega alla nostra presenza, la funzione ```dimenticanzanotturna``` lo fa solo all'orario che abbiamo scritto in configurazione: nell'ipotesi in cui fossimo in vacanze e non avessimo inserito l'antifurto, questo ci verrebbe notificato tutte le sere.

### Funzione ```gestisci_rfid```

Per come ho configurato il mio sistema di lettura dei tag RFID, ho a disposizione su Home Assistant un ```input_text.rfid``` in cui viene scritto il codice identificativo del tag che viene letto. La funzione ```gestisci_rfid``` legge il valore di questa entità e la confronta con i tag specificati in configurazione come ```"nome": codice_identificativo```: se c'è una corrispondenza attiva o disattiva l'antifurto loggando chi ha effettuato il cambio di stato.

## Conclusioni

Mi sembra di aver mostrato le potenzialità di AppDaemon nell'integrare le funzioni di un classico sistema di allarme con Home Assistant. La scrittura delle funzioni che vi ho descritto nasce dalle mie particolari esigenze, e sono sicuro che le vostre non siano identiche: ciò che vi ho spiegato dovrebbe aiutarvi a colmare questo *gap* e scrivere la vostra versione.

Vi ricordo di consultare la [documentazione](https://appdaemon.readthedocs.io/en/latest/) del progetto per ottenere ulteriori spunti, anche tramite i numerosi esempi che lì sono riportati. Per quanto riguarda la possibilità di integrare AppDaemon ad altri sistemi diversi da Home Assistant, vi consiglio di dare un'occhiata alla [sezione](https://appdaemon.readthedocs.io/en/latest/MQTT_API_REFERENCE.html) relativa al plugin MQTT, uno standard *de facto* nel settore della domotica.

Grazie per aver letto fino a qui e buon lavoro! Vi meritate una foto di un gattino per la pazienza.

??? quote "Miao."
    ![gattino](gattino.jpg)

## Breve sitografia

- [AppDaemon addon per Home Assistant](https://github.com/hassio-addons/addon-appdaemon)
- [Homeassistant, Actionable Notifications und Appdaemon](https://www.triumvirat.org/2020/02/09/hass-notifications-appdaemon/)
- [REST post from Python](https://community.home-assistant.io/t/rest-post-from-python/18329)
- [Libreria Requests](https://requests.readthedocs.io/projects/it/it/latest/#)