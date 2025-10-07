# Metagenomics diazinon

## Quality control
The first step in metagenomic data analysis is to perform quality control on the raw reads. For this, we use [fastp](https://github.com/OpenGene/fastp), an efficient tool for quality filtering, trimming adapters, and generating quality reports.
Asumming all your samples (in fastq.gz format) are in the same folder.
```bash
for i in `ls -1 *_1.fastq.gz | sed 's/_1.fastq.gz//'`; do  fastp -i $i\_1.fastq.gz -I $i\_2.fastq.gz --detect_adapter_for_pe -o trimmed/$i\_1.fq.gz -O trimmed/$i\_2.fq.gz -h trimmed/$i\_fastq.html -e 25
```
New data will be saved in a new folder named trimmed.

Then use bowtie2 to remove reads associated with the human genome. Please review this [link](https://benlangmead.github.io/aws-indexes/bowtie)to download the indexed human genome. 

```bash
#Download hg19
wget https://genome-idx.s3.amazonaws.com/bt/hg19.zip
unzip hg19.zip
```
Aquí hay que revisar el nombre del output, porque no me acuerdo dónde coloca el 1 y 2 en el nombre.
```bash
for i in $(ls *_1.fastq.gz | sed 's/_1.fastq.gz//'); do
  bowtie2 -p 8 \
    -x databases/references/H.sapiens_hg19/bowtie2/hg19 \
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
for i in `ls -1 *_1.fastq.gz | sed 's/_1.fastq.gz//'`; do
metaspades.py \
  -1 filtered/$i\_filtered.fq.gz.1 \ #corregir basado en el output de arriba
  -2 filtered/$i\_filtered.fq.gz.2 \
  --meta \
  -k 33,55,77,99,127 \
  -t 64 \
  -m 1000 \
  -o Assembly/$i/
```
