---
layout: post
date: 2025-06-04
title: "Pandas UDFs in Apache Spark"
date: 2025-06-04
categories: [Apache Spark, Pandas, UDFs]
tags: [spark, pyspark, pandas, udf, big data]
author: "wrschneider"
description: investigating usage and performance of Python UDFs in Spark with Pandas 
---
One of the reasons I've preferred Scala for working with Spark, is the ability to define complex logic in a UDF without as big of a performance penalty as Python UDFs.  (Aside from the [performance risk of UDFs in general]({% post_url 2025-05-23-spark-performance %}).)

That changed when [`pandas_udf`](https://spark.apache.org/docs/3.5.6/api/python/reference/pyspark.sql/api/pyspark.sql.functions.pandas_udf.html) 
was introduced.  There is still overhead of calling the Python interpreter from the JVM rather than executing the UDF
natively in the JVM, but data serialization is done in batch rather than row-by-row.  Also, using Pandas gives you the option of using vectorized
operations for performance; in a vectorized operation, iteration over rows in a Pandas dataframe is handled in underlying native libraries rather than
interpreted Python code.

The `pandas_udf` has been around for a while, and now it has higher salience for me as I am now encountering situations where Python might be
a better choice than Scala:

- Python and Pandas are ubiquitous in the data science community, while Scala can be a barrier to adoption or collaboration.
- Libraries like [MLflow](https://docs.databricks.com/aws/en/machine-learning/model-inference/dl-model-inference) tend to have more comprehensive support for Python. MLflow uses `pandas_udf` under the hood to do batch inference on Spark dataframes.

Here's an example that I experimented with.  First an example of how I might have written code in Scala:

```scala
case class Foo(
    id: Long,
    value: String
)

def concatUdf = udf((f: Foo) => s"concatenated ${f.id} ${f.value}")

df.select(concatUdf(struct($"id", $"value")).as("concat_value"))
```

An equivalent, basic Python UDF would be something like this

```python
@udf(returnType="string")
def ordinary_python_udf(row):
    return f"concatenated {row.id} {row.value}"
```

This will incur overhead of serializing every row to the Python interpreter individually.

String concatenation is simple enough that it can be done with native, vectorized Pandas operations, which will be much faster than
the first pass above.  Note that the `struct` type, which can translate to either a `Row` or case class in Scala UDFs, translates to a Pandas `DataFrame`, and
the return type will be a Pandas `Series`, analogous to a Spark `Column`:

```python
import pandas as pd

@pandas_udf(returnType="string")
def vectorized_pandas_udf(df: pd.DataFrame) -> pd.Series:
    return "concatenated " + df["id"].astype("string") + " " + df["value"].astype("string")
```

This above approach is good when you have logic that works on Pandas dataframes but not Spark dataframes, and can be vectorized.

Here's another example that would work with arbitrary Python code:

```python
import pandas as pd

@pandas_udf(returnType="string")
def pandas_apply_udf(df: pd.DataFrame) -> pd.Series:
    return df.apply(lambda row: f"pandas formatted: {row.id} {row.value}", axis=1)
```

In this case `row` will be passed in a `Series` object, and the `axis=1` is necessary to specify that the series should be all
the columns for a single row, rather than all the rows for a single column.

This is similar to the `ordinary_python_udf` example, where arbitrary Python code is executed per row, but using Pandas for serialization.

*Surprise!* in my contrived example I found the `pandas_apply_udf` version was actually *slower* than the `ordinary_python_udf` example.

So I tried another thing, to use `raw` to improve performance within `apply`:

```python
@pandas_udf(returnType="string")
def pandas_apply_udf(df: pd.DataFrame) -> pd.Series:
    cols = df.columns.tolist()
    id = cols.index('id')
    value = cols.index('value')
    
    return df.apply(lambda row: f"pandas formatted: {row[id]} {row[value]}", axis=1, raw=True)
```

This was almost as fast as the `pandas_vectorized_udf` version.

The Pandas `DataFrame` documentation says [this about `apply`](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.apply.html): 
> If `raw=True`, the passed function will receive ndarray objects instead of Series. This can improve performance when applying NumPy reduction functions or 
> other operations that work directly on arrays, as it avoids the overhead of creating Series objects for each row or column.

My best guess here is, the overhead of creating a Series object for each row was sufficient to negate
the benefits of serialization from Pandas UDFs.

The catch with `raw` is, since the passed function gets an ndarray instead of a Series, you index into individual struct fields by position rather
than name.  So in order to preserve similar behavior you can get the column list from the DataFrame and determine the position of each
field.

So, `pandas_udf` is a valuable tool for PySpark UDF performance, but you need to be aware of how it works to see the benefits.