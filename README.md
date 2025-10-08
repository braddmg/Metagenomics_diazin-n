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
Si 24h no son suficientes para ensamblar pueden borrar el k-mer 33 pero habría que revisar el output para ver dónde se cortó. Sino si habría que pedir más tiempo en el cluster.
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
done
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
## Reformatear y remover contigs
Vamos a usar Anvio para cambiar el nombre a los contigs generados para que no incluyan caracteres no convencionales y además eliminar los contigs menores a 1000bp. Coloque los archivos fasta en la misma carpeta y ejecute el siguiente comando. 

```bash
conda activate anvio-8.0
for i in `ls *fasta | awk 'BEGIN{FS=".fasta"}{print $1}'`
do
anvi-script-reformat-fasta $i.fasta \
                           -o $i.fa \
                           -l 1000 --simplify-names --prefix $i --seq-type NT
done
```
Esto va a generar nuevos achivos con extensión "*.fa" (para diferenciar de los antiguos)

## Binning con metawrap
Utilice el módulo de metawrap para el siguiente comando. Les recomiendo tener los archivos fasta descomprimidos en una carpeta con el nombre de cada muestra a analizar. Puede correrlos individual si son pocas o en loop.
```bash
gzip -d *.gz # para descomprimir, puede durar un rato. 
metawrap binning -o sample -t 64 -a sample.fa --metabat2 --maxbin2 --concoct Raw/sample/*_1.fastq Raw/sample/*_2.fastq
```

## Refinamiento
Puede que este no funcione si checkm2 no se instaló bien, pero me avisan el output que genera y el error. Igual se puede procesar con checkm2 después.
Aquí yo uso un 70% de completness y un 10% de contaminación, es decir, lo que no cumpla con eso se va. Pero esto es arbritario. Por ejemplo para un MAG de calidad "media" se sugiere un 50% de completness y 10%de contaminación, mientras que para uno de "buena o alta" calidad 90 y 5. Pueden bajarlo a 50 y 10 que luego igual lo podemos refinar más. En [este paper](https://www.nature.com/articles/nbt.3893) hay más info. 
```bash
metawrap bin_refinement -o SAMPLE_BIN_REFINEMENT -t 64 -A sample_bin/metabat2_bins/ -B sample_bin/maxbin2_bins -C sample_bin/concoct_bins -c 70 -x 10 -m 1000
```
