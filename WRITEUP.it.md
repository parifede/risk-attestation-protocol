# Un custode privato per la tua AI cloud: il Risk Attestation Protocol

Volevo usare i modelli cloud di frontiera sulla mia vita — le mie note, il mio
calendario, il mio profilo, il contesto accumulato che rende un assistente
davvero *mio* — senza spedire niente di tutto questo a un fornitore che non
controllo. Tutta la storia nasce da questa tensione. I modelli capaci stanno nel
cloud. I dati a cui tengo stanno sulla mia macchina e lì devono restare. E ogni
modo ovvio di colmare quella distanza è sbagliato.

Questo è il resoconto di come ci sono arrivato: non con un filtro di redazione
ingegnoso, ma con un piccolo protocollo dove due agenti negoziano, per iscritto,
su cosa esattamente può attraversare il confine — e dove la parte che possiede i
dati ha l'ultima parola, incluso il potere di vietare al cloud di rispondermi del
tutto.

Lo chiamo RAP — Risk Attestation Protocol. Il nome viene dall'artefatto al suo
centro: un *attestato di rischio*, un documento strutturato che un agente compila
e l'altro corregge. Questo articolo spiega il problema, il design, un esempio
concreto, perché ho deliberatamente tenuto lo scambio in testo in chiaro invece
che in qualcosa di più sofisticato, e dove il sistema protegge davvero e dove no.

## Il problema che nessuno risolve bene

Diciamo che chiedi a un assistente: *"Che macchina dovrei comprare, in base a chi
sono?"*

Una buona risposta deve sapere alcune cose di te. Il tuo budget. Dove vivi. Cosa
guidi adesso. Magari le tue preferenze dichiarate. Una risposta generica —
"valuta una utilitaria affidabile" — è inutile. Una risposta personale richiede
dati personali.

Se il tuo assistente gira su un modello cloud, quei dati personali devono andare
*da qualche parte* per essere elaborati. Le opzioni standard:

- **Mandare tutto.** Il modello riceve il tuo reddito, il tuo indirizzo, la tua
  storia, e dà un'ottima risposta. Hai anche appena consegnato a una terza parte
  un profilo dettagliato di te stesso. Privacy barattata per capacità.
- **Non mandare niente.** Tieni i tuoi dati, ma il modello può dare solo consigli
  generici. Capacità barattata per privacy.
- **Redarre in uscita.** Togli gli identificatori ovvi — email, numeri esatti — e
  manda il resto. Sembra responsabile ma è fragile, e peggio ancora è deciso dalla
  parte sbagliata: l'agente che *vuole* i dati è quello che sceglie quanti
  prenderne. Non c'è separazione di interesse.

Nessuna di queste è buona perché trattano tutte la domanda come "quanti dati
facciamo trapelare?" La domanda migliore è: *chi decide cosa attraversa, e a quale
fedeltà, e quel decisore sta dalla tua parte?*

## Due agenti, un confine

RAP divide il sistema in due ruoli con fiducia deliberatamente diseguale.

Il **Cloud Agent** è quello capace. Gira su un grande modello cloud. Ti parla,
ragiona, abbozza risposte. È *non fidato* — non perché sia malevolo, ma perché vive
fuori dal tuo confine dei dati. Non tocca mai i tuoi dati privati grezzi. Mai.

Il **Private Custodian** è quello fidato. Gira dove vivono i tuoi dati — sul tuo
hardware. È l'unica cosa che legge il tuo contesto privato grezzo, e ha potere di
veto sul Cloud Agent. Il Custodian decide cosa viene rilasciato, a quale
granularità, e — fondamentale — se il Cloud Agent ha il permesso di consegnarti una
risposta, oppure se deve restituire il proprio lavoro per la rifinitura privata.

I due non condividono mai memoria. Comunicano solo attraverso un documento
strutturato che viaggia tra loro: l'attestato. Non c'è canale laterale. Se un valore
non è nella proiezione approvata dell'attestato, il Cloud Agent non lo ha e deve
comportarsi come se non esistesse.

Quest'ultimo punto è la spina dorsale del design. Il confine non è imposto sperando
che il modello si comporti bene. È imposto dal fatto che al modello i dati non
vengono dati in primo luogo, e da una regola netta su cosa gli è permesso fare con
ciò che *riceve*.

## Come funziona la negoziazione

L'attestato attraversa un ciclo di vita fisso.

**1. Il Cloud Agent abbozza.** Quando una richiesta ha bisogno di contesto privato,
il Cloud Agent scrive un attestato iniziale. Include la *richiesta utente completa e
verbatim* — non un riassunto — più un elenco di cosa pensa di aver bisogno e perché,
più la sua stessa stima della sensibilità di ogni elemento.

Perché la richiesta completa e non un riassunto? Perché un riassunto è già esso
stesso una proiezione, e le decisioni di proiezione spettano al Custodian, non al
richiedente. Se il Cloud Agent potesse pre-riassumere, potrebbe far passare di
nascosto un'inquadratura davanti all'unica parte autorizzata a giudicare la
richiesta reale. Quindi il testo completo è obbligatorio.

**2. Il Custodian corregge.** Il Custodian esamina la bozza confrontandola con i dati
privati reali che custodisce. Produce una *proiezione approvata*: per ogni campo
richiesto, un valore reale più una **modalità di rilascio** che ne fissa la
granularità. Può anche *negare* campi del tutto. E imposta una **direttiva di
instradamento** che decide la consegna.

**3. Il Cloud Agent obbedisce.** Valida l'attestato corretto. Se qualcosa manca o è
contraddittorio, fallisce chiuso — nessuna risposta. Altrimenti esegue usando
*solo* la proiezione approvata, e instrada il proprio output esattamente come dice
la direttiva.

Le modalità di rilascio sono il selettore di granularità. Un campo può tornare come
`plain` (così com'è), `generalized` (una categoria più ampia invece del valore
esatto), `range_only` (un numero sostituito da una fascia), `tokenized` (un'etichetta
stabile), `summarized`, `redacted` (riconosciuto ma trattenuto), o `yes_no` (un
booleano e niente più). Il Cloud Agent deve trattarli come tetti invalicabili — non
può tentare di recuperare un dettaglio più fine di quanto la modalità consenta.

## L'esempio dell'auto, dall'inizio alla fine

Torniamo a *"Che macchina dovrei comprare, in base a chi sono?"*

Il Cloud Agent abbozza un attestato che richiede tre cose: la macchina attuale
(bassa sensibilità, la vuole esatta), la posizione (media, la vuole generalizzata),
e il reddito o budget (alta, vuole una fascia). Valuta la richiesta complessiva come
ad alto rischio.

Il Custodian esamina e restituisce questa proiezione:

- macchina attuale → `Golf 7`, modalità `plain`
- posizione → `Nord Italia`, modalità `generalized` (non la città, non l'indirizzo —
  solo la regione)
- reddito → `fascia 20k–40k`, modalità `range_only` (una fascia, mai la cifra)

E poi imposta la direttiva di instradamento su **ritorno al custode**: il Cloud
Agent *non* può rispondermi direttamente. Deve produrre una bozza e restituirla.

Perché? Perché la proiezione che ha ricevuto è intenzionalmente grossolana. Il
Custodian vuole che il Cloud Agent faccia il lavoro pesante — passare in rassegna le
macchine adatte, ragionare sui compromessi — su dati generalizzati, e poi rifinirà
privatamente quella bozza confrontandola con il mio profilo *esatto* prima che io la
veda. Il ragionamento costoso avviene su dati grossolani nel cloud; il passo finale
preciso e personale resta dentro il confine.

Così il modello cloud ragiona a fondo su "macchine affidabili per un guidatore del
Nord Italia in una fascia di reddito 20–40k che guida attualmente una Golf 7",
produce una buona bozza, e il Custodian — che conosce i miei numeri veri, la mia
città vera, le mie preferenze vere — la affina localmente. Il cloud non ha mai
saputo dove vivo né quanto guadagno. Ha saputo una regione e una fascia, ci ha
lavorato bene, ed è stato strutturalmente impedito dal rivolgersi a me direttamente.

Questo design a due flussi è la parte di cui vado più fiero. Il flusso semplice —
consegna diretta, dove la proiezione basta e il Cloud Agent risponde e basta — è il
caso comune. Ma il flusso di ritorno-al-custode è ciò che rende il confine *utile*
invece che meramente restrittivo. Permette a un modello potente-ma-non-fidato di
contribuire con lavoro reale a una domanda che non gli è permesso vedere per intero.

## L'invariante invalicabile

C'è una regola che l'implementazione impone in modo assoluto:

```
Se il Cloud Agent deve restituire l'output al Custodian,
allora NON può rispondere all'utente direttamente,
e la destinazione dell'output dev'essere il Custodian.
```

Se un attestato corretto viola mai questa regola — dice "ritorna al custode" ma
anche "puoi rispondere all'utente" — il Cloud Agent fallisce chiuso e resta in
silenzio. La contraddizione è trattata come pericolo, non come qualcosa da risolvere
tirando a indovinare. È il tipo di proprietà che vuoi noioso e assoluto, perché è
l'ultima linea tra "il cloud abbozza una risposta privata" e "il cloud *consegna*
una risposta privata".

## Perché testo in chiaro, e non qualcosa di più sofisticato

C'è una domanda che vale la pena affrontare direttamente, perché l'obiezione ovvia a
RAP è che è *inefficiente*. Gli agenti si passano avanti e indietro un documento YAML
verboso. Costa token e latenza. C'è lavoro di ricerca recente e davvero notevole —
per esempio il filone RecursiveMAS — che mostra come si possano far collaborare
sistemi multi-agente nello *spazio latente* invece che nel testo: passare gli stati
interni dei modelli direttamente da un agente all'altro, saltare del tutto il passo
di decodifica in testo, e risparmiare enormi quantità di token e tempo. Fino al 75%
di token in meno nei risultati riportati.

Quindi perché non ho fatto qualcosa del genere?

Perché per un confine di *privacy*, la verbosità è la feature, non il difetto.

Uno stato latente è veloce ma opaco. Non lo puoi leggere, non lo puoi controllare,
non lo puoi sanificare, non lo puoi registrare in un modo che un umano o un
verificatore deterministico possa poi verificare. È un vettore di numeri il cui
contenuto non puoi ispezionare. Ed è esattamente ciò che *non* vuoi che attraversi un
confine di fiducia il cui intero scopo è controllare cosa lo attraversa.

L'attestato RAP è l'opposto. È lento e prolisso, ma ogni singolo campo che attraversa
è nominato, classificato, assegnato a una modalità di rilascio, e messo per iscritto.
Lo puoi registrare. Lo puoi rieseguire. Lo puoi controllare a posteriori. Lo puoi far
fallire chiuso quando è malformato. L'intera cosa è costruita perché "cosa ha potuto
vedere il cloud?" abbia una risposta precisa e ispezionabile.

I due approcci ottimizzano assi diversi. La collaborazione latente ottimizza
l'*efficienza* — meno token, round più veloci. RAP ottimizza la *governance* — cosa
può attraversare, a quale fedeltà, e chi può parlare all'utente. Sull'asse
dell'efficienza, l'ispezionabilità è un costo. Sull'asse della governance,
l'ispezionabilità è l'intero punto. Non sto competendo con il lavoro sullo spazio
latente; sto risolvendo un problema diverso, e sul mio problema l'artefatto in testo
in chiaro vince proprio perché è in chiaro.

(C'è un futuro ipotetico in cui entrambe le parti girano in locale — un modello
locale potente che fa il ruolo del Cloud Agent sul tuo hardware. In quel mondo il
confine di fiducia cambia natura e uno scambio latente tra i *tuoi stessi* modelli
diventa pensabile. Ma è un sistema diverso, e non cambia il calcolo per il caso del
confine-cloud, che è quello che conta oggi.)

## Cosa RAP non protegge

Un meccanismo di privacy che si vende più di quanto vale è peggio di niente, perché
le persone si appoggiano a garanzie che non ci sono. Quindi, chiaramente, i limiti:

**RAP non difende da un Custodian compromesso.** Il Custodian è la radice di fiducia
per i tuoi dati privati — legge tutto. Se è compromesso, la partita è finita. RAP
assume che il Custodian sia affidabile per definizione; protegge i tuoi dati *dal
cloud*, non dalla cosa che li custodisce.

**RAP non elimina i side channel di volume e tempistica.** Anche se nessun valore di
campo trapela mai, il *fatto che un attestato esista* — la sua dimensione, la sua
frequenza — può dire a un fornitore cloud che osserva che era coinvolto del contesto
privato. Nel mio deployment lo accetto: non tratto il fornitore come un avversario
che cerca di profilarmi tramite analisi del traffico, sul ragionamento che un
fornitore che volesse informazioni generali su di me avrebbe vie ben più facili. Ma è
una scelta che ho fatto per il *mio* threat model. Se il tuo tratta il fornitore
cloud come un avversario attivo, devi affrontare questo separatamente, e RAP da solo
non lo farà.

**RAP non specifica trasporto, autenticazione o cifratura.** Quelli stanno sotto il
protocollo. RAP è il contratto di negoziazione — cosa attraversa e chi decide — non il
canale sicuro su cui viaggia. Sotto ci metti le solite cose.

**RAP non è un sistema di privacy completo.** È un pezzo ben definito: la parte che
dice cosa può attraversare il confine, a quale fedeltà, e chi può rispondere
all'utente. Il datastore, la logica di classificazione, le regole di modalità di
rilascio, la consegna finale — quelli sono implementazione, ed è lì che vive la
maggior parte del lavoro reale e del rischio reale.

Credo che essere espliciti su questo sia ciò che separa un design credibile da uno
speranzoso. RAP traccia una linea netta attorno a un problema difficile e risolve
*quello*, e dice chiaramente cosa sta fuori dalla linea.

## Da dove viene, e dove va

Ho costruito questo come parte di un sistema personale — un setup di AI locale dove un
agente basato su cloud lavora contro un vault privato sul mio hardware, mediato
esattamente da questo tipo di scambio di attestati. RAP è il contratto di confine
estratto da quel sistema e generalizzato così da non dipendere dal mio stack
specifico. Chiunque costruisca un agente che debba toccare dati privati senza cederli
può implementare i due ruoli e il documento che passa tra loro.

La specifica del protocollo — lo schema completo, i campi obbligatori, le modalità di
rilascio, l'invariante, i criteri di conformità — è pubblicata separatamente come
documento autonomo, deliberatamente neutra rispetto all'implementazione. Questo
articolo è il *perché*; la specifica è il *cosa*.

Se hai mai voluto puntare un modello di frontiera sulla tua vita senza consegnarla,
questo è un modo di tracciare quella linea in modo che la parte che tiene la penna sia
la tua.

---

*La specifica RAP è disponibile come repository autonomo. Questo scritto la
accompagna come razionale di design.*
