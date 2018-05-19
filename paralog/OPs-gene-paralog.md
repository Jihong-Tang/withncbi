# Gene-paralog-repeats

[TOC levels=1-3]: # " "
- [Gene-paralog-repeats](#gene-paralog-repeats)
- [Sources](#sources)
- [TODO](#todo)
- [Stats](#stats)
- [Repeats](#repeats)
    - [MITE](#mite)
    - [Other repeats](#other-repeats)
- [Scripts](#scripts)
    - [`proc_prepare.sh`](#proc_preparesh)
    - [`proc_repeat.sh`](#proc_repeatsh)
    - [`proc_mite.sh`](#proc_mitesh)
    - [`proc_paralog.sh`](#proc_paralogsh)
    - [`proc_all_gene.sh`](#proc_all_genesh)
    - [`proc_sep_gene.sh`](#proc_sep_genesh)
    - [`proc_sep_gene_jrunlist.sh`](#proc_sep_gene_jrunlistsh)
- [Atha](#atha)
- [Plants aligned with full chromosomes](#plants-aligned-with-full-chromosomes)
- [Plants aligned with partitioned chromosomes](#plants-aligned-with-partitioned-chromosomes)
- [Plants with annotations from JGI](#plants-with-annotations-from-jgi)


# Sources

* Annotations from
  [Ensembl gff3 files](https://github.com/wang-q/withncbi/blob/master/ensembl/README.md#gff3)
* Annotations from JGI PhytozomeV11
* Paralogs from
  [self-aligning](https://github.com/wang-q/withncbi/blob/master/paralog/OPs-selfalign.md)

# TODO

* Remove full transposons (Retro, DNA and RC transposons) from `paralog.yml`.
* Subfamilies.

# Stats

* Coverages on chromosomes of all feature types.
* Paralogs and adjacent regions intersect with all repeat families.
    * paralog
    * paralog_adjacent: paralog + 2000 bp
    * paralog_gene: intersections between paralogs and gene + 2000 bp
* Genes, upstreams, downstreams intersect with paralogs and all repeat families.
    * Up/down-streams are 2000 bp.
* Exons, introns, CDSs, five_prime_UTRs and three_prime_UTRs intersect with paralogs and all repeat
  families.

# Repeats

## MITE

* [Plant MITE database](http://pmite.hzau.edu.cn/download_mite/)
* http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/

## Other repeats

Ensembl gff3 files contain correct descriptions for dust and trf, but repeatmasker's descriptions
are not usable.

So I rerun RepeatMasker on every genomes and get reports from `genome.fa.out`.

1. RepeatMasker

    RepeatMasker runned with `-species Viridiplantae`.

    Repeat families listed in `genome.fa.tbl`. Families with proportions less than **0.0005** were
    dropped.

    * DNA: DNA transposons
    * LINE
    * LTR
    * Low_complexity
    * RC: Rolling-circles
    * SINE
    * Satellite
    * Simple_repeat

2. Ensembl gff3 repeats

    * dust: Low-complexity regions
    * trf: Tandem repeats

# Scripts

Same for each species.

## `proc_prepare.sh`

Genome, RepeatMasker, dustmasker and gff3.

```bash

cat <<'EOF' > ~/data/alignment/gene-paralog/proc_prepare.sh
#!/bin/bash

USAGE="Usage: $0 GENOME_NAME BASE_DIR"

if [ "$#" -lt 1 ]; then
    echo >&2 "$USAGE"
    exit 1
fi

echo "==> parameters <=="
echo "    " $@

GENOME_NAME=$1
BASE_DIR=${2:-~/data/alignment/Ensembl}

cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data

echo "==> get genome"
if [ -f ~/data/alignment/gene-paralog/${GENOME_NAME}/data/genome.fa ];
then
    echo "genome.fa exists"
else
    find ${BASE_DIR}/${GENOME_NAME} -type f -name "*.fa" \
        | sort | xargs cat \
        | perl -nl -e '/^>/ or $_ = uc; print' \
        > genome.fa
fi

echo "==> run RepeatMasker"
if [ -f ~/data/alignment/gene-paralog/${GENOME_NAME}/data/genome.fa.out ];
then
    echo "genome.fa.out exists"
else
    RepeatMasker genome.fa -species Viridiplantae -xsmall --parallel 8
    rm genome.fa.cat.gz  genome.fa.masked
    rm -fr RM_*
fi

echo "==> run dustmasker"
dustmasker -in genome.fa -infmt fasta -out - -outfmt interval \
    | perl -nl -e '
        BEGIN { $name = qq{}}
        if ( /^>(\S+)/ ) {
            $name = $1;
        } elsif ( /(\d+)\s+\-\s+(\d+)/ ) {
            print qq{$name:$1-$2};
        }
    ' \
    > dustmasker.output.txt
runlist cover dustmasker.output.txt -o dustmasker.yml

echo "==> Convert gff3 to runlists"
cd ~/data/alignment/gene-paralog/${GENOME_NAME}/feature

# For Atha, 0m41.121s. With --clean, 50m49.994s
time perl ~/Scripts/withncbi/util/gff2runlist.pl \
    --file ../data/gff3.gz \
    --size ../data/chr.sizes \
    --range 2000

EOF

```

## `proc_repeat.sh`

Repeats from RepeatMasker and gff3. Create `repeat.family.txt`.

```bash

cat <<'EOF' > ~/data/alignment/gene-paralog/proc_repeat.sh
#!/bin/bash

USAGE="Usage: $0 GENOME_NAME"

if [ "$#" -lt 1 ]; then
    echo "$USAGE"
    exit 1
fi

echo "==> parameters <=="
echo "    " $@

GENOME_NAME=$1
cd ~/data/alignment/gene-paralog/${GENOME_NAME}/repeat
find . -type f -name "*.yml" -or -name "*.csv" | parallel rm

echo "==> gff types"
gzip -d -c ../data/gff3.gz \
    | perl -nla -e '/^#/ and next; print $F[2]' \
    | sort | uniq -c \
    > gff.type.txt

gzip -d -c ../data/gff3.gz \
    | perl -nla -e '
        /^#/ and next;
        $F[2] eq q{repeat_region} or next;
        $F[8] =~ /description\=(\w+)/i or next;
        print qq{$F[2]\t$F[1]\t$1};
        ' \
    | sort | uniq -c \
    > gff.repeat.txt

echo "==> rmout families"
cat ../data/genome.fa.out \
    | perl -nla -e '/^\s*\d+/ or next; print $F[10]' \
    | sort | uniq -c \
    > rmout.family.txt
cp ../data/genome.fa.out ../repeat
cp ../data/genome.fa.tbl ../repeat

echo "==> rmout results"
perl ~/Scripts/withncbi/util/rmout2runlist.pl \
    --file ../data/genome.fa.out \
    --size ../data/chr.sizes

runlist split ../feature/all-repeat.yml -o .
runlist merge *.yml -o all-repeat.1.yml
find . -type f -name "*.yml" -not -name "all-*" | parallel rm

runlist span --op excise -n 10 --mk all-repeat.1.yml -o all-repeat.2.yml   # remove small spans
runlist span --op fill   -n 10 --mk all-repeat.2.yml -o all-repeat.3.yml   # fill small holes
runlist split all-repeat.3.yml -o .
mv all-repeat.3.yml all-repeat.yml
rm all-repeat.*.yml

echo "==> basic stat"
runlist stat all-repeat.yml -s ../data/chr.sizes --mk --all

echo "==> find repeat families large enough"
cat all-repeat.yml.csv \
    | perl -nla -F"," -e '
        next if $F[3] =~ /^c/;
        print $F[0] if $F[3] > 0.0005
    ' \
    > repeat.family.txt
cat repeat.family.txt \
    | parallel -j 8 "
        cp {}.yml ../yml
    "

echo "==> clean"
mv all-repeat.yml.csv ../stat
cp repeat.family.txt ../yml

EOF

```

## `proc_mite.sh`

MITE. Append to `repeat.family.txt`.

```bash

cat <<'EOF' > ~/data/alignment/gene-paralog/proc_mite.sh
#!/bin/bash

USAGE="Usage: $0 GENOME_NAME"

if [ "$#" -lt 1 ]; then
    echo "$USAGE"
    exit 1
fi

echo "==> parameters <=="
echo "    " $@

GENOME_NAME=$1
cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data

echo "==> mite_stat"
faops size mite.fa \
    | perl -nla -F"\t" -MStatistics::Descriptive -e '
        BEGIN {$stat = Statistics::Descriptive::Full->new;}
        next unless defined $F[1];
        $stat->add_data($F[1]);
        END {
            printf qq{Total:\t%d\nSum:\t%d\n}, $stat->count, $stat->sum;
            printf qq{Median:\t%d\nMean:\t%.1f\n}, $stat->median, $stat->mean;
            printf qq{Min:\t%d\nMax:\t%d\n}, $stat->min, $stat->max;
            printf qq{\n};
            printf qq{Length\t#Seqs\n};
            %distrib = $stat->frequency_distribution(10);
            for (sort {$a <=> $b} keys %distrib) {
                printf qq{%.1f\t%d\n}, $_, $distrib{$_};
            }
            printf qq{\n};
        }' \
    > mite_stat.txt

echo "==> genome blast"
faops filter -a 40  mite.fa mite.filter.fa
perl ~/Scripts/egaz/fasta_blastn.pl -f mite.filter.fa -g genome.fa -o mite.bg.blast --parallel 8
perl ~/Scripts/egaz/blastn_genome.pl -f mite.bg.blast -g genome.fa -o mite.bg.fasta -c 0.95 --parallel 8
cat mite.fa mite.bg.fasta \
    | faops filter -u stdin stdout \
    | faops filter -a 40 stdin stdout \
    > mite.all.fasta

echo "==> sparsemem_exact"
perl ~/Scripts/egaz/sparsemem_exact.pl -f mite.all.fasta -l 40 -g genome.fa -o mite.replace.tsv
cat mite.replace.tsv \
    | perl -nla -F"\t" -e ' print for @F' \
    | grep ':' \
    | sort | uniq \
    > mite.position.txt

echo "==> mite covers"
wc -l mite.position.txt >> mite_stat.txt
runlist cover mite.position.txt -o mite.yml

runlist span --op excise -n 10 mite.yml   -o mite.1.yml # remove small spans
runlist span --op fill   -n 10 mite.1.yml -o mite.2.yml # fill small holes
mv mite.2.yml mite.yml
rm mite.1.yml

runlist stat mite.yml -s chr.sizes

echo "==> clean"
rm mite.all.fasta mite.bg.blast mite.bg.fasta mite.filter.fa mite.replace.tsv

mv mite.yml ../yml
mv mite.yml.csv ../stat
mv mite_stat.txt ../stat
echo mite >> ../yml/repeat.family.txt

EOF

```

## `proc_paralog.sh`

```bash

cat <<'EOF' > ~/data/alignment/gene-paralog/proc_paralog.sh
#!/bin/bash

USAGE="Usage: $0 GENOME_NAME"

if [ "$#" -lt 1 ]; then
    echo "$USAGE"
    exit 1
fi

echo "==> parameters <=="
echo "    " $@

GENOME_NAME=$1

cd ~/data/alignment/gene-paralog/${GENOME_NAME}/yml
cp ../data/paralog.yml ../yml

echo "==> paralog_adjacent"
runlist span    --op pad    -n 2000 paralog.yml               -o paralog_adjacent.1.yml
runlist compare --op diff  paralog_adjacent.1.yml paralog.yml -o paralog_adjacent.2.yml
runlist span    --op excise -n 100  paralog_adjacent.2.yml    -o paralog_adjacent.3.yml
runlist span    --op fill   -n 100  paralog_adjacent.3.yml    -o paralog_adjacent.4.yml
runlist genome ../data/chr.sizes -o genome.yml
runlist compare --op intersect paralog_adjacent.4.yml genome.yml -o paralog_adjacent.5.yml
mv paralog_adjacent.5.yml paralog_adjacent.yml
rm paralog_adjacent.*.yml

echo "==> paralog_gene"
mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/yml/gene
runlist split ../feature/all-gene.yml -o ~/data/alignment/gene-paralog/${GENOME_NAME}/yml/gene
runlist compare --op union gene/gene.yml gene/upstream.yml -o paralog_gene.1.yml
runlist compare --op union gene/downstream.yml paralog_gene.1.yml -o paralog_gene.2.yml
runlist compare --op intersect paralog_gene.2.yml paralog.yml -o paralog_gene.3.yml
mv paralog_gene.3.yml paralog_gene.yml
rm paralog_gene.*.yml

for ftr in paralog paralog_adjacent paralog_gene
do
    echo "==> ${ftr} coverages"
    sleep 1;
    runlist stat -s ../data/chr.sizes ../yml/${ftr}.yml -o ../stat/${ftr}.yml.csv

    echo "==> ${ftr} stats"
    cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
    cat ../yml/repeat.family.txt \
        | parallel -j 1 --keep-order "
            echo \"==> {}\";
            runlist stat2 ../yml/${ftr}.yml ../yml/{}.yml -s ../data/chr.sizes \
                --op intersect --all \
                -o ../stat/${GENOME_NAME}.${ftr}.{}.csv
        "
    echo "key,chr_length,${ftr}_size,key_length,key_size,c1,c2,ratio" \
        > ../stat/${GENOME_NAME}.${ftr}.all-repeat.csv
    cat ../yml/repeat.family.txt \
        | parallel -j 1 --keep-order "
            cat ../stat/${GENOME_NAME}.${ftr}.{}.csv \
                | perl -nl -e '/^chr_length/ and next; print qq({},\$_)'
        " >> ../stat/${GENOME_NAME}.${ftr}.all-repeat.csv
    cat ../yml/repeat.family.txt \
        | parallel -j 1 --keep-order "rm ../stat/${GENOME_NAME}.${ftr}.{}.csv"
done

EOF

```

## `proc_all_gene.sh`

```bash

cat <<'EOF' > ~/data/alignment/gene-paralog/proc_all_gene.sh
#!/bin/bash

USAGE="Usage: $0 GENOME_NAME FEATURE_FILE"

if [ "$#" -lt 2 ]; then
    echo "$USAGE"
    exit 1
fi

echo "==> parameters <=="
echo "    " $@

GENOME_NAME=$1
FEATURE_FILE=$2
FEATURE_BASE=`basename "${FEATURE_FILE%.*}"`

echo "==> all-gene"
cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat

runlist stat2 ../feature/all-gene.yml ${FEATURE_FILE} -s ../data/chr.sizes \
    --op intersect --mk --all \
    -o ${GENOME_NAME}.all-gene.${FEATURE_BASE}.csv

EOF

```

## `proc_sep_gene.sh`

```bash

cat <<'EOF' > ~/data/alignment/gene-paralog/proc_sep_gene.sh
#!/bin/bash

USAGE="Usage: $0 GENOME_NAME FEATURE_FILE"

if [ "$#" -lt 2 ]; then
    echo "$USAGE"
    exit 1
fi

echo "==> parameters <=="
echo "    " $@

GENOME_NAME=$1
FEATURE_FILE=$2
FEATURE_BASE=`basename "${FEATURE_FILE%.*}"`

cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat

echo "==> intersect"
for ftr in gene upstream downstream exon CDS intron five_prime_UTR three_prime_UTR
do
    echo ${ftr}
done \
    | parallel -j 8 --keep-order "
        echo \"==> {} ${FEATURE_BASE}\";
        sleep 1;
        runlist stat2 ../feature/sep-{}.yml ${FEATURE_FILE} -s ../data/chr.sizes \
            --op intersect --mk --all \
            -o stat.sep-{}.${FEATURE_BASE}.csv;
        cat stat.sep-{}.${FEATURE_BASE}.csv \
            | cut -d ',' -f 1,3,5 \
            > stat.sep-{}.${FEATURE_BASE}.csv.tmp;
    "

echo "==> concat gene"
printf "gene_id," > ${GENOME_NAME}.gene.${FEATURE_BASE}.csv
for ftr in gene upstream downstream
do
    printf "${ftr}_length,${ftr}_${FEATURE_BASE},"
done >> ${GENOME_NAME}.gene.${FEATURE_BASE}.csv
echo >> ${GENOME_NAME}.gene.${FEATURE_BASE}.csv

for ftr in gene upstream downstream
do
    cat stat.sep-${ftr}.${FEATURE_BASE}.csv.tmp
done \
    | grep -v "^key" \
    | perl ~/Scripts/withncbi/util/merge_csv.pl --concat -f 0 -o stdout \
    >> ${GENOME_NAME}.gene.${FEATURE_BASE}.csv

echo "==> concat trans"
printf "trans_id," > ${GENOME_NAME}.trans.${FEATURE_BASE}.csv
for ftr in exon CDS intron five_prime_UTR three_prime_UTR
do
    printf "${ftr}_length,${ftr}_${FEATURE_BASE},"
done >> ${GENOME_NAME}.trans.${FEATURE_BASE}.csv
echo >> ${GENOME_NAME}.trans.${FEATURE_BASE}.csv

for ftr in exon CDS intron five_prime_UTR three_prime_UTR
do
    cat stat.sep-${ftr}.${FEATURE_BASE}.csv.tmp
done \
    | grep -v "^key" \
    | perl ~/Scripts/withncbi/util/merge_csv.pl --concat -f 0 -o stdout \
    >> ${GENOME_NAME}.trans.${FEATURE_BASE}.csv

echo "==> clean"
rm stat.sep-*.${FEATURE_BASE}.csv.tmp
rm stat.sep-*.${FEATURE_BASE}.csv

EOF

```

## `proc_sep_gene_jrunlist.sh`

```bash

cat <<'EOF' > ~/data/alignment/gene-paralog/proc_sep_gene_jrunlist.sh
#!/bin/bash

USAGE="Usage: $0 GENOME_NAME FEATURE_FILE"

if [ "$#" -lt 2 ]; then
    echo "$USAGE"
    exit 1
fi

echo "==> parameters <=="
echo "    " $@

GENOME_NAME=$1
FEATURE_FILE=$2
FEATURE_BASE=`basename "${FEATURE_FILE%.*}"`

cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat

echo "==> intersect"
# parallel in 1 threads to save memory
for ftr in gene upstream downstream exon CDS intron five_prime_UTR three_prime_UTR
do
    echo ${ftr}
done \
    | parallel -j 1 --keep-order "
        echo \"==> {} ${FEATURE_BASE}\";
        sleep 1;
        java -jar ~/Scripts/egaz/jar/jrunlist.jar statop \
            ../data/chr.sizes ../feature/sep-{}.yml ${FEATURE_FILE}  \
            --op intersect --all \
            -o stat.sep-{}.${FEATURE_BASE}.csv;
        cat stat.sep-{}.${FEATURE_BASE}.csv \
            | cut -d ',' -f 1,3,5 \
            > stat.sep-{}.${FEATURE_BASE}.csv.tmp;
    "

echo "==> concat gene"
printf "gene_id," > ${GENOME_NAME}.gene.${FEATURE_BASE}.csv
for ftr in gene upstream downstream
do
    printf "${ftr}_length,${ftr}_${FEATURE_BASE},"
done >> ${GENOME_NAME}.gene.${FEATURE_BASE}.csv
echo >> ${GENOME_NAME}.gene.${FEATURE_BASE}.csv

for ftr in gene upstream downstream
do
    cat stat.sep-${ftr}.${FEATURE_BASE}.csv.tmp
done \
    | grep -v "^key" \
    | perl ~/Scripts/withncbi/util/merge_csv.pl --concat -f 0 -o stdout \
    >> ${GENOME_NAME}.gene.${FEATURE_BASE}.csv

echo "==> concat trans"
printf "trans_id," > ${GENOME_NAME}.trans.${FEATURE_BASE}.csv
for ftr in exon CDS intron five_prime_UTR three_prime_UTR
do
    printf "${ftr}_length,${ftr}_${FEATURE_BASE},"
done >> ${GENOME_NAME}.trans.${FEATURE_BASE}.csv
echo >> ${GENOME_NAME}.trans.${FEATURE_BASE}.csv

for ftr in exon CDS intron five_prime_UTR three_prime_UTR
do
    cat stat.sep-${ftr}.${FEATURE_BASE}.csv.tmp
done \
    | grep -v "^key" \
    | perl ~/Scripts/withncbi/util/merge_csv.pl --concat -f 0 -o stdout \
    >> ${GENOME_NAME}.trans.${FEATURE_BASE}.csv

echo "==> clean"
rm stat.sep-*.${FEATURE_BASE}.csv.tmp
rm stat.sep-*.${FEATURE_BASE}.csv

EOF

```

# Atha

Full processing time is about 1 hour.

1. [Data](https://github.com/wang-q/withncbi/blob/master/paralog/OPs-selfalign.md#arabidopsis)

2. Prepare

    ```bash
    GENOME_NAME=Atha

    echo "====> create directories"
    mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/data
    mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/feature
    mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/repeat
    mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
    mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/yml

    echo "====> copy or download needed files here"
    cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data
    cp ~/data/alignment/self/plants/Genomes/${GENOME_NAME}/chr.sizes chr.sizes
    cp ~/data/alignment/self/plants/Results/${GENOME_NAME}/${GENOME_NAME}.chr.runlist.yml paralog.yml

    cp ~/data/ensembl82/gff3/arabidopsis_thaliana/Arabidopsis_thaliana.TAIR10.29.gff3.gz gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/03_arabidopsis_mite_seq.fa -O mite.fa
    ```

    ```bash
    cd ~/data/alignment/gene-paralog/Atha/data

    # 0m44.430s
    time bash ~/data/alignment/gene-paralog/proc_prepare.sh Atha
    # 0m46.052s
    time bash ~/data/alignment/gene-paralog/proc_repeat.sh Atha
    # 0m43.666s
    time bash ~/data/alignment/gene-paralog/proc_mite.sh Atha
    ```

3. Paralog-repeats stats

    ```bash
    cd ~/data/alignment/gene-paralog/Atha/stat
    # 0m28.356s
    time bash ~/data/alignment/gene-paralog/proc_paralog.sh Atha
    ```

4. Gene-paralog stats

    ```bash
    cd ~/data/alignment/gene-paralog/Atha/stat
    # 0m6.888s
    time bash ~/data/alignment/gene-paralog/proc_all_gene.sh Atha ../yml/paralog.yml
    # 0m7.283s
    time bash ~/data/alignment/gene-paralog/proc_all_gene.sh Atha ../yml/paralog_adjacent.yml

    # E5-2690 v3
    # real    8m45.668s
    # user    57m58.984s
    # sys     0m1.808s
    # i7-6700k
    # real	15m18.045s
    # user	104m21.728s
    # sys	0m13.363s
    time bash ~/data/alignment/gene-paralog/proc_sep_gene.sh Atha ../yml/paralog.yml

    # real	0m35.566s
    # user	1m12.613s
    # sys	0m7.601s
    time bash ~/data/alignment/gene-paralog/proc_sep_gene_jrunlist.sh Atha ../yml/paralog.yml

    bash ~/data/alignment/gene-paralog/proc_sep_gene_jrunlist.sh Atha ../yml/paralog_adjacent.yml
    ```

5. Gene-repeats stats

    ```bash
    cd ~/data/alignment/gene-paralog/Atha/stat

    cat ../yml/repeat.family.txt \
        | parallel -j 8 --keep-order "
            bash ~/data/alignment/gene-paralog/proc_all_gene.sh Atha ../yml/{}.yml
        "

    # 12 hours?
    # time \
    # cat ../yml/repeat.family.txt \
    #     | parallel -j 1 --keep-order "
    #         bash ~/data/alignment/gene-paralog/proc_sep_gene.sh Atha ../yml/{}.yml
    #     "    

    # real	10m34.669s
    # user	17m27.433s
    # sys	3m20.765s
    time \
        cat ../yml/repeat.family.txt \
            | parallel -j 1 --keep-order "
                bash ~/data/alignment/gene-paralog/proc_sep_gene_jrunlist.sh Atha ../yml/{}.yml
            "
    ```

6. Pack up

    ```bash
    cd ~/data/alignment/gene-paralog
    find Atha -type f -not -path "*/data/*" -print | zip Atha.zip -9 -@
    ```

# Plants aligned with full chromosomes

1. [Data](https://github.com/wang-q/withncbi/blob/master/paralog/OPs-selfalign.md#full-chromosomes)

    * OsatJap
    * Bdis
    * Alyr
    * Sbic

2. Prepare

    ```bash
    for GENOME_NAME in OsatJap Bdis Alyr Sbic
    do
        echo "====> create directories"
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/data
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/feature
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/repeat
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/yml

        echo "====> copy or download needed files here"
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data
        cp ~/data/alignment/self/plants/Genomes/${GENOME_NAME}/chr.sizes chr.sizes
        cp ~/data/alignment/self/plants/Results/${GENOME_NAME}/${GENOME_NAME}.chr.runlist.yml paralog.yml
    done

    # http://stackoverflow.com/questions/1494178/how-to-define-hash-tables-in-bash
    # OSX has bash 3. So no easy hashmaps. Do it manually.
    cd ~/data/alignment/gene-paralog

    # OsatJap
    cp ~/data/ensembl82/gff3/oryza_sativa/Oryza_sativa.IRGSP-1.0.29.gff3.gz \
        ~/data/alignment/gene-paralog/OsatJap/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/26_nipponbare_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/OsatJap/data/mite.fa

    # Bdis
    cp ~/data/ensembl82/gff3/brachypodium_distachyon/Brachypodium_distachyon.v1.0.29.gff3.gz \
        ~/data/alignment/gene-paralog/Bdis/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/25_brachypodium_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Bdis/data/mite.fa

    # Alyr
    cp ~/data/ensembl82/gff3/arabidopsis_lyrata/Arabidopsis_lyrata.v.1.0.29.gff3.gz \
        ~/data/alignment/gene-paralog/Alyr/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/02_lyrata_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Alyr/data/mite.fa

    # Sbic
    # Ensemblgenomes 82 didn't provide a full gff3
    gt merge -gzip -force \
        -o ~/data/alignment/gene-paralog/Sbic/data/gff3.gz \
        ~/data/ensembl82/gff3/sorghum_bicolor/Sorghum_bicolor.Sorbi1.29.chromosome.*.gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/28_sorghum_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Sbic/data/mite.fa

    ```

    ```bash
    for GENOME_NAME in OsatJap Alyr Sbic
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data

        bash ~/data/alignment/gene-paralog/proc_prepare.sh ${GENOME_NAME}

        bash ~/data/alignment/gene-paralog/proc_repeat.sh ${GENOME_NAME}
        bash ~/data/alignment/gene-paralog/proc_mite.sh ${GENOME_NAME}
    done
    ```

3. Paralog-repeats stats

    ```bash
    for GENOME_NAME in OsatJap Alyr Sbic
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
        bash ~/data/alignment/gene-paralog/proc_paralog.sh ${GENOME_NAME}
    done
    ```

4. Gene-paralog stats

    ```bash
    for GENOME_NAME in OsatJap Alyr Sbic
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat

        bash ~/data/alignment/gene-paralog/proc_all_gene.sh ${GENOME_NAME} ../yml/paralog.yml
        bash ~/data/alignment/gene-paralog/proc_all_gene.sh ${GENOME_NAME} ../yml/paralog_adjacent.yml

        bash ~/data/alignment/gene-paralog/proc_sep_gene.sh ${GENOME_NAME} ../yml/paralog.yml
        bash ~/data/alignment/gene-paralog/proc_sep_gene.sh ${GENOME_NAME} ../yml/paralog_adjacent.yml
    done
    ```

5. Gene-repeats stats

    ```bash
    for GENOME_NAME in OsatJap Alyr Sbic
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
        cat ../yml/repeat.family.txt \
            | parallel -j 8 --keep-order "
                bash ~/data/alignment/gene-paralog/proc_all_gene.sh ${GENOME_NAME} ../yml/{}.yml
            "

        cat ../yml/repeat.family.txt \
            | parallel -j 1 --keep-order "
                bash ~/data/alignment/gene-paralog/proc_sep_gene.sh ${GENOME_NAME} ../yml/{}.yml
            "
    done
    ```

6. Pack up

    ```bash
    for GENOME_NAME in OsatJap Alyr Sbic
    do
        cd ~/data/alignment/gene-paralog
        find ${GENOME_NAME} -type f -not -path "*/data/*" -print | zip ${GENOME_NAME}.zip -9 -@
    done
    ```

# Plants aligned with partitioned chromosomes

1. [Data](https://github.com/wang-q/withncbi/blob/master/paralog/OPs-selfalign.md#partitioned-chromosomes)

    * Mtru
    * Gmax
    * Brap
    * Vvin
    * Slyc
    * Stub

    * Bole (no mite)

2. Prepare

    ```bash
    for GENOME_NAME in Mtru Gmax Brap Vvin Slyc Stub
    do
        echo "====> create directories"
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/data
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/feature
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/repeat
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}/yml

        echo "====> copy or download needed files here"
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data
        cp ~/data/alignment/self/plants_parted/Genomes/${GENOME_NAME}/chr.sizes chr.sizes
        cp ~/data/alignment/self/plants_parted/Results/${GENOME_NAME}/${GENOME_NAME}.chr.runlist.yml paralog.yml
    done

    cd ~/data/alignment/gene-paralog

    # Mtru
    cp ~/data/ensembl82/gff3/medicago_truncatula/Medicago_truncatula.GCA_000219495.2.29.gff3.gz \
        ~/data/alignment/gene-paralog/Mtru/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/20_medicago_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Mtru/data/mite.fa

    # Gmax
    cp ~/data/ensembl82/gff3/glycine_max/Glycine_max.V1.0.29.gff3.gz \
        ~/data/alignment/gene-paralog/Gmax/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/18_soybean_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Gmax/data/mite.fa

    # Brap
    cp ~/data/ensembl82/gff3/brassica_rapa/Brassica_rapa.IVFCAASv1.29.gff3.gz \
        ~/data/alignment/gene-paralog/Brap/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/04_brassica_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Brap/data/mite.fa

    # Vvin
    cp ~/data/ensembl82/gff3/vitis_vinifera/Vitis_vinifera.IGGP_12x.29.gff3.gz \
        ~/data/alignment/gene-paralog/Vvin/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/39_grape_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Vvin/data/mite.fa

    # Slyc
    cp ~/data/ensembl82/gff3/solanum_lycopersicum/Solanum_lycopersicum.GCA_000188115.2.29.gff3.gz \
        ~/data/alignment/gene-paralog/Slyc/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/37_tomato_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Slyc/data/mite.fa

    # Stub
    cp ~/data/ensembl82/gff3/solanum_tuberosum/Solanum_tuberosum.3.0.29.gff3.gz \
        ~/data/alignment/gene-paralog/Stub/data/gff3.gz
    wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/38_potato_mite_seq.fa \
        -O ~/data/alignment/gene-paralog/Stub/data/mite.fa

    # # Sita
    # cp ~/data/ensembl82/gff3/setaria_italica/Setaria_italica.JGIv2.0.29.gff3.gz \
    #     ~/data/alignment/gene-paralog/Sita/data/gff3.gz
    # wget http://pmite.hzau.edu.cn/MITE/MITE-SEQ-V2/27_foxtail_mite_seq.fa \
    #     -O ~/data/alignment/gene-paralog/Sita/data/mite.fa
    ```

    ```bash
    for GENOME_NAME in Mtru Gmax Brap Vvin Slyc Stub Sita
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data

        bash ~/data/alignment/gene-paralog/proc_prepare.sh ${GENOME_NAME}

        bash ~/data/alignment/gene-paralog/proc_repeat.sh ${GENOME_NAME}
        bash ~/data/alignment/gene-paralog/proc_mite.sh ${GENOME_NAME}
    done
    ```

# Plants with annotations from JGI

1. [Data](https://github.com/wang-q/withncbi/blob/master/paralog/OPs-selfalign.md#full-chromosomes)

    * AthaJGI
    * OsatJapJGI

2. Prepare

    ```bash
    for GENOME_NAME in Atha OsatJap
    do
        echo "====> create directories"
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}JGI/data
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}JGI/feature
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}JGI/repeat
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}JGI/stat
        mkdir -p ~/data/alignment/gene-paralog/${GENOME_NAME}JGI/yml

        echo "====> copy or download needed files here"
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}JGI/data
        cp ~/data/alignment/gene-paralog/${GENOME_NAME}/data/* .
    done

    cd ~/data/alignment/gene-paralog

    # Atha
    cp -f ~/data/PhytozomeV11/Athaliana/annotation/Athaliana_167_TAIR10.gene_exons.gff3.gz \
        ~/data/alignment/gene-paralog/AthaJGI/data/gff3.gz

    # OsatJap
    cp ~/data/PhytozomeV11/Osativa/annotation/Osativa_323_v7.0.gene_exons.gff3.gz \
        ~/data/alignment/gene-paralog/OsatJapJGI/data/gff3.gz
    ```

    ```bash
    for GENOME_NAME in AthaJGI OsatJapJGI
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/feature
        perl ~/Scripts/withncbi/util/gff2runlist.pl \
            --file ../data/gff3.gz \
            --size ../data/chr.sizes \
            --range 2000 --remove

        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/data

        #bash ~/data/alignment/gene-paralog/proc_prepare.sh ${GENOME_NAME}

        bash ~/data/alignment/gene-paralog/proc_repeat.sh ${GENOME_NAME}
        bash ~/data/alignment/gene-paralog/proc_mite.sh ${GENOME_NAME}
    done
    ```

3. Paralog-repeats stats

    ```bash
    for GENOME_NAME in AthaJGI OsatJapJGI
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
        bash ~/data/alignment/gene-paralog/proc_paralog.sh ${GENOME_NAME}
    done
    ```

4. Gene-paralog stats

    ```bash
    for GENOME_NAME in AthaJGI OsatJapJGI
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat

        bash ~/data/alignment/gene-paralog/proc_all_gene.sh ${GENOME_NAME} ../yml/paralog.yml
        bash ~/data/alignment/gene-paralog/proc_all_gene.sh ${GENOME_NAME} ../yml/paralog_adjacent.yml

        bash ~/data/alignment/gene-paralog/proc_sep_gene.sh ${GENOME_NAME} ../yml/paralog.yml
        bash ~/data/alignment/gene-paralog/proc_sep_gene.sh ${GENOME_NAME} ../yml/paralog_adjacent.yml
    done
    ```

5. Gene-repeats stats

    ```bash
    for GENOME_NAME in AthaJGI OsatJapJGI
    do
        cd ~/data/alignment/gene-paralog/${GENOME_NAME}/stat
        cat ../yml/repeat.family.txt \
            | parallel -j 8 --keep-order "
                bash ~/data/alignment/gene-paralog/proc_all_gene.sh ${GENOME_NAME} ../yml/{}.yml
            "

        cat ../yml/repeat.family.txt \
            | parallel -j 1 --keep-order "
                bash ~/data/alignment/gene-paralog/proc_sep_gene_jrunlist.sh ${GENOME_NAME} ../yml/{}.yml
            "
    done
    ```

6. Pack up

    ```bash
    for GENOME_NAME in AthaJGI OsatJapJGI
    do
        cd ~/data/alignment/gene-paralog
        find ${GENOME_NAME} -type f -not -path "*/data/*" -print | zip ${GENOME_NAME}.zip -9 -@
    done
    ```

