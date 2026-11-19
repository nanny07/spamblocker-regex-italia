# Prefissi DCM → SpamBlocker
 
Applicativo web per bloccare le chiamate dai numeri fissi assegnati a un operatore telefonico. Legge il file ufficiale delle numerazioni assegnate, estrae tutti i prefissi dell’operatore scelto e genera una regola in formato JSON da importare nell’app Android [SpamBlocker](https://github.com/aj3423/SpamBlocker).
 
Funziona anche al contrario: dato un numero di telefono, indica a quale operatore è assegnato.
 
Tutto avviene nel browser: il file Excel non viene caricato su nessun server.
 
## Il file di partenza
 
Il file da usare si scarica dal sito del Ministero delle Imprese e del Made in Italy, alla pagina [Risorse di numerazione](https://www.mimit.gov.it/index.php/it/comunicazioni/telefonia/risorse-di-numerazione), fra gli allegati, alla voce «File Assegnati».
 
Il file contiene molti fogli. L’applicativo usa solo il foglio **DCM**, e in particolare queste colonne:
 
| Colonna | Contenuto | Uso |
|---|---|---|
| C | Decamigliaio, prima cifra (sempre `0`) | prima parte del prefisso |
| D | Decamigliaio, indicativo distrettuale senza lo zero (es. `144`) | seconda parte del prefisso |
| E | Decamigliaio, cifre successive (es. `01`) | terza parte del prefisso |
| F | Operatore | nome dell’azienda da cercare |
 
Nel file l’intestazione «Decamigliaio» copre insieme le colonne C, D ed E. Le colonne A (Distretto) e B (Area Locale) vengono solo mostrate a video.
 
Il Ministero aggiorna il file periodicamente: conviene scaricare la versione più recente prima di generare nuove regole.
 
## Avvio rapido
 
1. Apri `dcm-spamblocker.html` in un browser (serve la connessione a internet per caricare la libreria che legge i file Excel).
2. Carica il file «File Assegnati» scaricato dal sito del Ministero.
3. Cerca l’operatore per nome (basta una parte, per esempio `enegan` o `retelit`) e selezionalo.
4. Premi **Copia JSON**.
5. In SpamBlocker apri **Impostazioni › Impostazioni regex › Regole del numero**, tocca **Nuovo**, scegli **Importa**, incolla il JSON e conferma con **Importa**.
## Funzioni
 
### Ricerca per operatore
 
La ricerca confronta il testo con la colonna F, senza distinguere maiuscole, minuscole e accenti. Si possono selezionare più operatori insieme, per esempio tutte le varianti di un gruppo: «Seleziona tutti i risultati» aggiunge in un colpo solo tutti quelli trovati.
 
### Ricerca per numero («Di chi è un numero?»)
 
Scrivi un numero fisso e l’applicativo mostra operatore, prefisso, distretto e area locale. Il numero può essere scritto in vari modi, per esempio `+39 0144 011234`, `0039 0144 011234`, `390144011234` oppure `0144011234`; spazi, trattini e parentesi vengono ignorati.
 
La ricerca individua il prefisso più lungo del foglio DCM con cui inizia il numero. Con **Aggiungi alla selezione** l’operatore trovato entra direttamente nella regola.
 
### Opzioni della regola
 
| Opzione | Scelte | Effetto |
|---|---|---|
| Formato del numero che chiama | Solo internazionale (predefinito) / Internazionale o nazionale | vedi [Come viene costruita la regex](#come-viene-costruita-la-regex) |
| Applica a | Chiamate e SMS (predefinito) / Solo chiamate | campo `flags` |
| Notifica quando blocca | Nessuno, Bassa (predefinita), Media, Alta, Alta silenziata, oppure un canale personalizzato | campo `channel` |
| Descrizione della regola | testo libero, proposto in automatico dal nome dell’operatore | campo `description` |
| Regex compatta | sì / no (predefinito) | stessa regola, testo più corto |
 
Il JSON si può copiare negli appunti o scaricare come file `.json`.
 
## Come viene costruita la regex
 
Ogni riga del foglio DCM produce un prefisso concatenando le colonne C, D ed E. Per esempio la riga con C = `0`, D = `144`, E = `01` dà il prefisso `014401`.
 
Tutti i prefissi dell’operatore vengono riuniti in un’unica regex:
 
```
39(?:014401|014402|…)\d+
```
 
dove `39` è il prefisso internazionale dell’Italia e `\d+` rappresenta il resto del numero.
 
La regex inizia con `39` e non con `\39` né con `\+39`, per due motivi:
 
- in una regex `\3` vale come «ripeti il gruppo 3», quindi `\39…` non riconoscerebbe mai alcun numero;
- prima del confronto SpamBlocker toglie il `+` e gli zeri iniziali, quindi `+39 0144 011234` e `0039 0144 011234` vengono confrontati entrambi come `390144011234`.
Con il formato **Solo internazionale** la regola riconosce quindi i numeri che arrivano come `+39…` o `0039…`.
 
Con il formato **Internazionale o nazionale** la regex diventa `(?:\+39|0039)?(?:014401|…)\d+` e la regola attiva l’opzione «Numero grezzo» di SpamBlocker (`patternFlags: 1024`), che confronta il numero così come arriva, senza togliere `+` e zeri. In questo modo vengono riconosciuti anche i numeri che arrivano in formato nazionale, come `0144 011234`.
 
Nella regex finale i prefissi doppi vengono eliminati, così come quelli già coperti da un prefisso più corto (con `\d+` alla fine, `014401` include già tutto ciò che inizia con `0144011`).
 
### Regex compatta
 
Per gli operatori con molti prefissi la regex diventa molto lunga: per Telecom Italia supera i 240.000 caratteri. L’opzione **Regex compatta** raggruppa le cifre iniziali comuni (per esempio `0(?:1(?:0786|11869|…)…)`) e scende a circa 37.000 caratteri. Le due forme bloccano esattamente gli stessi numeri.
 
## Formato del JSON
 
SpamBlocker non pubblica una specifica del formato. Questo è ricavato dal codice sorgente dell’app (classe `RegexRule`) ed è lo stesso che l’app accetta in **Regole del numero › Nuovo › Importa**.
 
```json
{
  "pattern": "39(?:014401|014402)\\d+",
  "patternFlags": 0,
  "description": "Blocco Enegan S.p.A.",
  "isBlacklist": true,
  "flags": 3,
  "channel": "Low"
}
```
 
| Campo | Valori | Significato |
|---|---|---|
| `pattern` | regex | numeri da bloccare |
| `patternFlags` | `0` oppure `1024` | `1024` attiva «Numero grezzo» |
| `description` | testo | nome della regola in SpamBlocker |
| `isBlacklist` | `true` | la regola blocca (non autorizza) |
| `flags` | `3` oppure `1` | `3` chiamate e SMS, `1` solo chiamate |
| `channel` | `None`, `Low`, `Medium`, `High`, `High Muted` o l’ID di un canale creato nell’app | livello di notifica quando la regola blocca |
 
I campi non presenti prendono i valori predefiniti di SpamBlocker.
 
## Limiti
 
- **Una regola per volta.** SpamBlocker importa il JSON di una singola regola incollandolo nel campo di testo; non lo legge da file. Il file `.json` scaricato serve per conservarlo o per passarlo al telefono.
- **Solo numeri fissi.** Il foglio DCM contiene le numerazioni geografiche (che iniziano con `0`). Cellulari, numeri verdi e altre numerazioni non sono coperti.
- **Portabilità.** Il file indica l’operatore a cui è stato assegnato un blocco di numeri. Se un numero è passato a un altro operatore, il file non lo riporta.
- **Ripristino del backup non usato.** SpamBlocker può importare un backup completo da file (`.gz`), ma il ripristino cancella tutte le regole numeriche esistenti: per questo l’applicativo genera solo la singola regola.
## Dettagli tecnici
 
- Un solo file HTML con CSS e JavaScript incorporati, senza installazione né build.
- Lettura dei file Excel con [SheetJS](https://sheetjs.com/) 0.18.5, caricato da cdnjs (con jsDelivr come riserva). Viene letto solo il foglio DCM, così anche file di qualche megabyte si aprono in pochi secondi.
- Font da Google Fonts, con caratteri di sistema come riserva.
- Tema chiaro e scuro automatici, in base alle impostazioni del dispositivo.
La logica principale si trova nell’oggetto `DcmCore` dentro la pagina:
 
| Funzione | Cosa fa |
|---|---|
| `parseRows` | legge le righe del foglio DCM e costruisce i prefissi C+D+E |
| `groupByOperator` | raggruppa le righe per operatore (colonna F) |
| `buildPrefixes` | elimina prefissi doppi o già coperti |
| `buildPattern` / `trieGroup` | genera la regex, normale o compatta |
| `buildRule` | genera l’oggetto JSON per SpamBlocker |
| `parseItalianNumber` / `lookupNumber` | interpretano un numero e trovano l’operatore |
 
## Riferimenti
 
- Ministero delle Imprese e del Made in Italy, [Risorse di numerazione](https://www.mimit.gov.it/index.php/it/comunicazioni/telefonia/risorse-di-numerazione) (allegato «File Assegnati»)
- SpamBlocker: <https://github.com/aj3423/SpamBlocker>
 
