# Adding Column Level Lineage

## Why Would You Add Column Level Lineage? 
Column level lineage helps to track the lineage of a specific column within a data asset, ensuring data quality and tracing data issues. 
It also provides a detailed audit trail of how data has been transformed and used within an organization, which can be useful for compliance and regulatory requirements. 
Additionally, it helps to understand the impact of changes to a column on downstream data consumers.

:::note
Adding column level lineage via GraphQL is not supported yet.
:::


## Prerequisites
For this tutorial, you need to deploy DataHub Quickstart and ingest sample data. 
For detailed steps, please refer to [Prepare Local DataHub Environment](/docs/tools/tutorials/references/prepare-datahub.md).

:::note
Before adding lineage, you need to ensure the targeted dataset is already present in your datahub. 
If you attempt to manipulate entities that do not exist, your operation will fail. 
In this guide, we will be using data from a sample ingestion.
:::

In this example, we will add column level lineage between two hive dataset named `fct_users_created` and `fct_users_deleted`.

## Add Lineage With Python SDK

You can refer to the related code in [lineage_emitter_dataset_finegrained.py](https://github.com/datahub-project/datahub/blob/master/metadata-ingestion/examples/library/lineage_emitter_dataset_finegrained.py).
```python
# inlined from metadata-ingestion/examples/library/lineage_emitter_dataset_finegrained.py
import datahub.emitter.mce_builder as builder
from datahub.emitter.mcp import MetadataChangeProposalWrapper
from datahub.emitter.rest_emitter import DatahubRestEmitter
from datahub.metadata.com.linkedin.pegasus2avro.dataset import (
    DatasetLineageType,
    FineGrainedLineage,
    FineGrainedLineageDownstreamType,
    FineGrainedLineageUpstreamType,
    Upstream,
    UpstreamLineage,
)


def datasetUrn(tbl):
    return builder.make_dataset_urn("hive", tbl)


def fldUrn(tbl, fld):
    return builder.make_schema_field_urn(datasetUrn(tbl), fld)

fineGrainedLineages = [
    FineGrainedLineage(
        upstreamType=FineGrainedLineageUpstreamType.FIELD_SET,
        upstreams=[fldUrn("fct_users_created", "user_id")],
        downstreamType=FineGrainedLineageDownstreamType.FIELD,
        downstreams=[fldUrn("fct_users_deleted", "user_id")],
    )
]

# this is just to check if any conflicts with existing Upstream, particularly the DownstreamOf relationship
upstream = Upstream(dataset=datasetUrn("fct_users_deleted"), type=DatasetLineageType.TRANSFORMED)

fieldLineages = UpstreamLineage(
    upstreams=[upstream], fineGrainedLineages=fineGrainedLineages
)

lineageMcp = MetadataChangeProposalWrapper(
    entityUrn=datasetUrn("fct_users_created"),
    aspect=fieldLineages,
)

# Create an emitter to the GMS REST API.
emitter = DatahubRestEmitter("http://localhost:8080")

# Emit metadata!
emitter.emit_mcp(lineageMcp)

```

We're using the `MetdataChangeProposalWrapper` to change entities in this example.
For more information about the `MetadataChangeProposal`, please refer to [MetadataChangeProposal & MetadataChangeLog Events](/docs/advanced/mcp-mcl.md)


## Expected Outcomes
You can now see lineage between `fct_users_created` and `fct_users_deleted`.

[//]: # (![lineage-added]&#40;../../imgs/tutorials/lineage-added.png&#41;)

