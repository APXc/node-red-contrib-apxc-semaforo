# SEMAFORO Globale Multi Istanza in Backup
Nodo che Gestisce un oggetto o liste di oggeti,
con dei valori da utilizzare e variare in maniera
trasfersale al singolo flusso, 
il tutto mantenendo un backUp dei valori in caso
di blocco/recupero


## Propietà Generali
 -  **stoplight**
     Stato Monitoraggio Semaforo 
 - **Road**
    Topic di Riferimento per l'istanza corrente
 - **TypeReq**
     il tipo di richiesta utilizzata
 - **PropsType**
     Tipologia di Oggetto da Gesite, SingleValue(a Oggetto Singolo), ListMultiObject (a multi Oggetto a struttura similare)
 - **NamePropsID** 
     nome della Proprieta che identifica l'oggeto/valore da aggiornare,
 - **NameAdditionaListProp** 
     Oggetto con le Proprietà Aggiuntive da Aggiorare
 - **BackupName** 
     Nome Assegnato all'istanza di backUp
 - **Overwrite**
     Sovrascrittura dei valori in corso

### Stoplight
 Propietà che definisce se il Semaforo Monitorerà lo status (Ciclo Chiuso) o lo ignorera (Ciclo Aperto e Infinito) 

### Road 
 stinga con la quale si identifichera il context da utilizare
 per i valori, cosi da poter mantere anche piu istanze con oggeti differenti da monitorare

### TypeReq
 il Tipo di Richiesto può avere i seguenti valori:
 - **INIT** - Inizializatore, obbligatorio nel flusso di instanza per consentire il correto funzionamento del tutto,
 - **GET** - Recupero dei valori attuali
 - **UPD** - Aggiornamento Dati, passando la proprietà con valore buleano `msg.status` aggiorna lo stato,
 - **RESET** - Eliminazione dello Stato in Corso del BK - in consiglia l'utilizzo solo per pulire sitazzioni in errore,
 - **GET BACKUP** - Restituisce il contenuto del file di BACKUP 

### PropsType
 il tipo di Oggetto da gestire, può avere i seguenti valori:
 - **SingleObject** - Oggetto singolo con un Unico status da monitorare
 - **ListMultiObject** - lista di oggeti a multi chiavi con più staus da monitorare

#### _Returned SingleObject Exsample_
   `singleValue`: `{"value":{"name":"Alfa","time":30, "test": true},"status":false}` 

#### _Returned ListMultiObject Exsample_
   `tableList`: `[{"name":"alfa","time":30,"status":true},{"name":"beta","time":56,"status":false}]` 

### NamePropsID
 _(Obbligatorio per ListMultiObject in UPD)_ nome delle chiave in `msg` che contiene i valore per identifica nella lista di oggeti l'elemento da aggiornare

### NameAdditionaListProp
_(utilizabile solo in UPD)_ nome delle chiave in `msg` che contiene un oggeto le proprietà e valori, diverse dallo status da aggiornare nel elemento monitorato

### BackupName
 stringa da utilizzare come nome del file di Backup della Road corrente

### Overwrite 
 _(utilizabile solo in INIT)_ proprietà che permete in Inizializzaione di ignorare il backUp e gli oggeti attualmente monitorari per creare una situzione inizale pulita

---

## Propietà Oggetto
 - **`OnReq`** - Type: `Bool` - Indentifica il fatto che almeno uno dei valori e con `status` a false,
 - **`singleValue`** - Type: `Object` - contiene all'interno della proprieta chiave `value` il valore dell'oggeto singolo da monitorare passatogli nella inizializaione o dal backup e un nella chiave `status` contiene lo stato dell'oggeto che determina `OnReq` attuale,
 - **`tableList`** - Type: `Array<Object>` -contiene la lista di oggetti passatogli nella inizializaione con l'aggiunta in ogni oggeto della chiave `status` che contiene lo stato dell'oggeto, la somma degli stati di tutti gli oggeti determina `OnReq` attuale
---

# Flusso istanza 
il flusso di istanza prevede:
 1. un blocco **INIT** per inizalizare i valori/ recuperare il backUp, blocca il SEMAFORO in stato `OnReq` a `true` e inizializa i valori di stato degli oggeti monitorati a `false`
 2. un blocco **GET** per ottenere i dati dello stato in corso 
 3. un blocco **UPD** per aggiornare i valori di stato e/o proprieta delgli elementi
 4. in caso di necesita un blocco **RESET** per pulire i valori

il flusso risulta "_Sbloccato_"(cioe in `OnReq` a `false`) quanto lo stato di tutti i suoi elementi è a `true`
---

# _Returned Object Exsample_

`msg.payload` : `{"onReq":true,"singleValue":{"value":{"name":"Alfa","time":30, "test": true},"status":false},"tableList":[{"name":"alfa","time":30,"status":true},{"name":"beta","time":56,"status":false}]}`

---
# Logica Backup
il node genera per ogni **Road** un file di backUp chiamato come specificato in `BackupName`,
questo file contine la storia degli aggiornamenti effetuati agli oggeti `singleValue` e `tableList`,
il tutto finche la chiave `OnReq` rimane in stato `true`, al suo cambio di stato i valori salvati verrano cancelti per consentire un nuovo ciclo,
il file viene utilizato per recuperare i valori e stati degli oggeti monitorati, in modo tale in caso di nuova inizializaione i valori non verranno sovrascritti (ove non impostata la proprieta **Overwrite**) e nel caso il flusso dovesse interompersi inaspetamente i valore potranno essere recuperati

# Visione a Ciclo Chiuso
la logica fondante del nodo è di rendere recuperabile lo stato anche in caso di blocco dell'engine, 
di conseguena i dati delle modifiche degli oggeti monitorati verrano mantenuti nel file di backUp fino alla conclusione del cliclo,
in caso di blocco alla nuova inizializaione se il cliclo era stato completato verrà inializato uno nuovo, 
in caso contrario il nodo ignorera i valori passati è recuperara i dati dal file di backUp

# File Backup
il contenuto del file di Backup è il seguente:
 - **`name`** - Type: `String` - Nome file di Test,
 - **`stoplight`** - Type: `Bool` - Stato del Monitoriggio Semaforo per l'istanza sotto backup,
 - **`completed`** - Type: `Bool` - Indentifica lo stato del ciclo corrente,
 - **`createdAt`** - Type: `DataTime` - data di creazione del file,
 - **`road`** - Type: `String`- **Road** di apparteneza,
 - **`history`** - Type: `Array<Object>` - contiene la lista delle varie verisioni degli oggetti monitorari, traciandone i parametri passati e data di aggiornamento

# _Returned Object Exsample_

`msg.payload` : `{"name":"TestList", "stoplight":true,"completed":false,"createdAt":"2022-11-30T16:11:50.531Z","road":"ALFA_ONE","history":[{"onReq":true,"singleValue":null,"tableList":[{"name":"alfa","time":"TR","status":false},{"name":"beta","time":"TH","status":false}],"file":"BK_GBL_PROPS/ALFA_ONE_TestList.json","version":0,"datetime":"2022-12-05T09:37:53.648Z","propGetRif":"name"},{"onReq":true,"singleValue":null,"tableList":[{"name":"alfa","time":"TR","status":false},{"name":"beta","time":"PROVE","status":true}],"file":"BK_GBL_PROPS/ALFA_ONE_TestList.json","version":1,"datetime":"2022-12-05T09:38:06.798Z"}]}`







