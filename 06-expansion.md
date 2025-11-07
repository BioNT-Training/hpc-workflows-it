---
title: Elaborazione di elenchi di input
teaching: 50
exercises: 30
---


::: questions

- "Come posso elaborare più file contemporaneamente?"
- "Come si combinano più file insieme?"

:::

::: objectives

- "Usare Snakemake per elaborare tutti i campioni in una volta sola"
- "Creare un grafico di scalabilità che riunisca i nostri risultati"

:::

Abbiamo creato una regola che può generare un singolo file di output, ma non abbiamo intenzione di creare più regole per ogni file di output. Vogliamo generare tutti i file di esecuzione con una sola regola, se possibile, ma Snakemake può accettare un elenco di file di input:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  "p_{parallel_proportion}/runs/amdahl_run_2.json", "p_{parallel_proportion}/runs/amdahl_run_6.json"
    shell:
        "echo {input} done > {output}"
```

È fantastico, ma non vogliamo dover elencare singolarmente tutti i file che ci interessano. Come possiamo fare?

## Definizione di un elenco di campioni da elaborare

Per fare questo, possiamo definire alcuni elenchi come **variabili globali** di Snakemake.

Le variabili globali devono essere aggiunte prima delle regole nel file Snake.

```python
# Dimensioni dei task che vogliamo eseguire
NTASK_SIZES = [1, 2, 3, 4, 5]
```

- A differenza di quanto avviene con le variabili negli script di shell, possiamo mettere degli spazi intorno al segno `=`, ma non sono obbligatori.
- Gli elenchi di stringhe quotate sono racchiusi tra parentesi quadre e separati da virgole. Se conoscete un po' di Python, riconoscerete che si tratta della sintassi delle liste di Python.
- Una buona convenzione è quella di usare nomi in maiuscolo per queste variabili, ma non è obbligatorio.
- Sebbene questi elenchi siano definiti variabili, non è possibile modificare i valori una volta che il flusso di lavoro è in esecuzione, quindi gli elenchi definiti in questo modo sono più simili a costanti.

## Utilizzo di una regola di Snakemake per definire un gruppo di uscite

Ora aggiorniamo il nostro Snakefile per sfruttare la nuova variabile globale e creare un elenco di file:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  expand("p_{{parallel_proportion}}/runs/amdahl_run_{count}.json", count=NTASK_SIZES)
    shell:
        "echo {input} done > {output}"
```

La funzione `expand(...)` di questa regola genera un elenco di nomi di file, prendendo la prima cosa tra le parentesi singole come modello e sostituendo `{count}` con tutti gli `NTASK_SIZES`. Poiché ci sono 5 elementi nell'elenco, si otterranno 5 file che vogliamo creare. Si noti che abbiamo dovuto proteggere il nostro carattere jolly in una seconda serie di parentesi, in modo che non venisse interpretato come qualcosa che doveva essere espanso.

Nel nostro caso attuale ci basiamo ancora sul nome del file per definire il valore del carattere jolly `parallel_proportion`, quindi non possiamo chiamare direttamente la regola, ma dobbiamo richiedere un file specifico:

```bash
snakemake --profile cluster_profile/ p_0.999_runs.txt
```

Se non si specifica un nome di regola di destinazione o un nome di file sulla riga di comando quando si esegue Snakemake, l'impostazione predefinita è di usare **la prima regola** nel file Snake come destinazione.

::: callout

## Regole come target

Dare il nome di una regola a Snakemake sulla riga di comando funziona solo se quella regola non ha *parole jolly* in uscita, perché Snakemake non ha modo di sapere quali potrebbero essere le parole jolly desiderate. Verrà visualizzato l'errore "Le regole di destinazione non possono contenere caratteri jolly" Questo può accadere anche quando non si fornisce alcun target esplicito sulla riga di comando e Snakemake cerca di eseguire la prima regola definita nel file Snake.

:::

## Regole che combinano più input

La nostra regola `generate_run_files` è una regola che accetta un elenco di file di input. La lunghezza di tale elenco non è fissata dalla regola, ma può cambiare in base a `NTASK_SIZES`.

Nel nostro flusso di lavoro il passo finale consiste nel prendere tutti i file generati e combinarli in una trama. Per fare questo, forse avete sentito che alcune persone usano una libreria python chiamata `matplotlib`. Non è compito di questo tutorial scrivere lo script python per creare un grafico finale, quindi vi forniremo lo script come parte di questa lezione. È possibile scaricarlo con

```bash
curl -O https://ocaisa.github.io/hpc-workflows/files/plot_terse_amdahl_results.py
```

Lo script `plot_terse_amdahl_results.py` ha bisogno di una riga di comando che assomiglia a:

```bash
python plot_terse_amdahl_results.py --output <output image filename> <1st input file> <2nd input file> ...
```

Introduciamo questa funzione nella nostra regola `generate_run_files`:

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_runs.txt"
    input:  expand("p_{{parallel_proportion}}/runs/amdahl_run_{count}.json", count=NTASK_SIZES)
    shell:
        "python plot_terse_amdahl_results.py --output {output} {input}"
```

::: challenge

Questo script si basa su `matplotlib`, è disponibile come modulo d'ambiente? Aggiungere questo requisito alla regola.

:::::: solution

```python
rule generate_run_files:
    output: "p_{parallel_proportion}_scalability.jpg"
    input:  expand("p_{{parallel_proportion}}/runs/amdahl_run_{count}.json", count=NTASK_SIZES)
    envmodules:
      "matplotlib"
    shell:
        "python plot_terse_amdahl_results.py --output {output} {input}"
```

::::::

:::

Ora finalmente possiamo generare un grafico in scala! Eseguire il comando finale di Snakemake:

```bash
snakemake --profile cluster_profile/ p_0.999_scalability.jpg
```

::: challenge

Generare il grafico della scalabilità per tutti i valori da 1 a 10 core.

:::::: solution

```python
NTASK_SIZES = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

::::::

:::

::: challenge

Eseguire nuovamente il flusso di lavoro per un valore `p` di 0,8

:::::: solution

```bash
snakemake --profile cluster_profile/ p_0.8_scalability.jpg
```

::::::

:::

::: challenge

## Bonus round

Creare una regola finale che possa essere richiamata direttamente e che generi un grafico in scala per 3 diversi valori di `p`.

:::

::: keypoints

- "Utilizzare la funzione `expand()` per generare elenchi di nomi di file da combinare"
- "Ogni `{input}` a una regola può essere un elenco di lunghezza variabile"

:::


