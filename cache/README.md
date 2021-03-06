# Cache builders

This includes a pair of builders, `save_cache` and `restore_cache`, that work together to cache files between builds to a GCS bucket (or local file).

## Using the `save_cache` builder

All options use the form `--option=value` or `-o=value` so that they look nice in Yaml files.

| Option          | Description                                                 |
| --------------- | ----------------------------------------------------------- |
| -b, --bucket    | The cloud storage bucket to upload the cache to. [optional] |
| -o, --out       | The output directory to write the cache to. [optional]      |
| -k, --key       | The cache key used for this cache file. [optional]          |
| -p, --path      | The files to store in the cache. Can be repeated.           |
| -t, --threshold | The parallel composite upload threshold [default: 50M]      |

One of `--bucket` or `--out` parameters are required.  If `--bucket` then the cache file will be uploaded to the provided GCS bucket path.  If `--out` then the cache file will be stored in the directory specified on disk.

The key provided by `--key` is used to identify the cache file. Any other cache files for the same key will be overwritten by this one.

The `--path` parameters can be repeated for as many folders as you'd like to cache.  When restored, they will retain folder structure on disk.

## Using the `restore_cache` builder

All options use the form `--option=value` or `-o=value` so that they look nice in Yaml files.

| Option       | Description                                                      |
| ------------ | ---------------------------------------------------------------- |
| -b, --bucket | The cloud storage bucket to download the cache from. [optional]  |
| -s, --src    | The local directory in which the cache is stored. [optional]     |
| -k, --key    | The cache key used for this cache file. [optional]           |

One of `--bucket` or `--src` parameters are required.  If `--bucket` then the cache file will be downloaded from the provided GCS bucket path.  If `--src` then the cache file will be read from the directory specified on disk.

The key provided by `--key` is used to identify the cache file.

### `checksum` Helper

As apps develop, cache needs change. For instance when dependencies are removed from a project, or versions are updated, there is no need to cache the older versions of dependencies. Therefore it's recommended that you update the cache key when these changes occur.

This builder includes a `checksum` helper script, which you can use to create a simple checksum of files in your project to use as a cache key.

To use it in the `--key` arguemnt, simply surround the command with `$()`:

```bash
--key=build-cache-$(checksum build.gradle)-$(checksum dependencies.gradle)
```

To ensure you aren't paying for storage of obsolete cache files you can add an Object Lifecycle Rule to the cache bucket to delete object older than 30 days.

## Examples

The following examples demonstrate build requests that use this builder.

### Saving a cache with checksum to GCS bucket

This `cloudbuild.yaml` saves the files and folders in the `path` arguments to a cache file in the GCS bucket `gs://$CACHE_BUCKET/`.  In this example the key will be updated, resulting in a new cache, every time the `cloudbuild.yaml` build file is changed.

```yaml
- name: 'gcr.io/$PROJECT_ID/save_cache'
  args:
  - '--bucket=gs://$CACHE_BUCKET/'
  - '--key=resources-$( checksum cloudbuild.yaml )'
  - '--path=.cache/folder1'
  - '--path=.cache/folder2/subfolder3'
```

### Saving a cache with checksum to a local file

This `cloudbuild.yaml` saves the files and folders in the `path` arguments to a cache file in the directory passed to the `out` parameter.  In this example the key will be updated, resulting in a new cache, every time the `cloudbuild.yaml` build file is changed.

```yaml
- name: 'gcr.io/$PROJECT_ID/save_cache'
  args:
  - '--out=/cache/'
  - '--key=resources-$( checksum cloudbuild.yaml )'
  - '--path=.cache/folder1'
  - '--path=.cache/folder2/subfolder3'
  volumes:
  - name: 'cache'
    path: '/cache'
```

### Restore a cache from a GCS bucket

This `cloudbuild.yaml` restores the files from the compressed cache file identified by `key` on the cache bucket provided, if it exists.

```yaml
- name: 'gcr.io/$PROJECT_ID/restore_cache'
  args:
  - '--bucket=gs://$CACHE_BUCKET/'
  - '--key=resources-$( checksum cloudbuild.yaml )'
```

### Restore a cache from a local file

This `cloudbuild.yaml` restores the files from the compressed cache file identified by `key` on the local filesystem, if it exists.

```yaml
- name: 'gcr.io/$PROJECT_ID/restore_cache'
  args:
  - '--src=/cache/'
  - '--key=resources-$( checksum cloudbuild.yaml )'
  volumes:
  - name: 'cache'
    path: '/cache'
```
