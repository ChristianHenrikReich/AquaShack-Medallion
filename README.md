# AquaShack-Medallion

A pico-example of a meta-data driven lakehouse for Microsoft Fabric, forked from
[AquaShack](https://github.com/ChristianHenrikReich/AquaShack). Two differences
from the original:

1. The three layers are renamed to the classic medallion names: **Bronze**,
   **Silver**, and **Gold** (previously Landing, Base, Curated).
2. The meta data schema supports a `custom_transform` property. When set, the
   Bronze to Silver stage dispatches to a named custom function instead of
   running the default dedupe + fill-nulls flow. This is the extension point
   for entities that need shaping the default flow does not cover.

## Installation

`Setup.ipynb` is a single-script installer; run it end to end and everything
needed is provisioned.

## Documentation

Medallion architecture, three lakehouses:

- **Bronze**: data in original format where possible. Relational sources land
  as parquet. No Delta, no re-writing, no querying.
- **Silver**: aligned, cleaned data. All Delta tables.
- **Gold**: serving layer. Business logic, star schemas, etc.

Notebooks:

- `1_AquaShack_Bronze_To_Silver.ipynb` moves meta-data-described entities from
  Bronze to Silver. Default flow is dedupe + fill nulls. If an entity has a
  `custom_transform` in the meta data, that function runs instead.
- `2_AquaShack_Silver_To_Gold.ipynb` coordinates the Gold loaders.
- `3_*` and `4_*` load dimensions and facts into Gold.

## The `custom_transform` extension point

Default flow in Bronze to Silver:

```python
df = df.dropDuplicates()
df = df.na.fill(0)
df = df.na.fill("N/A")
```

That is correct for most entities. For entities that need more, name a custom
function in `data/meta-data/meta_data.json`:

```json
{
    "source": "Files/data/Sales/Transactions/Transactions.csv",
    "format": "csv",
    "destination": "sales_Transactions",
    "projected_columns": [],
    "custom_transform": "enrich_transactions"
}
```

And register the function in `notebooks/AquaShack_functions.ipynb`:

```python
@register_custom_transform("enrich_transactions")
def enrich_transactions(df: DataFrame, meta_entity: dict) -> DataFrame:
    df = df.dropDuplicates()
    df = df.na.fill(0)
    df = df.na.fill("N/A")
    df = df.withColumn("TransactionDateId",
                       F.expr("cast(date_format(to_date(TransactionDate), 'yyyyMMdd') as int)"))
    df = df.withColumn("LoadedAt", F.current_timestamp())
    return df
```

Signature contract: `(df, meta_entity) -> df`. The caller still handles the
write, so all write policy (IDs, audit, mode, projection) stays in one place.

## Articles

Same articles as the upstream project:

- [Delta and Parquet: Integer, GUID/UUID or SHA256 as ID?](https://medium.com/@christianhenrikreich/delta-and-parquet-integer-guid-uuid-or-sha256-as-id-67ba15b4437f)
- [Spark SQL: Why the choice of language does not impact performance](https://medium.com/@christianhenrikreich/spark-sql-why-the-choice-of-language-doesnt-impact-performance-747ff3f854ae)
- [Data Architecture: Data capture time and event time in medallion architecture](https://medium.com/@christianhenrikreich/data-architecture-data-capture-time-and-event-time-in-medallion-architecture-dceb93980e0a)
- [Microsoft Fabric: Building Pseudo Identity Columns Without monotonically_increasing_id() in Spark](https://medium.com/@christianhenrikreich/microsoft-fabric-building-pseudo-identity-columns-without-monotonically-increasing-id-in-spark-09b1efb577d1)
- [Lakehousing: Removing one of the biggest performance killers in Silver-to-Gold processing](https://medium.com/@christianhenrikreich/lakehousing-removing-one-of-the-biggest-performance-killers-in-bronze-to-silver-processing-8ed4ce372de6)
