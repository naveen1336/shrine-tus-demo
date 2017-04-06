# Shrine Tus Demo

This is a demo app for the [tus resumable upload protocol], which integrates
[tus-ruby-server] and [tus-js-client] with [Shrine] file attachment library.

## Setup

* Add .env with your Amazon S3 credentials:

  ```sh
  # .env
  S3_BUCKET="..."
  S3_REGION="..."
  S3_ACCESS_KEY_ID="..."
  S3_SECRET_ACCESS_KEY="..."
  ```

* Run `bundle install`

* Run `bundle exec rackup`

## Implementation

Since tus-ruby-server is only meant to be a temporary storage, after
tus-js-client uploads the file to tus-ruby-server, we need to move the uploaded
file to a permanent storage.

### Option A: Downloading through the tus server

The simplest option is to assign the tus URL as the cached file, using
[shrine-url] as the `:cache` storage.

```rb
gem "shrine-url", "~> 0.2"
```
```rb
Shrine.storages = {
  cache: Shrine::Storage::Url.new, # we save URLs to files served by the tus server
  store: Shrine::Storage::YourMainStorage.new(...),
}
```

After the file is uploaded to the tus server, we write its tus URL to the `id`
field of the uploaded file representation, and this way the file uploaded to
the tus server effectively acts as a cached Shrine file, making it easy to
upload the file to permanent storage in a background job.

```js
var file = {
  id: urlToFile,
  storage: "cache",
  metadata: {...},
}
```

On promoting the uploaded file will be downloaded from tus server and uploaded
to permanent storage. The [Down] gem that shrine-url uses doesn't currently
support resumable downloads, so if you want to be resilient to network errors
during large downloads, you can create a modified URL storage that uses `wget`.

```rb
require "tempfile"
require "open3"

class TusUrl < Shrine::Storage::Url
  # ...

  def download(id)
    tempfile = Tempfile.new("shrine-tus", binmode: true)
    cmd = %W[wget --no-verbose #{id} -O #{tempfile.path}]
    stdout, stderr, status = Open3.capture3(*cmd)

    if !status.success?
      tempfile.close!
      raise stderr
    end

    tempfile.tap(&:open)
  end

  def open(id)
    download(id)
  end

  # ...
end
```

If you also want `Shrine::UploadedFile#exists?` and
`Shrine::UploadedFile#delete` to work, you need to modify the URL storage to
add the `Tus-Resumable` header, because the tus specification (and thus
tus-ruby-server) requires it on HEAD and DELETE requests.

```rb
class TusUrl < Shrine::Storage::Url
  # ...

  private

  def request(*args)
    super do |req|
      req["Tus-Resumable"] = "1.0.0"
    end
  end

  # ...
end
```

### Option B: Downloading directly from the storage

While option **A** is simple and agnostic to the implemetation of the tus
server, downloading the uploaded file through the tus server has some
performance implications:

* The file will have to temporarily be downloaded to disk before it is uploaded
  to permanent storage, so you need to ensure that your main server has enough
  disk space to support this.

* Downloading files though tus-ruby-server impacts throughput of
  tus-ruby-server, because its workers need to wait until the whole file is
  downloaded before they become available again to serve other requests.

* If tus-ruby-server is hosted on Heroku or otherwise its web server has a
  request timeout configured, the request might time out before the file
  manages to get fully downloaded, depending on the size of the file and
  network conditions.

* If your permanent storage is of the same kind as the tus storage, the file
  can potentially be copied much faster by using features of that storage. For
  example, if both tus server and your app use Amazon S3 storage, you could
  issue a S3 COPY request to the permanent location, thus avoiding having to
  download and re-upload the file.

The simplest approach is to configure the tus storage inside your main app (if
it's not already), and rely on the common interface of all tus storages. That
way the code is the same regardless of which storage the tus server uses or
how that storage saves uploaded files.

```rb
require "uri"
require "down"
require "tempfile"

class TusUrl < Shrine::Storage::Url
  # ...

  def download(id)
    tempfile = Tempfile.new("shrine-tus", binmode: true)
    response = get_file(id)
    response.each { |chunk| tempfile << chunk }
    response.close
    tempfile.tap(&:open)
  end

  def open(id)
    response = get_file(id)

    Down::ChunkedIO.new(
      size:     response.length,
      chunks:   response.each,
      on_close: ->{response.close},
    )
  end

  private

  def get_file(url)
    uid = url.split("/").last
    storage.get_file(uid)
  end

  def tus_storage
    Tus::Server.opts[:storage]
  end

  # ...
end
```

Note that for filesystem storage this will only work if the tus server and your
main app share the same disk.

### Option C: Utilizing storage-specific optimizations

Option **B** is probably best in terms of performance if you're using a
different kind of permanent storage than the one tus-ruby-server uses. But if
you intend to use the **same kind of permanent storage as tus-ruby-server
uses**, you can optimize the file transfer from tus storage to permanent Shrine
storage even more by utilizing storage-specific optimizations that Shrine
storages have built-in.

This can be achieved by defining your temporary Shrine storage, instead of
being `Shrine::Storage::Url` like in options **A** and **B**, to be the same as
the storage that tus-ruby-server uses. So, if your tus-ruby-server is configured
with either of the following storages:

```rb
Tus::Server.opts[:storage] = Tus::Storage::Filesystem.new("data")
Tus::Server.opts[:storage] = Tus::Storage::Gridfs.new(client: mongo, prefix: "tus")
Tus::Server.opts[:storage] = Tus::Storage::S3.new(prefix: "tus", **s3_options)
```

Your Shrine temporary storage is configured with either of the following
storages:

```rb
Shrine.storages[:cache] = Shrine::Storage::FileSystem.new("data")
Shrine.storages[:cache] = Shrine::Storage::Gridfs.new(client: mongo, prefix: "tus")
Shrine.storages[:cache] = Shrine::Storage::S3.new(prefix: "tus", **s3_options)
```

On the client-side, once the file has finished uploading to your tus server,
you can continue sending the tus file data in the same format as if you were
using `Shrine::Storage::Url` as temporary Shrine storage (the same as in
options **A** and **B**):

```rb
{
  "id": "http://tus-server.org/96955a2864634ec2df84ce83d2013d66",
  "storage": "cache",
  "metadata": {
    "mime_type" => "video/mpeg",
    "filename" => "myvideo.mp4",
    "size" => 580598712
  }
}
```

Once this JSON data is submitted to your app, you need to transform the `id`
field into an identifier that your temporary Shrine storage will recognize.

```rb
movie_params = params["movie"]
file_data = JSON.parse(movie_params["video"])
tus_uid = file_data["id"].split("/").last
# ... transform `file_data` depending on the storage, see below ...
Movie.create(movie_params.merge("video" => file_data.to_json))
```

How you need to transform the `id` field depends on the storage you're using:

#### File System

```rb
require "shrine/storage/file_system"

Shrine.storages = {
  cache: Shrine::Storage::FileSystem.new("data"), # has to match tus storage
  store: Shrine::Storage::FileSystem.new("public", prefix: "uploads/store"),
}
```

The filesystem tus storage saves uploaded files to `<directory>/<uid>.file`, so
in your application you need to translate the tus URL sent by the client into
the correct file ID.

```rb
# ...
file_data #=> {"id" => "http://tus-server.org/68db42638388ae645ab747b36a837a79", ...}
file_data["id"] = "#{tus_uid}.file"
file_data #=> {"id" => "68db42638388ae645ab747b36a837a79.file", ...}
# ...
```

So far this will have the same performance as the generic version, because the
file is still being copied, if you load the `moving` Shrine plugin, the file
will be *moved* from temporary to permanent storage with the `mv` command,
which is instantaneous regardless of the filesize.

```rb
Shrine.plugin :moving
```

#### MongoDB GridFS

```rb
gem "shrine-gridfs"
```
```rb
require "shrine/storage/gridfs"

client = Mongo::Client.new("mongodb://127.0.0.1:27017/mydb", logger: Logger.new(nil))

Shrine.storages = {
  cache: Shrine::Storage::Gridfs.new(client: client, prefix: "tus"), # has to match tus storage
  store: Shrine::Storage::Gridfs.new(client: client),
}
```

The only difference between `Tus::Storage::Gridfs` and
`Shrine::Storage::Gridfs` is that the former saves the file ID into the
`filename` field, while the latter saves it into the `_id` field, so the code
for transformation of tus URL to Gridfs ID just requires the extra step of
finding the `_id` by `filename`.

```rb
# ...
file_data #=> {"id" => "http://tus-server.org/68db42638388ae645ab747b36a837a79", ...}
grid_info = Shrine.storages[:cache].bucket.find(filename: tus_uid).limit(1).first
file_data["id"] = grid_info[:_id].to_s
file_data #=> {"id" => "58d8d186c389e010d9e350b5", ...}
# ...
```

MongoDB doesn't support moving documents between different collections
(prefixes), so there is no moving support in `Shrine::Storage::Gridfs`. But it
still implements more efficient copying when it detects that source storage is
also Gridfs.

#### Amazon S3

```rb
require "shrine/storage/s3"

Shrine.storages = {
  cache: Shrine::Storage::S3.new(prefix: "tus", **s3_options), # has to match tus storage
  store: Shrine::Storage::S3.new(prefix: "store", **s3_options),
}
```

The S3 tus storage saves uploaded files to object with key `<prefix>/<uid>`,
so it's easy to translate the tus URL from attachment data sent by the client
into the corresponding object key.

```rb
# ...
file_data #=> {"id" => "http://tus-server.org/68db42638388ae645ab747b36a837a79", ...}
file_data["id"] = tus_id
file_data #=> {"id" => "68db42638388ae645ab747b36a837a79.file", ...}
# ...
```

When `Shrine::Storage::S3` detects that both source and destination storage are
S3, instead of downloading and re-uploading it will issue an S3 COPY request,
which should be very fast.

[tus resumable upload protocol]: http://tus.io
[tus-ruby-server]: https://github.com/janko-m/tus-ruby-server
[tus-js-client]: https://github.com/tus/tus-js-client
[Shrine]: https://github.com/janko-m/shrine
[shrine-url]: https://github.com/janko-m/shrine-url
[Down]: https://github.com/janko-m/down
