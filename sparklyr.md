# Sparklyr
[Sparklyr](http://spark.rstudio.com/) is an easy way to work with insilica datasets in an R based spark environment. Insilica datasource documentation is at [Lets create a dataset explorer at our home page TODO].

Setting up sparklyr is fast:

```R
install.packages("sparklyr")
sc <- spark_connect(master = "local", spark_home="spark.conf"))
```
We now have a spark context that we can use to perform spark tasks.  These examples rely on a simple single node hadoop cluster and local spark context.  The config file referenced above is:

```
# use 8 threads on a standalone spark context
spark.master                     localhost[8]
spark.driver.memory              5g
#use a single node hadoop cluster (requires you to set up hadoop)
spark.hadoop.fs.defaultFS        hdfs://localhost:9000/
```
<center>spark.conf</center>

