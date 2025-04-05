---
title: Esecuzione di comandi con Snakemake
teaching: 30
exercises: 30
---


::: questions

- "Come si esegue un semplice comando con Snakemake?"

:::

:::objectives

- "Creare una ricetta di Snakemake (uno Snakefile)"

:::

## Qual è il flusso di lavoro che mi interessa?

In questa lezione faremo un esperimento che prende un'applicazione che gira in parallelo e ne studia la scalabilità. Per farlo dovremo raccogliere dati, in questo caso significa eseguire l'applicazione più volte con un numero diverso di core della CPU e registrare il tempo di esecuzione. Una volta fatto questo, dobbiamo creare una visualizzazione dei dati per vedere come si confronta con il caso ideale.

Dalla visualizzazione possiamo decidere a quale scala ha più senso eseguire l'applicazione in produzione per massimizzare l'uso della CPU allocata nel sistema.

Potremmo fare tutto questo manualmente, ma ci sono strumenti utili per aiutarci a gestire le pipeline di analisi dei dati come quelle del nostro esperimento. Oggi ne conosceremo uno: Snakemake.

Per iniziare a usare Snakemake, iniziamo con un semplice comando e vediamo come eseguirlo tramite Snakemake. Scegliamo il comando `hostname` che stampa il nome dell'host in cui viene eseguito il comando:

```bash
[ocaisa@node1 ~]$ hostname
```

```output
node1.int.jetstream2.hpc-carpentry.org
```

Questo stampa il risultato, ma Snakemake si affida ai file per conoscere lo stato del flusso di lavoro, quindi reindirizziamo l'output a un file:

```bash
[ocaisa@node1 ~]$ hostname > hostname_login.txt
```

## Creazione di un file Snake

Modificare un nuovo file di testo chiamato `Snakefile`.

Contenuto di `Snakefile`:

```python
rule hostname_login:
    output: "hostname_login.txt"
    input:  
    shell:
        "hostname > hostname_login.txt"
```

::: callout

## Punti chiave di questo file

1. Il file si chiama `Snakefile` - con la maiuscola `S` e senza estensione.
1. Alcune righe sono rientrate. I rientri devono essere fatti con caratteri di spazio, non con tabulazioni. Si veda la sezione di impostazione per sapere come far sì che il proprio editor di testo faccia questo.
1. La definizione della regola inizia con la parola chiave `rule` seguita dal nome della regola, poi da due punti.
1. Abbiamo chiamato la regola `hostname_login`. Si possono usare lettere, numeri o trattini bassi, ma il nome della regola deve iniziare con una lettera e non può essere una parola chiave.
1. Le parole chiave `input`, `output` e `shell` sono tutte seguite da due punti (":").
1. I nomi dei file e il comando di shell sono tutti in `"quotes"`.
1. Il nome del file di output viene dato prima del nome del file di input. In realtà, a Snakemake non interessa l'ordine in cui appaiono, ma in questo corso daremo prima l'output. Vedremo presto perché.
1. In questo caso d'uso non c'è un file di input per il comando, quindi lo lasciamo vuoto.

:::

Torniamo alla shell ed eseguiamo la nostra nuova regola. A questo punto, se ci sono virgolette mancanti, rientri errati e così via, potremmo vedere un errore.

```bash
snakemake -j1 -p hostname_login
```

::: callout

## `bash: snakemake: command not found...`

Se la shell vi dice che non riesce a trovare il comando `snakemake`, allora dobbiamo rendere il software disponibile in qualche modo. Nel nostro caso, ciò significa cercare il modulo che dobbiamo caricare:

```bash
module spider snakemake
```

```output
[ocaisa@node1 ~]$ module spider snakemake

--------------------------------------------------------------------------------------------------------
  snakemake:
--------------------------------------------------------------------------------------------------------
     Versions:
        snakemake/8.2.1-foss-2023a
        snakemake/8.2.1 (E)

Names marked by a trailing (E) are extensions provided by another module.


--------------------------------------------------------------------------------------------------------
  For detailed information about a specific "snakemake" package (including how to load the modules) use the module's full name.
  Note that names that have a trailing (E) are extensions provided by other modules.
  For example:

     $ module spider snakemake/8.2.1
--------------------------------------------------------------------------------------------------------
```

Ora vogliamo il modulo, quindi carichiamolo per rendere disponibile il pacchetto

```bash
[ocaisa@node1 ~]$ module load snakemake
```

e poi assicurarsi di avere a disposizione il comando `snakemake`

```bash
[ocaisa@node1 ~]$ which snakemake
```

```output
/cvmfs/software.eessi.io/host_injections/2023.06/software/linux/x86_64/amd/zen3/software/snakemake/8.2.1-foss-2023a/bin/snakemake
```

```bash
snakemake -j1 -p hostname_login
```

:::

::: challenge

## Esecuzione di Snakemake

Eseguire `snakemake --help | less` per vedere la guida per tutte le opzioni disponibili. Che cosa fa l'opzione `-p` nel comando `snakemake` di cui sopra?

1. protegge i file di output esistenti
1. stampa i comandi di shell che vengono eseguiti sul terminale
1. Indica a Snakemake di eseguire solo un processo alla volta
1. chiede all'utente il file di input corretto

:::::: hint

È possibile cercare nel testo premendo <kbd>/</kbd>, e tornare alla shell con <kbd>q</kbd>.

::::::

:::::: solution

(2) Stampa i comandi di shell che vengono eseguiti sul terminale

Questa è una cosa così utile che non sappiamo perché non sia l'opzione predefinita! L'opzione `-j1` indica a Snakemake di eseguire solo un processo alla volta, e per ora ci atterremo a questa opzione perché rende le cose più semplici. La risposta 4 è un'assurdità, poiché Snakemake non richiede mai un input interattivo da parte dell'utente.

::::::


:::

::: keypoints

- "Prima di eseguire Snakemake è necessario scrivere uno Snakefile"
- "Uno Snakefile è un file di testo che definisce un elenco di regole"
- "Le regole hanno ingressi, uscite e comandi di shell da eseguire"
- "Si dice a Snakemake quale file creare e questo esegue il comando di shell definito nella regola appropriata"

:::


