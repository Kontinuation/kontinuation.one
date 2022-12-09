---
title: "Improving Geometry Serde Performance of Apache Sedona"
date: 2022-12-06T22:27:13+08:00
draft: false
toc: false
images:
tags:
  - geospatial
---

[Apache Sedona](https://sedona.apache.org/) is an amazing software. It brings
distributed geospatial processing capabilities to popular cluster computing
systems such as Apache Spark and Apache Flink. In this post we'll investigate
on the performance characteristics of Spatial SQL on Apache Spark, and improve
the performance of Spatial SQL by tinkering with the geometry object
serializer/deserializer in Apache Sedona.

## Why Do We Care About the Performance of Geometry Serdes?

Spark SQL do not have native support for geometry columns and geometry
values. In order to bring geospatial support to Spark SQL, Apache Sedona
defined a UDT
([`GeometryUDT`](https://github.com/apache/incubator-sedona/blob/sedona-1.3.0/sql/src/main/scala/org/apache/spark/sql/sedona_sql/UDT/GeometryUDT.scala))
for representing geospatial values in Spark SQL. UDT values are composed by
primitive Spark SQL values, Apache Sedona defines `GeometryUDT` as an array of
bytes, which contains the WKB representation of the geometry value.

WKB serialization/deserialization happens all the time when you run Spatial SQL
queries. Each time a spatial function was evaluated, the arguments of the
spatial function need to be deserialized. If the return value of that spatial
function is a geometry object, then that geometry object needs to be serialized
in order to be represented as a UDT value. Let's take
[`ST_Transform`](https://github.com/apache/incubator-sedona/blob/sedona-1.3.0/sql/src/main/scala/org/apache/spark/sql/sedona_sql/expressions/Functions.scala#L177-L186)
as an example, you can easily spot the serialization and deserialization code
in the `eval` method.

```scala
override def eval(input: InternalRow): Any = {
  val geometry = inputExpressions(0).toGeometry(input)
  val sourceCRSString = inputExpressions(1).asString(input)
  val targetCRSString = inputExpressions(2).asString(input)
  val lenient = inputExpressions(3).eval(input).asInstanceOf[Boolean]
  (geometry,sourceCRSString,targetCRSString,lenient) match {
    case (null,_,_,_)  => null
    case _ => Functions.transform(geometry, sourceCRSString, targetCRSString, lenient).toGenericArrayData
  }
}
```

Now we know that serialization and deserialization of geometry objects are the
common steps of evaluating spatial functions. If geometry serde takes fair
amount of time when running Spatial SQL, it deserves careful optimization to
reduce the performance overhead. Actually geometry serde is a big performance
killer when running some commonly seen queries. Here is a Spatial SQL for
selecting geometries within a given rectangular region:

```sql
SELECT COUNT(1) FROM ms_buildings
WHERE ST_Within(geom, ST_GeomFromText('POLYGON((-71.07418550113192 42.37012196853071,-71.0453463898038 42.37012196853071,-71.0453463898038 42.35096853399424,-71.07418550113192 42.35096853399424,-71.07418550113192 42.37012196853071))'))
```

It is a simple range query. Let's profile one of the executor nodes when
running this query using
[async-profiler](https://github.com/jvm-profiling-tools/async-profiler) and
obtain a flamegraph. We can easily spot the performance bottleneck of this
query: the purple highlighted horizontal bar in the following flamegraph
indicates the cost of geometry object deserialization, which takes 64% of total
running time. Spark SQL is not throwing most of our computation power to the
geometry serde instead of doing useful work.

![flamegraph_unopt_range_query](/sedona-serde-performance/flamegraph_unoptimized_range_query.png)

Now we know that geometry serde is the bottleneck of some particular kind of
Spatial SQL query, it is time to optimize it.

## You Have To Measure It In Order To Improve It.

Geometry serde is a small piece of CPU-intensive code, which seems to be fairly
easy to benchmark. However, benchmarking things running on JVM is not an easy
thing - you have to combat with the complicated, profile guided JIT compiler to
obtain correct benchmark results. Fortunately, there's a project named
[JMH](https://github.com/openjdk/jmh) for helping us getting rid of lots of
traps and doing benchmarks correctly. We can setup our [benchmark
project](https://github.com/Kontinuation/play-with-geometry-serde) based on JMH
and start measuring.

There are 2 geometry serdes in Apache Sedona. We'll benchmark both of them and
pick up the better one as our starting point of rolling a faster serde. These 2
geometry serdes are:

1. [`GeometrySerializer`](https://github.com/apache/incubator-sedona/blob/sedona-1.3.0/sql/src/main/scala/org/apache/sedona/sql/utils/GeometrySerializer.scala):
   This is the geometry serde used by `GeometryUDT` and Spatial SQL. It is using
   WKB as serialization format.
2. [`GeometrySerde`](https://github.com/apache/incubator-sedona/blob/sedona-1.3.0/core/src/main/java/org/apache/sedona/core/geometryObjects/GeometrySerde.java): This is a serde based on Kryo for efficient serialization and deserialization of geometry objects in RDDs.

`GeometrySerde` does not only serialize geometry itself, it also serializes
user data attached to the geometry objects. So it is not completely fair to
compare `GeometrySerde` with `GeometrySerializer`. `GeometrySerde` uses
[`ShapeSerde`](https://github.com/apache/incubator-sedona/blob/sedona-1.3.0/core/src/main/java/org/apache/sedona/core/formatMapper/shapefileParser/parseUtils/shp/ShapeSerde.java)
to serialize the pure geometry part. This is also the geometry serde for
reading Shapefiles.

To sum up, we have 3 serdes to benchmark: a WKB based `GeometrySerializer`, a
Kryo serde `GeometrySerde` and a shapefile compliant `ShapeSerde`. In order to
make things clear, we'll refer to them as `WkbSerde`, `KryoSerde` and
`ShapeSerde` in the following texts.

### Define the workload

We'll benchmark the performance of serializing/deserializing 2D geometries with
X/Y coordinates since this is the most common use case. We'll run these serdes
on various types of geometry objects (Point, LineString, Polygon,
MultiPolygon). For geometries with multiple coordinates, we run the tests on
geometry objects of various sizes. The `segment` parameter indicates the number
of nodes in the individual parts of the
geometry. [BenchmarkedGeometries.java](https://github.com/Kontinuation/play-with-geometry-serde/blob/main/src/jmh/java/com/spatialx/BenchmarkedGeometries.java)
is the code for preparing benchmarked geometries.

### Run the Benchmark

Let's run the benchmark with `-f 3 -w 5 -r 5`, which is a good balance between
running time and accuracy of measurements. This is the result obtained by
running the benchmark on an ECS instanace with 4 Intel(R) Xeon(R) Platinum
8269CY CPUs. Java version was OpenJDK 1.8.0_352. We've only included the result
of `segment = 16` for saving screen space.

```
Benchmark                                         (segments)   Mode  Cnt         Score         Error  Units
BenchmarkWkbSerde.deserializeGeometryCollection           16  thrpt   15    148420.082 ±    8395.253  ops/s
BenchmarkKryoSerde.deserializeGeometryCollection          16  thrpt   15    179118.144 ±    2361.610  ops/s
BenchmarkWkbSerde.deserializeLineString                   16  thrpt   15   1294007.710 ±   46441.569  ops/s
BenchmarkKryoSerde.deserializeLineString                  16  thrpt   15   1419209.227 ±   43513.643  ops/s
BenchmarkShapeSerde.deserializeLineString                 16  thrpt   15   2113479.014 ± 1003938.612  ops/s
BenchmarkWkbSerde.deserializeMultiPolygon                 16  thrpt   15    340951.943 ±   12441.200  ops/s
BenchmarkKryoSerde.deserializeMultiPolygon                16  thrpt   15    468347.426 ±   13647.060  ops/s
BenchmarkShapeSerde.deserializeMultiPolygon               16  thrpt   15    928877.615 ±   23477.555  ops/s
BenchmarkWkbSerde.deserializePoint                        16  thrpt   15   7693355.984 ±  193782.092  ops/s
BenchmarkKryoSerde.deserializePoint                       16  thrpt   15   8174952.289 ±  318876.854  ops/s
BenchmarkShapeSerde.deserializePoint                      16  thrpt   15  12494806.724 ±  634448.854  ops/s
BenchmarkWkbSerde.deserializePolygon                      16  thrpt   15   1197937.023 ±   33104.487  ops/s
BenchmarkKryoSerde.deserializePolygon                     16  thrpt   15   1234112.233 ±   74607.021  ops/s
BenchmarkShapeSerde.deserializePolygon                    16  thrpt   15   2668153.092 ±   49593.190  ops/s
BenchmarkWkbSerde.deserializePolygonWithHoles             16  thrpt   15    392725.518 ±   25334.622  ops/s
BenchmarkKryoSerde.deserializePolygonWithHoles            16  thrpt   15    467200.201 ±   15385.118  ops/s
BenchmarkShapeSerde.deserializePolygonWithHoles           16  thrpt   15    939063.600 ±   27115.595  ops/s
BenchmarkWkbSerde.serializeGeometryCollection             16  thrpt   15    134069.692 ±    9140.104  ops/s
BenchmarkKryoSerde.serializeGeometryCollection            16  thrpt   15    244533.759 ±    5986.474  ops/s
BenchmarkWkbSerde.serializeLineString                     16  thrpt   15   1226963.139 ±   48017.455  ops/s
BenchmarkKryoSerde.serializeLineString                    16  thrpt   15   2068722.298 ±  118988.199  ops/s
BenchmarkShapeSerde.serializeLineString                   16  thrpt   15   2460851.815 ±   41971.344  ops/s
BenchmarkWkbSerde.serializeMultiPolygon                   16  thrpt   15    411487.421 ±   19040.631  ops/s
BenchmarkKryoSerde.serializeMultiPolygon                  16  thrpt   15    693294.294 ±   12503.655  ops/s
BenchmarkShapeSerde.serializeMultiPolygon                 16  thrpt   15    782453.061 ±   33955.116  ops/s
BenchmarkWkbSerde.serializePoint                          16  thrpt   15   9222249.626 ±  306121.802  ops/s
BenchmarkKryoSerde.serializePoint                         16  thrpt   15  10050255.878 ± 4309908.204  ops/s
BenchmarkShapeSerde.serializePoint                        16  thrpt   15  29068624.264 ± 1483390.980  ops/s
BenchmarkWkbSerde.serializePolygon                        16  thrpt   15   1164400.303 ±   42605.393  ops/s
BenchmarkKryoSerde.serializePolygon                       16  thrpt   15   2035363.615 ±   71694.688  ops/s
BenchmarkShapeSerde.serializePolygon                      16  thrpt   15   1896086.966 ±  516254.434  ops/s
BenchmarkWkbSerde.serializePolygonWithHoles               16  thrpt   15    442221.837 ±   10232.831  ops/s
BenchmarkKryoSerde.serializePolygonWithHoles              16  thrpt   15    660124.301 ±   19330.398  ops/s
BenchmarkShapeSerde.serializePolygonWithHoles             16  thrpt   15    765501.356 ±   89840.549  ops/s
```

## Reading these Numbers

We can conclude that the performance of `KryoSerde` is slightly better than
`WkbSerde` in this particular environment, but significantly poorer than
`ShapeSerde`. It seems that `ShapeSerde` is good starting point for further
optimizing serde performance. However, we need to interpret these numbers with
caution to fully understand the benchmark results.

To understand the make up of computational costs when serializing and
deserializing geometry objects, we can profile it using a profiler. JMH has
builtin integration with async-profiler, so that we can instruct JMH to profile
it when running the benchmark. Let's run the benchmark of serializing
MultiPolygons with `segment = 16` as an example:

```bash
java -jar ./build/libs/play-with-geometry-serde-1.0-SNAPSHOT-jmh.jar "BenchmarkWkbSerde.serializeMultiPolygon" -f 1 -p "segments=16" -r 5s -w 5s -prof async:libPath=/path/to/libasyncProfiler.so\;output=flamegraph
```

First let's find out why WKB is slow. Most of its time was spent on
[`WKBWriter.writeCoordinate`](https://github.com/locationtech/jts/blob/1.19.0/modules/core/src/main/java/org/locationtech/jts/io/WKBWriter.java#L434-L452). Its
implementation involves reordering eight bytes for floating point numbers
manually in
[`ByteOrderValues.putLong`](https://github.com/locationtech/jts/blob/1.19.0/modules/core/src/main/java/org/locationtech/jts/io/ByteOrderValues.java#L82-L104)
and reallocations of the byte array in `ByteArrayOutputStream.write`. We may
wonder if the JIT compiler is smart enough to skip reordering when the byte
order of `WKBWriter` is native byte order, unfortunatally it is not the
case. We've also benchmarked `WkbSerdeBigEndian` and the result is about the
same with `WkbSerde`.

![flamegraph_wkb_serde](/sedona-serde-performance/flamegraph_wkb_serde.png)

The next thing to see is the profiling result of `KryoSerde`. We can
immediately spot one big anomaly in the profiling result: why do we need to
create JTS points when serializing something? We should not create any geometry
objects in the serialization process.

![flamegraph_kryo_serde](/sedona-serde-performance/flamegraph_kryo_serde.png)

The `GeometryFactory.createPoint` call took lots of time when serializing
geometry objects. We expect a huge performance improvement after pulling out
this sting. It is in
[`ShapeSerde.putPoints`](https://github.com/apache/incubator-sedona/blob/sedona-1.3.0-incubating/core/src/main/java/org/apache/sedona/core/formatMapper/shapefileParser/parseUtils/shp/ShapeSerde.java#L254). We
can replace invocation of `getPointN` with `getCoordinateN` to avoid the
creation of point objects.

When it comes to deserialization, `WkbSerde` is slow in the same way with
serialization. `KryoSerde` is much slower than `ShapeSerde`, while it is mostly
based on `ShapeSerde`. We found that the extra performance cost introduced to
`KryoSerde` was in the
[`ShapeReader`](https://github.com/apache/incubator-sedona/blob/sedona-1.3.0-incubating/core/src/main/java/org/apache/sedona/core/formatMapper/shapefileParser/parseUtils/shp/ShapeReaderFactory.java#L71-L75). Each
time the deserializer reads a floating point number, it fetches 8 bytes from
the Kryo input object, create a ByteBuffer object, then get a floating point
number out of it. There are too many small reads to the Kryo input stream and
we've wasted too much time creating ByteBuffer objects when the geometry to be
deserialized contains lots of coordinates.

![flamegraph_kryo_deserialize_bytebuffer](/sedona-serde-performance/flamegraph_kryo_deserialize_bytebuffer.png)

```java
@Override
public double readDouble() {
  return toByteBuffer(input, 8).getDouble();
}

private static ByteBuffer toByteBuffer(Input input, int numBytes) {
  byte[] bytes = new byte[numBytes];
  input.read(bytes);
  return ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN);
}
```

A better way of reading coordinates from Kryo input is to read multiple
coordinates at once. We can read a large chunk of byte array from Kryo input
each time we want to read a CoordinateSequence of multiple coordinates.

```java
@Override
public void readCoordinates(CoordinateSequence coordinateSequence, int numCoordinates) {
  ByteBuffer bb = toByteBuffer(input, numCoordinates * 16);
  for (int k = 0; k < numCoordinates; k++) {
    coordinateSequence.setOrdinate(k, 0, bb.getDouble(16 * k));
    coordinateSequence.setOrdinate(k, 1, bb.getDouble(16 * k + 8));
  }
}
```

## Improving the Performance of `KryoSerde` and `ShapeSerde`

We've fixed the performance problems mentioned above in our own fork
[here](https://github.com/Kontinuation/play-with-geometry-serde/tree/v1.0.0/src/main/java/org/apache/sedona/core/optimized/formatMapper/shapefileParser/parseUtils/shp). Benchmark
shows that our fix improved serde performance by roughly 30%, and the
performance gap between `KryoSerde` and `ShapeSerde` has also became smaller.

```
Benchmark                                                   (segments)   Mode  Cnt         Score         Error  Units
BenchmarkKryoSerde.deserializeMultiPolygon                          16  thrpt   15    445317.627 ±    8713.265  ops/s
BenchmarkKryoSerdeOptimized.deserializeMultiPolygon                 16  thrpt   15    637855.796 ±   22901.547  ops/s
BenchmarkShapeSerdeOptimized.deserializeMultiPolygon                16  thrpt   15    583509.084 ±  240406.288  ops/s
BenchmarkKryoSerde.serializeMultiPolygon                            16  thrpt   15    678090.693 ±   12838.073  ops/s
BenchmarkKryoSerdeOptimized.serializeMultiPolygon                   16  thrpt   15    827620.426 ±  180365.455  ops/s
BenchmarkShapeSerdeOptimized.serializeMultiPolygon                  16  thrpt   15   1296729.904 ±   16064.890  ops/s
```

## How Can We Do Better?

Til now, the obvious performance problems in `ShapeSerde` have already been
resolved. We need to dig deeper to get better performance. We can hardly find
anything to improve by looking at `ShapeSerde`'s code, it is already a very
compact, hassle free implementation of geometry serde using
`java.nio.ByteBuffer`. The profiling result shows that the next thing to
improve might be `ByteBuffer.getDouble` (though profiling result was not that
accurate when everything was heavily inlined by the C2 optimizer), what can we
do to improve the performance of reading 64-bit floating point number from a
byte buffer?

![flamegraph_byte_buffer_slow_get_double](/sedona-serde-performance/flamegraph_byte_buffer_slow_get_double.png)

