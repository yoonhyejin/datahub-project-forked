# Adding Terms

## Why Would You Add Terms? 
Terms are something you can ~~~. You can do 
* This
* That

Fore more information about terms, refer to [About Datahub Terms](https://datahubproject.io/docs/terms/).

## Pre-requisites

### Deploy Datahub Quickstart 
You need a datahub running in local environment. First, install acryl-datahub if you did't yet. 
```shell
python3 -m pip install --upgrade pip wheel setuptools
python3 -m pip install --upgrade acryl-datahub
```
If you can see datahub version like this, you're good to go. 
```shell
$ datahub version
DataHub CLI version: 0.10.0.1
Python version: 3.9.6 (default, Jun 16 2022, 21:38:53)
[Clang 13.0.0 (clang-1300.0.27.3)]
```

Run datahub quickstart. This will deploy local datahub server to http://localhost:9002 
```shell
datahub docker quickstart
```
After logging in with the default credential(`username: datahub / password: datahub`), you can see Datahub ready for you. 

![datahub-main-ui](../../imgs/tutorials/datahub-main-ui.png)

Please refer to [DataHub Quickstart Guide](https://datahubproject.io/docs/quickstart) for more information. 

### Ingest Sample Data
We will use sample data provided with datahub quickstart. 
If you already have data on your datahub, you might skip this part. 

```shell
datahub docker ingest-sample-data 
```
This will ingest various entities like datasets, tags and terms to your local Datahub.
![datahub-main-ui](../../imgs/tutorials/sample-ingestion.png)

## Example Usage

:::note
Adding terms assumes that you already have a dataset and terms on your datahub.
If you try to manipulate with entities that does not exist, it might return errors or failed to reference them.
:::


### Add Terms With Python SDK

Following codes add a glossary term named `CustomerAccount` to a column `user_name` of a hive dataset named `fct_users_created`.
You can refer to a full code in [dataset_add_column_term.py](https://github.com/datahub-project/datahub/blob/master/metadata-ingestion/examples/library/dataset_add_column_term.py).


```python
# inlined from metadata-ingestion/examples/library/dataset_add_column_term.py
import logging
import time

from datahub.emitter.mce_builder import make_dataset_urn, make_term_urn
from datahub.emitter.mcp import MetadataChangeProposalWrapper

# read-modify-write requires access to the DataHubGraph (RestEmitter is not enough)
from datahub.ingestion.graph.client import DatahubClientConfig, DataHubGraph

# Imports for metadata model classes
from datahub.metadata.schema_classes import (
    AuditStampClass,
    EditableSchemaFieldInfoClass,
    EditableSchemaMetadataClass,
    GlossaryTermAssociationClass,
    GlossaryTermsClass,
)

log = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)


def get_simple_field_path_from_v2_field_path(field_path: str) -> str:
    """A helper function to extract simple . path notation from the v2 field path"""
    if not field_path.startswith("[version=2.0]"):
        # not a v2, we assume this is a simple path
        return field_path
        # this is a v2 field path
    tokens = [
        t for t in field_path.split(".") if not (t.startswith("[") or t.endswith("]"))
    ]

    return ".".join(tokens)


# Inputs -> the column, dataset and the term to set
column = "user_name"
dataset_urn = make_dataset_urn(platform="hive", name="fct_users_created", env="PROD")
term_to_add = make_term_urn("User")


# First we get the current editable schema metadata
gms_endpoint = "http://localhost:8080"
graph = DataHubGraph(DatahubClientConfig(server=gms_endpoint))


current_editable_schema_metadata = graph.get_aspect(
    entity_urn=dataset_urn, aspect_type=EditableSchemaMetadataClass
)


# Some pre-built objects to help all the conditional pathways
now = int(time.time() * 1000)  # milliseconds since epoch
current_timestamp = AuditStampClass(time=now, actor="urn:li:corpuser:ingestion")

term_association_to_add = GlossaryTermAssociationClass(urn=term_to_add)
term_aspect_to_set = GlossaryTermsClass(
    terms=[term_association_to_add], auditStamp=current_timestamp
)
field_info_to_set = EditableSchemaFieldInfoClass(
    fieldPath=column, glossaryTerms=term_aspect_to_set
)

need_write = False
field_match = False
if current_editable_schema_metadata:
    for fieldInfo in current_editable_schema_metadata.editableSchemaFieldInfo:
        if get_simple_field_path_from_v2_field_path(fieldInfo.fieldPath) == column:
            # we have some editable schema metadata for this field
            field_match = True
            if fieldInfo.glossaryTerms:
                if term_to_add not in [x.urn for x in fieldInfo.glossaryTerms.terms]:
                    # this tag is not present
                    fieldInfo.glossaryTerms.terms.append(term_association_to_add)
                    need_write = True
            else:
                fieldInfo.glossaryTerms = term_aspect_to_set
                need_write = True

    if not field_match:
        # this field isn't present in the editable schema metadata aspect, add it
        field_info = field_info_to_set
        current_editable_schema_metadata.editableSchemaFieldInfo.append(field_info)
        need_write = True

else:
    # create a brand new editable schema metadata aspect
    current_editable_schema_metadata = EditableSchemaMetadataClass(
        editableSchemaFieldInfo=[field_info_to_set],
        created=current_timestamp,
    )
    need_write = True

if need_write:
    event: MetadataChangeProposalWrapper = MetadataChangeProposalWrapper(
        entityUrn=dataset_urn,
        aspect=current_editable_schema_metadata,
    )
    graph.emit(event)
    log.info(f"Term {term_to_add} added to column {column} of dataset {dataset_urn}")

else:
    log.info(f"Term {term_to_add} already attached to column {column}, omitting write")

```

### Add Terms With GraphQL
There are several ways to do it with GraphQL : GraphQL Explorer, HTTP CURL, Postman or GraphQL SDKs. 

#### GraphQL Explorer
Navigate to GraphQL Explorer (`http://localhost:9002/api/graphiql`) and run the following query.

```python
mutation addTerms {
    addTerms(
      input: { 
        termUrns: ["urn:li:glossaryTerm:CustomerAccount"], 
        resourceUrn: "urn:li:dataset:(urn:li:dataPlatform:hive,fct_users_created,PROD)",
        subResourceType:DATASET_FIELD,
        subResource:"user_name"})
}
```
You are succeeded if you see the following response.
```python
{
  "data": {
    "addTerms": true
  },
  "extensions": {}
}
```

#### CURL

With CURL, you need to provide tokens. To generate token, please refer [Generate Access Token](http://localhost:3000/docs/dev-guides/tutorials/generate-access-token). 
With `accessToken`, you can run the following command.

```shell
curl --location --request POST 'http://localhost:8080/api/graphql' \
--header 'Authorization: Bearer <my-access-token>' \
--header 'Content-Type: application/json' \
--data-raw '{ "query": "mutation addTerm { addTerms(input: { termUrns: [\"urn:li:glossaryTerm:CustomerAccount\"], resourceUrn: \"urn:li:dataset:(urn:li:dataPlatform:hive,fct_users_created,PROD)\" }) }", "variables":{}}'
```

Expected Response:

```json
{"data":{"addTerms":true},"extensions":{}}
```

## Expected Outcomes
You can now see `CustomerAccount` tag has been added to `user_name` column. 
![term-added](../../imgs/tutorials/term-added.png)


## What’s Next

