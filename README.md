# Atlante Stipendio

Simulatore interattivo di stipendio e tassazione marginale in Italia (snapshot normativo 2026).

## Utilizzo
Apri `index.html` nel browser. Non sono necessari server, build, npm o librerie esterne.

Il progetto contiene due viste collegate:
- **Crescita e tassazione:** prelievo marginale sulla RAL, grafico alternativo degli importi annui, simulazioni degli aumenti.
- **Stipendio e previdenza:** retribuzione, bonus, buoni pasto, welfare, costo aziendale, versamenti previdenziali e TFR, confrontati su base annuale, mensile e oraria.

## Parametri e limiti
- Il profilo fiscale e la RAL sono condivisi.
- Le addizionali Roma–Lazio sono incorporate. Per altri territori occorre inserire le aliquote, facendo riferimento ai dati MEF; la modalità manuale non simula automaticamente scaglioni/esenzioni locali articolate.
- Le percentuali contributive aziendali sono un **profilo di stima modificabile**, non aliquote universali.
- Le simulazioni della liquidazione futura di TFR e fondo pensione sono indicative, non previsioni né importi subito riscattabili.
- I benefit sono valorizzati nominalmente: non sempre equivalgono a contanti.
- Il tasto **Scarica HTML** conserva le personalizzazioni in una copia locale.

## Pubblicazione su Vercel
Questo sito è statico: importa il repository GitHub in Vercel, scegli **Framework Preset: Other**, nessun build command e **Output Directory: .** (radice). Il file `index.html` viene servito come homepage. Dopo la connessione, i push sul ramo principale attivano normalmente nuovi deploy.

## Avvertenza
Strumento didattico e orientativo, non sostituisce un cedolino, una consulenza fiscale o una verifica previdenziale. Verificare sempre il profilo contributivo specifico, il CCNL e l'anno normativo.

Realizzato da [Paolo Refuto](https://www.linkedin.com/in/paolo-refuto-6a5b52236/).
