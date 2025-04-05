---
title: Esecuzione di Snakemake sul cluster
teaching: 30
exercises: 20
---


::: objectives

- "Definire le regole da eseguire localmente e sul cluster"

:::

::: questions

- "Come si esegue la mia regola di Snakemake nel cluster?"

:::

Cosa succede quando vogliamo che la nostra regola venga eseguita sul cluster piuttosto che sul nodo di accesso? Il cluster che stiamo usando usa Slurm e si dà il caso che Snakemake abbia un supporto integrato per Slurm, dobbiamo solo dirgli che vogliamo usarlo.

Snakemake utilizza l'opzione `executor` per consentire di selezionare il plugin che si desidera eseguire la regola. Il modo più rapido per applicarlo al proprio file Snake è definirlo sulla riga di comando. Proviamo

```bash
[ocaisa@node1 ~]$ snakemake -j1 -p --executor slurm hostname_login
```

```output
Building DAG of jobs...
Retrieving input from storage.
Nothing to be done (all requested files are present and up to date).
```

Non è successo nulla! Perché no? Quando gli viene chiesto di costruire un target, Snakemake controlla il "tempo di ultima modifica" del target e delle sue dipendenze. Se una dipendenza è stata aggiornata dopo il target, le azioni vengono rieseguite per aggiornare il target. Utilizzando questo approccio, Snakemake sa che deve ricostruire solo i file che, direttamente o indirettamente, dipendono dal file che è stato modificato. Questo è chiamato _costruzione incrementale_.

::: callout

## Le build incrementali migliorano l'efficienza

Ricostruendo i file solo quando necessario, Snakemake rende l'elaborazione più efficiente.

:::

::: challenge

## In esecuzione sul cluster

Ora abbiamo bisogno di un'altra regola che esegua la `hostname` sul _cluster_. Creare una nuova regola nel file Snake e provare a eseguirla sul cluster con le opzioni da `--executor slurm` a `snakemake`.

:::::: solution

La regola è quasi identica alla regola precedente, tranne che per il nome della regola e il file di output:

```python
rule hostname_remote:
    output: "hostname_remote.txt"
    input:
    shell:
        "hostname > hostname_remote.txt"
```

Si può quindi eseguire la regola con

```bash
[ocaisa@node1 ~]$ snakemake -j1 -p --executor slurm hostname_remote
```

```output
Building DAG of jobs...
Retrieving input from storage.
Using shell: /cvmfs/software.eessi.io/versions/2023.06/compat/linux/x86_64/bin/bash
Provided remote nodes: 1
Job stats:
job                count
---------------  -------
hostname_remote        1
total                  1

Select jobs to execute...
Execute 1 jobs...

[Mon Jan 29 18:03:46 2024]
rule hostname_remote:
    output: hostname_remote.txt
    jobid: 0
    reason: Missing output files: hostname_remote.txt
    resources: tmpdir=<TBD>

hostname > hostname_remote.txt
No SLURM account given, trying to guess.
Guessed SLURM account: def-users
No wall time information given. This might or might not work on your cluster.
If not, specify the resource runtime in your rule or as a reasonable default
via --default-resources. No job memory information ('mem_mb' or 
'mem_mb_per_cpu') is given - submitting without.
This might or might not work on your cluster.
Job 0 has been submitted with SLURM jobid 326 (log: /home/ocaisa/.snakemake/slurm_logs/rule_hostname_remote/326.log).
[Mon Jan 29 18:04:26 2024]
Finished job 0.
1 of 1 steps (100%) done
Complete log: .snakemake/log/2024-01-29T180346.788174.snakemake.log
```

Notate tutti gli avvertimenti che Snakemake ci dà sul fatto che la regola potrebbe non essere in grado di essere eseguita sul nostro cluster, perché forse non abbiamo fornito informazioni sufficienti. Fortunatamente per noi, questa regola funziona sul nostro cluster e possiamo dare un'occhiata al file di output creato dalla nuova regola, `hostname_remote.txt`:

```bash
[ocaisa@node1 ~]$ cat hostname_remote.txt
```

```output
tmpnode1.int.jetstream2.hpc-carpentry.org
```

::::::

:::

## Profilo di Snakemake

L'adattamento di Snakemake a un particolare ambiente può comportare molti flag e opzioni. Pertanto, è possibile specificare un profilo di configurazione da utilizzare per ottenere le opzioni predefinite. Questo profilo si presenta come

```bash
snakemake --profile myprofileFolder ...
```

La cartella del profilo deve contenere un file chiamato `config.yaml` che è quello che memorizzerà le nostre opzioni. La cartella può contenere anche altri file necessari per il profilo. Creiamo il file `cluster_profile/config.yaml` e inseriamo alcune delle nostre opzioni esistenti:

```yaml
printshellcmds: True
jobs: 3
executor: slurm
```

Ora dovremmo essere in grado di rilanciare il nostro flusso di lavoro puntando al profilo invece di elencare le opzioni. Per forzare l'esecuzione del nostro flusso di lavoro, dobbiamo prima rimuovere il file di output `hostname_remote.txt` e poi possiamo provare il nostro nuovo profilo

```bash
[ocaisa@node1 ~]$ rm hostname_remote.txt
[ocaisa@node1 ~]$ snakemake --profile cluster_profile hostname_remote
```

Il profilo è estremamente utile nel contesto del nostro cluster, poiché l'esecutore Slurm ha molte opzioni e a volte è necessario usarle per poter inviare lavori al cluster a cui si ha accesso. Sfortunatamente, i nomi delle opzioni in Snakemake non sono esattamente gli stessi di Slurm, quindi abbiamo bisogno dell'aiuto di una tabella di traduzione:

| SLURM             | Snakemake         | Description                                                    |
| ----------------- | ----------------- | -------------------------------------------------------------- |
| `--partition`     | `slurm_partition` | the partition a rule/job is to use                             |
| `--time`          | `runtime`         | the walltime per job in minutes                                |
| `--constraint`    | `constraint`      | may hold features on some clusters                             |
| `--mem`           | `mem, mem_mb`     | memory a cluster node must                                     |
|                   |                   | provide (mem: string with unit), mem_mb: int                   |
| `--mem-per-cpu`   | `mem_mb_per_cpu`  | memory per reserved CPU                                        |
| `--ntasks`        | `tasks`           | number of concurrent tasks / ranks                             |
| `--cpus-per-task` | `cpus_per_task`   | number of cpus per task (in case of SMP, rather use `threads`) |
| `--nodes`         | `nodes`           | number of nodes                                                |

Le avvertenze fornite da Snakemake suggeriscono che potrebbe essere necessario fornire queste opzioni. Un modo per farlo è fornirle come parte della regola di Snakemake usando la parola chiave `resources`, ad esempio,

```python
rule:
    input: ...
    output: ...
    resources:
        partition = <partition name>
        runtime = <some number>
```

e possiamo anche usare il profilo per definire valori predefiniti per queste opzioni da usare con il nostro progetto, usando la parola chiave `default-resources`. Ad esempio, la memoria disponibile sul nostro cluster è di circa 4 GB per core, quindi possiamo aggiungerla al nostro profilo:

```yaml
printshellcmds: True
jobs: 3
executor: slurm
default-resources:
  - mem_mb_per_cpu=3600
```

:::challenge

Sappiamo che il nostro problema viene eseguito in un tempo molto breve. Cambiare la durata predefinita dei nostri lavori a due minuti per Slurm.

::::::solution

```yaml
printshellcmds: True
jobs: 3
executor: slurm
default-resources:
  - mem_mb_per_cpu=3600
  - runtime=2
```

::::::

:::

Esistono varie opzioni `sbatch` non supportate direttamente dalle definizioni delle risorse nella tabella precedente. È possibile utilizzare la risorsa `slurm_extra` per specificare uno qualsiasi di questi flag aggiuntivi a `sbatch`:

```python
rule myrule:
    input: ...
    output: ...
    resources:
        slurm_extra="--mail-type=ALL --mail-user=<your email>"
```

## Esecuzione di regole locali

La nostra regola iniziale era di ottenere il nome host del nodo di accesso. Vogliamo sempre eseguire la regola sul nodo di accesso perché abbia senso. Se diciamo a Snakemake di eseguire tutte le regole tramite l'esecutore Slurm (cosa che stiamo facendo tramite il nostro nuovo profilo), questo non accadrà più. Quindi come si fa a forzare l'esecuzione della regola sul nodo di login?

Beh, nel caso in cui una regola di Snakemake esegua un compito banale, l'invio di un lavoro potrebbe essere eccessivo (ad esempio, meno di un minuto di tempo di calcolo). Come nel nostro caso, sarebbe meglio che queste regole venissero eseguite localmente (cioè dove viene eseguito il comando `snakemake`) invece che come lavoro. Snakemake consente di indicare quali regole devono essere sempre eseguite localmente con la parola chiave `localrules`. Definiamo `hostname_login` come regola locale vicino alla parte superiore del nostro `Snakefile`.

```python
localrules: hostname_login
```

::: keypoints

- "Snakemake genera e invia i propri script batch per lo scheduler"
- "È possibile memorizzare le impostazioni di configurazione predefinite in un profilo Snakemake"
- "`localrules` definisce le regole che vengono eseguite localmente e non vengono mai inviate a un cluster"

:::


