---
title: "sparklyr: R interface for Apache Spark"
output:
  github_document:
    fig_width: 9
    fig_height: 5
---

![Github workflow status](https://github.com/sparklyr/sparklyr/workflows/CI/badge.svg) [![Travis build Status](https://travis-ci.org/sparklyr/sparklyr.svg?branch=master)](https://travis-ci.org/sparklyr/sparklyr) [![AppVeyor build status](https://ci.appveyor.com/api/projects/status/qjosuhlp55nwv42y?svg=true)](https://ci.appveyor.com/project/javierluraschi71148/sparklyr) [![CRAN_Status_Badge](https://www.r-pkg.org/badges/version/sparklyr)](https://cran.r-project.org/package=sparklyr) <a href="https://www.r-pkg.org/pkg/sparklyr"><img src="https://cranlogs.r-pkg.org/badges/sparklyr?color=brightgreen" style=""></a> [![codecov](https://codecov.io/gh/sparklyr/sparklyr/branch/master/graph/badge.svg)](https://codecov.io/gh/sparklyr/sparklyr) [![Join the chat at https://gitter.im/sparklyr/sparklyr](https://badges.gitter.im/sparklyr/sparklyr.svg)](https://gitter.im/sparklyr/sparklyr?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

<img src="tools/readme/sparklyr-illustration.png" width="320" align="right" style="margin-left: 20px; margin-right: 20px"/>



- Install and connect to [Spark](http://spark.apache.org/) using YARN, Mesos, Livy or Kubernetes.
- Use [dplyr](https://spark.rstudio.com/dplyr/) to filter and aggregate Spark datasets and [streams](https://spark.rstudio.com/guides/streaming/) then bring them into R for analysis and visualization.
- Use [MLlib](http://spark.apache.org/docs/latest/mllib-guide.html), [H2O](https://spark.rstudio.com/guides/h2o/), [XGBoost](https://github.com/rstudio/sparkxgb) and [GraphFrames](https://github.com/rstudio/graphframes) to train models at scale in Spark.
- Create interoperable machine learning [pipelines](https://spark.rstudio.com/guides/pipelines/) and productionize them with [MLeap](https://spark.rstudio.com/guides/mleap/).
- Create [extensions](http://spark.rstudio.com/extensions.html) that call the full Spark API or run [distributed R](https://spark.rstudio.com/guides/distributed-r/) code to support new functionality.

## Installation

You can install the **sparklyr** package from [CRAN](https://CRAN.r-project.org) as follows:


```r
install.packages("sparklyr")
```

You should also install a local version of Spark for development purposes:


```r
library(sparklyr)
spark_install()
```

To upgrade to the latest version of sparklyr, run the following command and restart your r session:


```r
install.packages("devtools")
devtools::install_github("sparklyr/sparklyr")
```

## Connecting to Spark

You can connect to both local instances of Spark as well as remote Spark clusters. Here we'll connect to a local instance of Spark via the [spark_connect](https://spark.rstudio.com/reference/spark-connections/) function:


```r
library(sparklyr)
```

```
## Error in library(sparklyr): there is no package called 'sparklyr'
```

```r
sc <- spark_connect(master = "local")
```

```
## Error in spark_connect(master = "local"): could not find function "spark_connect"
```

The returned Spark connection (`sc`) provides a remote dplyr data source to the Spark cluster.

For more information on connecting to remote Spark clusters see the [Deployment](http://spark.rstudio.com/deployment.html) section of the sparklyr website.

## Using dplyr

We can now use all of the available dplyr verbs against the tables within the cluster.

We'll start by copying some datasets from R into the Spark cluster (note that you may need to install the nycflights13 and Lahman packages in order to execute this code):


```r
install.packages(c("nycflights13", "Lahman"))
```


```r
library(dplyr)
iris_tbl <- copy_to(sc, iris)
```

```
## Error in copy_to(sc, iris): object 'sc' not found
```

```r
flights_tbl <- copy_to(sc, nycflights13::flights, "flights")
```

```
## Error in copy_to(sc, nycflights13::flights, "flights"): object 'sc' not found
```

```r
batting_tbl <- copy_to(sc, Lahman::Batting, "batting")
```

```
## Error in copy_to(sc, Lahman::Batting, "batting"): object 'sc' not found
```

```r
src_tbls(sc)
```

```
## Error in src_tbls(sc): object 'sc' not found
```

To start with here's a simple filtering example:


```r
# filter by departure delay and print the first few records
flights_tbl %>% filter(dep_delay == 2)
```

```
## Error in eval(lhs, parent, parent): object 'flights_tbl' not found
```

[Introduction to dplyr](https://CRAN.R-project.org/package=dplyr) provides additional dplyr examples you can try. For example, consider the last example from the tutorial which plots data on flight delays:


```r
delay <- flights_tbl %>%
  group_by(tailnum) %>%
  summarise(count = n(), dist = mean(distance), delay = mean(arr_delay)) %>%
  filter(count > 20, dist < 2000, !is.na(delay)) %>%
  collect
```

```
## Error in eval(lhs, parent, parent): object 'flights_tbl' not found
```

```r
# plot delays
library(ggplot2)
```

```
## Error in library(ggplot2): there is no package called 'ggplot2'
```

```r
ggplot(delay, aes(dist, delay)) +
  geom_point(aes(size = count), alpha = 1/2) +
  geom_smooth() +
  scale_size_area(max_size = 2)
```

```
## Error in ggplot(delay, aes(dist, delay)): could not find function "ggplot"
```


### Window Functions

dplyr [window functions](https://CRAN.R-project.org/package=dplyr) are also supported, for example:


```r
batting_tbl %>%
  select(playerID, yearID, teamID, G, AB:H) %>%
  arrange(playerID, yearID, teamID) %>%
  group_by(playerID) %>%
  filter(min_rank(desc(H)) <= 2 & H > 0)
```

```
## Error in eval(lhs, parent, parent): object 'batting_tbl' not found
```

For additional documentation on using dplyr with Spark see the [dplyr](http://spark.rstudio.com/dplyr.html) section of the sparklyr website.

## Using SQL

It's also possible to execute SQL queries directly against tables within a Spark cluster. The `spark_connection` object implements a [DBI](https://github.com/rstats-db/DBI) interface for Spark, so you can use `dbGetQuery` to execute SQL and return the result as an R data frame:


```r
library(DBI)
iris_preview <- dbGetQuery(sc, "SELECT * FROM iris LIMIT 10")
```

```
## Error in dbGetQuery(sc, "SELECT * FROM iris LIMIT 10"): object 'sc' not found
```

```r
iris_preview
```

```
## Error in eval(expr, envir, enclos): object 'iris_preview' not found
```

## Machine Learning

You can orchestrate machine learning algorithms in a Spark cluster via the [machine learning](http://spark.apache.org/docs/latest/mllib-guide.html) functions within **sparklyr**. These functions connect to a set of high-level APIs built on top of DataFrames that help you create and tune machine learning workflows.

Here's an example where we use [ml_linear_regression](https://spark.rstudio.com/reference/ml_linear_regression/) to fit a linear regression model. We'll use the built-in `mtcars` dataset, and see if we can predict a car's fuel consumption (`mpg`) based on its weight (`wt`), and the number of cylinders the engine contains (`cyl`). We'll assume in each case that the relationship between `mpg` and each of our features is linear.


```r
# copy mtcars into spark
mtcars_tbl <- copy_to(sc, mtcars)
```

```
## Error in copy_to(sc, mtcars): object 'sc' not found
```

```r
# transform our data set, and then partition into 'training', 'test'
partitions <- mtcars_tbl %>%
  filter(hp >= 100) %>%
  mutate(cyl8 = cyl == 8) %>%
  sdf_partition(training = 0.5, test = 0.5, seed = 1099)
```

```
## Error in eval(lhs, parent, parent): object 'mtcars_tbl' not found
```

```r
# fit a linear model to the training dataset
fit <- partitions$training %>%
  ml_linear_regression(response = "mpg", features = c("wt", "cyl"))
```

```
## Error in eval(lhs, parent, parent): object 'partitions' not found
```

```r
fit
```

```
## Error in eval(expr, envir, enclos): object 'fit' not found
```

For linear regression models produced by Spark, we can use `summary()` to learn a bit more about the quality of our fit, and the statistical significance of each of our predictors.


```r
summary(fit)
```

```
## Error in summary(fit): object 'fit' not found
```

Spark machine learning supports a wide array of algorithms and feature transformations and as illustrated above it's easy to chain these functions together with dplyr pipelines. To learn more see the [machine learning](https://spark.rstudio.com/mlib/) section.

## Reading and Writing Data

You can read and write data in CSV, JSON, and Parquet formats. Data can be stored in HDFS, S3, or on the local filesystem of cluster nodes.


```r
temp_csv <- tempfile(fileext = ".csv")
temp_parquet <- tempfile(fileext = ".parquet")
temp_json <- tempfile(fileext = ".json")

spark_write_csv(iris_tbl, temp_csv)
```

```
## Error in spark_write_csv(iris_tbl, temp_csv): could not find function "spark_write_csv"
```

```r
iris_csv_tbl <- spark_read_csv(sc, "iris_csv", temp_csv)
```

```
## Error in spark_read_csv(sc, "iris_csv", temp_csv): could not find function "spark_read_csv"
```

```r
spark_write_parquet(iris_tbl, temp_parquet)
```

```
## Error in spark_write_parquet(iris_tbl, temp_parquet): could not find function "spark_write_parquet"
```

```r
iris_parquet_tbl <- spark_read_parquet(sc, "iris_parquet", temp_parquet)
```

```
## Error in spark_read_parquet(sc, "iris_parquet", temp_parquet): could not find function "spark_read_parquet"
```

```r
spark_write_json(iris_tbl, temp_json)
```

```
## Error in spark_write_json(iris_tbl, temp_json): could not find function "spark_write_json"
```

```r
iris_json_tbl <- spark_read_json(sc, "iris_json", temp_json)
```

```
## Error in spark_read_json(sc, "iris_json", temp_json): could not find function "spark_read_json"
```

```r
src_tbls(sc)
```

```
## Error in src_tbls(sc): object 'sc' not found
```


## Distributed R

You can execute arbitrary r code across your cluster using `spark_apply`. For example, we can apply `rgamma` over `iris` as follows:


```r
spark_apply(iris_tbl, function(data) {
  data[1:4] + rgamma(1,2)
})
```

```
## Error in spark_apply(iris_tbl, function(data) {: could not find function "spark_apply"
```

You can also group by columns to perform an operation over each group of rows and make use of any package within the closure:


```r
spark_apply(
  iris_tbl,
  function(e) broom::tidy(lm(Petal_Width ~ Petal_Length, e)),
  columns = c("term", "estimate", "std.error", "statistic", "p.value"),
  group_by = "Species"
)
```

```
## Error in spark_apply(iris_tbl, function(e) broom::tidy(lm(Petal_Width ~ : could not find function "spark_apply"
```

## Extensions

The facilities used internally by sparklyr for its dplyr and machine learning interfaces are available to extension packages. Since Spark is a general purpose cluster computing system there are many potential applications for extensions (e.g. interfaces to custom machine learning pipelines, interfaces to 3rd party Spark packages, etc.).

Here's a simple example that wraps a Spark text file line counting function with an R function:


```r
# write a CSV
tempfile <- tempfile(fileext = ".csv")
write.csv(nycflights13::flights, tempfile, row.names = FALSE, na = "")
```

```
## Error in loadNamespace(name): there is no package called 'nycflights13'
```

```r
# define an R interface to Spark line counting
count_lines <- function(sc, path) {
  spark_context(sc) %>%
    invoke("textFile", path, 1L) %>%
      invoke("count")
}

# call spark to count the lines of the CSV
count_lines(sc, tempfile)
```

```
## Error in spark_context(sc): could not find function "spark_context"
```


To learn more about creating extensions see the [Extensions](http://spark.rstudio.com/extensions.html) section of the sparklyr website.


## Table Utilities

You can cache a table into memory with:


```r
tbl_cache(sc, "batting")
```

and unload from memory using:


```r
tbl_uncache(sc, "batting")
```


## Connection Utilities

You can view the Spark web console using the `spark_web` function:


```r
spark_web(sc)
```

You can show the log using the `spark_log` function:


```r
spark_log(sc, n = 10)
```

```
## Error in spark_log(sc, n = 10): could not find function "spark_log"
```

Finally, we disconnect from Spark:


```r
  spark_disconnect(sc)
```

```
## Error in spark_disconnect(sc): could not find function "spark_disconnect"
```

## RStudio IDE

The latest RStudio [Preview Release](https://www.rstudio.com/products/rstudio/download/preview/) of the RStudio IDE includes integrated support for Spark and the sparklyr package, including tools for:

- Creating and managing Spark connections
- Browsing the tables and columns of Spark DataFrames
- Previewing the first 1,000 rows of Spark DataFrames

Once you've installed the sparklyr package, you should find a new **Spark** pane within the IDE. This pane includes a **New Connection** dialog which can be used to make connections to local or remote Spark instances:

<img src="tools/readme/spark-connect.png" class="screenshot" width=389 />

Once you've connected to Spark you'll be able to browse the tables contained within the Spark cluster and preview Spark DataFrames using the standard RStudio data viewer:

<img src="tools/readme/spark-dataview.png" class="screenshot" width=639 />

You can also connect to Spark through [Livy](http://livy.io) through a new connection dialog:

<img src="tools/readme/spark-connect-livy.png" class="screenshot" width=389 />

<div style="margin-bottom: 15px;"></div>

The RStudio IDE features for sparklyr are available now as part of the [RStudio Preview Release](https://www.rstudio.com/products/rstudio/download/preview/).

## Using H2O

[rsparkling](https://cran.r-project.org/package=rsparkling) is a CRAN package from [H2O](http://h2o.ai) that extends [sparklyr](http://spark.rstudio.com) to provide an interface into [Sparkling Water](https://github.com/h2oai/sparkling-water). For instance, the following example installs, configures and runs [h2o.glm](http://docs.h2o.ai/h2o/latest-stable/h2o-docs/data-science/glm.html):


```r
library(rsparkling)
library(sparklyr)
library(dplyr)
library(h2o)

sc <- spark_connect(master = "local", version = "2.3.2")
mtcars_tbl <- copy_to(sc, mtcars, "mtcars")

mtcars_h2o <- as_h2o_frame(sc, mtcars_tbl, strict_version_check = FALSE)

mtcars_glm <- h2o.glm(x = c("wt", "cyl"),
                      y = "mpg",
                      training_frame = mtcars_h2o,
                      lambda_search = TRUE)
```


```r
mtcars_glm
```
```
## Model Details:
## ==============
##
## H2ORegressionModel: glm
## Model ID:  GLM_model_R_1527265202599_1
## GLM Model: summary
##     family     link                              regularization
## 1 gaussian identity Elastic Net (alpha = 0.5, lambda = 0.1013 )
##                                                                lambda_search
## 1 nlambda = 100, lambda.max = 10.132, lambda.min = 0.1013, lambda.1se = -1.0
##   number_of_predictors_total number_of_active_predictors
## 1                          2                           2
##   number_of_iterations                                training_frame
## 1                  100 frame_rdd_31_ad5c4e88ec97eb8ccedae9475ad34e02
##
## Coefficients: glm coefficients
##       names coefficients standardized_coefficients
## 1 Intercept    38.941654                 20.090625
## 2       cyl    -1.468783                 -2.623132
## 3        wt    -3.034558                 -2.969186
##
## H2ORegressionMetrics: glm
## ** Reported on training data. **
##
## MSE:  6.017684
## RMSE:  2.453097
## MAE:  1.940985
## RMSLE:  0.1114801
## Mean Residual Deviance :  6.017684
## R^2 :  0.8289895
## Null Deviance :1126.047
## Null D.o.F. :31
## Residual Deviance :192.5659
## Residual D.o.F. :29
## AIC :156.2425
```


```r
spark_disconnect(sc)
```

## Connecting through Livy

[Livy](https://github.com/cloudera/livy) enables remote connections to Apache Spark clusters. However, please notice that connecting to Spark clusters through Livy is much slower than any other connection method.

Before connecting to Livy, you will need the connection information to an existing service running Livy. Otherwise, to test `livy` in your local environment, you can install it and run it locally as follows:


```r
livy_install()
```


```r
livy_service_start()
```

```
## Error in livy_service_start(): could not find function "livy_service_start"
```

To connect, use the Livy service address as `master` and `method = "livy"` in `spark_connect`. Once connection completes, use `sparklyr` as usual, for instance:


```r
sc <- spark_connect(master = "http://localhost:8998", method = "livy", version = "2.4.0")
```

```
## Error in spark_connect(master = "http://localhost:8998", method = "livy", : could not find function "spark_connect"
```

```r
copy_to(sc, iris)
```

```
## Error in copy_to(sc, iris): object 'sc' not found
```

```r
spark_disconnect(sc)
```

```
## Error in spark_disconnect(sc): could not find function "spark_disconnect"
```

Once you are done using `livy` locally, you should stop this service with:


```r
livy_service_stop()
```

```
## Error in livy_service_stop(): could not find function "livy_service_stop"
```

To connect to remote `livy` clusters that support basic authentication connect as:


```r
config <- livy_config(username="<username>", password="<password>")
sc <- spark_connect(master = "<address>", method = "livy", config = config)
spark_disconnect(sc)
```

## Connecting through Databricks Connect
[Databricks Connect](https://docs.databricks.com/dev-tools/databricks-connect.html#databricks-connect) allows you to connect
sparklyr to a remote Databricks Cluster. You can install [Databricks Connect Python pakcage](https://pypi.org/project/databricks-connect/) and use it to submit Spark jobs written in sparklyr APIs and have them execute remotely on a Databricks cluster instead of in the local Spark session.

To use sparklyr with Databricks Connect first launch a Cluster on Databricks. Then follow [these instructions](https://docs.databricks.com/dev-tools/databricks-connect.html#client-setup) to setup the client:

1. Make sure pyspark is not installed
2. Install the latest version of Databricks Connect python package.
3. Run `databricks-connect configure` and provide the configuration information
    * Databricks account URL of the form `ttps://<account>.cloud.databricks.com`.
    * [User token](https://docs.databricks.com/dev-tools/api/latest/authentication.html#token-management)
    * Cluster ID
    * Port (default port number is `15001`)

To configure `sparklyr` with Databricks Connect, set the following environment variables:

```bash
$export SPARK_VERSION=2.4.4
$export SPARK_HOME=`databricks-connect get-spark-home`
```

```
## Error in running command bash
```

Now simply create a spark connection as follows

```r
sc <- spark_connect(method = "databricks")
copy_to(sc, iris)
```


```r
spark_disconnect(sc)
```
