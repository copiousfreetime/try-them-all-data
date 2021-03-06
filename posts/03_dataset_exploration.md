# Exploring the Unsplash Dataset

In [01] we decided to use the [Unsplash Dataset](https://unsplash.com/data) and enhance it with the data from [GeoNames](https://www.geonames.org/) and [GADM](https://gadm.org/).

So lets look at these datasets and see what we have. In all likelihood we'll have to do some mucking about with the data to get it into a shape we want.

## Fetching and loading the unsplash data

The Unsplash data is the core of our examples, so lets load it up according to their instructions and see where it gets us. We'll be working with the lite dataset for these articles since that is the one that is fully available for everyone.

All the documentation for this is in the [Unsplash datasets github repository.](https://github.com/unsplash/datasets)

1. Make a scratch directory.
    `mkdir unsplash && cd unsplash`
2. Download the dataset.
    `curl -L https://unsplash.com/data/lite/latest -# -o unsplash-research-dataset-lite-latest.zip`
3. Extract the files
    `unzip unsplash-research-dataset-lite-latest.zip`
4. Quick check on sizes
      ```sh
      % wc -l *.tsv000
       1646598 collections.tsv000
       4075505 conversions.tsv000
       2689741 keywords.tsv000
         25001 photos.tsv000
       8436845 total
      ```
5. Download the creation and loading scripts mentioned in [Loading data in PostgreSQL](https://github.com/unsplash/datasets/blob/master/how-to/psql/README.md).
    ```sh
    curl -L -O https://raw.githubusercontent.com/unsplash/datasets/master/how-to/psql/create_tables.sql
    curl -L -O https://raw.githubusercontent.com/unsplash/datasets/master/how-to/psql/load-data-client.sql
    ```
6. Create the postgresql db - this does assume you have a postgresql server up and running locally. You'll probably need to add adjust the commandline as appropriate for your situation.
    `createdb -h localhost unsplash_lite`
7. Create the tables.
    ```sh
    % psql -U jeremy  -d unsplash_lite -f create_tables.sql
    CREATE TABLE
    CREATE TABLE
    CREATE TABLE
    CREATE TABLE
    ```
8. Edit the `load-data-client.sql` file to replace the `{path}` section to the full path to this unsplash scratch directory you are in.
9. Load the data - the numbers output should 1 less than those  in the `wc -l` check from Step 4 above. That's because of header lines on all the files. This will probably take a few minutes. Given that there is a header row on all the tsv files, the rows reported by sql should all be 1 less than the rows listed above from `wc`.
    ```sh
    % time psql -h localhost -U jeremy -d unsplash_lite -f load-data-client.sql
    COPY 25000
    COPY 2689739 # <-- Hmm.. this one is NOT 1 less than keywords.tsv000 above - will have to investigate
    COPY 1646597
    COPY 4075504

    real    1m48.452s
    user    0m57.944s
    sys     0m2.320s
    ```

## Data Quality Checks

All of the following assume you are at the `psql` commandline. So connect up: `psql -U jeremy -h localhost unplash_lite`

Its always a good idea when you start looking at a dataset to do some cursory data quality checks and see if anything jumps out.

### Check Referential Integrity

According to the [dataset documentation](https://github.com/unsplash/datasets/blob/master/DOCS.md) the `photo_id` column on the `unsplash_photos.photo_id` field is the primary key of all the photos, and the `photo_id` column in the other tables should refer to the photos table. The [create_tables.sql](https://github.com/unsplash/datasets/blob/master/how-to/psql/create_tables.sql) does not create the referential constraint - or indexes on these columns. Lets add those. Doing so will confirm the documented referential integrity.

```sql
unsplash_lite=# alter table unsplash_collections add foreign key (photo_id) references unsplash_photos(photo_id);
ALTER TABLE
unsplash_lite=# alter table unsplash_conversions add foreign key (photo_id) references unsplash_photos(photo_id);
ALTER TABLE
unsplash_lite=# alter table unsplash_keywords add foreign key (photo_id) references unsplash_photos(photo_id);
ALTER TABLE
```

No errors. Excellent.

Just to save our sanity while doing some exploring - lets go ahead and add indexes on those foreign key columns. No need to add one for `unsplash_collections` as its part of the compound primary key.

```sql
unsplash_lite=# create index on unsplash_conversions(photo_id);
CREATE INDEX
unsplash_lite=# create index on unsplash_keywords(photo_id);
CREATE INDEX
```

### Cardinality check

One of the first things I do when looking at a new set of data is check out the cardinality of the columns. In other words, the list of distinct values in the column. This can show you potential errors or just undocumented assumptions in the dataset.

This is effectively doing the following type of query for column in each table. You could also use a tool like `[xsv stats --cardinality -d '\t' photos.tsv000  | xsv table](https://github.com/BurntSushi/xsv)` to do an initial report if all the data can be held in memory.

```sql
=# select photo_featured, count(*) from unsplash_photos group by 1;
 photo_featured | count
----------------+-------
 t              | 25000
(1 row) 
```

As you can see here, this is a boolean column, and all the records in the lite dataset are true. When a column has a cardinality of 1, it is always [worth confirming](https://github.com/unsplash/datasets/issues/25) that this cardinality is correct. In this case [it was](https://github.com/unsplash/datasets/issues/25#issuecomment-677794892) it was.

  > It is expected that all the photos in the Lite dataset are featured photos. It won't be the case in the Full dataset
  > -- [@TimmyCarbone](https://github.com/unsplash/datasets/issues/25#issuecomment-677794892)

The first version of the Unsplash dataset I downloaded was the initial release. When I did this check on the various `unsplash_photos.ai_primary_landmark_*` columns, it turned out all the values where `NULL`. I [asked about this](https://github.com/unsplash/datasets/issues/12). This one turned out to be a bug, was fixed, and a new release of the dataset was published. 

So always worth worth confirming. Initial clarifications can save hours, days, weeks, even months of person time to find, fix, and reprocess incorrect initial data assumptions.


### Check for leading and trailing whitespace on text fields.

If we look at the `unsplash_keywords` table - we see that there is a keywords column - I wonder what the shape of that is. Are the keywords stripped of leading and trailing spaces? Using the `LIKE` operator we check for leading and trailing spaces.

```text
unsplash_lite=# select count(*) from unsplash_keywords where keyword like ' %';
 count
-------
  1248

unsplash_lite=# select count(*) from unsplash_keywords where keyword like '% ';
 count
-------
   291
```

That could be an issue -- lets see if there's any any keywords that are just padded left or right with spaces.

```text
unsplash_lite=# select keyword, count(*)  from unsplash_keywords where keyword like '% ' group by 1 order by 2 desc limit 5;
     keyword      | count
------------------+-------
 wallpaper        |     5
 california       |     5
 cat photography  |     4
 beautiful        |     4
  silhouette      |     3

unsplash_lite=# select '>' || keyword || '<' as keyword, count(*) from unsplash_keywords where keyword IN (' wallpaper', 'wallpaper', 'wallpaper ') group by 1 order by 2 desc;
   keyword    | count
--------------+-------
 >wallpaper<  |  1951
 > wallpaper< |    12
 >wallpaper < |     5
```

<em>That || operator is the SQL string concatenation operator, we're using it here visually show us what the padding looks like</em>

Yup -- looks like there might be something to this -- [I asked Unsplash about it](https://github.com/unsplash/datasets/issues/13). There's a whole lot of text fields in the Unsplash data, and humans being humans, we never enter data consistently. Turns out this is a known factor in this dataset and [they are are open to community input](https://github.com/unsplash/datasets/issues/13#issuecomment-672635482):

  > Also, if anyone wants to give it a shot, we'd be happy to implement good solutions from the open-source community!
  >
  > -- @TimmyCarbone

### How about if the keywords are normalized on case?

```
unsplash_lite=# select count(*) from unsplash_keywords where keyword != lower(keyword);
 count
-------
     0
```

Okay - that looks good.

### That 1 line difference on keywords

When we look at the original `keywords.tsv000` file it 2,689,741 rows, which should be 1 header row and 2,689,740 data rows. When we imported, postgresql reported 2,689,739 rows. Lets double check.

```sh
% wc -l keywords.tsv000
2689741 keywords.tsv000
```

```sql
unsplash_lite=# select count(*) from unsplash_keywords ;
  count
---------
 2689739
(1 row)
```

Still a row off, maybe there's an extra newline on the tsv?
```
% tail -2 keywords.tsv000
--2IBUMom1I     people  62.514862060546903              f
--2IBUMom1I     electronics     43.613410949707003              f
```

Nope. Well, maybe there was a row eliminated on import. In the [create_tables.sql](https://github.com/unsplash/datasets/blob/master/how-to/psql/create_tables.sql) the primary key on the `unsplash_keywords` table is a compound key of `photo_id` and `keyword`.

```sql
CREATE TABLE unsplash_keywords (
  photo_id varchar(11),
  keyword text,
  ai_service_1_confidence float,
  ai_service_2_confidence float,
  suggested_by_user boolean,
  PRIMARY KEY (photo_id, keyword)
);
```

How about we reimport this data file and see how it looks. We'll create a new table, that's just like the original one but without the primary key and then import the keywords tsv into it. The `\COPY` command there is adapted from [load-data-client.sql](https://github.com/unsplash/datasets/blob/master/how-to/psql/load-data-client.sql)

```sql
unsplash_light# CREATE TABLE unsplash_keywords_raw ( photo_id varchar(11), keyword text, ai_service_1_confidence float, ai_service_2_confidence float, suggested_by_user boolean
CREATE TABLE
unsplash_light# \COPY unsplash_keywords_raw FROM PROGRAM 'awk FNR-1 ./keywords.tsv* | cat' WITH ( FORMAT csv, DELIMITER E'\t', HEADER false);
COPY 2689739
```

No joy, same as before. Well - looks like we need to write a program. What we want to do is go through the keywords file and make sure that all the records have the same number of fields, and that we have the right number for records and the right number of lines. Since my primary language is Ruby - I'll use it.

```ruby
#!/usr/bin/env ruby

line_number    = 0
unique_counts  = Hash.new(0)

File.open("keywords.tsv000") do |f|

  header          = f.readline.strip
  line_number     += 1
  header_parts    = header.split("\t")
  puts "Headers: #{header_parts.join(" -- ")}"


  f.each_line do |line|
    line_number += 1
    parts       = line.strip.split("\t")
    primary_key = parts[0..1].join("-")

    unique_counts[primary_key] += 1

    if parts.size != header_parts.size
      $stderr.puts "[#{line_number} - #{primary_key}] parts count #{parts.size} != #{header_parts.size}"
    end
  end
end

$stderr.puts "lines in file   : #{line_number}"
$stderr.puts "data lines      : #{line_number - 1}"
$stderr.puts "unique row count: #{unique_counts.size}"

unique_counts.each do |key, count|
  if count != 1
    $stderr.puts "Primary key #{key} has count #{count}"
  end
end
```

And then we run it.

```sh
Headers: photo_id -- keyword -- ai_service_1_confidence -- ai_service_2_confidence -- suggested_by_user
[1590611 - PF4s20KB678-"fujisan] parts count 2 != 5
[1590612 - mount fuji"-] parts count 4 != 5
lines in file   : 2689741
data lines      : 2689740
unique row count: 2689740
```

Looks like we have a row count that we expect from `wc -l` but we have 2 rows, that are adjacent, with the wrong parts count. There is probably an embedded `\n` in the keyword field of photo `PF4s20KB678`. Lets dump those lines of the file and see what we see.

```sh
% sed -n '1590610,1590613p' keywords.tsv000
PF4s20KB678     night   22.3271160125732                f
PF4s20KB678     "fujisan
mount fuji"                     t
PF4s20KB678     pier    22.6900939941406                f
```

Yup - definitely an embedded newline. And here we encounter the difference between *record count* and *line count*. In this case our assumption that there was 1 line in the file per record was incorrect. One of the keywords has an embedded newline. Lets go check the database.

```sql
unsplash_lite=# select * from unsplash_keywords where photo_id = 'PF4s20KB678' and keyword like '%fujisan%';
  photo_id   |  keyword   | ai_service_1_confidence | ai_service_2_confidence | suggested_by_user
-------------+------------+-------------------------+-------------------------+-------------------
 PF4s20KB678 | fujisan   +|                         |                         | t
             | mount fuji |                         |                         |
(1 row)
```

Excellent! We were wrong! That's always a really good feeling.

Looks like the import tool did the right thing and the data is consistent. We should probably notify Unsplash though, and make sure that this is to be expected. It is possible that other people using this dataset may parse it simply like we did in our ruby program and in doing so process the data incorrectly.





### Just do some fun queries

So now lets just poke around a bit and see what looks interesting.

* most common camera make / model
* long exposures - star trails / time laps
* small aperture - scenery
* popular keywords
*


