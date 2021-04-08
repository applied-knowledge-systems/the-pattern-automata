Scripts to import UMLS Metathesaurus into Redis cluster

need to run 

```
sudo mysql  umls_meta --skip-column-names --raw  < cui_term.sql | redis-cli -p 30004 --pipe
```
on all nodes. 
For demo load to single node:

```
sudo mysql  umls_meta --skip-column-names --raw  < cui_term.sql | redis-cli -h 10.144.17.211 --pipe 
```

To Extract All search terms for building Aho-Corasick automata:

```
#dump all words to CUI into file for Aho Corasick Automata
sudo mysql --skip-column-names --raw umls_meta -e "select WD, CUI from MRXW_ENG" >words_cui.tsv
```
# So far best performing Aho-Corasick automata is made using query above and aho_corasick_intake.py

To install UMLS Metathesaurus follow: 
https://www.nlm.nih.gov/research/umls/knowledge_sources/metathesaurus/index.html


Select canonical names for concepts from UMLS

'''
select CUI, STR from MRCONSO where LAT='ENG' AND TS='P' AND STT='PF' AND ISPREF='Y' limit 5
'''

```
select a.CUI, a.STR from MRCONSO a, MRREL b where a.cui=b.cui2 AND a.LAT='ENG' AND a.TS='P' AND a.STT='PF' AND a.ISPREF='Y' limit 5

SELECT * FROM MRREL WHERE cui2 = 'C0032344' AND stype2 = 'AUI'

```

```
sudo mysql --raw umls_meta  -e "select a.CUI, a.STR,b.* from MRCONSO a, MRREL b where a.cui=b.cui1 AND b.STYPE1='AUI' AND a.LAT='ENG' AND a.TS='P' AND a.STT='PF' AND a.ISPREF='Y' limit 5"
```
Find atom matching a word (non normalised)
```
SELECT b.* FROM MRXW_ENG a, MRCONSO b WHERE a.wd = 'bleeding' AND a.cui = b.cui AND a.sui = b.sui AND b.LAT='ENG' AND b.TS='P' AND b.STT='PF' AND b.ISPREF='Y';
```
(count 7127 for just bleeding)
todo: Find atom matching a word with canonical concept 

1. Find atoms matching a normalized string.

SELECT b.* FROM MRXNS_ENG a, MRCONSO b
WHERE a.nstr = 'abnormal bleed'
     AND a.cui = b.cui
     AND a.lui = b.lui;
sudo mysql --raw umls_meta  -e "SELECT a.nstr,a.cui,b.* FROM MRXNS_ENG a, MRCONSO b WHERE a.nstr = 'abnormal bleed' AND a.cui = b.cui AND a.lui = b.lui;"

sudo mysql --raw umls_meta  -e "SELECT a.nstr,a.cui,b.STR FROM MRXNS_ENG a, MRCONSO b WHERE a.nstr = 'abnormal bleed' AND a.cui = b.cui AND a.lui = b.lui AND b.LAT='ENG' AND b.TS='P' AND b.STT='PF' AND b.ISPREF='Y' ORDER BY CHAR_LENGTH(b.STR);"

select non normalised term, cui and canonical name

sudo mysql --raw umls_meta  -e "SELECT a.wd,a.cui,b.STR FROM MRXW_ENG a, MRCONSO b WHERE a.wd = 'bleeding' AND a.cui = b.cui AND a.sui = b.sui AND b.LAT='ENG' AND b.TS='P' AND b.STT='PF' AND b.ISPREF='Y' ORDER BY CHAR_LENGTH(b.STR);"

# Final solution
(to be validated)
```
sudo mysql --skip-column-names  --raw umls_meta  -e "select STR, CUI from MRCONSO where LAT='ENG' AND TS='P' AND STT='PF' AND ISPREF='Y'" >canonical_str_cui.tsv
```

# TODO:

- [ ] test wd (non normalised term) and nd - normalised term for aho-corasick
- [ ] another approach is to select relevant dictionary based on role: Defined Role (Surgeon vs Rentgenolog vs Nurse), select relevant semantic layer from UMLS (based on Semantic Types: Sign or Symptom) and then select corresponding wd or nstr

Latest Automata available: 

https://s3.eu-west-2.amazonaws.com/assets.thepattern.digital/automata_syns.lzma
both produce good enough results:
- automata_syns.lzma - no filtering applied (filtering on matching)
- automata_fresh_words_cui.pkl - filtered less than 4 characters, not stop words