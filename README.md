# IMNGSanalysis

This code involves a series of scripts for the analysis, filtering and classification of the data obtained through IMNGS.
IMNGS, the Integrated Microbial NGS Platform, is an online tool that screens for and processes all prokaryotic 16S rRNA gene amplicon datasets available in SRA and uses them to build sample-specific sequence databases (Lagkouvardos et al., 2016). Therefore, this tool can quantify the prevalence and abundance of any 16S rRNA sequence of interest introduced as input, giving information about the number of coincidences with any of the samples included in its database, and also about the type of sample (e.g., soil, water, anaerobic digester, environmental, etc.).

For our anaerobic digestion microbiome study, sequences were submitted to IMNGS and analysed through a series of python scripts, including scraping.

From the files obtained from IMNGS, *IMNGS_97_counts_matrix.tab* was used, at first only taking into account the samples in *IMNGS_97_abun_0.1_matrix.tab*, which contains only those samples in which the abundance of the desired sequence is above **0.1**.

### Parameters
```
imngs_counts = "path/to/IMNGS_97_counts_matrix.tab"
abun01 = "path/to/IMNGS_97_abun_0.1_matrix.tab"
Output = "path/to/Output"
```

### Selection of the samples (abundance > 0.1%)
```
dbs_yes = []
for line in abun01:
    dbs_yes.append(line.split('\t')[0])
```

### Writing a new file with only the 0.1% abundance samples
```
output = 'path/to/imngs_counts01'

for line in imngs_counts:
    db = line.split('\t')[0]
    if db in dbs_yes:
        output.write(line)
```

### Definition of the scraping function
```
from bs4 import BeautifulSoup
import requests

def scraping(url):
    req = requests.get(url)
    html_content = req.text
    soup = BeautifulSoup(html_content, 'html.parser')
    summary = 'no results'
    description = 'no results'
    for word in soup.find_all('p', {'class' : 'ncbi-doc-excerpt'}):
        summary = word.text
    for word in soup.find_all('span', {'class' : 'ncbi-doc-authors'}):
        description = word.text
    return (summary+'\t'+description)
```

### Filtering and scraping of the samples
In this step, if none of the words related to *anaerobic digestion* is present (i.e, words *biogas, anaerobic, reactor, digester*), then the scraping is performed. 
Then, if some word related to AD is present either in the description or in the sample type, it will be classified as positive. "methan" and "aceto" terms are added to be looked for in the description.
```
counts_abund01 = open(Output + '/counts_abund01.csv')
output2 = open(Output + '/scraping_result.csv')

for line in counts_abund01:
    sample_type = line.split('\t')[1].rstrip()
    db = line.split('\t')[0].rstrip()
    if 'biogas' not in sample_type and 'anaerobic' not in sample_type and 'reactor' not in sample_type and 'digester' not in sample_type:
        url = 'https://www.ncbi.nlm.nih.gov/search/all/?term='+db
        check = scraping(url).lower()
        if 'biogas' in check or 'anaerobic' in check or 'reactor' in check or 'methan' in check or 'aceto' in check or 'digester' in check:
            output2.write(db+'\t'+sample+'\t+\t'+check+'\n')
        else:
            output2.write(db+'\t'+sample+'\t-\t'+check+'\n')
```
### Manual curation
The results in *scraping_result.csv* are manually curated. This is included, on the one hand, to make sure that every tag related to anaerobic digestion has been reclassified as such. On the other hand, there are many tags which are ambiguous or inespecific, so it is necessary to reclassify them into more concrete groups (for example, *metagenome* into *soil metagenome* or *human metagenome*, depending on the information recovered from the scraping).
After this step, *positive* samples (tagged with a + next to the sample number) will be the ones related to anaerobic digestion.

```
scrap = open('/results_scrapping.csv')
scrap_positive = []
for line in scrap:
    db = line.split('\t')[0]
    if line.split('\t')[2] == '+':
        scrap_positive.append(db)

for line in counts_abund01:
    line = line.rstrip()
    sample = line.split('\t')[1]
    db = line.split('\t')[0]
    abund = float(line.split('\t')[19])
    if 'biogas' in sample or 'anaerobic' in sample or 'bioreactor' in sample or db in scrap_positive:
        sample = 'anaerobic digestion'
    if sample not in sampleList:
        sampleList[sample] = abund
    else:
        sampleList[sample] += abund
```
At the end, the object *sampleList* would give a list with all the groups and its abundances.
