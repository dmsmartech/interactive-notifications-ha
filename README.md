# 🔔 Notifiche Interattive in Home Assistant

Questo repository nasce come materiale di approfondimento legato a un video in cui mostriamo come Home Assistant può **coinvolgerti in un'automazione**, chiedendoti quale azione eseguire direttamente da una notifica push — invece di eseguire sempre e comunque la stessa azione in automatico.

Qui trovi:

- una spiegazione chiara di **cosa sono** le notifiche interattive e **cosa puoi farci**;
- una **lista di idee** d'uso reale, ognuna argomentata con un esempio concreto;
- una **guida passo-passo** per costruirne una da zero, in Home Assistant, senza toccare file YAML a mano;
- un'**automazione demo pronta da scaricare** (cartella [`automazione-demo/`](automazione-demo/)) che puoi importare subito per vedere il meccanismo funzionare dal vivo.

---

## Cosa sono le notifiche interattive

Il più delle volte creiamo automazioni che eseguono da sole una o più azioni prestabilite e, al massimo, ci notificano quello che hanno fatto. Ma in tanti casi l'azione giusta da compiere **dipende dalla situazione**, e in questi casi sarebbe utile che fosse Home Assistant a chiedere a noi cosa fare.

Le notifiche interattive sono semplicemente le normali notifiche push del tuo telefono o del tuo smartwatch, arricchite con **fino a 4 pulsanti azione**. Quando tocchi un pulsante, l'app companion di Home Assistant genera un evento (`mobile_app_notification_action`) che una tua automazione può intercettare per eseguire l'azione scelta.

In altre parole: invece di aprire l'app, cercare la dashboard, trovare il dispositivo giusto e azionarlo manualmente, **agisci direttamente sulla notifica**, in un tocco.

> I pulsanti nelle notifiche sono una caratteristica dei singoli sistemi operativi (iOS, Android, watchOS). Home Assistant non li "inventa": semplicemente li sfrutta, passando ai sistemi operativi la lista di pulsanti da mostrare insieme al testo della notifica.

### Come compaiono, a seconda del dispositivo

| Dispositivo | Come si vedono i pulsanti |
|---|---|
| **Android** | Compaiono subito sotto la notifica, sempre visibili |
| **iPhone** | Compaiono con una pressione prolungata sulla notifica |
| **Apple Watch** | Compaiono aprendo o toccando la notifica |

## Cosa puoi ottenere

Le possibilità sono praticamente infinite, perché dipendono dalle tue esigenze specifiche. In generale, le notifiche interattive ti permettono di:

- **Delegare la decisione finale a te stesso**, quando un'automazione non può (o non deve) decidere da sola quale azione eseguire;
- **Reagire più in fretta**, senza aprire l'app e navigare fino al dispositivo giusto;
- **Ridurre il numero di automazioni "rigide"**, sostituendo automazioni che fanno sempre la stessa cosa con automazioni che ti propongono opzioni;
- **Chiudere il cerchio** di automazioni che oggi si limitano a "informarti" trasformandole in automazioni che ti permettono anche di "agire".

---

## Idee di utilizzo

Ogni idea qui sotto è pensata come punto di partenza: adattala ai tuoi dispositivi e alle tue entità reali.

### ⚡ Alert soglia potenza contatore → stacco carichi

**Il problema:** hai un contatore con un limite di potenza (es. 3 kW) e un sensore che misura la potenza istantanea assorbita in casa (`sensor.potenza_istantanea`). Quando ti avvicini troppo al limite, rischi il distacco generale.

**L'automazione tradizionale** spegnerebbe sempre lo stesso carico (es. lo scaldabagno), ma magari in quel momento ti serve proprio quello, e potresti spegnere qualcos'altro senza problemi.

**Con una notifica interattiva**, invece, quando mancano ad esempio 300 W al distacco, ricevi un avviso con più opzioni tra cui scegliere:

- "Spegni scaldabagno"
- "Spegni asciugatrice"
- "Ignora per ora"

```yaml
triggers:
  - trigger: numeric_state
    entity_id: sensor.potenza_istantanea
    above: 2700   # 300 W sotto il limite di 3000 W
actions:
  - action: notify.mobile_app_iphone_di_mario
    data:
      title: "⚡ Attenzione al distacco!"
      message: "Mancano 300 W al distacco. Cosa vuoi spegnere?"
      data:
        actions:
          - action: "SPEGNI_SCALDABAGNO"
            title: "Spegni scaldabagno"
          - action: "SPEGNI_ASCIUGATRICE"
            title: "Spegni asciugatrice"
          - action: "IGNORA_DISTACCO"
            title: "Ignora per ora"
```

Poi, in una seconda automazione con trigger `event` su `mobile_app_notification_action`, intercetti l'azione scelta e spegni l'interruttore corrispondente (`switch.turn_off`).

### 🚪 Porta/serratura lasciata aperta

**Il problema:** una serratura smart, ad esempio quella della "porta vetro portico", risulta `unlocked`. Se resta così troppo a lungo e in casa non c'è nessuno, è un rischio.

**Con una notifica interattiva:** se la serratura resta sbloccata oltre N minuti (es. 10) e nessuna persona tracciata risulta `home`, ricevi una notifica con un pulsante **"Blocca ora"** che, se premuto, richiama direttamente `lock.lock` su quella serratura — senza dover aprire l'app.

Questa è anche l'idea che usiamo come **esempio completo passo-passo** più avanti in questa guida.

### 🤖 Vacuum bloccato o in errore

**Il problema:** il robot aspirapolvere si blocca (attributo/stato `error`) durante la pulizia — magari perché rimane incastrato, o non trova la via di ritorno.

**Con una notifica interattiva:** ricevi un avviso con due pulsanti azione:

- **"Rimanda alla base"** → chiama `vacuum.return_to_base`
- **"Riprova"** → chiama `vacuum.start` per fargli ritentare la pulizia da dove si trovava

Così decidi tu, caso per caso, se vale la pena insistere o se è meglio farlo rientrare.

### 🗑️ Rifiuti

**Il problema:** hai già un helper `input_text.rifiuti_stasera` che ogni giorno viene valorizzato (es. da un'automazione o da un calendario) con il tipo di rifiuto da buttare quella sera ("Organico", "Plastica", "Carta"...). Il rischio è dimenticarselo.

**Con una notifica interattiva:** la sera prima ricevi un promemoria con scritto cosa buttare, e un pulsante **"Fatto"** che, una volta premuto, azzera l'helper (`input_text.set_value` con valore vuoto), così sai sempre se l'hai già segnato come fatto oppure no.

```yaml
triggers:
  - trigger: time
    at: "20:00:00"
conditions:
  - condition: not
    conditions:
      - condition: state
        entity_id: input_text.rifiuti_stasera
        state: ""
actions:
  - action: notify.mobile_app_iphone_di_mario
    data:
      title: "🗑️ Rifiuti stasera"
      message: "Stasera tocca a: {{ states('input_text.rifiuti_stasera') }}"
      data:
        actions:
          - action: "RIFIUTI_FATTO"
            title: "Fatto"
```

Nella seconda automazione, alla pressione di "Fatto", chiami `input_text.set_value` con `value: ""` sull'helper.

### 🛎️ Citofono → apertura cancello/cancelletto

**Il problema:** squilla il citofono (es. un evento o un `binary_sensor` generato dall'integrazione del tuo citofono/videocitofono) e devi decidere al volo cosa aprire: il cancello carraio o il cancelletto pedonale — o niente.

**Con una notifica interattiva:** alla chiamata del citofono ricevi subito una notifica (magari con anche lo snapshot della videocamera, se disponibile) con i pulsanti:

- **"Apri cancello"** → `cover.open_cover` (o `lock.unlock`, a seconda di come è integrato il tuo cancello)
- **"Apri cancelletto"** → sul cover/lock del cancelletto pedonale
- **"Ignora"** → nessuna azione

In questo modo decidi in tempo reale chi far entrare, senza dover correre all'app o al citofono fisico.

### 🚨 Sirena allarme attivata (retro, palo)

**Il problema:** hai una sirena d'allarme (es. `siren.allarme_retro`) collegata a Home Assistant. Quando scatta vuoi saperlo **immediatamente**, ma a volte è un falso positivo — un gatto che passa, una raffica di vento — e vorresti poterla silenziare al volo, senza dover aprire l'app e cercare l'entità giusta.

**Con una notifica interattiva:** appena la sirena si attiva, ricevi una notifica **critica** (che, se il tuo dispositivo lo supporta, suona anche a telefono in modalità silenziosa) con un pulsante **"Silenzia sirena"** che richiama subito `siren.turn_off`.

```yaml
triggers:
  - trigger: state
    entity_id: siren.allarme_retro
    to: "on"
actions:
  - action: notify.mobile_app_iphone_di_mario
    data:
      title: "🚨 Sirena allarme attivata!"
      message: "La sirena del retro/palo è scattata."
      data:
        push:
          interruption-level: critical   # iOS: prova a suonare anche a telefono silenziato
        importance: high                 # Android: notifica ad alta priorità
        actions:
          - action: "SILENZIA_SIRENA"
            title: "Silenzia sirena"
```

Nella seconda automazione, alla pressione di "Silenzia sirena", richiami `siren.turn_off` sull'entità della sirena. Nota: le notifiche critiche su iOS richiedono che l'app companion abbia l'apposita autorizzazione attiva nelle impostazioni del telefono, altrimenti la notifica arriva comunque ma senza forzare l'audio.

---

## Come funzionano tecnicamente

Ogni notifica interattiva ha due metà: **l'invio** e **la gestione della risposta**. Servono quindi sempre **due automazioni** (o due parti della stessa logica):

### 1. L'invio: `data.actions`

Il servizio `notify.mobile_app_<nome_dispositivo>` (uno per ogni telefono/tablet collegato) accetta, dentro `data.data.actions`, fino a 4 oggetti con:

> **Nota:** su molte installazioni la versione "moderna" e generica `notify.send_message` (quella che punta a un'entità `notify.*` invece che a un servizio per-dispositivo) **non accetta** il campo `data` con le azioni — lo schema del servizio ammette solo `message` e `title` e la chiamata fallisce con un errore tipo `extra keys not allowed @ data['data']`. Per i pulsanti azione usa quindi sempre `notify.mobile_app_<nome_dispositivo>`, non `notify.send_message` (vedi anche la sezione [Inviare a "tutti i dispositivi"](#inviare-a-tutti-i-dispositivi) più sotto per il caso di più dispositivi).

- `action`: un identificatore **tuo**, univoco, che userai per riconoscere quale pulsante è stato premuto (es. `"BLOCCA_PORTA"`). Non deve avere spazi, per convenzione si scrive in maiuscolo.
- `title`: il testo visibile sul pulsante (es. `"Blocca ora"`).

```yaml
data:
  data:
    actions:
      - action: "BLOCCA_PORTA"
        title: "Blocca ora"
```

### 2. La risposta: evento `mobile_app_notification_action`

Quando l'utente tocca un pulsante, l'app companion genera automaticamente un evento Home Assistant di tipo `mobile_app_notification_action`, con dentro `event_data.action` uguale all'identificatore che avevi scelto. Una seconda automazione, con un `trigger` di tipo `event`, lo intercetta:

```yaml
triggers:
  - trigger: event
    event_type: mobile_app_notification_action
    event_data:
      action: "BLOCCA_PORTA"
actions:
  - action: lock.lock
    target:
      entity_id: lock.porta_vetro_portico
```

Da qui in poi è un'automazione come tutte le altre: puoi aggiungere condizioni, inviare una notifica di conferma, aggiornare un helper, e così via.

### Inviare a "tutti i dispositivi"

Per raggiungere un solo dispositivo basta chiamare il suo `notify.mobile_app_<nome>`. Ma se vuoi che una notifica **con pulsanti** raggiunga ogni telefono/tablet collegato a Home Assistant, il problema è che non esiste un servizio unico che accetti sia "più dispositivi in un colpo solo" sia il campo `data` con le azioni (`notify.send_message` accetta più dispositivi ma, come visto sopra, spesso non accetta `data`; i servizi `notify.mobile_app_<nome>` accettano `data` ma uno alla volta, un dispositivo per chiamata).

La soluzione è **ciclare** con `repeat: for_each` sulle entità `notify.*` (una per dispositivo con l'app companion) e, per ciascuna, ricostruire il nome del servizio legacy corrispondente sostituendo il prefisso `notify.` con `notify.mobile_app_`:

```yaml
- repeat:
    for_each: >-
      {{ states.notify
         | map(attribute='entity_id')
         | map('regex_replace', '^notify\.', 'notify.mobile_app_')
         | list }}
    sequence:
      - action: "{{ repeat.item }}"
        data:
          title: "Titolo"
          message: "Messaggio"
          data:
            actions:
              - action: "UN_AZIONE"
                title: "Un pulsante"
        continue_on_error: true
```

`continue_on_error: true` evita che un dispositivo che fallisce (es. offline, disinstallato) blocchi l'invio agli altri. Questo è esattamente il meccanismo usato nell'[automazione demo](automazione-demo/notifica-interattiva-demo.yaml) inclusa in questo repository.

**Se non arriva nulla a nessun dispositivo:** apri la trace dell'automazione (**Impostazioni → Automazioni e scene → apri l'automazione → icona "..." in alto → Traccia**) e controlla l'ultima esecuzione: se vedi un errore tipo `extra keys not allowed @ data['data']`, significa che da qualche parte stai ancora chiamando `notify.send_message` con un campo `data` — sostituiscilo con il pattern sopra.

**Se `states.notify` risulta vuoto:** verifica in **Impostazioni → Dispositivi e servizi → Entità**, filtrando per dominio `notify`. Se non compare nulla, la tua installazione espone i dispositivi solo come servizi legacy senza entità corrispondente: in quel caso dovrai elencare a mano i tuoi servizi `notify.mobile_app_<nome>` invece di ricavarli da `states.notify`.

---

## Guida passo-passo: crea la tua prima notifica interattiva

Usiamo come esempio completo l'idea "porta/serratura lasciata aperta", ma il procedimento è identico per qualunque altra idea della lista sopra.

### Prerequisiti

1. Hai installato l'app companion di Home Assistant sul telefono (iOS o Android) ed è collegata alla tua istanza.
2. Le notifiche dell'app companion sono attive nelle impostazioni del telefono.
3. Conosci il nome del `notify.*` associato al tuo dispositivo (**Impostazioni → App companion**, oppure controlla in **Impostazioni → Dispositivi e servizi → Dispositivi**, cercando il tuo telefono: il servizio si chiama tipicamente `notify.mobile_app_<nome_dispositivo>`).

### Passo 1 — Crea l'automazione che invia la notifica

1. Vai su **Impostazioni → Automazioni e scene → Crea automazione → Crea nuova automazione vuota**.
2. Come **trigger**, scegli uno stato: entità `lock.porta_vetro_portico`, da `locked` a `unlocked`, con una durata (`for`) di 10 minuti — così scatta solo se resta sbloccata davvero, non per un attimo.
3. Come **condizione**, aggiungi che nessuna persona tracciata sia in stato `home` (puoi usare una condizione `state` su ogni `person.*`, oppure — se ne hai più di una — un gruppo/helper che riassume la presenza in casa).
4. Come **azione**, scegli il servizio `notify.mobile_app_<tuo_dispositivo>` e componi il messaggio (per raggiungere più dispositivi insieme vedi [Inviare a "tutti i dispositivi"](#inviare-a-tutti-i-dispositivi)). Nella sezione dati aggiuntivi (in modalità YAML dell'azione) aggiungi `data.actions` con il pulsante:

```yaml
data:
  actions:
    - action: "BLOCCA_PORTA_VETRO"
      title: "Blocca ora"
```

5. Salva l'automazione con un nome chiaro, es. *"Porta portico sbloccata - avviso"*.

### Passo 2 — Crea l'automazione che gestisce la risposta

1. Crea una **seconda** automazione, con **trigger** di tipo *Evento*, `event_type: mobile_app_notification_action`.
2. Nei dati dell'evento (`event_data`), aggiungi `action: BLOCCA_PORTA_VETRO` — così l'automazione scatta solo per questo pulsante specifico (puoi aggiungere altri trigger, con altri `id`, se hai più pulsanti da gestire nella stessa automazione).
3. Come **azione**, richiama `lock.lock` sull'entità `lock.porta_vetro_portico`.
4. (Facoltativo ma consigliato) Aggiungi un'azione finale di notifica di conferma, così sai che il comando è stato eseguito: *"Porta bloccata ✅"*.
5. Salva con un nome chiaro, es. *"Porta portico sbloccata - blocca su richiesta"*.

### Passo 3 — Testa

Sblocca manualmente la serratura e aspetta il tempo impostato (o riducilo temporaneamente a 1 minuto per fare un test veloce). Quando arriva la notifica, prova il pulsante e verifica che la porta si blocchi davvero.

### Note utili

- **Un'automazione, più pulsanti:** puoi mettere più trigger `event` nella stessa automazione (uno per ogni `action`, ognuno con un `id` diverso) e poi usare `condition: trigger` / `choose` per smistare le azioni — è quello che fa l'[automazione demo](automazione-demo/notifica-interattiva-demo.yaml) inclusa qui.
- **`tag`:** puoi aggiungere `data.tag: "un-identificatore"` per far sì che notifiche successive con lo stesso tag sostituiscano quella precedente invece di accumularsi.
- **Icone e altre opzioni:** su Android puoi personalizzare colore, icona e canale di notifica; su iOS puoi usare `data.push.category` per notifiche più avanzate. Non sono necessarie per iniziare: i pulsanti funzionano già con la configurazione minima mostrata sopra.
- **Più dispositivi:** se vuoi che la notifica arrivi a più persone/telefoni contemporaneamente, cicla sui servizi `notify.mobile_app_<nome>` con `repeat: for_each` (vedi [Inviare a "tutti i dispositivi"](#inviare-a-tutti-i-dispositivi)) — `notify.send_message` da solo non basta perché spesso non supporta il campo `data` con le azioni.

---

## Automazione demo pronta all'uso

Nella cartella [`automazione-demo/`](automazione-demo/) trovi **un'unica automazione** di test, generica e "mock", pensata per essere scaricata e provata da chiunque, indipendentemente dai dispositivi che hai in casa. Non fa nulla di reale: serve solo a farti vedere il meccanismo delle notifiche interattive dal vivo, in due minuti.

Cosa fa **[`notifica-interattiva-demo.yaml`](automazione-demo/notifica-interattiva-demo.yaml)**, in un solo automazione:

1. Quando premi l'helper "Test Notifiche Interattive" (un `input_button`, il trigger più semplice che chiunque può creare in pochi secondi), invia una notifica push con **4 pulsanti** a **tutti** i dispositivi collegati a Home Assistant.
2. Qualunque pulsante tu prema, la stessa automazione lo intercetta e invia subito una **notifica di conferma** (di nuovo a tutti i dispositivi) che ti dice quale pulsante hai premuto.

Un `choose` interno, basato su `condition: trigger`, distingue se l'automazione è scattata per inviare la notifica iniziale oppure per gestire la pressione di uno dei 4 pulsanti.

### Come installarla

1. **Crea l'helper pulsante:** vai su **Impostazioni → Automazioni e scene → Helper → Crea helper → Pulsante**, chiamalo `Test Notifiche Interattive` (l'`entity_id` risultante sarà `input_button.test_notifiche_interattive` — se Home Assistant genera un nome leggermente diverso, aggiornalo nel file YAML prima di importarlo).
2. **Importa l'automazione:** vai su **Impostazioni → Automazioni e scene → Crea automazione → Crea nuova automazione vuota**, poi apri il menù **⋮ (tre puntini) → Modifica in YAML** e incolla il contenuto di `notifica-interattiva-demo.yaml`. Salva.
3. **Prova:** vai sulla dashboard, premi il pulsante helper "Test Notifiche Interattive". Dovresti ricevere subito la notifica con i 4 pulsanti su tutti i tuoi dispositivi con l'app companion. Premine uno: arriverà la notifica di conferma.

Usa questo file come base: copialo, rinominalo, e sostituisci il contenuto delle azioni con la tua logica reale, prendendo spunto dalla lista di idee più sopra.

---

## Licenza

Distribuito sotto licenza MIT. Consulta il file [LICENSE](LICENSE) per il testo completo.
