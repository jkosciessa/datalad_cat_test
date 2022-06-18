# DataCat Brainhack 2022 notes

## Overview / FAQ

- [datalad catalog handbook](http://docs.datalad.org/projects/catalog/en/latest/index.html)

### Where are the files going to be stored?

The catalog will be stored in a standalone folder adjacent to the catalogued dataset. One may for example have a superdataset encompassing different subdatasets (e.g., raw data, preprocessing pipelines, analysis data), with a standalone catalogue encompassing metadata across these different subdatasets. 

### Where do I get the metadata from?

For this purpose, we use the datalad metadata extension in conjunction with translation scripts. There are multiple templates for the extraction of dataset-level information, BIDS metadata, etc. 

## Step-by-step

### Set up virtual environment and install the catalog package

To avoid software environment issues, it is recommended to install the datacat package in a virtual environment. See the [handbook](http://docs.datalad.org/projects/catalog/en/latest/installation.html) for instructions.

### 0. Structuring the superdataset

When there are multiple datasets that should be jointly captured within a catalog, it makes sense to encapsulate them within a subdataset. If subdatasets are nested, it helps catalog creation to have the relevant subdatasets available on the same version. Once the superdataset has been created, we can add metadata for the superdataset ([template](https://github.com/mslw/AIDAqc-dataset/blob/main/.studyminimeta.yaml)), which we will opt to ultimately render at the main page of the catalog.

The superdataset associated with this example catalog can be found [here](https://github.com/jkosciessa/eegmp).

### 1. Extracting metadata using datalad metadata extention

The metadata in the .txt file usually can come from two levels, the (1) dataset level, (2) file level. There are moreover additional extractors, such as for structured BIDS data (see next section). 

```
datalad -f json_pp meta-extract -d . metalad_core
```

The flag `-f json_pp` can be used to display json information in a "pretty print" version in the terminal. We can use this command to visualize the metadata that are being extracted prior to writing them into an external text file. Next, we export the relevant metadata to a .txt file.

```
datalad meta-extract -d . metalad_core > ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d . metalad_studyminimeta >> ./../eegmp_cat/supermetadata.txt
```

We can now also do metadata extraction for the subdatasets.

```
datalad meta-extract -d ./eegmp_data/ metalad_core >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_preproc/ metalad_core >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_analysis/ metalad_core >> ./../eegmp_cat/supermetadata.txt
```

*Note: So far, we have manually extracted these different types of information, but there is also the option to recursively obtain metadata. [TO DO]*

### 2. Extracting BIDS metadata

The datalad `neuroimaging` extension provides tools for the structured extraction of metadata from an existing BIDS directory. We will first use this tool to extract BIDS metadata, and then use a conversion script to translate this metadata into the format expected by `datacat`.

First, we install the neuroimaging package. At the moment, we also need a dev branch containing ongoing updates to the bids extractor that have not been merged yet. Once those have been merged and published, an installation could also be done from `pipy` via the `pip` command.

```
%pip install datalad-neuroimaging
git clone https://github.com/datalad/datalad-neuroimaging.git
cd datalad-neuroimaging/
git fetch origin pull/104/head:update-bids-extractor
git checkout update-bids-extractor
pip install -e .
```

We can now proceed with metadata extraction. The example shows the pretty print visualization in the terminal. Note that in the current example, the actual BIDS directory is `/eegmp_data/eeg_BIDS/`, but we need to point the meta-extract call to a datalad dataset. The function will now look for a `participants.tsv` file to identify the root directory of a valid BIDS directory within a specified dataset. Long story short, the target of the call needs to be the dataset `/eegmp_data/`, not the BIDS root therein (`/eegmp_data/eeg_BIDS/`).

```
datalad -f json_pp meta-extract -d ./eegmp_data/ bids_dataset
```

Ultimately we want to append this metadata to the metadata txt file.

```
datalad meta-extract -d ./eegmp_data/ bids_dataset >> ./../eegmp_cat/supermetadata.txt
```

*Note related to eegmp dataset: before running BIDS metadata extraction, all events.json files were deleted beacuse they were empty cells and could not be retrieved via `datalad get`, thus producing an error when trying to run the `meta-extract` command.*: `rm ./sub-*/eeg/sub-*_task-xxxx_events.json` 

### 3. Extracting datacite.yml information

If you have pushed your data to gin, you may have also created a DOI. This would have required you to create a datacite file containing metadata, which we can now also use to include in the catalogue.

```
datalad meta-extract -d <path-to-dataset> datacite_gin
```

```
datalad meta-extract -d ./eegmp_data/ datacite_gin >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_preproc/ datacite_gin >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_analysis/ datacite_gin >> ./../eegmp_cat/supermetadata.txt
```

### 4. Extract metadata at the file level

```
datalad -f json meta-conduct <path-to-file-extract_file_metadata.json> \
    traverser.top_level_dir=<path-to-specific-dataset-for-which-file-metadata-is-to-be-extracted> \
    traverser.item_type=file \
    extractor.extractor_type=file \
    extractor.extractor_name=metalad_core \
    | jq -c '.["pipeline_data"]["result"]["metadata"][0]["metadata_record"]' >> <path-to-output-file>
```

- *Note: `extract_file_metadata.json` is part of the `fairly-big-catalog-workflow` [repository](https://github.com/jsheunis/fairly-big-catalog-workflow.git); see below for clone instructions*
- *Note: There may be an issue with relative paths, safer to specify absolute paths for now ([see this issue](https://github.com/datalad/datalad-metalad/issues/253))*
- *Note: Be aware that this step may take a while, depending on how many files need to be indexed.*
- *Question: What would happen to indexing if file indices are not displayed in the filesystem? Are the indexers independent of these filesystem-level issues (e.g., in Mac's Finder, see this [issue on broken symlinks in the handbook](http://handbook.datalad.org/en/stable/basics/101-115-symlinks.html?highlight=Finder#broken-symlinks)).*

e.g., applied to my specific example:

```
datalad -f json meta-conduct /Users/kosciessa/hackathon2022/fairly-big-catalog-workflow/conduct_pipelines/extract_file_metadata.json \
    traverser.top_level_dir=/Users/kosciessa/hackathon2022/eegmp/eegmp_data/ \
    traverser.item_type=file \
    extractor.extractor_type=file \
    extractor.extractor_name=metalad_core \
    | jq -c '.["pipeline_data"]["result"]["metadata"][0]["metadata_record"]' >> /Users/kosciessa/hackathon2022/eegmp_cat/eegmp_data_filemeta.txt

datalad -f json meta-conduct /Users/kosciessa/hackathon2022/fairly-big-catalog-workflow/conduct_pipelines/extract_file_metadata.json \
    traverser.top_level_dir=/Users/kosciessa/hackathon2022/eegmp/eegmp_preproc/ \
    traverser.item_type=file \
    extractor.extractor_type=file \
    extractor.extractor_name=metalad_core \
    | jq -c '.["pipeline_data"]["result"]["metadata"][0]["metadata_record"]' >> /Users/kosciessa/hackathon2022/eegmp_cat/eegmp_preproc_filemeta.txt

datalad -f json meta-conduct /Users/kosciessa/hackathon2022/fairly-big-catalog-workflow/conduct_pipelines/extract_file_metadata.json \
    traverser.top_level_dir=/Users/kosciessa/hackathon2022/eegmp/eegmp_analysis/ \
    traverser.item_type=file \
    extractor.extractor_type=file \
    extractor.extractor_name=metalad_core \
    | jq -c '.["pipeline_data"]["result"]["metadata"][0]["metadata_record"]' >> /Users/kosciessa/hackathon2022/eegmp_cat/eegmp_analysis_filemeta.txt
```

### Intermediate overview

From the eegmp superdataset, run

```
datalad meta-extract -d . metalad_core > ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d . metalad_studyminimeta >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_data/ metalad_core >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_preproc/ metalad_core >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_analysis/ metalad_core >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_data/ bids_dataset >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_data/ datacite_gin >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_preproc/ datacite_gin >> ./../eegmp_cat/supermetadata.txt
datalad meta-extract -d ./eegmp_analysis/ datacite_gin >> ./../eegmp_cat/supermetadata.txt
cd ..
```

### Translate the datalad-metadata .txt output into the correct nomenclature for datalad-catalog

Now we should have one metadata.txt file containing dataset-level metadata, as well as multiple dataset-specific matadata.txt files containing the respective file information. First, we merge these files to derive one .txt file containing all relevant infos.

```
cd ..
cat ./eegmp_cat/eegmp_data_filemeta.txt ./eegmp_cat/eegmp_preproc_filemeta.txt ./eegmp_cat/eegmp_analysis_filemeta.txt >> eegmp_cat/supermetadata.txt
```
The current metadata contained in the .txt file now needs to be translated into the format expected by the datalad catalog. Handily, there are scripts for this translation process.

```
git clone https://github.com/jsheunis/fairly-big-catalog-workflow.git
cd fairly-big-catalog-workflow
chmod -R u+rwx ./extractor_translators/*
chmod -R u+rwx ./local/*
cd local
./local_translate_metadata.sh ./../../eegmp_cat/supermetadata.txt ./../../eegmp_cat/supermetadata_translated.txt 
```

The key command here is `./local_translate_metadata.sh <metadata.txt> <metadata_translated.txt>`

This produces a file called `<original-name>_translated.txt` that contains the metadata as required by datalad catalog.

- *Note: The most recent conversion script is located in the developer branch: `git checkout stephan_dev`.*
- *Note: The conversion script required `zsh` as a shell.*
- *Note: When many file-level data are included, the conversion process may also be lengthy.*

### Set up a config file to specify metadata to be used to construct the catalog 

- [yml template](https://github.com/datalad/datalad-catalog/blob/main/datalad_catalog/templates/config.yml)
- [json template](https://github.com/datalad/datalad-catalog/blob/main/datalad_catalog/templates/config.json)

This config file can be specified during the creation of the catalog. The core functionality of the config is to specify the source of the metadata for some fields of the catalog. The config currently applies on the whole catalog level, but may be changed in the future to apply to the dataset level.There are three options: (1) merge all sources (==> "merge"), (2) select a single source (e.g. "metalad_core"), (3) or include the value from all sources as a list (e.g. ["metalad_core", "bids_dataset"]). A "source" is the name of an extractor. Here are some examples for how sources can be specified. The sources are named after the metadata tools used to extract the data.

- e.g. dataset_id (and version) are the same coming from all extractors, so selecting a single one is fine (i.e. metalad_core)
- same for type
- children are "merge" because different children objects could be added by different extractors and we want them all, so don't change that
- interim comment: the config currently applies on the whole catalog level, but I think it should be changed to apply on the dataset level. for example for one dataset you might have the metadata from metalad_studyminimeta and for another from datacite_gin. with the current config you can only specify on the catalog level what a field's source should be, which wouldn't account for datasets getting metadata from different extractors.
- e.g. in your case: name would come from metalad_studyminimeta for the superdataset, and from datacite_gin for the others. in these cases, i think the best for now is to set those fields in the config to empty strings
- In case a field is left empty, datalad-catalog will assign it the first value it encounters.

## Creating the catalog, using the config

```
datalad catalog create -c <path-to-your-catalog-directory> -m <path-to-your-translated-metadata-file> -y <path-to-your-config-file>
```

The catalog directory does not need to exist yet, otherwise a `-f` force flag has to be provided.

In this particular case: 

```
datalad catalog create -c ./eegmp_catalog -m ./eegmp_cat/supermetadata_translated.txt -y ./eegmp_cat/config.json
```

If successful, we should see something like this:

```
catalog create(ok): /Users/kosciessa/hackathon2022 [Catalog successfully created at: eegmp_catalog]
catalog add(ok): /Users/kosciessa/hackathon2022 [Metadata items successfully added to catalog]
action summary:
  catalog add (ok: 1)
  catalog create (ok: 1)
```
### Set the superdataset of the catalog

Now we want to set the superdataset of the catalog, to tell it which one to navigate to when opening the catalog in the browser. We need the superdataset's id and version for that, which we get from the metadata.

```
datalad catalog set-super -c <path-to-your-catalog-directory> -i <dataset_id> -v <dataset_version>
```

Example:

```
datalad catalog set-super -c eegmp_catalog/ -i e3076b62-d690-4b85-9c2d-e65aee6c057d -v 770f5729d088d5c491f370d24f3d7a9e0235027d
catalog set-super(ok): /Users/kosciessa/hackathon2022 [Superdataset successfully set for catalog]
```
*Hint: The id and version information can be manually extracted from the metadata.txt files.*

This will save a `super.json` file in the `metadata` directory of the catalog folder.

## Serving the catalog

Now we can check whether the catalog setup has worked by hosting a version of the catalog locally. 

```
datalad catalog serve -c <path-to-your-catalog-directory>
```

In the example case:

```
datalad catalog serve -c ./eegmp_catalog
```
We can then inspect the catalog in a browser of our choice by navigating to `http://localhost:8000/`.
