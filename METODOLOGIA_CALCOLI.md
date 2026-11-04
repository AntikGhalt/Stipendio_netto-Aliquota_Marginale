# Metodologia di calcolo — Atlante Stipendio

**Versione documentata:** simulatore v4.8, file [index.html](./index.html), anno fiscale di riferimento **2026**.  
**Scopo:** rendere riproducibili le formule effettivamente eseguite dal codice JavaScript, il significato economico delle grandezze, le ipotesi e le limitazioni.  
**Natura del documento:** descrizione del modello implementato, **non** certificazione dell'esattezza o dell'aggiornamento di ogni regola tributaria. In caso di discrepanza tra questo testo e il programma, per sapere che cosa il sito *calcola* fa fede il codice; la discrepanza va comunque corretta.

---

## 1. Impostazione generale e filosofia del simulatore

Il simulatore osserva la retribuzione del lavoro dipendente privato italiano da due punti di vista collegati:

1. **Crescita e tassazione**: per ciascuna RAL si stimano netto, costo dell'azienda e prelievo marginale associato a un aumento di retribuzione. La pagina isola la **RAL ordinaria**: bonus e benefit sono esclusi da queste curve.
2. **Stipendio e previdenza**: alla medesima RAL si aggiungono bonus, welfare, buoni pasto, tempo di lavoro, fondo pensione e TFR. Il risultato viene mostrato per anno, per mensilità equivalente e per ora effettivamente lavorata.

Le due viste condividono **una sola RAL**, un solo profilo fiscale e un solo motore di calcolo. La seconda vista imposta opzioni aggiuntive senza sostituire le formule della prima.

L'ordine logico è:

~~~text
RAL contrattuale
  + bonus lordo
  + parte imponibile dei buoni pasto [solo base contributiva/fiscale]
    -> contributi obbligatori del lavoratore
    -> premio soggetto a imposta sostitutiva, se spettante
    -> imponibile IRPEF e detrazioni
    -> imposte e agevolazioni
    -> netto monetario
  + valore nominale di welfare e buoni pasto
    -> netto e benefit utilizzabili nell'anno

In parallelo:
  -> contributi obbligatori dell'azienda e INAIL
  -> contributo aziendale al fondo pensione e solidarietà
  -> TFR maturato
    -> costo complessivo aziendale
~~~

**Tre grandezze non equivalenti**:

- **Disponibilità corrente**: denaro netto più benefit valorizzati al nominale. Non tutti i benefit si convertono liberamente in denaro.
- **Accantonamenti previdenziali**: contributi al fondo pensione e TFR. Sono valore maturato, ma non disponibilità immediata; il TFR non equivale a un secondo stipendio.
- **Costo aziendale**: tutto il costo rappresentato dal modello per remunerare e assicurare il lavoratore, inclusi oneri non incassati direttamente dalla persona.

Non si possono sommare indiscriminatamente i subtotali delle tre categorie: alcune voci sono già contenute nelle altre.

## 2. Convenzioni, simboli e ordine di calcolo

Usiamo questi simboli (nomi descrittivi, non variabili aggiuntive nel codice):

| Simbolo | Significato | Variabile nel motore |
|---|---|---|
| R | RAL contrattuale annua | ral |
| B | Bonus lordo annuo | bonus |
| V | Valore nominale annuo dei buoni | mealTotal |
| V_imp | Parte imponibile dei buoni | mealTaxable |
| W | Welfare aziendale esente valorizzato al nominale | welfare |
| A | Base per contributi obbligatori | base |
| C_l | Contributi obbligatori lavoratore | contrib |
| C_d | Contributi obbligatori datore, esclusa INAIL | employer |
| P_l | Versamento dipendente al fondo | pensionWorker |
| P_d | Versamento aziendale al fondo | pensionEmployer |
| Y | Imponibile IRPEF ordinaria | y |
| I | IRPEF lorda | gross |
| T | Imposte da pagare, incluso premio sostitutivo | taxes |
| F | Trasferimenti/bonus fiscali monetari | transfers |
| N | Netto monetario | net |
| Q | TFR annuo maturato | tfr |
| K | Costo complessivo aziendale | cost |

Tutte le formule fondamentali sono **annuali**. I valori mensili e orari si ottengono **dopo** il calcolo annuale, mediante una divisione: non sono buste paga effettive.

Il codice usa numeri JavaScript con precisione floating point; arrotonda soprattutto per visualizzare, non ogni singolo passaggio. Le detrazioni che impiegano rapporti TUIR applicano una **troncatura a quattro decimali** (funzione ratio). Importi e percentuali visibili possono quindi presentare differenze di pochi centesimi per arrotondamento.

Il codice si trova principalmente nelle funzioni **calculate**, **tax**, **workDed**, **familyDed**, **mandatoryContrib**, **timeData**, **packageOptions**, **drawRateChart**, **payComponents** e **newPayCard**.

---

## 3. Parametri incorporati nello snapshot 2026

Sono valori iniziali **modificabili**: descrivono un dipendente privato ordinario senza esoneri specifici e non una determinata azienda.

### 3.1 IRPEF e detrazioni

| Parametro | Valore iniziale |
|---|---:|
| Prima aliquota IRPEF, fino a imponibile 28.000 € | 23% |
| Seconda aliquota, quota fra 28.000 e 50.000 € | 33% |
| Terza aliquota, quota oltre 50.000 € | 43% |
| Detrazione lavoro, soglia iniziale fino a 15.000 € | 1.955 € |
| Base detrazione lavoro nel secondo/terzo tratto | 1.910 € |
| Maggiorazione da ridurre nel tratto 15.000–28.000 € | 1.190 € |
| Maggiorazione specifica nel tratto 25.000–35.000 € | 65 € |
| Detrazione massima da cuneo, secondo tratto | 1.000 € |
| Trattamento integrativo massimo | 1.200 € |

### 3.2 Contributi e altri parametri

| Parametro | Valore iniziale |
|---|---:|
| Contributi lavoratore, aliquota ordinaria | 9,19% |
| Contributo aggiuntivo lavoratore sopra soglia | 1% |
| Soglia contributo aggiuntivo nella simulazione | 56.224 € |
| Contributi datore IVS/pensione | 23,81% |
| NASpI datore | 1,31% |
| Fondi interprofessionali datore | 0,30% |
| Fondo garanzia TFR datore | 0,20% |
| CUAF datore | 0,68% |
| Maternità datore | 0,24% |
| Malattia datore | 2,44% |
| FIS/altri fondi datore | 0,33% |
| **Totale contributi datore nel profilo iniziale, esclusa INAIL** | **29,31%** |
| INAIL, stima separata | 0,40% |
| Divisore TFR | 13,5 |
| Recupero contributivo nella stima TFR | 0,50% |
| Quota personale al fondo, se attivo | 0,55% della RAL |
| Quota aziendale al fondo, se attivo | 1,55% della RAL |
| Limite annuo di deducibilità dei contributi al fondo | 5.300 € |
| Solidarietà a carico dell'azienda sulla quota al fondo | 10% |
| Premio di risultato: imposta sostitutiva | 1% |
| Premio di risultato: massimo lordo agevolabile | 5.000 € |
| Reddito di lavoro dipendente precedente: limite dichiarato | 80.000 € |
| Buono pasto elettronico, esenzione giornaliera | 10 € |
| Buono pasto cartaceo, esenzione giornaliera | 4 € |

**Attenzione:** 23,81% è soltanto la quota IVS aziendale, **non** il totale dei contributi che l'azienda versa. Il 29,31% è la somma delle componenti selezionate nel profilo iniziale; **non è un'aliquota universale**. INAIL viene aggiunta separatamente.

I valori percentuali del fondo sono applicati **convenzionalmente all'intera RAL**. I CCNL possono prevedere basi diverse (minimi tabellari, retribuzione utile al TFR ecc.) e subordinare il contributo datoriale a un versamento minimo del dipendente. Il simulatore consente di cambiare le percentuali ma **non verifica automaticamente l'adesione minima prevista dal CCNL**. Un riquadro propone i minimi di alcuni CCNL e li inserisce nei campi con un clic: Terziario Confcommercio · Fon.Te. 0,55% / 1,55%; Metalmeccanici · Cometa 1,20% / 2,00%; Chimico-farmaceutico · Fonchim 1,20% / 2,10% (manuale operativo aziende, febbraio 2026); Alimentare · Alifond 1,00% / 1,50% (fonti secondarie, da verificare sul CCNL); Tessile-abbigliamento · Previmoda 1,50% / 2,30% (da luglio 2026). Nei contratti la base può essere il minimo tabellare o la retribuzione utile al TFR; qui si applicano alla RAL.

---

## 4. Formazione della base contributiva e contributi obbligatori

### 4.1 Base contributiva

La base costruita dal codice è:

~~~text
A = R + B + V_imp
~~~

La parte fiscalmente imponibile dei buoni entra nella base contributiva, ma il loro valore nominale **non** è aggiunto alla RAL come denaro incassato.

### 4.2 Lavoratore

~~~text
C_l = A × aliquota_lavoratore
    + max(0, A − soglia_aggiuntiva) × aliquota_aggiuntiva
~~~

Con i parametri iniziali:

~~~text
C_l = A × 9,19% + max(0, A − 56.224) × 1%
~~~

Si tratta di una funzione **continua a tratti**: oltre soglia si tassa con l'1% **solo la parte eccedente**, non tutta la retribuzione. Il codice non introduce automaticamente un tetto contributivo universalmente applicabile, né riproduce tutte le quote minori dipendente per FIS/CIGS o altre gestioni.

**Nota per gli sviluppi:** il calcolo usa una soglia annua semplificata; non ricostruisce il meccanismo di versamento e conguaglio mensile del contributo aggiuntivo.

### 4.3 Datore e coperture sociali

Per ciascuna voce contributiva datoriale j:

~~~text
C_d,j = A × aliquota_datore,j
C_d   = somma_j C_d,j
~~~

Nel profilo iniziale:

~~~text
C_d = A × 29,31%
INAIL_stimata = A × 0,40%
~~~

Nel 29,31% sono già ricompresi IVS, NASpI, fondi interprofessionali, Fondo garanzia TFR, CUAF, maternità, malattia e FIS/altri fondi. **Non si deve aggiungere di nuovo il 23,81%.**

I contributi del datore sono **fuori dalla RAL**; quelli del lavoratore sono **già interni alla RAL**. INAIL e le prestazioni di assicurazione sociale rappresentano costi e coperture, non denaro versato nel conto corrente del lavoratore né automaticamente imposte IRPEF.

La contribuzione effettiva dipende da matricola e inquadramento INPS, dimensioni, settore, rapporto, apprendistato, sgravi e fondi applicabili. Gli importi del simulatore sono pertanto **stimati**.

---

## 5. Premi: trattamento ordinario e premio di risultato agevolato

Il bonus è aggiuntivo rispetto alla RAL contrattuale. La **retribuzione lorda erogata** è:

~~~text
Retribuzione_lorda_erogata = R + B
~~~

La RAL in senso stretto rimane R; **RAL + bonus** non è una seconda definizione di RAL, bensì il lordo monetario complessivamente corrisposto.

### 5.1 Condizioni per il regime agevolato

La modalità agevolata è applicata solo se sono contemporaneamente selezionati:

1. Premio di risultato agevolato.
2. Requisiti contrattuali/di legge confermati.
3. Reddito da lavoro dipendente nell'anno precedente entro la soglia dichiarata (80.000 € nello snapshot).

Queste sono **dichiarazioni dell'utente**, non verifiche documentali del simulatore.

La parte lorda agevolabile è:

~~~text
B_agev_lordo = min(B, limite_premio_lordo)   [solo se tutte le condizioni sono vere]
B_agev_lordo = 0                              [altrimenti]
~~~

La parte soggetta a imposta sostitutiva è calcolata **al netto dei contributi obbligatori imputabili a quella quota**:

~~~text
C_agev = C_l(R + V_imp + B_agev_lordo) − C_l(R + V_imp)

B_agev_imponibile = max(0, B_agev_lordo − C_agev)

Imposta_sostitutiva = B_agev_imponibile × aliquota_sostitutiva
~~~

L'eccedenza di bonus rispetto al limite agevolabile confluisce nella normale base IRPEF. I contributi obbligatori sono calcolati su **tutto** il bonus, anche quello agevolato.

Non vengono simulati speciali regimi di **decontribuzione** eventualmente previsti per particolari premi. Se manca una delle condizioni selezionate, tutto il bonus torna alla tassazione ordinaria.

### 5.2 Bonus netto: metodo controfattuale

Nella vista riepilogativa, il **bonus netto aggiuntivo** non si ottiene moltiplicando B per (1 − un'aliquota presunta). Si procede invece per differenza tra due scenari:

~~~text
N_bonus = N(R, bonus B, senza buoni e welfare)
        − N(R, bonus 0, senza buoni e welfare)
~~~

È un effetto incrementale: considera l'eventuale variazione di scaglioni, detrazioni, contributi, trattamento integrativo e imposte. **Non equivale necessariamente al netto del bonus in un cedolino specifico.**

### 5.3 Il bonus entra nel TFR?

L'input avanzato **Bonus incluso nella base TFR** determina se B è sommato a R nel calcolo del TFR (v. §10). L'opzione iniziale è **No**. Non va dedotta automaticamente dal regime fiscale agevolato: la base utile al TFR dipende da contratto e natura dell'erogazione.

---

## 6. Previdenza complementare: versamenti, deducibilità e costo

La previdenza complementare può essere attivata o disattivata. Quando è disattivata:

~~~text
P_l = 0
P_d = 0
solidarietà = 0
TFR_al_fondo = 0
~~~

Con fondo attivo:

~~~text
P_l = R × percentuale_personale_fondo
P_d = R × percentuale_aziendale_fondo

Versamenti_deducibili = min(limite_annuo, P_l + P_d)
Versamenti_non_dedotti = max(0, P_l + P_d − limite_annuo)

Solidarietà_azienda = P_d × percentuale_solidarietà
~~~

Il dipendente **rinuncia a una quota di netto monetario** per versarla nel fondo. Quella quota rimane **risparmio del dipendente**, non un'imposta. Il datore aggiunge un proprio contributo, che entra nei costi aziendali e nel patrimonio previdenziale del lavoratore.

La contribuzione aziendale transita nella costruzione dell'imponibile fiscale con una corrispondente deduzione entro il limite disponibile (vedi §7). Il TFR versato nel fondo **non usa il limite ordinario di deducibilità**.

Il contributo di solidarietà del 10% iniziale si applica alla **quota aziendale versata al fondo**, non è accreditato al fondo pensione e non riduce il netto del lavoratore.

**Limiti:** non si simulano ulteriori contributi versati autonomamente, conversioni del premio in previdenza, tutte le eccezioni di deducibilità, agevolazioni sulla base del CCNL né eventuali misure compensative per l'impresa che conferisce TFR.

---

## 7. Imponibile IRPEF e imposta lorda

Con A, C_l, B_agev_imponibile e i contributi al fondo definiti sopra:

~~~text
Y = max(0,
        A
      − C_l
      − B_agev_imponibile
      + P_d
      − Versamenti_deducibili)
~~~

Questa è **la base IRPEF ordinaria**; non coincide con la RAL.

L'IRPEF lorda si calcola per **scaglioni marginali**:

~~~text
I(Y) = min(Y, b1) × a1
     + max(0, min(Y, b2) − b1) × a2
     + max(0, Y − b2) × a3
~~~

con b1 = 28.000 €, b2 = 50.000 €, a1 = 23%, a2 = 33%, a3 = 43% nello snapshot.

Passando da 49.000 a 50.000 € di **RAL** non si attraversa necessariamente la soglia IRPEF dei 50.000 €, perché essa riguarda **Y**, che è generalmente inferiore alla RAL. La quota di reddito imponibile eccedente lo scaglione, non tutto il reddito, è tassata all'aliquota superiore.

---

## 8. Detrazioni d'imposta

### 8.1 Detrazione da lavoro dipendente

La funzione **workDed(Y)** è a tratti:

~~~text
Y <= 15.000:
    D_l = 1.955

15.000 < Y <= 28.000:
    D_l = 1.910 + 1.190 × trunc4((28.000 − Y) / 13.000)

28.000 < Y <= 50.000:
    D_l = 1.910 × trunc4((50.000 − Y) / 22.000)

Y > 50.000:
    D_l = 0

Inoltre, se 25.000 < Y <= 35.000:
    D_l = D_l + 65
~~~

I parametri numerici sono modificabili nel file. Il codice applica la detrazione al reddito fiscale Y ed esclude il caso di base contributiva nulla.

### 8.2 Familiari a carico

**Coniuge:** la funzione per il coniuge è a tratti, secondo Y:

~~~text
0 < Y <= 15.000:
    D_c = 800 − 110 × trunc4(Y / 15.000)

15.000 < Y <= 40.000:
    D_c = 690 + maggiorazioni puntuali:
      +10 se 29.000 < Y <= 29.200
      +20 se 29.200 < Y <= 34.700
      +30 se 34.700 < Y <= 35.000
      +20 se 35.000 < Y <= 35.100
      +10 se 35.100 < Y <= 35.200

40.000 < Y < 80.000:
    D_c = 690 × trunc4((80.000 − Y) / 40.000)

Altrimenti:
    D_c = 0
~~~

**Figli ammessi alla detrazione:** per n figli e quota scelta s (50% o 100%):

~~~text
Limite_figli = 95.000 + 15.000 × (n − 1)

D_figli = 950 × n × s × trunc4((Limite_figli − Y) / Limite_figli)
~~~

La funzione restituisce zero quando non ci sono figli. La quota è unica per tutti i figli; con coniuge fiscalmente a carico il codice forza s = 100%. Si assumono dodici mesi di spettanza. Il numero immesso si riferisce ai **figli ammessi alla detrazione IRPEF**, non a tutti i figli del nucleo familiare: l'Assegno Unico non è un componente dello stipendio simulato.

Non sono riprodotte casistiche familiari particolari, periodi infrannuali, altri redditi, diverse percentuali per ciascun figlio o verifiche anagrafiche automatiche.

### 8.3 Capienza fiscale e ordine di applicazione

Il codice applica le detrazioni nell'ordine:

~~~text
D_l_utilizzata     = min(I, D_l)                 [se opzione attiva]
D_fam_utilizzata   = min(max(0, I − D_l_utilizzata), D_fam)
D_cuneo_utilizzata = min(max(0, I − D_l_utilizzata − D_fam_utilizzata),
                        D_cuneo_teorica)

IRPEF_netta = max(0, I − D_l_utilizzata − D_fam_utilizzata − D_cuneo_utilizzata)
~~~

La detrazione non sfruttata per **incapienza** non diventa automaticamente un trasferimento monetario. Questo ordine di utilizzo è rilevante anche per le simulazioni controfattuali e il trattamento integrativo.

---

## 9. Misure di cuneo fiscale e trattamento integrativo

### 9.1 Somma non imponibile fino a 20.000 €

Quando la misura è attiva:

~~~text
Se Y <= 8.500:          Somma_cuneo = Y × 7,1%
Se 8.500 < Y <= 15.000: Somma_cuneo = Y × 5,3%
Se 15.000 < Y <= 20.000: Somma_cuneo = Y × 4,8%
Se Y > 20.000:          Somma_cuneo = 0
~~~

La percentuale è applicata, nel codice, a **Y intero** nel tratto considerato, non alla sola differenza sopra soglia. Questa somma non è un contributo pensionistico, né una detrazione IRPEF: è un trasferimento che aumenta il netto monetario.

### 9.2 Detrazione cuneo oltre 20.000 € fino a 40.000 €

~~~text
Se 20.000 < Y <= 32.000: D_cuneo_teorica = 1.000
Se 32.000 < Y <= 40.000: D_cuneo_teorica = 1.000 × (40.000 − Y)/8.000
Altrimenti:              D_cuneo_teorica = 0
~~~

Questa è una **detrazione**: riduce l'IRPEF fino alla capienza dopo le altre detrazioni, ma non produce automaticamente un rimborso eccedente l'imposta.

### 9.3 Trattamento integrativo

La funzione parte da un massimo iniziale di 1.200 €. Se il trattamento integrativo è abilitato, il codice applica:

~~~text
Se Y <= 15.000:
    TI = 1.200 se I(Y + B_agev_imponibile) > D_l_teorica − 75
         altrimenti 0

Se 15.000 < Y <= 28.000:
    TI = min(1.200, max(0, D_l_teorica + D_fam_teorica − I(Y)))

Se Y > 28.000:
    TI = 0
~~~

La verifica nel primo tratto comprende nella **base del test** anche l'imponibile del premio a imposta sostitutiva, mentre la soglia di reddito è confrontata con Y. Nel secondo tratto il modello **non** incorpora tutte le ulteriori detrazioni per oneri e spese pregresse che possono incidere sul trattamento spettante.

Trattamento integrativo e somma cuneo sono aggregati dal motore in:

~~~text
F = TI + Somma_cuneo
~~~

Nel riepilogo del costo del lavoro, quando si parla di **imposte nette dei benefici fiscali**, si sottrae F dalle imposte pagate: è una convenzione di lettura del **prelievo netto**, non un'affermazione che i trasferimenti siano imposte negative in senso giuridico.

---

## 10. Addizionali regionale e comunale (v4.7)

Le addizionali sono calcolate soltanto se **IRPEF netta > 0**, sulla base imponibile Y. L'utente sceglie **regione** (Trento e Bolzano sono distinte) e **comune di residenza**; le aliquote 2026 sono applicate automaticamente.

### 10.1 Fonte dei dati

Dipartimento delle Finanze, **elenchi generali in CSV** delle addizionali (pagine «Elenchi generali» di addizionale regionale e comunale), scaricati il 4 novembre 2026:

- **Regionali:** elenco 2026 (aggiornato al 19/06/2026), 21 enti. Per Puglia e Molise vale la delibera più recente (rideterminazione del commissario ad acta per il disavanzo sanitario).
- **Comunali:** elenco 2026 per 7.897 comuni. Il CSV riporta «0*» per i comuni che, alla data, non hanno pubblicato la delibera 2026: in questo caso si applica la regola dell'elenco 2025 (le aliquote restano in vigore fino a nuova delibera; nell'elenco 2025, definitivo, «0*» indica addizionale non istituita). Risultato: 4.057 comuni con delibera 2026, 2.956 con regola 2025 confermata, 884 senza addizionale.

### 10.2 Regola comunale

Per ogni comune è memorizzata una delle forme previste dal MEF:

~~~text
Aliquota unica:         Comune = Y × aliquota
Aliquote a scaglioni:   Comune = Σ_k (quota di Y nello scaglione k) × aliquota_k
Soglia di esenzione:    Comune = 0 se Y ≤ soglia; altrimenti la formula sopra su tutto Y
~~~

La soglia è un'esenzione, non una franchigia: superata, si paga sull'intero imponibile. Sono segnalati come **casi particolari** (117 comuni) quelli con esenzioni riservate a categorie (pensionati, dipendenti…) o con dati incoerenti nell'elenco MEF: per questi si applicano le aliquote senza l'esenzione per categoria e l'interfaccia invita a verificare la delibera (link diretto alla pagina MEF del comune).

### 10.3 Regola regionale

Gli scaglioni sono quelli dell'elenco 2026, applicati in modo progressivo. Le disposizioni speciali scritte nel campo «disposizione» sono codificate a mano:

| Ente | Regola 2026 implementata |
|---|---|
| Lazio | 1,73% su tutto Y fino a 28.000 €; oltre, 1,73% fino a 15.000 € e 3,33% sul resto; −60 € tra 28.001 e 30.000 € |
| Umbria | 1,23% su tutto Y fino a 28.000 €; oltre, 1,73 / 3,02 / 3,12 / 3,33% a scaglioni, −150 € fino a 50.000 € |
| Friuli-Venezia Giulia | 0,70% su tutto Y fino a 15.000 €; oltre, 1,23% su tutto Y |
| Valle d'Aosta | esenzione fino a 15.000 €; oltre, 1,23% su tutto Y |
| Provincia di Trento | nessuna addizionale fino a 30.000 € (deduzione di 30.000 €); oltre, 1,23% fino a 50.000 € e 1,73% sul resto |
| Provincia di Bolzano | 1,23% fino a 50.000 € e 1,73% oltre; −430,50 € fino a 90.000 €; oltre 50.000 € ulteriore −125 € × (Y − 50.000)/25.000, massimo 125 € |

Non sono applicate le agevolazioni regionali legate a figli, disabilità o categorie particolari (es. Campania, Piemonte, Puglia, Marche, Veneto, Sardegna, detrazioni per figli di Trento e Bolzano): il campo figli del simulatore conta solo i figli con detrazione IRPEF (21–29 anni), mentre queste agevolazioni usano definizioni diverse. L'interfaccia lo indica nella sezione «Aliquote applicate».

### 10.4 Aliquote personalizzate

La scelta «Aliquote personalizzate» sospende il calcolo automatico e usa quattro input: aliquota regionale unica, aliquota comunale unica e le rispettive soglie di esenzione (Regione = Y × aliquota se Y > soglia, altrimenti 0; idem per il comune).

### 10.5 Verifica

Con la configurazione iniziale (Roma, Lazio) il netto è identico a quello delle versioni precedenti, che incorporavano Roma–Lazio a mano. Controlli a campione ricostruiti a mano: Milano (Lombardia a scaglioni; comune 0,8% con esenzione fino a 23.000 €) e Torino (Piemonte a scaglioni; comune a quattro scaglioni con esenzione fino a 11.790 €).

## 11. Imposte finali e netto monetario

La somma delle imposte effettivamente incluse dal motore è:

~~~text
T = IRPEF_netta + Add_regionale + Add_comunale + Imposta_sostitutiva_premio
~~~

Il netto monetario annuo è:

~~~text
N = R + B − C_l − P_l − T + F
~~~

**Il netto monetario non include** welfare, valore nominale dei buoni pasto, contributi aziendali al fondo o TFR.

L'importo complessivo **utilizzabile nell'anno** mostrato come primo indicatore della Dashboard 2 è invece:

~~~text
Netto_e_benefit_utilizzabili = N + W + V
~~~

Welfare e buoni pasto sono valorizzati **al nominale**, ma non sono necessariamente fungibili con il denaro. Il netto monetario N ha già scontato il prelievo su eventuali buoni eccedenti la soglia di esenzione.

### 11.1 Scomposizione della voce principale

Per dare un contenuto comprensibile alla prima voce della dashboard, il simulatore procede per tre calcoli successivi:

~~~text
N_RAL = N(R, senza bonus, senza buoni, senza welfare)

N_bonus = N(R, con bonus, senza buoni, senza welfare) − N_RAL

Welfare_utilizzabile = W

Buoni_pasto_netti_stimati = V + N(R, con bonus e buoni) −
                            N(R, con bonus ma senza buoni)

Totale = N_RAL + N_bonus + Welfare_utilizzabile
         + Buoni_pasto_netti_stimati
       = N + W + V
~~~

**Perché il netto dei buoni può essere minore del valore nominale?** I buoni non entrano direttamente nella voce denaro netto, ma la loro **parte imponibile** può comportare ulteriori contributi e imposte, riducendo N. Il valore finale viene quindi calcolato per differenza, senza sommare impropriamente la parte imponibile alla retribuzione monetaria.

**Precisazione metodologica:** la scomposizione è una **attribuzione sequenziale** degli effetti: prima RAL, poi bonus, poi buoni. Se sono presenti soglie e agevolazioni, l'effetto del singolo componente può dipendere dall'ordine di inserimento. Le quattro righe si riconciliano con il totale, ma non sono effetti causali autonomi o invarianti al percorso.

---

## 12. TFR: maturazione, destinazione e recupero

### 12.1 Maturazione nell'anno

Il modello costruisce una base utile:

~~~text
Base_TFR = R + B     se il bonus è incluso nel TFR
Base_TFR = R         se il bonus è escluso
~~~

e calcola:

~~~text
Q = max(0,
        Base_TFR / divisore_TFR
        − A × aliquota_recupero_TFR)
~~~

Nello snapshot:

~~~text
Q = max(0, Base_TFR / 13,5 − A × 0,50%)
~~~

Il recupero dello 0,50% è una correzione della **quota TFR stimata**, non una nuova trattenuta applicata al netto monetario. Welfare e valore esente dei buoni non sono sommati alla base TFR.

**Base del recupero (verificata in v3.8):** lo 0,50% è calcolato sulla retribuzione imponibile ai fini previdenziali dell'anno, cioè su A (RAL + bonus + buoni imponibili), anche quando il bonus non entra nella base del TFR. È la regola di legge (art. 3 L. 297/1982), non una semplificazione. Il contributo è già compreso nell'IVS a carico dell'azienda e viene recuperato sulla quota TFR: nel costo aziendale compare una sola volta. Non vengono simulate rivalutazioni del TFR maturato in anni precedenti.

### 12.2 Destinazione

Se il fondo è attivo e l'opzione **Conferito al fondo pensione** è selezionata:

~~~text
TFR_al_fondo = Q
TFR_fuori_fondo = 0
~~~

Altrimenti:

~~~text
TFR_al_fondo = 0
TFR_fuori_fondo = Q
~~~

La destinazione **non cambia Q né il costo aziendale del TFR** nel modello; cambia dove viene contabilizzato l'accantonamento. Il regime ordinario può corrispondere all'azienda o al Fondo di Tesoreria INPS, secondo il caso.

---

## 13. Fondo pensione e TFR: valore futuro netto illustrativo

Il simulatore calcola prima il **valore lordo annuo destinato al fondo**:

~~~text
Contributi_fondo_senza_TFR = P_l + P_d

Fondo_lordo = P_l + P_d + TFR_al_fondo
~~~

La base fiscalmente assoggettata all'aliquota illustrativa viene approssimata come:

~~~text
Base_imponibile_fondo_illustrativa
    = max(0, min(Fondo_lordo,
                 Versamenti_deducibili + TFR_al_fondo))

Netto_fondo_illustrativo
    = Fondo_lordo − Base_imponibile_fondo_illustrativa
      × aliquota_uscita_fondo
~~~

L'aliquota è selezionabile fra **15%, 12%, 9% e 23%**. Il 15% è il valore iniziale, non il 9%. L'aliquota del 9% è un possibile esito per determinate prestazioni pensionistiche con elevata anzianità; il 23% può essere rilevante per alcune anticipazioni o riscatti. **Non esiste un'aliquota unica applicabile a tutti i prelievi.**

**Bug corretto in v3.4:** fino alla v3.3 il cambio dell'aliquota del fondo (e di quella del TFR) veniva annullato da un ricalcolo che rimetteva il valore precedente. Ora le aliquote sono lette direttamente dai campi a ogni calcolo (funzione `futureRates`).

Se il TFR rimane fuori dal fondo, il simulatore mostra separatamente:

~~~text
Netto_TFR_ordinario_illustrativo
    = TFR_fuori_fondo × (1 − aliquota_TFR_manualizzata)
~~~

L'aliquota iniziale del secondo scenario è **23% modificabile**. Non viene ricostruita la tassazione separata effettiva.

**Limiti essenziali delle due stime:** riguardano **il flusso di un singolo anno**, non un montante accumulato nel tempo; non includono rendimenti, rivalutazioni, anticipazioni già ricevute, imposte sui rendimenti, specifici periodi di partecipazione, regimi transitori e ogni distinzione fiscale delle prestazioni. Non rappresentano il capitale immediatamente prelevabile. La parte non dedotta dei contributi è esclusa convenzionalmente dalla base indicata, senza ricostruire tutti gli adempimenti fiscali richiesti.

Il fondo complementare e la pensione pubblica **non sono equivalenti**: i contributi obbligatori FPLD non sono un conto individuale immediatamente riscattabile da sommare al netto futuro.

---

## 13-bis. Straordinari (v4.2)

Nella pagina «Analisi stipendio» si possono inserire ore e compenso orario lordo di straordinario diurno, festivo e notturno (ore al mese × 12 oppure all'anno):

~~~text
S        = Σ_i ore_i × compenso_orario_i              (straordinari lordi annui)
S_spec   = ore_festive × compenso_festivo + ore_notturne × compenso_notturno
A        = R + S + B + V_imp                          (base contributiva)
C_spec   = C_l(R + S + V_imp) − C_l(R + S − S_spec + V_imp)   (contributi imputabili a S_spec)
S_sost   = min(limite, max(0, S_spec − C_spec))       [se l'opzione è attiva]
Y        = A − C_l − B_agev_imponibile − S_sost + P_d − Versamenti_deducibili
Imposta_sostitutiva_straordinari = S_sost × 15%
N        = R + S + B − C_l − P_l − T + F
K        = R + S + B + C_d + INAIL + Q + V + W + P_d + Solidarietà
~~~

- Gli straordinari pagano contributi e IRPEF come la RAL e **non** entrano nella base del TFR (scelta prudente: dipende dal CCNL). Il recupero 0,50% sul TFR, calcolato sull'imponibile previdenziale, li include.
- **Imposta sostitutiva 2026:** la legge di bilancio 2026 (L. 199/2025, art. 1, commi 10-11) prevede il 15% al posto di IRPEF e addizionali sulle maggiorazioni per lavoro notturno, festivo e a turni; la circolare AdE 3/E del 24 giugno 2026 chiarisce che si applica all'intera retribuzione dello straordinario notturno o festivo. Il limite è 1.500 € di imponibile annuo. Requisiti (CCNL applicato, soglia di reddito dell'anno precedente) sono dichiarati dall'utente con una spunta, come per il premio di risultato. Aliquota e limite sono parametri modificabili. Non sono simulate le maggiorazioni per lavoro notturno o festivo svolto in orario ordinario, né le indennità di turno o reperibilità.
- Nella scomposizione del netto, gli straordinari sono aggiunti dopo la RAL e prima del bonus (attribuzione sequenziale, §11.1).
- Le ore di straordinario si sommano alle ore lavorate nella colonna oraria: con compenso nullo misurano la riduzione della paga effettiva per ora.

## 14. Buoni pasto, welfare e giorni lavorativi

### 14.1 Giorni e ore

La sezione sull'orario utilizza per convenzione **261 giorni lunedì–venerdì nel 2026** e una settimana su **5 giorni**.

~~~text
Ore_al_giorno = Ore_settimanali / 5

Giorni_effettivi_anno
    = max(0, 261 − Festività_e_patrono − Ferie_ed_ex_festività)

Ore_effettivamente_lavorate
    = Giorni_effettivi_anno × Ore_al_giorno − Ore_altri_permessi

Ore_contrattuali_anno = Ore_settimanali × 52
~~~

Impostazioni iniziali: RAL **30.000 €**, **40 ore/settimana**, **27 giorni** di ferie + ex festività utilizzate, **38 ore** di altri permessi, **1 giorno di smart working a settimana senza buono** e **9,8 giorni** di festività/patrono come media convenzionale modificabile.

L'input dei permessi in ore **esclude** le ex festività già inserite nel conteggio dei giorni. Le formule non ricostruiscono ogni calendario reale, turnazione, part-time verticale, malattia o maternità.

### 14.2 Smart working senza buono

Il numero di giorni di smart working **senza erogazione del buono** è convertito in una frazione di giorni potenzialmente lavorati:

~~~text
Smart_fraction =
    giorni_smart_settimanali / 5         [modalità settimanale]
    giorni_smart_mensili / (261 / 12)    [modalità mensile]

Giorni_smart_senza_buono
    = Giorni_effettivi_anno × min(1, Smart_fraction)

Giorni_con_buono
    = max(0, round(Giorni_effettivi_anno − Giorni_smart_senza_buono))
~~~

Le **ferie** sono già escluse dai giorni potenzialmente lavorati. Gli **altri permessi orari** non riducono i giorni dei buoni nella semplificazione adottata. Lo smart working non elimina automaticamente il buono: l'input misura proprio i **giorni di smart senza buono**.

### 14.3 Valore e imponibilità

~~~text
V = Giorni_con_buono × Valore_buono_giornaliero

V_imp = Giorni_con_buono
        × max(0, Valore_buono_giornaliero − Soglia_esente)
~~~

La soglia iniziale è 10 €/giorno per buoni elettronici, 4 €/giorno per cartacei. L'eventuale eccedenza incrementa A e può incrementare contributi e imposte, pur non essendo retribuzione monetaria.

Il welfare W è assunto **interamente esente**; non sono simulati fringe benefit imponibili, limiti condizionati o particolari regimi di conversione.

---

## 15. Costo aziendale complessivo

La formula del motore è:

~~~text
K = (R + B)
  + C_d
  + INAIL_stimata
  + Q
  + V
  + W
  + P_d
  + Solidarietà_azienda
~~~

L'ordine è intenzionale:

- R + B è la retribuzione lorda monetaria;
- C_d e INAIL sono obblighi e coperture dell'azienda;
- Q è il TFR maturato **una sola volta**, indipendentemente dalla destinazione;
- V e W sono benefit al valore nominale;
- P_d è la quota aziendale al fondo pensione;
- la solidarietà aziendale è un costo ulteriore, **non** un versamento al fondo.

**Non** si sommano nuovamente C_l e P_l a K: entrambi sono già compresi nella RAL, rispettivamente come contributo obbligatorio e come parte di retribuzione destinata a risparmio previdenziale.

### 15.1 Identità di riconciliazione

Definendo:

~~~text
Disponibilità_corrente = N + V + W

Prelievo_fiscale_netto = T − F

Contributi_e_assicurazioni_obbligatorie
    = C_l + C_d + INAIL_stimata + Solidarietà_azienda

Versamenti_fondo_senza_TFR = P_l + P_d
~~~

la formula del codice implica esattamente:

~~~text
K = Disponibilità_corrente
  + Prelievo_fiscale_netto
  + Contributi_e_assicurazioni_obbligatorie
  + Versamenti_fondo_senza_TFR
  + Q
~~~

**Uso nell'interfaccia (v3.3).** Questa identità è la struttura delle tre card della Dashboard 2 e dei box di aumento della Dashboard 1:

~~~text
G  = Imposte_future_stimate = Base_imponibile_fondo_illustrativa × aliquota_uscita_fondo
                            + TFR_fuori_fondo × aliquota_TFR_manualizzata        (§13)
IVS_d = A × aliquota_IVS_datore                       (23,81% nel profilo iniziale)

① Netto utilizzabile                 = N + V + W
② Previdenza, netto stimato          = P_l + P_d + Q − G
③ Contributi pensionistici IVS       = C_l + IVS_d
④ Imposte e assicurazioni sociali    = (T − F) + G + (C_d − IVS_d) + INAIL_stimata + Solidarietà_azienda
Costo aziendale K                    = ① + ② + ③ + ④
~~~

**Perché IVS separata (v3.6).** Nel sistema contributivo l'aliquota di computo del lavoro dipendente è il 33% (23,81% azienda + 9,19% lavoratore): l'intera quota IVS viene accreditata sul montante individuale, che determina la pensione. Invalidità e superstiti sono finanziate dalla stessa contribuzione e non hanno un'aliquota separata. Il legame fra versato e prestazione è quindi diretto, ma non è un risparmio: la gestione è a ripartizione e il valore dipende da rivalutazione (PIL nominale), coefficienti di trasformazione ed età di uscita, fissati per legge. Il simulatore non stima quel valore. Il contributo aggiuntivo dell'1% oltre la prima fascia (56.224 € nel 2026) è giuridicamente IVS ed è incluso in ③; non è stata trovata una fonte ufficiale che chiarisca se accresca il montante, e la nota informativa lo segnala come non verificato. Le altre aliquote datoriali (NASpI, malattia, maternità, CUAF, FIS, fondi interprofessionali, Fondo garanzia TFR), l'INAIL e la solidarietà sul fondo finanziano coperture e servizi senza accumulo individuale: sono in ④ con le imposte.


La quota di ciascuna sezione è mostrata, a scelta, sul **costo aziendale** (predefinito: ①+②+③ = 100%) o sulla **RAL** (①+②+③ = K/R, maggiore del 100%). **Valore per te (v3.7).** Un selettore unico per le tre card sceglie il valore principale: ① (netto utilizzabile), ① + ② (con la previdenza complementare e il TFR, al netto stimato) oppure ① + ② + ③ (con i contributi pensionistici IVS). Sono ammesse solo somme progressive perché le sezioni sono ordinate dalla più disponibile alla meno certa; ④ è sempre esclusa. I colori seguono la scelta: con ① le tre sezioni hanno colori autonomi (verde, blu viola, blu); con ① + ② le due sezioni sommate diventano due tonalità di verde; con ① + ② + ③ tre tonalità di verde. ④ resta sempre arancione. ③ e ④ compaiono nella barra come blocchi di colore unico e mostrano le sfumature delle singole voci solo quando la sezione è aperta.

Le sezioni ③ e ④ della v3.6 sostituiscono la precedente sezione unica «Imposte e contributi». Dalla v3.4 la sezione ② è **al netto** delle imposte future stimate, e quelle imposte sono spostate in ③: così ③ raccoglie *tutti* i prelievi, inclusi quelli riferibili a fondo e TFR (Fondo garanzia TFR dentro C_d, solidarietà 10% sulla quota aziendale, imposta futura sul fondo e sul TFR), e la quota di ② sul costo misura ciò che il lavoratore incasserà in futuro. G è una **stima** con le aliquote illustrative (§13), applicata al flusso dell'anno. La solidarietà, molto piccola, è sommata ai contributi azienda e INAIL (resta visibile nel testo informativo). Le imposte di ③ sono «nette» perché vi si sottraggono trattamento integrativo e somma cuneo (F): sono denaro versato in busta paga, già compreso in N; senza sottrarli l'identità non si chiuderebbe. Nella card la riga «− Bonus fiscali in busta paga» lo rende esplicito.

Nell'interfaccia le tre sezioni sono **chiuse di base** e mostrano accanto il totale e la quota sul costo; aprirne una la apre nelle tre colonne. La barra di composizione mostra sei colori: netto (①), previdenza netta (②), imposte nette, imposte future stimate, contributi lavoratore, contributi azienda+INAIL+solidarietà (③). Se una componente è negativa (es. F > T a redditi bassi) la barra rappresenta solo le parti positive e lo segnala.

Le stesse grandezze, in differenza, formano la scomposizione di un aumento (funzione `deltaParts`):

~~~text
ΔK = ΔN + Δ(V+W) + (ΔP_l + ΔP_d + ΔQ − ΔG) + Δ(T − F) + ΔG + ΔC_l + Δ(C_d + INAIL + Solidarietà)
~~~

L'identità è **contabile nel perimetro del modello**: si ottiene sostituendo la definizione di N e cancellando le voci già comprese nella RAL. Non implica che il lavoratore possa monetizzare tutte le assicurazioni sociali o il TFR.

**Nota di interpretazione:** il prelievo fiscale netto può essere inferiore alle imposte pagate, o anche negativo, se F è elevato. Le categorie «imposte nette» e «contributi e assicurazioni» non sono automaticamente convertibili in valore dei servizi pubblici ricevuti da una singola persona.

---

## 16. Annuale, mensile e orario

Il calcolo fiscale è **sempre annuo**. Le tre colonne nella Dashboard 2 applicano il medesimo denominatore d a **tutte le voci**, così da essere confrontabili:

~~~text
Colonna annuale: d = 1
Colonna mensile: d = 12, 13 oppure 14 [12 iniziale]
Colonna oraria:  d = Ore_effettivamente_lavorate

Valore_mostrato(voce) = Valore_annuo(voce) / d
~~~

Di conseguenza:

- **/12, /13, /14** sono medie equivalenti di un valore annuo, non una ricostruzione dei cedolini di dicembre/giugno/luglio;
- la colonna per ora misura il rapporto fra il valore annuo e **ore effettivamente lavorate**, tenendo conto delle assenze parametrizzate;
- anche contributi, imposte, TFR e costo aziendale sono divisi per lo stesso d, perché si sta osservando l'**equivalente** per mensilità o ora, non l'esborso effettivo del singolo periodo;
- in questa versione la presentazione non tenta di stimare l'effettiva tassazione delle tredicesime e quattordicesime.

La RAL contrattuale compare come **riferimento**, mentre il primo indicatore principale è il **netto monetario più benefit utilizzabili**. Sotto il valore principale sono mostrati due rapporti, identici nelle tre colonne perché indipendenti dal denominatore d:

~~~text
Netto_utilizzabile / RAL           = (N + V + W) / R
Netto_utilizzabile / costo_azienda = (N + V + W) / K
~~~

Il primo può superare la lettura «netto/RAL» della Dashboard 1 perché include benefit e bonus netto al numeratore ma non il bonus lordo al denominatore.

**Altri elementi della Dashboard 2 (v3.3).** Il riquadro TFR (accanto a quello del bonus) mostra Q, la destinazione e l'opzione «bonus incluso nella base TFR». Le aliquote illustrative di uscita (§13) sono in una riga dedicata sotto le card. Restano due approfondimenti: il costo aziendale per voce di spesa (con le singole aliquote datoriali e la quota di ogni voce su K) e il prospetto analitico lordo→netto e calcolo delle imposte. Le tabelle «benefit e disponibilità», «benefit», «risparmio previdenziale» e «costo» del prospetto sono state rimosse perché duplicavano le sezioni ① e ②.

---

## 17. Analisi della crescita della RAL e aliquota marginale effettiva

### 17.1 Due scenari della Dashboard 1

Per l'analisi marginale vengono calcolati due risultati con **profilo fiscale invariato**, senza bonus, welfare e buoni:

~~~text
Risultato_iniziale = calculate(R, profilo)
Risultato_finale   = calculate(R + aumento, profilo)
~~~

L'aumento **standard** è 1.000 €; quello **personalizzato** è selezionabile fino a 20.000 €. Le variazioni del netto, dei contributi, delle imposte e del costo aziendale sono le differenze fra i risultati.

Queste sono **variazioni finite**, non derivate infinitesime. Una curva a passo 1.000 € può intercettare una soglia dentro un intervallo: in quel punto la variazione rappresenta una **media sull'intervallo**.

### 17.2 Aliquota marginale fiscale e contributiva

Con incremento ΔR > 0:

~~~text
ΔN   = N(R + ΔR) − N(R)
ΔP_l = P_l(R + ΔR) − P_l(R)

Aliquota_marginale_effettiva
    = (ΔR − ΔN − ΔP_l) / ΔR

    = (ΔC_l + ΔT − ΔF) / ΔR
~~~

Il fondo pensione **è sottratto dal numeratore** perché non deve essere classificato come imposizione.

Nella curva rappresentata a passo fisso ΔR = 1.000 € il valore in percentuale è:

~~~text
Aliquota_1000_percento
    = 100 × (1.000 − ΔN − ΔP_l) / 1.000
~~~

Curve del grafico, raggruppate nella legenda; ogni voce riporta il valore alla RAL selezionata (passo R → R + 1.000 €). Dalla v3.4 un interruttore sceglie la **prospettiva**: «Quanto resta in tasca» (predefinita, solo gruppo «Quanto resta», attiva solo Netto su 1.000 € di RAL) oppure «Cuneo» (gruppi lavoratore e azienda; attive di base solo il prelievo marginale e la quota personale al fondo, le altre si accendono dalla legenda).

**Unità dell'asse (v3.5).** Le curve sono espresse in **euro su ogni 1.000 € in più** (di RAL o di costo aziendale, secondo la curva): il valore in euro è la frazione × 1.000, quindi 619 € equivale a un'aliquota del 61,9%. I riepiloghi laterali e i box restano anche in percentuale.

~~~text
Lavoratore · dentro la RAL (denominatore ΔR)
  Prelievo marginale          = (Δ(T − F) + ΔC_l) / ΔR        [predefinita]
    di cui imposte nette      = Δ(T − F) / ΔR                 [predefinita]
    di cui contributi lavor.  = ΔC_l / ΔR                     [predefinita]
  Quota personale al fondo    = ΔP_l / ΔR                     [opzionale, solo fondo attivo]

Azienda · fuori dalla RAL
  Contributi azienda e INAIL  = Δ(C_d + INAIL + Solidarietà) / ΔR   [predefinita]
  Cuneo marginale sul costo   = Δ(T − F + G + C_l + C_d + INAIL + Sol.) / ΔK

Quanto resta
  Netto / ΔRAL                = ΔN / ΔR                       [predefinita nella prospettiva «resta»]
  Netto / Δcosto aziendale    = ΔN / ΔK                       [opzionale]
~~~

Per costruzione: Prelievo marginale = imposte nette + contributi lavoratore, e Netto/ΔRAL = 1 − Prelievo marginale − ΔP_l/ΔR.

Il **cuneo marginale** qui definito ha al denominatore l'intero ΔK, che include TFR e quota aziendale al fondo: è coerente con la scomposizione ①②③ ma **non coincide con il tax wedge OCSE**, che misura il costo del lavoro senza TFR e previdenza complementare. Le curve «contributi azienda» e «netto/Δcosto» sostituiscono la precedente «costo aggiuntivo oltre la RAL», che mescolava prelievi e retribuzione differita; è stata rimossa anche la curva «imposte + contributi + fondo», ricavabile come 100% − Netto/ΔRAL.

**Scala verticale.** Dalla v3.5 la scala comprende sempre tutti i valori delle curve visibili, senza tagli, con tacche arrotondate.

**Lo scalino intorno a 9.000–10.000 € di RAL è reale, non un artefatto di calcolo.** Il trattamento integrativo (1.200 €) spetta nel primo tratto solo se I > D_l − 75 (§9.3), cioè da circa 8.200 € di imponibile: superata la soglia, il netto aumenta di colpo di 1.200 €, quindi su +1.000 € di RAL ΔN > ΔR (prelievo marginale negativo, netto/ΔRAL > 100%, e anche ΔN > ΔK). Subito prima, la somma cuneo passa dal 7,1% al 5,3% di tutto Y a 8.500 € (§9.1), con uno scalino opposto. Il passo di 1.000 € spalma su un intervallo ciò che nella norma avviene in un euro. È rilevante solo per redditi annui molto bassi (part-time, anno incompleto). Le tacche sono arrotondate a passi 1/2/2,5/5 × 10ⁿ.

**Maniglie.** Trascinando, RAL e aumento avanzano a passi di 1.000 € (nei campi si può comunque scrivere qualsiasi importo). La RAL di partenza si sposta cliccando o trascinando nel grafico; l'aumento si trascina dal punto verde posto **sopra** l'area del grafico, così le due maniglie restano separabili anche quando si sovrappongono in orizzontale.

**Due netti, un solo motore.** La pagina 01 mostra il **netto dalla sola RAL** (N senza bonus, buoni e welfare), che coincide con la prima riga di ① nella pagina 02. Il passaggio è: netto utilizzabile della pagina 02 = netto dalla sola RAL + bonus netto + welfare + buoni pasto netti. Le curve marginali restano sulla sola RAL, perché bonus e benefit hanno regimi propri e confonderebbero la lettura dell'aliquota.

**Riepiloghi a destra del grafico e box sottostanti.** Per lo scenario +1.000 € e per l'aumento personalizzato ΔR_p si mostrano: ΔN, ΔN/ΔR, ΔN/ΔK, prelievo marginale e ΔK. I box di dettaglio aggiungono la scomposizione completa di ΔK (barra + tabella in € , % di ΔR e % di ΔK) e i livelli medi prima/dopo (RAL, N, K, N/R, N/K).

### 17.3 Perché il passaggio dello scaglione IRPEF può essere poco visibile

Lo scaglione superiore del 43% inizia quando **Y supera 50.000 €**, non quando la RAL supera 50.000 €. Inoltre, nella fascia immediatamente precedente, la detrazione lavoro si riduce con l'aumento del reddito: parte del prelievo effettivo è già attribuibile alla **perdita di detrazione**. Quando questa si azzera, l'aumento dell'aliquota IRPEF nominale può essere parzialmente compensato dalla fine di tale perdita.

L'aliquota marginale effettiva osservata è quindi il risultato **congiunto** di aliquote, contributi, addizionali, detrazioni e trasferimenti: non coincide in generale con 23%, 33% o 43%.

### 17.4 Soglie e interazione col grafico

Il grafico ricava la **RAL che corrisponde a ciascuna soglia di imponibile** risolvendo numericamente:

~~~text
Y(RAL_soglia) = 28.000 oppure 50.000
~~~

tramite ricerca binaria sul motore di calcolo (funzione chartThresholds). È più corretto che collocare arbitrariamente le soglie in corrispondenza di RAL 28.000 e 50.000 €.

Le linee grigie indicano tali soglie (anche nel grafico dei valori annui, che dalla v3.3 include la serie «imposte nette» T − F); la linea verde indica la **RAL dopo l'aumento personalizzato**, modificabile trascinando il suo punto superiore. L'intervallo RAL iniziale–RAL dopo aumento è visualizzato sull'asse orizzontale. Le curve dei valori annui costituiscono una visualizzazione alternativa del medesimo profilo.

Le soglie sono cercate numericamente in un intervallo fino a 120.000 €; la grafica mostra il dominio RAL fino a 100.000 €. Quando la RAL dopo l'aumento supera 100.000 €, il valore finale è segnalato come **fuori scala** e il segno grafico viene limitato al bordo.

**Attenzione interpretativa:** la ricerca binaria presuppone una relazione tra RAL e Y sufficientemente regolare/monotona nel profilo selezionato. Modifiche estreme ai parametri possono rendere meno affidabile la localizzazione.

---

## 18. Rapporti riepilogativi e tabella RAL

Per una RAL selezionata, la Dashboard 1 mostra fra l'altro:

~~~text
Aliquota_media           = (C_l + T − F) / R
Netto / RAL              = N / R
Netto / costo_azienda    = N / K
~~~

Dalla v3.3 l'aliquota media sostituisce il rapporto Netto / imponibile IRPEF, che non era un'aliquota e confondeva basi diverse: affiancata alla curva marginale permette il confronto **media vs marginale** sullo stesso numeratore (imposte nette + contributi del lavoratore, fondo escluso).

Nel confronto ogni **5.000 €** il motore ricalcola tutto per R = 0, 5.000, ..., 100.000 € e riporta i valori annuali. La colonna prelievo/RAL usa:

~~~text
Prelievo_medio_su_RAL = (C_l + T − F) / R
~~~

per R > 0; a R = 0 il rapporto non è definito e non viene mostrato. La quota personale al fondo è esclusa dal **prelievo**, poiché è un investimento previdenziale.

La funzione di esportazione CSV salva una griglia di valori calcolati; i campi utilizzano il separatore punto e virgola e valori decimali convertiti in formato italiano.

---

## 19. Perimetro, esclusioni e punti di attenzione tecnica

### Esclusioni strutturali

Il simulatore non ricostruisce:

- cedolini mensili e conguagli effettivi, ratei di 13ª/14ª, rapporto iniziato o terminato nell'anno;
- altri redditi personali, deduzioni/detrazioni per spese private, tutte le situazioni familiari;
- ogni CCNL, livello, apprendistato, minimi contributivi e sgravi datoriali;
- aliquote territoriali progressive ed esenzioni locali non incorporate;
- straordinari, turni, lavoro notturno, malattia, maternità, trasferte e indennità;
- regimi fiscali specifici dei premi diversi dalla configurazione semplificata;
- rivalutazione pluriennale del TFR, rendimento del fondo pensione, fiscalità dei rendimenti e disponibilità effettiva delle prestazioni;
- eventuali costi amministrativi, commissioni e IVA non rappresentati nel costo dei benefit;
- il valore economico attuariale individuale di pensione pubblica, NASpI, malattia, maternità e altre assicurazioni.

### Questioni da non nascondere sotto un numero preciso

1. **Aliquote datore**: 29,31% + INAIL è un esempio modificabile, non una previsione esatta per ogni datore.
2. **Fuori Roma**: addizionali inizialmente nulle se non inserite; le aliquote uniche non equivalgono a un motore comunale completo.
3. **TFR**: il recupero dello 0,50% usa A (imponibile previdenziale), anche quando il bonus non è utile al TFR: è la regola di legge; restano fuori rivalutazioni e anni precedenti.
4. **Fondo**: quote calcolate sull'intera RAL e spettanza aziendale non verificata rispetto al versamento minimo contrattuale.
5. **Tassazione futura**: 15/12/9/23% e aliquota TFR manuale producono **scenari su un flusso annuale**, non una liquidazione certa.
6. **Bonus**: le condizioni agevolative sono flag dell'utente; non equivalgono a un controllo della documentazione contrattuale.
7. **Buoni**: la valorizzazione «netta» è residuale, dipende anche dalla presenza del bonus e dalle altre condizioni fiscali.
8. **Lavoro orario**: i valori sono divisi per ore effettive convenzionali, non per ore certificate da un sistema presenze.
9. **Rotture di soglia**: la progressività delle detrazioni, del cuneo e dei trasferimenti può generare discontinuità nella curva marginale.
10. **Snapshot**: nessuna fonte online viene interrogata automaticamente per aggiornare la normativa fiscale del simulatore.

### Controlli raccomandati nelle revisioni

Prima di modificare una formula, verificare almeno:

- **Identità del netto**: N = R + B − C_l − P_l − T + F.
- **Identità della disponibilità**: N + V + W coincide con la somma delle quattro componenti del riepilogo.
- **Identità del costo**: K coincide con la somma delle voci nella §15 e con la riconciliazione di §15.1.
- **Fondo OFF**: P_l, P_d, solidarietà e TFR al fondo risultano zero; il TFR ordinario rimane contabilizzato.
- **TFR ON/OFF**: cambia soltanto la destinazione; Q resta lo stesso.
- **Bonus agevolato ON/OFF**: limite applicato al **premio lordo**, imposta sostitutiva sulla quota imponibile dopo contributi.
- **Buono elettronico/cartaceo**: V non cambia a parità di valore nominale; V_imp può cambiare e modificare N.
- **RAL sincronizzata**: l'analisi marginale resta priva di bonus/benefit e la Dashboard 2 li aggiunge separatamente.
- **Mensilità**: cambiare 12/13/14 deve cambiare solo il denominatore, non i valori annuali.
- **Composizione ①+②+③**: per ogni RAL e scenario la somma delle tre sezioni coincide con K, e la scomposizione di ΔK coincide con il costo aggiuntivo (verificato in v3.3 su 63 combinazioni di RAL, bonus, benefit, fondo, TFR, familiari e territorio).
- **Aumenti**: il risultato per ΔR = 1.000 € coincide con la differenza fra due chiamate del motore, e la curva fiscale marginale **non** include P_l.
- **Ripristino e salvataggio**: la copia HTML conserva gli input, i flag e le selezioni principali.

Questi sono **test di coerenza interna**. Non sostituiscono una verifica con fonti normative vigenti e casi reali di busta paga.

---

## 20. Fonti di riferimento e manutenzione

Le regole e le stime sono documentate anche nel pannello «Metodo, perimetro e fonti» dell'HTML. Per aggiornamenti successivi, consultare in particolare:

- [Ministero del Lavoro — misure della legge di bilancio 2026](https://www.lavoro.gov.it/notizie/pagine/legge-di-bilancio-2026-le-principali-misure-lavoratori-imprese-e-famiglie)
- [TUIR art. 13 — detrazioni lavoro](https://www.normattiva.it/atto/caricaDettaglioAtto?atto.articolo.numero=13&atto.codiceRedazionale=086U0917&atto.dataPubblicazioneGazzetta=1986-12-31&vigenza=2026-01-01)
- [TUIR art. 12 — detrazioni familiari](https://www.normattiva.it/atto/caricaDettaglioAtto?atto.articolo.numero=12&atto.codiceRedazionale=086U0917&atto.dataPubblicazioneGazzetta=1986-12-31&vigenza=2026-01-01)
- [INPS — aliquote e variabilità dei contributi](https://www.inps.it/it/it/inps-comunica/diritti-e-obblighi-in-materia-di-sicurezza-sociale-nell-unione-e/per-le-imprese/aliquote-contributive.html)
- [INPS — Fondo di integrazione salariale](https://www.inps.it/it/it/dettaglio-scheda.it.schede-servizio-strumento.schede-servizi.50262.fondo-d-integrazione-salariale-fis-.html)
- [MEF — addizionali comunali](https://www1.finanze.gov.it/finanze2/dipartimentopolitichefiscali/fiscalitalocale/nuova_addcomirpef/download/tabella.htm)
- [MEF — addizionali regionali](https://www1.finanze.gov.it/finanze2/dipartimentopolitichefiscali/fiscalitalocale/addregirpef/download/tabella.htm)
- [Ministero del Lavoro — previdenza complementare](https://lavoro.gov.it/previdenza-complementare/educazione-previdenziale/come-funziona-e-quanto-costa)
- [Fon.Te. — contribuzione](https://www.fondofonte.it/adesione/contribuzione/)
- [Cometa](https://www.cometafondo.it/prestazioni-contribuzione/tfr/) e [Previmoda](https://www.previmoda.it/come-aderire/) — riferimenti su contribuzione contrattuale

**Procedura di manutenzione raccomandata:**

1. Modificare il codice del motore e i parametri del nuovo anno fiscale.
2. Verificare le regole con le fonti ufficiali, incluse le date di applicazione.
3. Aggiornare **questo documento** insieme alla logica effettiva, non soltanto alle etichette.
4. Eseguire controlli di riconciliazione e casi ai confini degli scaglioni.
5. Aggiornare la versione del simulatore e indicare gli eventuali comportamenti cambiati.

---

**Autore del progetto:** [Paolo Refuto](https://www.linkedin.com/in/paolo-refuto-6a5b52236/).  
**Stato:** metodologia della **v4.8**. Le note metodologiche descrivono la logica esistente; non costituiscono correzioni silenziose del programma.

---

## Registro modifiche

**v4.8**
- Profilo: territorio, fondo pensione e familiari in tre box di pari altezza, con i link informativi allineati in fondo; «Aliquote applicate» come riquadro sovrapposto che contiene anche i campi delle aliquote personalizzate e il pulsante «Usa queste aliquote».
- Costo aziendale per voce e prospetto dal lordo al netto riuniti in un'unica tendina di approfondimento.

**v4.7**
- Addizionali regionali e comunali 2026 automatiche da regione e comune (dati MEF incorporati per 7.897 comuni e 21 enti regionali); aliquote personalizzate come alternativa; sezione «Aliquote applicate» per la verifica, con link alla delibera.
- Profilo su tre box (territorio, familiari, fondo pensione); coniuge di nuovo come menu; detrazioni automatiche spostate nei parametri modificabili; icona ⚙︎ per i riquadri che contengono opzioni.

**v4.6**
- Spiegazioni ⓘ come riquadri sovrapposti (senza freccia e senza spostare il contenuto); su mobile pannello in basso.
- Profilo: territorio, coniuge, figli e detrazioni su 2×2 a sinistra, fondo pensione a destra; coniuge come interruttore No/Sì; figli da menu 0–6 o «altro numero».
- Tabella dei minimi CCNL con pulsante «Usa» (aggiunti Fonchim e Alifond); spiegazione del fondo pensione riscritta.

**v4.5**
- Retribuzione, bonus e TFR come tre box con la stessa struttura (campo, menu, riga di stato, tendina ⓘ) e righe allineate; il regime del bonus e la destinazione del TFR diventano menu a tendina; il TFR maturato è un campo in sola lettura.
- Nota sui familiari a carico riscritta (chi indicare, limiti di reddito, quota) e mostrata come riquadro sovrapposto.

**v4.4**
- Retribuzione e benefit: RAL, welfare e buoni pasto in colonna a sinistra, bonus e TFR affiancati a destra; cifra della RAL meno pesante.

**v4.3**
- RAL in evidenza nella pagina «Analisi stipendio».
- Box dell'orario: cinque campi su una riga (con «Giorni di smart senza buono» spostato qui) e «Stime ore e buoni» sempre visibili sotto i campi; ipotesi sul calendario in tendina.

**v4.2**
- Straordinari diurni, festivi e notturni nel box dell'orario (tendina facoltativa), con imposta sostitutiva 15% 2026 su festivo e notturno, effetto su netto, costo e paga oraria. Rimossa la tendina «Ipotesi su bonus, welfare e buoni pasto».

**v4.1**
- Profilo fiscale come scheda fissa: territorio, coniuge, figli (con quota) e detrazioni automatiche (menu a comparsa) su un'unica riga.
- Bonus e destinazione del TFR affiancati in due box; etichette dello smart working accorciate; campi allineati anche quando le etichette vanno a capo.

**v4.0**
- Pagine scambiate e rinominate: la pagina iniziale è **01 · Analisi stipendio** (nel testo di questo documento «Dashboard 2»), la seconda è **02 · Aliquota marginale e aumenti** («Dashboard 1»). Le formule non cambiano.
- Profilo fiscale aperto di base, con territorio e familiari affiancati; detrazioni automatiche, requisiti del premio di risultato, destinazione del TFR e spiegazioni lunghe in tendine chiuse con un riepilogo nell'intestazione.

**v3.9**
- Colori di ①②③ guidati dal perimetro «Valore per te» al posto della trasparenza; ② blu viola e ③ blu anche nei box della pagina 01.
- Rimossa la nota sulla pensione pubblica accanto al valore ①+②+③.

**v3.8**
- Campo svuotato = 0 (nessun errore; il valore 0 viene scritto quando si esce dal campo). Passo minimo 1 per frecce e pulsanti, tranne nei parametri fiscali.
- Barra di composizione senza spostamenti quando si apre una sezione; «100%» senza decimale; campi della retribuzione allineati.
- Controllo generale dei calcoli: costo e netto ricostruiti a mano coincidono con il motore al centesimo; verificate sulle fonti ufficiali l'addizionale Lazio 2026 (1,73% fino a 28.000 €; oltre, 1,73% fino a 15.000 € e 3,33% sul resto; detrazione 60 € fra 28.001 e 30.000 €), la soglia dell'1% aggiuntivo (56.224 €) e la base del recupero 0,50% sul TFR.

**v3.7**
- Selettore «Valore per te: ① · ①+② · ①+②+③»; barra con colori pieni per ③ e ④, sfumature solo a sezione aperta.
- Campi numerici: la rotellina non cambia i valori; frecce e pulsanti avanzano di un passo proporzionale (10^(cifre intere − 2): 2.000 → ±100, 543 → ±10, 30 → ±1; minimo 0,5 per lo smart working).
- Permessi iniziali: 38 ore.

**v3.6**
- Sezione ③ «Contributi pensionistici IVS» (parte lavoratore e parte azienda, blu) separata da ④ «Imposte e assicurazioni sociali» (arancione); stessa scomposizione nei box degli aumenti della pagina 01.
- Decimali a grandezza piena nella card oraria.
- Valori iniziali: RAL 30.000 €, 27 giorni di ferie ed ex festività, 1 giorno di smart working a settimana.

**v3.5**
- Asse del grafico marginale in euro su 1.000 € e scala senza tagli; vista «Cuneo» con attivi solo prelievo marginale e quota personale al fondo; legenda dei valori annui nello stesso stile, con importi alla RAL selezionata.
- Pagina 02: rimosso il rapporto sulla RAL accanto a quello sul costo; selettore «Percentuali su: costo aziendale | RAL» per tutte le quote delle card.

**v3.4**
- ② al netto delle imposte future stimate, spostate in ③; solidarietà accorpata ai contributi azienda e INAIL; sezioni ①②③ richiudibili con totale e quota affiancati; «= 100%» sul costo.
- Corretto il bug per cui le aliquote illustrative di fondo e TFR non si aggiornavano.
- Pagina 01: prospettiva «quanto resta» (predefinita) / «cuneo»; scala −30%/110% con marcatori; maniglia dell'aumento sopra il grafico; etichette dell'asse sotto la barra RAL; casella dell'aumento accanto al titolo; netto rinominato «dalla sola RAL» con collegamento alla pagina 02.
- Pagina 02: costo per voce e prospetto analitico affiancati; spiegazioni come ⓘ; aliquote datoriali mostrate accanto alle singole voci.

**v3.3** — nessuna formula del motore `calculate` è cambiata; cambiano presentazione e indicatori derivati.
- Grafico marginale: scomposizione predefinita del prelievo (imposte nette + contributi lavoratore) e contributi a carico dell'azienda; nuove curve opzionali cuneo marginale sul costo, netto/ΔRAL, netto/Δcosto; legenda raggruppata con valori alla RAL selezionata; scala con tacche arrotondate e taglio sotto −30%.
- Riepiloghi laterali e box degli aumenti con netto/ΔRAL, netto/Δcosto e scomposizione completa di ΔK.
- Dashboard 1: aliquota media al posto di netto/imponibile IRPEF; serie «imposte nette» nel grafico dei valori annui.
- Dashboard 2: card in tre sezioni ①②③ che si sommano al costo aziendale (rimosse dalla sezione costo le righe «versamenti al fondo» e «TFR», già in ②); rapporti netto/RAL e netto/costo; riquadro TFR accanto al bonus; approfondimenti ridotti a costo per voce e prospetto lordo→netto.
- Territori: menu raggruppato e pannello addizionali accanto al menu; corretto il testo che indicava valori iniziali 2% e 0,8% (sono 0%).
