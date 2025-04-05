---
title: Applicazioni MPI e Snakemake
teaching: 30
exercises: 20
---


::: objectives

- "Definire le regole da eseguire localmente e sul cluster"

:::

::: questions

- "Come si esegue un'applicazione MPI tramite Snakemake sul cluster?"

:::

Ora è il momento di tornare al nostro flusso di lavoro reale. Possiamo eseguire un comando sul cluster, ma che dire dell'esecuzione dell'applicazione MPI che ci interessa? La nostra applicazione si chiama `amdahl` ed è disponibile come modulo d'ambiente.

::: challenge

Individuare e caricare il modulo `amdahl` e quindi _sostituire_ la nostra regola `hostname_remote` con una versione che esegue `amdahl`. (Non preoccupatevi ancora dell'MPI parallelo, ma eseguitelo con una sola CPU, `mpiexec -n 1 amdahl`).

La regola viene eseguita correttamente? In caso contrario, cercare nei file di log per scoprirne il motivo?

::::::solution

```bash
module spider amdahl
module load amdahl
```

individuerà e caricherà il modulo `amdahl`. Possiamo quindi aggiornare/sostituire la nostra regola per eseguire l'applicazione `amdahl`:

```python
rule amdahl_run:
    output: "amdahl_run.txt"
    input:
    shell:
        "mpiexec -n 1 amdahl > {output}"
```

Tuttavia, quando si tenta di eseguire la regola si ottiene un errore (a meno che non sia già disponibile una versione diversa di `amdahl` nel proprio percorso). Snakemake riporta la posizione dei log e se guardiamo all'interno possiamo (alla fine) trovare

```output
...
mpiexec -n 1 amdahl > amdahl_run.txt
--------------------------------------------------------------------------
mpiexec was unable to find the specified executable file, and therefore
did not launch the job.  This error was first reported for process
rank 0; it may have occurred for other processes as well.

NOTE: A common cause for this error is misspelling a mpiexec command
      line parameter option (remember that mpiexec interprets the first
      unrecognized command line token as the executable).

Node:       tmpnode1
Executable: amdahl
--------------------------------------------------------------------------
...
```

Quindi, anche se abbiamo caricato il modulo prima di eseguire il flusso di lavoro, la nostra regola Snakemake non ha trovato l'eseguibile. Questo perché la regola Snakemake viene eseguita in un ambiente _runtime_ pulito e dobbiamo dirgli in qualche modo di caricare il modulo ambiente necessario prima di provare a eseguire la regola.

::::::


:::

## Snakemake e moduli di ambiente

La nostra applicazione si chiama `amdahl` ed è disponibile sul sistema tramite un modulo d'ambiente, quindi dobbiamo dire a Snakemake di caricare il modulo prima di provare a eseguire la regola. Snakemake conosce i moduli d'ambiente, che possono essere specificati tramite (un'altra) opzione:

```python
rule amdahl_run:
    output: "amdahl_run.txt"
    input:
    envmodules:
      "mpi4py",
      "amdahl"
    input:
    shell:
        "mpiexec -n 1 amdahl > {output}"
```

L'aggiunta di queste righe non è tuttavia sufficiente a far eseguire la regola. Non solo bisogna dire a Snakemake quali moduli caricare, ma bisogna anche dirgli di usare i moduli d'ambiente in generale (poiché si ritiene che l'uso dei moduli d'ambiente renda l'ambiente di runtime meno riproducibile, dato che i moduli disponibili possono differire da cluster a cluster). Per questo è necessario dare a Snakemake un'opzione aggiuntiva

```bash
snakemake --profile cluster_profile --use-envmodules amdahl_run
```

::: challenge

Utilizzeremo i moduli d'ambiente per tutto il resto del tutorial, quindi rendiamola un'opzione predefinita del nostro profilo (impostando il suo valore a `True`)

::::::solution

Aggiornare il profilo del cluster a

```yaml
printshellcmds: True
jobs: 3
executor: slurm
default-resources:
  - mem_mb_per_cpu=3600
  - runtime=2
use-envmodules: True
```

Se si vuole testare, è necessario cancellare il file di output della regola e rieseguire Snakemake.

::::::

:::

## Snakemake e MPI

Nell'ultima sezione non abbiamo eseguito un'applicazione MPI, in quanto abbiamo eseguito solo su un core. Come si fa a richiedere l'esecuzione su più core per una singola regola?

Snakemake ha un supporto generale per MPI, ma l'unico esecutore che attualmente supporta esplicitamente MPI è l'esecutore Slurm (per nostra fortuna!). Se guardiamo alla nostra tabella di traduzione da Slurm a Snakemake, notiamo che le opzioni rilevanti appaiono in fondo:

| SLURM             | Snakemake       | Description                                                    |
| ----------------- | --------------- | -------------------------------------------------------------- |
| ...               | ...             | ...                                                            |
| `--ntasks`        | `tasks`         | number of concurrent tasks / ranks                             |
| `--cpus-per-task` | `cpus_per_task` | number of cpus per task (in case of SMP, rather use `threads`) |
| `--nodes`         | `nodes`         | number of nodes                                                |

L'opzione che ci interessa è `tasks`, poiché aumenteremo solo il numero di ranghi. Possiamo definirli in una sezione `resources` della nostra regola e fare riferimento ad essi usando dei segnaposto:

```python
rule amdahl_run:
    output: "amdahl_run.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi='mpiexec',
      tasks=2
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```

Questo ha funzionato, ma ora abbiamo un piccolo problema. Vogliamo farlo per alcuni valori diversi di `tasks`, il che significa che abbiamo bisogno di un file di output diverso per ogni esecuzione. Sarebbe fantastico se potessimo indicare in qualche modo nel file `output` il valore che vogliamo usare per `tasks`... e far sì che Snakemake lo prenda.

Potremmo usare un _wildcard_ nel `output` per permetterci di definire il `tasks` che vogliamo usare. La sintassi di questo carattere jolly è la seguente

```python
output: "amdahl_run_{parallel_tasks}.txt"
```

dove `parallel_tasks` è il nostro carattere jolly.

::: callout

## Caratteri jolly

I caratteri jolly sono usati nelle righe `input` e `output` della regola per rappresentare parti di nomi di file. Come lo schema `*` nella shell, il carattere jolly può sostituire qualsiasi testo per comporre il nome del file desiderato. Come per la denominazione delle regole, si può scegliere un nome a piacere per i caratteri jolly, qui abbiamo usato `parallel_tasks`. L'uso degli stessi caratteri jolly nell'input e nell'output indica a Snakemake come abbinare i file di input a quelli di output.

Se due regole utilizzano un carattere jolly con lo stesso nome, Snakemake le tratterà come entità diverse - le regole in Snakemake sono autocontenute in questo modo.

Nella riga `shell` si può fare riferimento al carattere jolly con `{wildcards.parallel_tasks}`

:::

## Ordine delle operazioni di Snakemake

Abbiamo appena iniziato con alcune semplici regole, ma vale la pena di pensare a cosa fa esattamente Snakemake quando lo si esegue. Ci sono tre fasi distinte:

1. Prepara l'esecuzione:
    1. Legge tutte le definizioni delle regole dal file Snakefile
1. Pianifica cosa fare:
    1. vede quale/i file si sta chiedendo di creare
    1. Cerca una regola corrispondente guardando le `output` di tutte le regole che conosce
    1. Compila i caratteri jolly per ottenere il valore `input` per questa regola
    1. verifica che questo file di input (se richiesto) sia effettivamente disponibile
1. Esegue i passi:
    1. Crea la directory per il file di output, se necessario
    1. Rimuove il vecchio file di output, se è già presente
    1. Solo allora, esegue il comando di shell con i segnaposto sostituiti
    1. controlla che il comando sia stato eseguito senza errori *e* che il nuovo file di output sia stato creato come previsto

::: callout

## Modalità di esecuzione a secco (`-n`)

Spesso è utile eseguire solo le prime due fasi, in modo che Snakemake pianifichi i lavori da eseguire e li stampi sullo schermo, ma non li esegua mai. Questo viene fatto con il flag `-n`, ad esempio:

```bash
> $ snakemake -n ...
```

:::

La quantità di controlli può sembrare pedante in questo momento, ma con l'aumentare dei passi del flusso di lavoro questo diventerà davvero molto utile.

## Usando i caratteri jolly nella nostra regola

Vorremmo usare un carattere jolly nel `output` per permetterci di definire il numero di `tasks` che vogliamo usare. Sulla base di quanto visto finora, si potrebbe immaginare che questo potrebbe essere simile a

```python
rule amdahl_run:
    output: "amdahl_run_{parallel_tasks}.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      tasks="{parallel_tasks}"
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```

ma ci sono due problemi:

- L'unico modo per Snakemake di conoscere il valore del carattere jolly è che l'utente richieda esplicitamente un file di output concreto (invece di chiamare la regola):

```bash
  snakemake --profile cluster_profile amdahl_run_2.txt
```

questo è perfettamente valido, poiché Snakemake è in grado di capire che ha una regola che può corrispondere a quel nome di file.

- Il problema maggiore è che anche così non funziona, sembra che non si possa usare un carattere jolly per `tasks`:

  ```output
  WorkflowError:
  SLURM job submission failed. The error message was sbatch:
  error: Invalid numeric value "{parallel_tasks}" for --ntasks.
  ```

Purtroppo per noi, non c'è un modo diretto per accedere ai caratteri jolly per `tasks`. Il motivo è che Snakemake cerca di usare il valore di `tasks` durante la fase di inizializzazione, quindi prima di conoscere il valore del carattere jolly. È necessario rimandare la determinazione di `tasks` a un momento successivo. Questo si può ottenere specificando una funzione di input invece di un valore per questo scenario. La soluzione è quindi quella di scrivere una funzione da usare una sola volta per manipolare Snakemake in modo che lo faccia per noi. Poiché la funzione è specifica per la regola, possiamo usare una funzione di una riga senza nome. Questo tipo di funzioni sono chiamate funzioni anonime o funzioni lamdba (entrambe hanno lo stesso significato) e sono una caratteristica di Python (e di altri linguaggi di programmazione).

Per definire una funzione lambda in python, la sintassi generale è la seguente:

```python
lambda x: x + 54
```

Poiché la nostra funzione può accettare i caratteri jolly come argomenti, possiamo usarli per impostare il valore di `tasks`:

```python
rule amdahl_run:
    output: "amdahl_run_{parallel_tasks}.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```

Ora abbiamo una regola che può essere usata per generare output da esecuzioni di un numero arbitrario di task paralleli.

::: callout

## Commenti in Snakefiles

Nel codice precedente, la riga che inizia con `#` è una riga di commento. Si spera che abbiate già l'abitudine di aggiungere commenti ai vostri script. Un buon commento rende qualsiasi script più leggibile, e questo vale anche per Snakefiles.

:::

Poiché la nostra regola è ora in grado di generare un numero arbitrario di file di output, le cose potrebbero diventare molto affollate nella nostra directory corrente. Probabilmente è meglio mettere le esecuzioni in una cartella separata per mantenere l'ordine. Possiamo aggiungere la cartella direttamente al nostro `output` e Snakemake si occuperà della creazione della directory per noi:

```python
rule amdahl_run:
    output: "runs/amdahl_run_{parallel_tasks}.txt"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl > {output}"
```

::: challenge

Creare un file di output (sotto la cartella `runs`) per il caso in cui si abbiano 6 task paralleli

(SUGGERIMENTO: ricordate che Snakemake deve essere in grado di far corrispondere il file richiesto al `output` di una regola)

:::::: solution

```bash
snakemake --profile cluster_profile runs/amdahl_run_6.txt
```

::::::

:::

Un'altra cosa della nostra applicazione `amdahl` è che alla fine vogliamo elaborare l'output per generare il nostro grafico in scala. L'output in questo momento è utile per la lettura, ma rende più difficile l'elaborazione. in `amdahl` c'è un'opzione che ci facilita questo compito. Per vedere le opzioni di `amdahl` si può usare

```bash
[ocaisa@node1 ~]$ module load amdahl
[ocaisa@node1 ~]$ amdahl --help
```

```output
usage: amdahl [-h] [-p [PARALLEL_PROPORTION]] [-w [WORK_SECONDS]] [-t] [-e]

options:
  -h, --help            show this help message and exit
  -p [PARALLEL_PROPORTION], --parallel-proportion [PARALLEL_PROPORTION]
                        Parallel proportion should be a float between 0 and 1
  -w [WORK_SECONDS], --work-seconds [WORK_SECONDS]
                        Total seconds of workload, should be an integer greater than 0
  -t, --terse           Enable terse output
  -e, --exact           Disable random jitter
```

L'opzione che stiamo cercando è `--terse`, che farà stampare l'output di `amdahl` in un formato molto più facile da elaborare, JSON. Il formato JSON in un file usa tipicamente l'estensione `.json`, quindi aggiungiamo questa opzione al nostro comando `shell` e cambiamo il formato del file `output` per adattarlo al nostro nuovo comando:

```python
rule amdahl_run:
    output: "runs/amdahl_run_{parallel_tasks}.json"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl --terse > {output}"
```

C'era un altro parametro per `amdahl` che ha attirato la mia attenzione. `amdahl` ha un'opzione `--parallel-proportion` (o `-p`) che potremmo essere interessati a cambiare perché modifica il comportamento del codice e quindi ha un impatto sui valori che otteniamo nei nostri risultati. Aggiungiamo un altro livello di directory al nostro formato di output per riflettere una scelta particolare per questo valore. Possiamo usare un carattere jolly in modo da non dover scegliere subito il valore:

```python
rule amdahl_run:
    output: "p_{parallel_proportion}/runs/amdahl_run_{parallel_tasks}.json"
    input:
    envmodules:
      "amdahl"
    resources:
      mpi="mpiexec",
      # No direct way to access the wildcard in tasks, so we need to do this
      # indirectly by declaring a short function that takes the wildcards as an
      # argument
      tasks=lambda wildcards: int(wildcards.parallel_tasks)
    input:
    shell:
        "{resources.mpi} -n {resources.tasks} amdahl --terse -p {wildcards.parallel_proportion} > {output}"
```

::: challenge

Crea un file di output per un valore di `-p` di 0,999 (il valore predefinito è 0,8) per il caso in cui abbiamo 6 task paralleli.

:::::: solution

```bash
snakemake --profile cluster_profile p_0.999/runs/amdahl_run_6.json
```

::::::

:::

::: keypoints

- "Snakemake sceglie la regola appropriata sostituendo i caratteri jolly in modo che l'output corrisponda all'obiettivo"
- "Snakemake controlla varie condizioni di errore e si ferma se vede un problema"

:::


