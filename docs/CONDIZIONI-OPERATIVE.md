# Smart Absorption Radar – Condizioni operative (Pine v6, v1.0.0)

Documento di riferimento per `pine/smart_absorption_radar.pine`. Ogni parametro e' configurabile dagli input dell'indicatore.

## Stato di verifica

- Lo script **non e' stato compilato su TradingView** (in questo ambiente non esiste un compilatore Pine). E' stato scritto e rivisto staticamente.
- **Primo passo obbligatorio**: incollarlo nel Pine Editor, correggere eventuali errori di compilazione, poi eseguire la procedura di test (da consegnare con la documentazione finale).
- Nessun backtest e nessun paper trading eseguiti: **non usare con capitale reale**.

## Regole di non-repainting

- Tutta la logica di stato gira solo con `barstate.isconfirmed`. Nessun evento nasce da barre in formazione.
- HTF con "Solo candele HTF chiuse" attivo: `request.security(f()[1], lookahead_on)`, ultimo valore di una candela HTF chiusa.
- Con l'opzione disattivata il contesto usa la candela HTF aperta: `repainting_risk: true` nel JSON.
- Pivot (swing) confermati solo dopo `pivR` barre a destra.

## Contesto (RIALZISTA / RIBASSISTA / EQUILIBRIO)

Per ogni HTF (default 60 e 240): rialzista se ultimi due massimi e minimi pivot sono crescenti **e** close > EMA lenta **e** EMA veloce > EMA lenta. Ribassista speculare. Altrimenti neutro.

- "Entrambi allineati": RIALZISTA/RIBASSISTA solo se i due HTF concordano.
- "Superiore decide, intermedio non opposto": segue l'HTF superiore se l'intermedio non e' opposto.
- EQUILIBRIO, o HTF inferiore al TF del grafico: nessun ingresso.

## Zone di valutazione

Calcolate sul giorno del fuso configurato (massimo, minimo, apertura, range, VWAP cumulata).

- Sottovalutazione: parte bassa del range (default fino al 35%). Sopravvalutazione: dal 65%. Centro: equilibrio, mai zona di ingresso.
- Candela LONG: minimo nella zona bassa. Candela SHORT: massimo nella zona alta.
- Ingresso LONG solo sotto il 50% del range, SHORT solo sopra.
- Range valido solo con almeno `minDayBars` barre e ampiezza >= `minDayRangeATR` x ATR.
- Il VWAP richiede volume. Senza volume e' `null` e dichiarato in `missing_fields`.
- **Volume profile non implementato**: Pine non lo espone, nessun livello e' inventato.

## Swing e Fibonacci (0,705 – 0,886)

- Gamba LONG: pivot minimo -> pivot massimo; gamba SHORT: pivot massimo -> pivot minimo. Significativa se >= `swingMinATR` x ATR e >= `swingMinPct` %.
- Zona LONG: `B - r*(B-A)` con r = 0,705 e 0,886. SHORT: `B + r*(A-B)`.
- Il riferimento cambia solo se: swing violato (chiusura oltre A), nuovo estremo oltre B, o swing scaduto. Ogni cambio incrementa `swing_id`, archivia in grigio il disegno precedente e **invalida i setup in corso**. Nessuna riscrittura silenziosa.

## Rilevamento dell'assorbimento (candela)

Candela LONG (SHORT speculare), tutte vere, valutate a barra chiusa:

1. Nella zona di sottovalutazione e nella zona Fib (con tolleranza ATR).
2. Livello violato e recuperato: minimo sotto il minimo delle ultime `sweepN` barre, chiusura di nuovo sopra quel livello.
3. Range tra `minRangeATR` e `maxRangeATR` x ATR.
4. Chiusura nella parte alta (`minCloseLoc`) e ombra inferiore >= `minWick`.
5. `price_efficiency` = max(open-close, 0)/range <= `maxEff`.
6. Volume relativo >= `volMult` e sforzo/risultato (volume relativo / range in ATR) >= `minEffort`. Senza volume: nessun segnale, salvo `requireVolume` disattivato.
7. Mancata prosecuzione: per `followBars` barre successive nessun minimo oltre l'estremo della candela (tolleranza `followTolATR`).

Una candela rossa con volume alto **non** basta: servono recupero e discrepanza sforzo/risultato. "Assorbimento" resta una classificazione indiretta basata su OHLCV; order flow, delta, footprint e liquidazioni **non sono disponibili e non sono simulati**.

## Doppio fallimento

- Secondo tentativo distinto dal primo: distanza tra `minGap` e `maxGap` barre, rimbalzo di almeno `reboundATR` x ATR tra i due, estremo non piu' basso del primo oltre `levelTolATR` x ATR.
- Estremo piu' basso oltre la tolleranza: livello non tenuto, il vecchio setup decade e la nuova candela riparte come primo tentativo.
- Oltre `maxAttempts` tentativi: invalidazione `too_many_tests`.
- Invalidazioni: contesto cambiato o blocco giornaliero, swing invalido o cambiato, chiusura oltre il livello (`invalTolATR`), finestre scadute, tentativi eccessivi.

## Conferma, stop, target

- Conferma: chiusura oltre il massimo (o la meta') della candela del secondo tentativo, entro `confirmMaxBars`. Mai anticipata.
- Ingresso: chiusura della barra di conferma (prezzo di riferimento).
- Stop LONG: sotto il minimo piu' basso dei due tentativi, piu' `slAtrMargin` x ATR (SHORT speculare).
- Target: livello piu' vicino oltre `tpMinATR` x ATR tra massimo/minimo giornaliero, giorno precedente, apertura, VWAP, equilibrio, sessioni A/B (corrente e ultima), estremo dello swing e ultimo pivot. Alternative: estremo dello swing, o R fisso.
- Il target **non viene spostato** per ottenere il R:R: se non conviene, il setup e' scartato (`INVALIDATION` con il motivo).
- Motivi di scarto: `trade_already_open`, `daily_trades_limit`, `daily_stop_loss_limit`, `invalid_stop_geometry`, `stop_too_wide_atr`, `stop_too_wide_pct`, `entry_in_unfavorable_half_of_range`, `no_valid_target`, `risk_reward_below_minimum`, `position_size_zero`.

## Gestione del capitale

- Quantita' = (capitale x rischio%) / (distanza stop + slippage, per valore per punto + commissioni). Valore per punto: `syminfo.pointvalue` o override.
- Massimo 2 operazioni e 2 stop al giorno (configurabili), conteggio nel fuso scelto. Al raggiungimento scatta `DAILY_BLOCK` e i setup in corso vengono invalidati.
- Un'operazione alla volta. SL e TP nella stessa barra: si assume lo stop (ipotesi prudente).
- **Limite importante**: il conteggio e' una simulazione sui dati del grafico, non un registro di conto. Cambiando timeframe il conteggio si ricalcola sui nuovi dati. Il blocco reale e' applicato dal **server**, indicizzato per giorno e strumento e indipendente dal timeframe.
- Il capitale deve essere espresso nella valuta del simbolo: Pine non converte valute.

## Eventi JSON

`FIRST_ATTEMPT`, `SECOND_ATTEMPT`, `CONFIRMATION`, `INVALIDATION`, `DAILY_BLOCK`, piu' `TRADE_CLOSED` (aggiunto: serve al server per riconciliare gli stop giornalieri). Lo schema completo verra' consegnato con il server. Valori non disponibili restano `null` e il campo e' elencato in `missing_fields`.

## Pannello

CONTESTO, direzione di entrambi gli HTF, SETUP, DIREZIONE, DECISIONE CLAUDE, operazioni e stop giornalieri, rischio residuo, risultato del giorno, stato e avvisi sui dati. **DECISIONE CLAUDE e' un input manuale**: Pine non puo' ricevere la risposta del server.

## Limiti noti

- Nessuna verifica di compilazione, backtest o paper trading ancora eseguiti.
- Volume profile, order flow, delta, footprint e liquidazioni non disponibili in Pine.
- Parametri di default non ottimizzati: sono punti di partenza da validare.
- Il token webhook nel JSON e' un segreto condiviso, non una chiave API: tenere lo script privato.
