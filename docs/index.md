Come sanno coloro che frequentano il canale Slack *SmartHome* con il sottoscritto, di recente ho dovuto procurarmi un nuovo sistema di antifurto per la nuova casa in cui vivo. Tralascerò le tribolazioni che ho dovuto superare per trovare un prodotto che mi soddisfacesse: sappiate solo che alla fine ne ho trovato uno che aveva tutte le caratteristiche che cercavo, l'ho acquistato e montato.

Tutto *rose e fiori*, finchè il tarlo che ho citato sopra ha cominciato a scavare e farsi sentire, fino a convincermi che sarebbe stato **indispensabile** collegare il mio impianto alla già estesa domotica di casa, basata su Home Assistant.

!!! note ""
    Se esiste un "effetto collaterale" legato alla passione della Domotica, è sicuramente quello di non riuscire più ad accontertarsi facilmente.

Ecco che cominciavano a delinearsi i contorni di un nuovo progetto, riassumibili in:

- collegare il sistema di antifurto esistente ad Home Assistant;
- scrivere del software per potenziare le funzioni dell'antifurto, alla luce del nuovo collegamento appena creato.

Il primo punto merita un articolo a parte per interesse e complessità, e nel caso che vogliate seguirmi nella *tana del Bianconiglio* dipende dalle caratteristiche del vostro impianto. Vi basti sapere che nella mia configurazione mi ritrovo con due entità legate al mio antifurto:

Entità | Funzione
-------|---------
`switch.antifurto_template`|azionato come switch permette di inserire e disinserire l'antifurto: il suo stato è on ad antifurto inserito, off viceversa;
`binary_sensor.allarme_antifurto`|legato all'innesco vero e proprio dell'allarme: il suo stato è on se la sirena sta suonando, off viceversa.

Per scrivere la parte software del mio progetto ho deciso di cogliere l'occasione di continuare ad impratichirmi con [AppDaemon](https://appdaemon.readthedocs.io/en/stable/index.html): lo ritengo uno strumento estremamente potente e versatile, come avrò modo di dimostrare in seguito.

!!! info "Premessa"
    È bene chiarire fin da subito che non mi considero uno sviluppatore, e la mia conoscenza di Python è esclusivamente *strumentale* a soddisfare i miei scopi: ci saranno sicuramente molti modi più raffinati e *pythonistici* per ottenere ciò che ho ottenuto io, e mi scuso fin da subito per la mia ignoranza a riguardo.

Tornando al progetto, le funzioni *aggiuntive* che ho deciso di inserire nel mio Antifurto 2.0 sono:

- accendere le luci da me specificate nel caso in cui suoni la sirena;
- poter inserire/disinserire l'antifurto tramite badge o generico chip RFID;
- avvisarmi se sto aprendo una finestra ad antifurto inserito;
- avvisarmi se ho dimenticato di inserire l'antifurto dopo che siamo usciti da un certo tempo, o se siamo tutti a casa e la sera non l'abbiamo ancora inserito;
- potenziare il sistema di notifica su telefono degli eventi, inserendo la possibilità di usare notifiche *actionable* e *critical* di iOS.