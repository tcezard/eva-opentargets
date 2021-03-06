# Manual curation, part I, technical: fetch the latest ClinVar data, attempt automatic mapping, extract unmapped traits

## Run the automated protocol
Before running, set up the environment:
* [Common environment](../environment.md)
* [Protocol-specific environment](README.md#setting-up-environment)

```bash
# Create directories for data processing
mkdir -p ${CURATION_RELEASE_ROOT}

# Download the latest ClinVar variant summary data
wget -q \
  --directory-prefix ${CURATION_RELEASE_ROOT} \
  ${CLINVAR_PATH_BASE}/tab_delimited/variant_summary.txt.gz

# Run the trait mapping pipeline
cd ${CODE_ROOT} && ${BSUB_CMDLINE} -K -M 4G \
  -o ${CURATION_RELEASE_ROOT}/log.trait_mapping.out \
  -e ${CURATION_RELEASE_ROOT}/log.trait_mapping.err \
  python3 bin/trait_mapping.py \
  -i ${CURATION_RELEASE_ROOT}/variant_summary.txt.gz \
  -o ${CURATION_RELEASE_ROOT}/automated_trait_mappings.tsv \
  -c ${CURATION_RELEASE_ROOT}/traits_requiring_curation.tsv

# Download the latest eva_clinvar release from FTP. At this step, mappings produced by the pipeline on the previous
# iteration (including automated and manual) are downloaded to be used to aid the manual curation process.
wget -qO- ftp://ftp.ebi.ac.uk/pub/databases/eva/ClinVar/latest/eva_clinvar.txt \
  | cut -f4-5 | sort -u > ${CURATION_RELEASE_ROOT}/previous_mappings.tsv

# Create the final table for manual curation
cd ${CODE_ROOT} && python3 bin/trait_mapping/create_table_for_manual_curation.py \
  --traits-for-curation ${CURATION_RELEASE_ROOT}/traits_requiring_curation.tsv \
  --previous-mappings ${CURATION_RELEASE_ROOT}/previous_mappings.tsv \
  --output ${CURATION_RELEASE_ROOT}/table_for_manual_curation.tsv

# Sort and export to Google Sheets. Note that the number of columns in the output table is limited to 50, because only a
# few traits have that many mappings, and in virtually all cases these extra mappings are not meaningful. However, having
# a very large table degrades the performance of Google Sheets substantially.
cut -f-50 ${CURATION_RELEASE_ROOT}/table_for_manual_curation.tsv \
  | sort -t$'\t' -k2,2rV > ${CURATION_RELEASE_ROOT}/google_sheets_table.tsv
```

## Create a Google spreadsheet for curation

Duplicate a [template](https://docs.google.com/spreadsheets/d/1PyDzRs3bO1klvvSv9XuHmx-x7nqZ0UAGeS6aV2SQ2Yg/edit?usp=sharing). Paste the contents of `${CURATION_RELEASE_ROOT}/google_sheets_table.tsv` file into it, starting with column H “ClinVar label”. Example of a table fully populated with data can be found [here](https://docs.google.com/spreadsheets/d/1HQ08UQTpS-0sE9MyzdUPO7EihMxDb2e8N14s1BknjVo/edit?usp=sharing).
