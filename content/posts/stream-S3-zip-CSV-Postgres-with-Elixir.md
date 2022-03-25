---
title: "Stream S3 > zip > CSV > Postgres with Elixir"
date: 2022-03-25T11:00:00+02:00
draft: false
tags: ["elixir", "stream", "S3"]
---

I was recently confronted with this situation:
* an S3 bucket contains a zip file
* the zip file contains a CSV
* I want to take the content of that CSV, transform it, and push it to a Postgres database

## Without streaming
Without streaming, the steps to do that are the following:
* download the zip archive locally (maybe it is huge!)
* unzip the archive
* open the CSV file and load its content in memory (maybe it is huge!)
* transform the content
* save it to the database

If the zip archive is really big, but the CSV file you are interested in is small, downloading the zip file and unzipping it is a lot of wasted resources. But if the CSV is big, then loading it in memory will take a lot of memory space.

## Streaming all the way
The fact that you can in Elixir stream that task from the S3 bucket, all the way to your database completely blows my mind. And it means that no matter the size of the zip archive and the CSV file, you can do the job, one small part at a time, with a minimal memory footprint. 

Here is how to do it.

### Unzip
Elixir [Unzip](https://hexdocs.pm/unzip/readme.html) library allows you to stream the content of a single file inside a zip archive. To quote their documentation:

> For example, if a zip file has 100 files and we only want one file, Unzip reads only that file.

The zip file can be on your local file system, but [it can also be on an S3 bucket](https://hexdocs.pm/unzip/readme.html#s3-file).

### NimbleCSV

> [NimbleCSV](https://hexdocs.pm/nimble_csv/NimbleCSV.html#content) is a small and fast parsing and dumping library.

It especially provides a useful function, [to_line_stream/1](https://hexdocs.pm/nimble_csv/NimbleCSV.html#c:to_line_stream/1) to transform a stream "by chunk" to a stream "by line". That's because Unzip will stream the file by chunks, but to parse and transform the CSV, we need a stream "line by line".

### Ecto

No need to talk again about Ecto's [streaming capabilities](https://hexdocs.pm/ecto/Ecto.Repo.html#c:stream/2) if you have read my [previous post](https://francis.chabouis.fr/posts/stream-data-from-an-api-to-your-database-with-elixir/).

## Putting it all together

1. Install the dependencies

```elixir
def deps do
  [
    {:unzip, "~> x.x.x"},
    {:nimble_csv, "~> 1.1"},
    {:ecto_sql, "~> 3.0"},
    {:postgrex, ">= 0.0.0"}
  ]
end
```

2. Copy the `Unzip.S3File` module and the `Unzip.FileAccess` implementation, as provided by the [Unzip doc](https://hexdocs.pm/unzip/readme.html#s3-file).

3. Start streaming!

```elixir
    # Unzip
    aws_s3_config =
      ExAws.Config.new(:s3,
        access_key_id: ["xxx", :instance_role],
        secret_access_key: ["xxx", :instance_role]
      )

    file = new("zip_archive_name", "bucket_name", aws_s3_config)
    {:ok, unzip} = Unzip.new(file)
    unzip_stream = Unzip.file_stream!(unzip, file_name)

    # NimbleCSV + Ecto

    # everything is wrapped in a Ecto transaction
    MyRepo.transaction(fn ->
      unzip_stream
      # transform the stream to a stream of binaries
      |> Stream.map(fn c -> IO.iodata_to_binary(c) end)
      # transform the stream to a stream line by line
      |> NimbleCSV.RFC4180.to_line_stream()
      |> NimbleCSV.RFC4180.parse_stream(skip_headers: false)
      # transform the stream to a stream of maps %{column_name1: value1, ...}
      |> Stream.transform([], fn r, acc ->
        if acc == [] do
          # first row contains the column names, we put them in the accumulator
          {%{}, r}
        else
          # other rows contain the values, we zip them with the column names
          {[acc |> Enum.zip(r) |> Enum.into(%{})], acc}
        end
      end)
      |> Stream.map(fn m ->
        # we can transform the map to something we want to insert in the db
        %{
          stop_id: m |> Map.fetch!("stop_id"),
          ...,
          ...,
        }
      end)
      # we'll insert the data by groups of 1000 rows
      |> Stream.chunk_every(1000)
      |> Stream.each(fn chunk -> MyRepo.insert_all("your db schema module here", chunk) end)
      |> Stream.run()
    end)
```