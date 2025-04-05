---
title: Concatenazione di regole
teaching: 40
exercises: 30
---


::: questions

- "Come si combinano le regole in un flusso di lavoro?"
- "Come faccio a creare una regola con più ingressi e uscite?"

:::

::: objectives

- ""

:::

## Una pipeline di regole multiple

Ora abbiamo una regola che può generare output per qualsiasi valore di `-p` e per qualsiasi numero di task, dobbiamo solo chiamare Snakemake con i parametri che vogliamo:

```bash
snakemake --profile cluster_profile p_0.999/runs/amdahl_run_6.json
```

Questo però non è esattamente comodo, per generare un set di dati completo dobbiamo eseguire Snakemake molte volte con diversi target di file di output. Creiamo invece una regola che generi questi file per noi.

Il concatenamento delle regole in Snakemake è una questione di scelta di modelli di nomi di file che collegano le regole. Si tratta di un'arte: nella maggior parte dei casi ci sono diverse opzioni che funzionano:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  "p_{parallel_proportion}/runs/amdahl_run_6.json"
    shell:
        "echo {input} done > {output}"
```

::: challenge

La nuova regola non fa nulla, si assicura solo che venga creato il file desiderato. Non vale la pena eseguirla sul cluster. Come assicurarsi che venga eseguita solo sul nodo di accesso?

:::::: solution

Dobbiamo aggiungere la nuova regola alla nostra `localrules`:

```python
localrules: hostname_login, generate_run_files
```

:::

:::

Ora eseguiamo la nuova regola (ricordiamo che dobbiamo richiedere il file di output per nome, poiché il `output` nella nostra regola contiene un pattern jolly):

```bash
[ocaisa@node1 ~]$ snakemake --profile cluster_profile/ p_0.999_runs.txt
```

```output
Using profile cluster_profile/ for setting default command line arguments.
Building DAG of jobs...
Retrieving input from storage.
Using shell: /cvmfs/software.eessi.io/versions/2023.06/compat/linux/x86_64/bin/bash
Provided remote nodes: 3
Job stats:
job                   count
------------------  -------
amdahl_run                1
generate_run_files        1
total                     2

Select jobs to execute...
Execute 1 jobs...

[Tue Jan 30 17:39:29 2024]
rule amdahl_run:
    output: p_0.999/runs/amdahl_run_6.json
    jobid: 1
    reason: Missing output files: p_0.999/runs/amdahl_run_6.json
    wildcards: parallel_proportion=0.999, parallel_tasks=6
    resources: mem_mb=1000, mem_mib=954, disk_mb=1000, disk_mib=954,
               tmpdir=<TBD>, mem_mb_per_cpu=3600, runtime=2, mpi=mpiexec, tasks=6

mpiexec -n 6 amdahl --terse -p 0.999 > p_0.999/runs/amdahl_run_6.json
No SLURM account given, trying to guess.
Guessed SLURM account: def-users
Job 1 has been submitted with SLURM jobid 342 (log: /home/ocaisa/.snakemake/slurm_logs/rule_amdahl_run/342.log).
[Tue Jan 30 17:47:31 2024]
Finished job 1.
1 of 2 steps (50%) done
Select jobs to execute...
Execute 1 jobs...

[Tue Jan 30 17:47:31 2024]
localrule generate_run_files:
    input: p_0.999/runs/amdahl_run_6.json
    output: p_0.999_runs.txt
    jobid: 0
    reason: Missing output files: p_0.999_runs.txt;
            Input files updated by another job: p_0.999/runs/amdahl_run_6.json
    wildcards: parallel_proportion=0.999
    resources: mem_mb=1000, mem_mib=954, disk_mb=1000, disk_mib=954,
               tmpdir=/tmp, mem_mb_per_cpu=3600, runtime=2

echo p_0.999/runs/amdahl_run_6.json done > p_0.999_runs.txt
[Tue Jan 30 17:47:31 2024]
Finished job 0.
2 of 2 steps (100%) done
Complete log: .snakemake/log/2024-01-30T173929.781106.snakemake.log
```

Osservate i messaggi di log che Snakemake stampa nel terminale. Che cosa è successo?

1. Snakemake cerca una regola per creare `p_0.999_runs.txt`
1. Determina che "generate_run_files" può fare questo se `parallel_proportion=0.999`
1. Vede che l'input necessario è quindi `p_0.999/runs/amdahl_run_6.json`
1. Snakemake cerca una regola per creare `p_0.999/runs/amdahl_run_6.json`
1. Determina che "amdahl_run" può fare questo se `parallel_proportion=0.999` e `parallel_tasks=6`
1. Ora Snakemake ha raggiunto un file di input disponibile (in questo caso, non è richiesto alcun file di input), esegue entrambi i passaggi per ottenere l'output finale

Questo, in breve, è il modo in cui costruiamo i flussi di lavoro in Snakemake.

1. Definizione delle regole per tutte le fasi di elaborazione
1. Scegliere modelli di denominazione `input` e `output` che consentano a Snakemake di collegare le regole
1. Indica a Snakemake di generare i file di output finali

Se si è abituati a scrivere script regolari, ci vuole un po' di tempo per abituarsi. Invece di elencare i passi in ordine di esecuzione, si lavora sempre a ritroso** dal risultato finale desiderato. L'ordine delle operazioni è determinato dall'applicazione delle regole di corrispondenza dei pattern ai nomi dei file, non dall'ordine delle regole nel file Snake.

::: callout

## Prima le uscite?

L'approccio di Snakemake di lavorare a ritroso partendo dall'output desiderato per determinare il flusso di lavoro è il motivo per cui mettiamo le righe `output` per prime in tutte le nostre regole - per ricordarci che sono queste le prime cose che Snakemake guarda!

Molti utenti di Snakemake, e in effetti la documentazione ufficiale, preferiscono avere il `input` per primo, quindi in pratica si dovrebbe usare l'ordine che si ritiene più opportuno.

:::

::: callout

## Uscite `log` in Snakemake

Snakemake ha un campo di regole dedicato per le uscite che sono [file di log](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#log-files), e queste sono trattate per lo più come uscite normali, tranne per il fatto che i file di log non vengono rimossi se il lavoro produce un errore. Ciò significa che è possibile consultare il log per diagnosticare l'errore. In un flusso di lavoro reale questo può essere molto utile, ma in termini di apprendimento dei fondamenti di Snakemake ci atterremo ai normali campi `input` e `output`.

:::

::: callout

## Gli errori sono normali

Non scoraggiatevi se vedete degli errori quando testate per la prima volta le vostre nuove pipeline di Snakemake. Ci sono molte cose che possono andare storte quando si scrive un nuovo flusso di lavoro e di solito sono necessarie diverse iterazioni per ottenere le cose giuste. Un vantaggio dell'approccio di Snakemake rispetto ai normali script è che Snakemake fallisce rapidamente quando c'è un problema, invece di continuare e potenzialmente eseguire calcoli inutili su dati parziali o corrotti. Un altro vantaggio è che quando un passo fallisce si può riprendere tranquillamente da dove si era interrotto.

:::



::: keypoints

- "Snakemake collega le regole cercando iterativamente le regole che hanno ingressi mancanti"
- "Le regole possono avere più ingressi e/o uscite con nome"
- "Se un comando di shell non produce l'output previsto, Snakemake lo considererà un fallimento"

:::


