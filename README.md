# fluent-plugin-label-router

[Fluentd](https://fluentd.org/) output plugin to route records based on their Kubernetes metadata.

## Installation

### RubyGems

```
$ gem install fluent-plugin-label-router
```

### Specific install

```
$ gem install specific_install && gem specific_install -l https://github.com/banzaicloud/fluent-plugin-label-router.git
```

### Bundler

Add following line to your Gemfile:

```ruby
gem "fluent-plugin-label-router"
```

And then execute:

```
$ bundle
```

## Configuration

The configuration builds from `<route>` sections. Each `route` section
can have several `<match>` statement. These statements computed in order and
positive (or in case of *negate true* negative) results break the evaluation.
We can say that the sections are coupled in a **lazy evaluation OR**. 

```
<match example.tag**>
  @type label_router
  <route>
     ...
  </route>
  <route>
    <match>
      ...
    </match>
    <match> #Exclude
      negate true
      ...
    </match>
  </route>
</match>
```

| Parameter   | Description                                                                                            | Default |
|-------------|--------------------------------------------------------------------------------------------------------|---------|
| @label      | New @LABEL if selectors matched                                                                        | nil     |
| tag         | New tag if selectors matched                                                                           | ""      |
| emit_mode   | Emit mode. If `batch`, the plugin will emit events per labels matched. Enum: record, batch             | batch   |
| sticky_tags | Sticky tags will match only one record from an event stream. The same tag will be treated the same way | true    |
| match       | Select the log if match with parameters defined                                                        | nil     |

#### Selectors / Excludes
| Parameter  | Description                                                                   | Default |
|------------|-------------------------------------------------------------------------------|---------|
| labels     | Label definition to match record. Example: app:nginx                          | nil     |
| namespaces | Namespaces definition to filter the record. Ignored if left empty.            | []      |
| negate     | Negate the selector meaning to exclude matches                                | false   |

## Rules of thumb

1. Defining more than one namespace in `namespaces` inside a `match` statement
will check whether any of that namespaces matches.

2. Using `sticky_tags` means that only the **first** record will be analysed per `tag`.
Keep that in mind if you are ingesting traffic that is not unique on a per tag bases.
Fluentd and fluent-bit tail logs from Kubernetes are unique per container.

3. The plugin does not check if the configuration is valid so be careful to not define
impossible statements like identical `match` statement with negate.

## Examples

### 1. Route specific `labels` and `namespaces` to `@label` and new `tag`
Configuration to re-tag and re-label all logs from `default` namespace with label `app=nginx` and `env=dev`.
```
<match example.tag**>
  @type label_router
  <route>
    @label @NGINX
    tag new_tag
    <match>
      labels app:nginx,env:dev
      namespaces default
    </match>
  </route>
</match>
```

### 2. Exclude specific `labels` and `namespaces`
Configuration to re-tag and re-label all logs that **not** from `default` namespace **and not** have labels `ap=nginx` and `env=dev`
```
<match example.tag**>
  @type label_router
  <route>
    @label @NGINX
    tag new_tag
    <match>
      negate true
      labels app:nginx,env:dev
      namespaces default
    </match>
  </route>
</match>
```

#### Example records

Input
```ruby
@label = ""; tag = "raw.input"; {"log" => "", "kubernetes" => { "namespace_name" => "default", "labels" =>  {"app" => "nginx", "env" => "dev" } } }
@label = ""; tag = "raw.input"; {"log" => "", "kubernetes" => { "namespace_name" => "kube-system", "labels" =>  {"app" => "tiller" } } }

```

Output
```ruby
@label = "@NGINX"; tag = "new_tag"; {"log" => "", "kubernetes" => { "namespace_name" => "default", "labels" =>  {"app" => "nginx" } } }
nil
```
### 2. Both `labels` and `namespace` are optional
Only `labels`
```
<match example.tag**>
  @type label_router
  <route>
    @label @NGINX
    tag new_tag
    <match>
      labels app:nginx
    </match>
  </route>
</match>
```
Only `namespace`
```
<match example.tag**>
  @type label_router
  <route>
    @label @NGINX
    tag new_tag
    <match>
      namespaces default
    </match>
  </route>
</match>
```
Rewrite all
```
<match example.tag**>
  @type label_router
  <match>
    @label @NGINX
    tag new_tag
  </match>
</match>
```

### 3. One of `@label` ot `tag` configuration should be specified
If you don't rewrite either of them fluent will **likely to crash** because it will reprocess the same messages again.

## Copyright

* Copyright(c) 2019- Banzai Cloud
* License
  * Apache License, Version 2.0
