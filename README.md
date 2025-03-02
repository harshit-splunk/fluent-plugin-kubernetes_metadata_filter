# fluent-plugin-kubernetes_metadata_filter, a plugin for [Fluentd](http://fluentd.org)
[![Circle CI](https://circleci.com/gh/fabric8io/fluent-plugin-kubernetes_metadata_filter.svg?style=svg)](https://circleci.com/gh/fabric8io/fluent-plugin-kubernetes_metadata_filter)
[![Code Climate](https://codeclimate.com/github/fabric8io/fluent-plugin-kubernetes_metadata_filter/badges/gpa.svg)](https://codeclimate.com/github/fabric8io/fluent-plugin-kubernetes_metadata_filter)
[![Test Coverage](https://codeclimate.com/github/fabric8io/fluent-plugin-kubernetes_metadata_filter/badges/coverage.svg)](https://codeclimate.com/github/fabric8io/fluent-plugin-kubernetes_metadata_filter)
[![Ruby Style Guide](https://img.shields.io/badge/code_style-rubocop-brightgreen.svg)](https://github.com/rubocop-hq/rubocop)
[![Ruby Style Guide](https://img.shields.io/badge/code_style-community-brightgreen.svg)](https://rubystyle.guide)

The Kubernetes metadata plugin filter enriches container log records with pod and namespace metadata.

This plugin derives basic metadata about the container that emitted a given log record using the source of the log record. Records from journald provide metadata about the
container environment as named fields. Records from JSON files encode metadata about the container in the file name.  The initial metadata derived from the source is used
to lookup additional metadata about the container's associated pod and namespace (e.g. UUIDs, labels, annotations) when the kubernetes_url is configured.  If the plugin cannot
authoritatively determine the namespace of the container emitting a log record, it will use an 'orphan' namespace ID in the metadata. This behaviors supports multi-tenant systems
that rely on the authenticity of the namespace for proper log isolation.

## Requirements

| fluent-plugin-kubernetes_metadata_filter  | fluentd | ruby |
|-------------------|---------|------|
| >= 2.10.0 | >= v1.10.0 | >= 2.6 |
| >= 2.5.0 | >= v1.10.0 | >= 2.5 |
| >= 2.0.0 | >= v0.14.20 | >= 2.1 |
|  < 2.0.0 | >= v0.12.0 | >= 1.9 |

NOTE: For v0.12 version, you should use 1.x.y version. Please send patch into v0.12 branch if you encountered 1.x version's bug.

NOTE: This documentation is for fluent-plugin-kubernetes_metadata_filter-plugin-elasticsearch 2.x or later. For 1.x documentation, please see [v0.12 branch](https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter/tree/v0.12).

## Installation

    gem install fluent-plugin-kubernetes_metadata_filter

## Configuration

Configuration options for fluent.conf are:

* `kubernetes_url` - URL to the API server. Set this to retrieve further kubernetes metadata for logs from kubernetes API server. If not specified, environment variables `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT` will be used if both are present which is typically true when running fluentd in a pod.
* `apiVersion` - API version to use (default: `v1`)
* `ca_file` - path to CA file for Kubernetes server certificate validation
* `verify_ssl` - validate SSL certificates (default: `true`)
* `client_cert` - path to a client cert file to authenticate to the API server
* `client_key` - path to a client key file to authenticate to the API server
* `bearer_token_file` - path to a file containing the bearer token to use for authentication
* `tag_to_kubernetes_name_regexp` - the regular expression used to extract kubernetes metadata (pod name, container name, namespace) from the current fluentd tag.
This must use named capture groups for `container_name`, `pod_name`, `namespace`, and either `pod_uuid (/var/log/pods)` or `docker_id (/var/log/containers)`
* `cache_size` - size of the cache of Kubernetes metadata to reduce requests to the API server (default: `1000`)
* `cache_ttl` - TTL in seconds of each cached element. Set to negative value to disable TTL eviction (default: `3600` - 1 hour)
* `watch` - set up a watch on pods on the API server for updates to metadata (default: `true`)
* *DEPRECATED*`de_dot` - replace dots in labels and annotations with configured `de_dot_separator`, required for Datadog and ElasticSearch 2.x compatibility (default: `true`)
* *DEPRECATED*`de_dot_separator` - separator to use if `de_dot` is enabled (default: `_`)
* *DEPRECATED*`de_slash` - replace slashes in labels and annotations with configured `de_slash_separator`, required for Datadog compatibility (default: `false`)
* *DEPRECATED*`de_slash_separator` - separator to use if `de_slash` is enabled (default: `__`)
* *DEPRECATED* `use_journal` - If false, messages are expected to be formatted and tagged as if read by the fluentd in\_tail plugin with wildcard filename.  If true, messages are expected to be formatted as if read from the systemd journal.  The `MESSAGE` field has the full message.  The `CONTAINER_NAME` field has the encoded k8s metadata (see below).  The `CONTAINER_ID_FULL` field has the full container uuid.  This requires docker to use the `--log-driver=journald` log driver.  If unset (the default), the plugin will use the `CONTAINER_NAME` and `CONTAINER_ID_FULL` fields
if available, otherwise, will use the tag in the `tag_to_kubernetes_name_regexp` format.
* `container_name_to_kubernetes_regexp` - The regular expression used to extract the k8s metadata encoded in the journal `CONTAINER_NAME` field default: See [code](https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter/blob/master/lib/fluent/plugin/filter_kubernetes_metadata.rb#L68)
  * This corresponds to the definition [in the source](https://github.com/kubernetes/kubernetes/blob/release-1.6/pkg/kubelet/dockertools/docker.go#L317)
* `annotation_match` - Array of regular expressions matching annotation field names. Matched annotations are added to a log record.
* `allow_orphans` - Modify the namespace and namespace id to the values of `orphaned_namespace_name` and `orphaned_namespace_id`
when true (default: `true`)
* `orphaned_namespace_name` - The namespace to associate with records where the namespace can not be determined (default: `.orphaned`)
* `orphaned_namespace_id` - The namespace id to associate with records where the namespace can not be determined (default: `orphaned`)
* `lookup_from_k8s_field` - If the field `kubernetes` is present, lookup the metadata from the given subfields such as `kubernetes.namespace_name`, `kubernetes.pod_name`, etc.  This allows you to avoid having to pass in metadata to lookup in an explicitly formatted tag name or in an explicitly formatted `CONTAINER_NAME` value.  For example, set `kubernetes.namespace_name`, `kubernetes.pod_name`, `kubernetes.container_name`, and `docker.id` in the record, and the filter will fill in the rest. (default: `true`)
* `ssl_partial_chain` - if `ca_file` is for an intermediate CA, or otherwise we do not have the root CA and want
  to trust the intermediate CA certs we do have, set this to `true` - this corresponds to
  the `openssl s_client -partial_chain` flag and `X509_V_FLAG_PARTIAL_CHAIN` (default: `false`)
* `skip_labels` - Skip all label fields from the metadata.
* `skip_container_metadata` - Skip some of the container data of the metadata. The metadata will not contain the container_image and container_image_id fields.
* `skip_master_url` - Skip the master_url field from the metadata.
* `skip_namespace_metadata` - Skip the namespace_id field from the metadata. The fetch_namespace_metadata function will be skipped. The plugin will be faster and cpu consumption will be less.
* `watch_retry_interval` - The time interval in seconds for retry backoffs when watch connections fail. (default: `10`)


Reading from the JSON formatted log files with `in_tail` and wildcard filenames while respecting the CRI-o log format with the same config you need the fluent-plugin "multi-format-parser":

```
fluent-gem install fluent-plugin-multi-format-parser
```

The config block could look like this:
```
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file fluentd-docker.pos
  read_from_head true
  tag kubernetes.*
  <parse>
    @type multi_format
    <pattern>
      format json
      time_key time
      time_type string
      time_format "%Y-%m-%dT%H:%M:%S.%NZ"
      keep_time_key false
    </pattern>
    <pattern>
      format regexp
      expression /^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$/
      time_format '%Y-%m-%dT%H:%M:%S.%N%:z'
      keep_time_key false
    </pattern>
  </parse>
</source>

<filter kubernetes.var.log.containers.**.log>
  @type kubernetes_metadata
</filter>

<match **>
  @type stdout
</match>
```

Reading from the systemd journal (requires the fluentd `fluent-plugin-systemd` and `systemd-journal` plugins, and requires docker to use the `--log-driver=journald` log driver):
```
<source>
  @type systemd
  path /run/log/journal
  pos_file journal.pos
  tag journal
  read_from_head true
</source>

# probably want to use something like fluent-plugin-rewrite-tag-filter to
# retag entries from k8s
<match journal>
  @type rewrite_tag_filter
  rewriterule1 CONTAINER_NAME ^k8s_ kubernetes.journal.container
  ...
</match>

<filter kubernetes.**>
  @type kubernetes_metadata
  use_journal true
</filter>

<match **>
  @type stdout
</match>
```


## Environment variables for Kubernetes

If the name of the Kubernetes node the plugin is running on is set as
an environment variable with the name `K8S_NODE_NAME`, it will reduce cache
misses and needless calls to the Kubernetes API.

In the Kubernetes container definition, this is easily accomplished by:

```yaml
env:
- name: K8S_NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
```

## Example input/output

Kubernetes creates symlinks to Docker log files in `/var/log/containers/*.log`. Docker logs in JSON format.

Assuming following inputs are coming from a log file named `/var/log/containers/fabric8-console-controller-98rqc_default_fabric8-console-container-df14e0d5ae4c07284fa636d739c8fc2e6b52bc344658de7d3f08c36a2e804115.log`:

```
{
  "log": "2015/05/05 19:54:41 \n",
  "stream": "stderr",
  "time": "2015-05-05T19:54:41.240447294Z"
}
```

Then output becomes as belows
```
{
  "log": "2015/05/05 19:54:41 \n",
  "stream": "stderr",
  "docker": {
    "id": "df14e0d5ae4c07284fa636d739c8fc2e6b52bc344658de7d3f08c36a2e804115",
  }
  "kubernetes": {
    "host": "jimmi-redhat.localnet",
    "pod_name":"fabric8-console-controller-98rqc",
    "pod_id": "c76927af-f563-11e4-b32d-54ee7527188d",
    "pod_ip": "172.17.0.8",
    "container_name": "fabric8-console-container",
    "namespace_name": "default",
    "namespace_id": "23437884-8e08-4d95-850b-e94378c9b2fd",
    "namespace_annotations": {
      "fabric8.io/git-commit": "5e1116f63df0bac2a80bdae2ebdc563577bbdf3c"
    },
    "namespace_labels": {
      "product_version": "v1.0.0"
    },
    "labels": {
      "component": "fabric8Console"
    }
  }
}
```

If using journal input, from docker configured with `--log-driver=journald`, the input looks like the `journalctl -o export` format:
```
# The stream identification is encoded into the PRIORITY field as an
# integer: 6, or github.com/coreos/go-systemd/journal.Info, marks stdout,
# while 3, or github.com/coreos/go-systemd/journal.Err, marks stderr.
PRIORITY=6
CONTAINER_ID=b6cbb6e73c0a
CONTAINER_ID_FULL=b6cbb6e73c0ad63ab820e4baa97cdc77cec729930e38a714826764ac0491341a
CONTAINER_NAME=k8s_registry.a49f5318_docker-registry-1-hhoj0_default_ae3a9bdc-1f66-11e6-80a2-fa163e2fff3a_799e4035
MESSAGE=172.17.0.1 - - [21/May/2016:16:52:05 +0000] "GET /healthz HTTP/1.1" 200 0 "" "Go-http-client/1.1"
```

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Test it (`GEM_HOME=vendor bundle install; GEM_HOME=vendor bundle exec rake test`)
5. Push to the branch (`git push origin my-new-feature`)
6. Create new Pull Request

## Copyright
  Copyright (c) 2015 jimmidyson
