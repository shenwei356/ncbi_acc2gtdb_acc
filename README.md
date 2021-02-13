# ncbi_acc2gtdb_acc

Mapping NCBI genbank accession to GTDB accession

## Strategy

    # genbank assembly
    NCBI assembly_accession (1) <- (1) ftp-url/fna.gz (1) <- (n) NCBI accession
    
    # gtdb
    NCBI assembly_accession (1) -> (1) GTDB accession

## Tools

- [csvtk](https://github.com/shenwei356/csvtk)
- [rush](https://github.com/shenwei356/rush/)
    
## Steps

    # -------------- download meta data -----------------
    
    # download assembly summary files
    wget -c https://ftp.ncbi.nlm.nih.gov/genomes/genbank/archaea/assembly_summary.txt -O as-genbank-archaea.txt
    wget -c https://ftp.ncbi.nlm.nih.gov/genomes/genbank/bacteria/assembly_summary.txt -O as-genbank-bacteria.txt 
    
    # download gtdb metadata
    wget -c https://data.ace.uq.edu.au/public/gtdb/data/releases/release95/95.0/ar122_metadata_r95.tar.gz
    wget -c https://data.ace.uq.edu.au/public/gtdb/data/releases/release95/95.0/bac120_metadata_r95.tar.gz 
       
    
    # -------------- preparation -----------------
    
    # convert assembly summary files to TSV
    for f in as-*.txt; do 
        sed 1d $f | sed "1s/^# //" | sed "s/\"/$/g" > ${f%.*}.tsv
    done
    

    # concatenate gtdb metadata, and extract gets
    #     ncbi_genbank_assembly_accession ->  gtdb accession
    tar -zxvf ar122_metadata_r95.tar.gz
    tar -zxvf bac120_metadata_r95.tar.gz        
    cat ar122_metadata_r95.tsv <(sed 1d bac120_metadata_r95.tsv) \
        | csvtk cut -t -f ncbi_genbank_assembly_accession,accession \
        | csvtk replace -t -f ncbi_genbank_assembly_accession -p "\.\d+$" \
        | sed 1d \
        > ncbi_ass2gtdb_acc.tsv
        
    # extract NCBI assembly_accession and ftp path
    cat as-genbank-archaea.tsv <(sed 1d as-genbank-bacteria.tsv) \
        | csvtk cut -t -f assembly_accession,ftp_path \
        | csvtk replace -t -f assembly_accession -p "\.\d+$" \
        | csvtk uniq -t -f assembly_accession \
        > ncbi_ass2ftp.tsv
    
    # filter these in gtdb
    cat ncbi_ass2ftp.tsv \
        | sed 1d \
        | csvtk grep -H -t -f 1 -P <(cut -f 1 ncbi_ass2gtdb_acc.tsv) \
        > ncbi_ass2ftp.filter.tsv
    
    # counting
    wc -l ncbi_ass2gtdb_acc.tsv ncbi_ass2ftp.filter.tsv
    # 194600 ncbi_ass2gtdb_acc.tsv
    # 194436 ncbi_ass2ftp.filter.tsv
    
    # find out these not in genbank.
    # possible reasson: deleted fro genbank, e.g., GCA_007036345
    cat ncbi_ass2gtdb_acc.tsv \
        | csvtk grep -H -t -f 1 -v -P <(cut -f 1 ncbi_ass2ftp.tsv) \
        | cut -f 1 \
        > ncbi_ass2gtdb_acc.tsv.not-in-sa.tsv 

    
    # -------------- mapping NCBI accession to NCBI assembly_accession -----------------
    
    # parse all genomes, extract accession, and append NCBI assembly_accession.
    # it may report "gzip: stdin: unexpected end of file",
    # may be due to my bad internet connection.
    #
    # time: abount 10 hours with a VPS at San Francisco, CA
    threads=20
    time cat ncbi_ass2ftp.filter.tsv \
        | csvtk replace -H -t -f 2 -p "^ftp" -r "https" \
        | rush -k -j $threads 'wget {2}/{2%}_genomic.fna.gz -o /dev/null -O - \
            | gzip -cd | grep "^>" | cut -f 1 | sed "s/>//" | awk "{print \$1, \"{1}\"}"' \
        | sed 's/ /\t/' \
        | gzip -c \
        > ncbi_acc2ncbi_ass.tsv.gz
        
    # -------------- mapping NCBI accession to GTDB accession -----------------
    
    csvtk replace -H -t -f 2 -p '(.+)' -r '{kv}' --key-miss-repl __ \
        -k ncbi_ass2gtdb_acc.tsv ncbi_acc2ncbi_ass.tsv.gz \
        | gzip -c \
        > ncbi_acc2gtdb_acc.tsv.gz

        
## Result

stats

    # records of gtdb
    echo "records of GTDB: $(cat ncbi_ass2gtdb_acc.tsv | wc -l)"
    # without genbank assebmly assembly_accession
    echo "    these without assembly_accession: $(cat ncbi_ass2gtdb_acc.tsv | grep -c none)"    
    # these in genbank
    echo "    these available in current GenBank: $(cat ncbi_ass2ftp.filter.tsv | wc -l)"
    # actually downloaded
    echo "    actually downloaded: $(zcat ncbi_acc2gtdb_acc.tsv.gz | cut -f 2 | uniq | wc -l)"
    
    records of GTDB: 194600
        these without assembly_accession: 7
        these available in current GenBank: 194436
        actually downloaded: 194431

preview

    zcat ncbi_acc2gtdb_acc.tsv.gz | head -n 5
    LMVM01000001.1  RS_GCF_002287175.1
    LMVM01000002.1  RS_GCF_002287175.1
    LMVM01000003.1  RS_GCF_002287175.1
    LMVM01000004.1  RS_GCF_002287175.1
    LMVM01000005.1  RS_GCF_002287175.1
