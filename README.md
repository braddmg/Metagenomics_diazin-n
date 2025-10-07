# Metagenomics diazinon

## Quality control
The first step in metagenomic data analysis is to perform quality control on the raw reads. For this, we use [fastp](https://github.com/OpenGene/fastp), an efficient tool for quality filtering, trimming adapters, and generating quality reports.
Asumming all your samples (in fastq.gz format) are in the same folder.
```bash
for i in `ls -1 *_1.fastq.gz | sed 's/_1.fastq.gz//'`; do  fastp -i $i\_1.fastq.gz -I $i\_2.fastq.gz --detect_adapter_for_pe -o trimmed/$i\_1.fq.gz -O trimmed/$i\_2.fq.gz -h trimmed/$i\_fastq.html -e 25
```
New data will be saved in a new folder named trimmed.

Then use bowtie2 to remove reads associated with the human genome. Please review this [link](https://benlangmead.github.io/aws-indexes/bowtie) to download the indexed human genome. 

```bash
#Download hg19
wget https://genome-idx.s3.amazonaws.com/bt/hg19.zip
unzip hg19.zip
```
Aquí hay que revisar el nombre del output, porque no me acuerdo dónde coloca el 1 y 2 en el nombre.
```bash
for i in `ls *_1.fastq.gz | sed 's/_1.fastq.gz//'`; do
  bowtie2 -p 8 \
    -x databases/references/H.sapiens_hg19/bowtie2/hg19 \ # poner el path dónde está el genoma descargado
    -1 trimmed/$i\_1.fastq.gz \
    -2 trimmed/$i_2.fastq.gz \
    --very-sensitive-local \
    --quiet \
    --un-conc-gz filtered/$i\_filtered.fq.gz
done
```
Samples will be saved in a new folder named filtered.

## Metagenomic Assembly
Aquí pueden intentar correr un loop,esto procesa una y luego la otra. O pueden correrlas por separado haciendo dos carpetas nada más o usando el comando base. 
Tal vez sea mejor correrlas por separadop y enviar cada una a un nodo aparte para que terminen antes y no se corte en las 24h. 
Si 24h no son suficientes para ensamblar pueden borrar el k-mer 127 pero habría que revisar el output para ver dónde se cortó. Sino si habría que pedir más tiempo en el cluster.
```bash
for i in `ls -1 *_1.fastq.gz | sed 's/_1.fastq.gz//'`; do #Aquí pueden reemplazar toda la parte del ls con sólo los nombres de las dos muestras, como son solo 2.
metaspades.py \
  -1 filtered/$i\_filtered.fq.gz.1 \ #corregir basado en el output de arriba
  -2 filtered/$i\_filtered.fq.gz.2 \
  --meta \
  -k 33,55,77,99,127 \
  -t 64 \
  -m 1000 \
  -o Assembly/$i/
#rename the contigs.fa
for i in `ls -1 *_1.fastq.gz | sed 's/_1.fastq.gz//'`; do
mv Assembly/$i/contigs.fa Assembly/$i/$i\.fasta
done
```

## Evaluating coassembly

Assembled contigs can be evaluated using MetaQuast, which provides quality assessment based on alignment and assembly metrics. 
To process all samples efficiently, locate the FASTA files (.fasta) and use the following command within a loop:
```bash
for i in `ls -1 *.fasta | sed 's/.fasta//'`
do
metaquast.py -L -s Assembly/$i/$i\.fasta -o QUAST/$i/ --min-contig 500 # Aquí pueden elegir si usar 500 o 1000. Yo recomiendo 1000
done
```
