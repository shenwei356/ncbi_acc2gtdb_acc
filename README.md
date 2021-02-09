# ncbi_acc2gtdb_acc

Mapping NCBI genbank accession to GTDB accession

## Strategy

    NCBI assembly_accession (n) <- (1) ftp url -- fna.gz (n) <- (1) NCBI accession
    
    NCBI assembly_accession (1) -> (1) GTDB accession

## Tools

- [csvtk](https://github.com/shenwei356/csvtk)
- [rush](https://github.com/shenwei356/rush/)
    
## Steps

    # -------------- download meta data -----------------
    
    # download assembly summary files, for
    #   NCBI assembly_accession (n) <- (1) ftp url -- fna.gz (n) <- (1) NCBI accession
    wget -c https://ftp.ncbi.nlm.nih.gov/genomes/genbank/archaea/assembly_summary.txt -O as-genbank-archaea.txt
    wget -c https://ftp.ncbi.nlm.nih.gov/genomes/genbank/bacteria/assembly_summary.txt -O as-genbank-bacteria.txt
    
    # download gtdb metadata, for
    #   NCBI assembly_accession (1) -> (1) GTDB accession
    wget -c https://data.ace.uq.edu.au/public/gtdb/data/releases/release95/95.0/ar122_metadata_r95.tar.gz
    wget -c https://data.ace.uq.edu.au/public/gtdb/data/releases/release95/95.0/bac120_metadata_r95.tar.gz 
       
    
    # -------------- preparation -----------------
    
    # convert assembly summary files to TSV
    for f in as-*.txt; do 
        sed 1d $f | sed "1s/^# //" | sed "s/\"/$/g" > ${f%.*}.tsv
    done
    
    # extract NCBI assembly_accession and ftp path
    cat as-genbank-archaea.tsv <(sed 1d as-genbank-bacteria.tsv) \
        | csvtk cut -t -f assembly_accession,ftp_path \
        > assacc2ftp.tsv
        
    # concatenate gtdb metadata, and extract gets
    #     ncbi_genbank_assembly_accession ->  gtdb accession
    tar -zxvf ar122_metadata_r95.tar.gz
    tar -zxvf bac120_metadata_r95.tar.gz        
    cat ar122_metadata_r95.tsv <(sed 1d bac120_metadata_r95.tsv) \
        | csvtk cut -t -f ncbi_genbank_assembly_accession,accession \
        | sed 1d \
        > ncbi_ass2gtdb_acc.tsv
    
    
    # -------------- mapping NCBI accession to NCBI assembly_accession -----------------
    
    # parse all genomes, extract accession, and append NCBI assembly_accession.
    # it may report "gzip: stdin: unexpected end of file",
    # may be due to my bad internet connection
    
    time cat assacc2ftp.tsv \
        | csvtk replace -t -f ftp_path -p "^ftp" -r "https" \
        | sed 1d \
        | rush -k -j 10 'wget {2}/{2%}_genomic.fna.gz -o /dev/null -O - \
            | gzip -cd | grep "^>" | cut -f 1 | sed "s/>//" | awk "{print \$1, \"{1}\"}"' \
        | sed 's/ /\t/' \
        | gzip -c \
        > ncbi_acc2ncbi_ass.tsv.gz
        
    # -------------- mapping NCBI accession to GTDB accession -----------------
    
    csvtk replace -H -t -f 2 -p '(.+)' -r '$1--{kv}' --key-miss-repl __ \
        -k ncbi_ass2gtdb_acc.tsv ncbi_acc2ncbi_ass.tsv \
        | sed 's/--/\t/' \
        | gzip -c \
        > ncbi_acc2gtdb_acc.tsv.gz
        
Note that not all NCBI assembly_accessions can be mapped to GTDB.
    
    # filtering:
    zcat ncbi_acc2gtdb_acc.tsv.gz \
        | grep -v __ \
        | gzip -c \
        -o ncbi_acc2gtdb_acc.filter.tsv.gz
        
## Result

    zcat ncbi_acc2gtdb_acc.tsv.gz | head -n 5
    LMVM01000001.1  GCA_002287175.1 RS_GCF_002287175.1
    LMVM01000002.1  GCA_002287175.1 RS_GCF_002287175.1
    LMVM01000003.1  GCA_002287175.1 RS_GCF_002287175.1
    LMVM01000004.1  GCA_002287175.1 RS_GCF_002287175.1
    LMVM01000005.1  GCA_002287175.1 RS_GCF_002287175.1
