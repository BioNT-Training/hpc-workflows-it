---
title: Segnaposto
teaching: 40
exercises: 30
---


::: questions

- "Come si crea una regola generica?"

:::

::: objectives

- "Vedere come Snakemake gestisce alcuni errori"

:::

Il nostro Snakefile presenta alcune duplicazioni. Per esempio, i nomi dei file di testo sono ripetuti in alcuni punti delle regole dello Snakefile. Gli Snakefile sono una forma di codice e, in qualsiasi codice, la ripetizione può portare a problemi (ad esempio, rinominiamo un file di dati in una parte dello Snakefile, ma dimentichiamo di rinominarlo altrove).

::: callout

## D.R.Y. (Don't Repeat Yourself)

In molti linguaggi di programmazione, la maggior parte delle caratteristiche del linguaggio serve a consentire al programmatore di descrivere lunghe routine di calcolo come codice breve, espressivo e bello. Le caratteristiche di Python, R o Java, come le variabili e le funzioni definite dall'utente, sono utili in parte perché ci evitano di dover scrivere (o pensare) tutti i dettagli più volte. Questa buona abitudine di scrivere le cose solo una volta è nota come principio "Non ripeterti" o D.R.Y.

:::

Rimuoviamo alcune ripetizioni dal nostro Snakefile.

## Segnaposto

Per creare una regola più generica abbiamo bisogno di **placeholder**. Vediamo l'aspetto di un segnaposto

```python
rule hostname_remote:
    output: "hostname_remote.txt"
    input:
    shell:
        "hostname > {output}"

```

Come promemoria, ecco la versione precedente dell'ultimo episodio:

```python
rule hostname_remote:
    output: "hostname_remote.txt"
    input:
    shell:
        "hostname > hostname_remote.txt"

```

La nuova regola ha sostituito i nomi di file espliciti con cose in `{curly brackets}`, in particolare `{output}` (ma avrebbe potuto essere anche `{input}`... se avesse avuto un valore e fosse stato utile).

### `{input}` e `{output}` sono **dei segnaposto**

I segnaposto sono usati nella sezione `shell` di una regola e Snakemake li sostituisce con valori appropriati - `{input}` con il nome completo del file di input e `{output}` con il nome completo del file di output - prima di eseguire il comando.

anche `{resources}` è un segnaposto e si può accedere a un elemento nominato di `{resources}` con la notazione `{resources.runtime}` (per esempio).

:::keypoints

- "Le regole di Snakemake sono rese più generiche con i segnaposto"
- "I segnaposto nella parte shell della regola vengono sostituiti con valori basati sui caratteri jolly scelti"

:::


