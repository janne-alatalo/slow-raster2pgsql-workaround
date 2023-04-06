# Workaround for slow `raster2pgsql` upload speed when using out-db rasters and GeoTIFF files


TLDR: `export GTIFF_DIRECT_IO=YES` Only works with uncompressed GeoTIFFs,
however you can upload the GeoTIFF to PostGIS when it is uncompressed and
re-compress the file later.


The `raster2pgsql` command is extremely slow with default options when it is
used to upload `NODATA` enabled GeoTIFF files to PostGIS when using out-db
rasters. The slowness is caused by the `NODATA` value check that is enabled by
default. The `NODATA` check avoids adding tiles to the PostGIS where all pixel
values are `NODATA`, therefore it is very useful to keep on, however the
simplest fix is just to turn it off with the `-k` argument. Obviously, this
results to many useless tiles in the PostGIS table that are filled with only
`NODATA` values.


A better workaround is to use the GDAL `GTIFF_DIRECT_IO` or
`GTIFF_VIRTUAL_MEM_IO` options. The options are documented
[here](https://gdal.org/drivers/raster/gtiff.html), however there is no much
information about them, except that they only work with uncompressed TIFF
files.


This repository includes a test GeoTIFF file that can be used to measure the
performance improvement when setting the environment variable. The results are
computed on a Linux system. The version information for the `raster2pgsql` is
`RELEASE: 3.3.2 GDAL_VERSION=36 (POSTGIS_REVISION)` (GDAL version `3.6.3`).


To run the following tests, first uncompress the compressed tiff file from this
repository:

```
gdal_translate compressed-cropped-with-nodata-small.tif cropped-with-nodata-small.tif -co COMPRESS=NONE -co BIGTIFF=YES
```

```
# Baseline
perf stat -r 10 bash -c 'raster2pgsql -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1'
```

```
Performance counter stats for 'bash -c raster2pgsql -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1' (10 runs):

         12,086.80 msec task-clock:u                     #    9.950 CPUs utilized               ( +-  9.62% )
                 0      context-switches:u               #    0.000 /sec                      
                 0      cpu-migrations:u                 #    0.000 /sec                      
         1,076,170      page-faults:u                    #  158.469 K/sec                       ( +-  9.57% )
    12,245,285,654      cycles:u                         #    1.803 GHz                         ( +-  9.63% )
     9,220,877,548      instructions:u                   #    1.32  insn per cycle              ( +-  9.57% )
     2,125,160,960      branches:u                       #  312.936 M/sec                       ( +-  9.57% )
       106,203,865      branch-misses:u                  #    9.09% of all branches             ( +-  9.68% )

            1.2147 +- 0.0255 seconds time elapsed  ( +-  2.10% )
```


```
# When setting the env variable
perf stat -r 10 bash -c 'GTIFF_DIRECT_IO=YES raster2pgsql -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1'
```

```
Performance counter stats for 'bash -c GTIFF_DIRECT_IO=YES raster2pgsql -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1' (10 runs):

          3,630.24 msec task-clock:u                     #    9.724 CPUs utilized               ( +-  9.58% )
                 0      context-switches:u               #    0.000 /sec                      
                 0      cpu-migrations:u                 #    0.000 /sec                      
            35,063      page-faults:u                    #   17.860 K/sec                       ( +-  9.58% )
     5,746,675,315      cycles:u                         #    2.927 GHz                         ( +-  9.65% )
     5,766,619,492      instructions:u                   #    1.86  insn per cycle              ( +-  9.57% )
     1,348,166,920      branches:u                       #  686.724 M/sec                       ( +-  9.57% )
        58,917,479      branch-misses:u                  #    7.95% of all branches             ( +-  9.71% )

            0.3733 +- 0.0132 seconds time elapsed  ( +-  3.54% )
```

From the "time elapsed" we calculate `1.2147 / 0.3733` -> `> 300%` speed
increase. With larger file size the speed increase is even higher. For a `265M`
GeoTIFF file the benchmark takes `26.480 +- 0.278 seconds` for baseline, and
`2.709 +- 0.129 seconds` when `GTIFF_DIRECT_IO=YES` is set.


The `export GTIFF_VIRTUAL_MEM_IO=YES` has almost as good performance increase,
however in my benchmark `GTIFF_DIRECT_IO` was slightly better.

### Workaround for compressed GeoTIFFs

PostGIS does not care that the GeoTIFF file compression is changed after the
file metadata is already uploaded to the database. Therefore, the workaround is:

1. Rename the original file e.g. `mv compressed-cropped-with-nodata-small.tif compressed-cropped-with-nodata-small.tif.orig`
2. Uncompress the compressed file `gdal_translate compressed-cropped-with-nodata-small.tif.orig compressed-cropped-with-nodata-small.tif -co COMPRESS=NONE -co BIGTIFF=YES`
3. Upload to PostGIS using the direct io option `GTIFF_DIRECT_IO=YES raster2pgsql -a -t 100x100 -R -Y compressed-cropped-with-nodata-small.tif radarimages | psql -q`
4. Replace the uncompressed file with the compressed file `mv compressed-cropped-with-nodata-small.tif.orig compressed-cropped-with-nodata-small.tif`


### Sanity checks

To confirm that the result is exactly the same.

```
diff -s \
  <(bash -c 'raster2pgsql -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages') \
  <(bash -c 'GTIFF_DIRECT_IO=YES raster2pgsql -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages')

Files /proc/self/fd/11 and /proc/self/fd/12 are identical
```

To confirm that the `NODATA` check actually does something we can disable it
with `-k` for the baseline and check that the results differ.

```
diff -s \
  <(bash -c 'raster2pgsql -k -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages') \
  <(bash -c 'GTIFF_DIRECT_IO=YES raster2pgsql -a -t 100x100 -R -Y cropped-with-nodata-small.tif radarimages')

Files /proc/self/fd/11 and /proc/self/fd/12 differ
```

### in-db rasters

With a quick benchmark, you get similar performance increase when setting
`GTIFF_DIRECT_IO` when using `in-db` rasters too.

```
perf stat -r 10 bash -c 'raster2pgsql -a -t 100x100 -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1'                                                                [13:47:19]
                                                                                          

 Performance counter stats for 'bash -c raster2pgsql -a -t 100x100 -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1' (10 runs):

          6,237.50 msec task-clock:u                     #    9.932 CPUs utilized               ( +-  9.44% )
                 0      context-switches:u               #    0.000 /sec                      
                 0      cpu-migrations:u                 #    0.000 /sec                      
           113,180      page-faults:u                    #   32.974 K/sec                       ( +-  9.60% )
     9,466,631,683      cycles:u                         #    2.758 GHz                         ( +-  9.50% )
    10,319,739,898      instructions:u                   #    2.00  insn per cycle              ( +-  9.57% )
     1,994,234,552      branches:u                       #  581.006 M/sec                       ( +-  9.57% )
        70,480,996      branch-misses:u                  #    6.43% of all branches             ( +-  9.35% )

            0.6280 +- 0.0105 seconds time elapsed  ( +-  1.67% )
```

```
perf stat -r 10 bash -c 'GTIFF_DIRECT_IO=YES raster2pgsql -a -t 100x100 -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1'                                            [13:47:34]
                                                   

 Performance counter stats for 'bash -c GTIFF_DIRECT_IO=YES raster2pgsql -a -t 100x100 -Y cropped-with-nodata-small.tif radarimages > /dev/null 2>&1' (10 runs):

          3,141.00 msec task-clock:u                     #    9.966 CPUs utilized               ( +-  9.40% )
                 0      context-switches:u               #    0.000 /sec                      
                 0      cpu-migrations:u                 #    0.000 /sec                      
            58,267      page-faults:u                    #   33.450 K/sec                       ( +-  9.58% )
     5,249,417,091      cycles:u                         #    3.014 GHz                         ( +-  9.28% )
     7,128,043,419      instructions:u                   #    2.41  insn per cycle              ( +-  9.57% )
     1,289,803,414      branches:u                       #  740.452 M/sec                       ( +-  9.57% )
        41,614,856      branch-misses:u                  #    5.87% of all branches             ( +-  8.65% )

           0.31516 +- 0.00755 seconds time elapsed  ( +-  2.40% )
```

### Flamegraph

Before finding the workaround with the `GTIFF_DIRECT_IO=YES` option, I tried to
see if it would be possible to improve the performance of `raster2pgsql`. This
repository includes `flamegraph.svg` file for those that are interested. The
`rt_band_check_is_nodata` calls the `rt_band_get_pixel` function in a loop for
every pixel in the tile tanking the performance.
