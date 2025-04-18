---
mapped_pages:
  - https://www.elastic.co/guide/en/logstash/current/multiline.html
---

# Managing Multiline Events [multiline]

Several use cases generate events that span multiple lines of text. In order to correctly handle these multiline events, Logstash needs to know how to tell which lines are part of a single event.

Multiline event processing is complex and relies on proper event ordering. The best way to guarantee ordered log processing is to implement the processing as early in the pipeline as possible.

The [multiline](logstash-docs-md://lsr/plugins-codecs-multiline.md) codec is the preferred tool for handling multiline events in the Logstash pipeline. The multiline codec merges lines from a single input using a simple set of rules.

::::{important}
If you are using a Logstash input plugin that supports multiple hosts, such as the [beats](logstash-docs-md://lsr/plugins-inputs-beats.md) input plugin, you should not use the [multiline](logstash-docs-md://lsr/plugins-codecs-multiline.md) codec to handle multiline events. Doing so may result in the mixing of streams and corrupted event data. In this situation, you need to handle multiline events before sending the event data to Logstash.
::::


The most important aspects of configuring the multiline codec are the following:

* The `pattern` option specifies a regular expression. Lines that match the specified regular expression are considered either continuations of a previous line or the start of a new multiline event. You can use [grok](logstash-docs-md://lsr/plugins-filters-grok.md) regular expression templates with this configuration option.
* The `what` option takes two values: `previous` or `next`. The `previous` value specifies that lines that match the value in the `pattern` option are part of the previous line. The `next` value specifies that lines that match the value in the `pattern` option are part of the following line.* The `negate` option applies the multiline codec to lines that *do not* match the regular expression specified in the `pattern` option.

See the full documentation for the [multiline](logstash-docs-md://lsr/plugins-codecs-multiline.md) codec plugin for more information on configuration options.

## Examples of Multiline Codec Configuration [_examples_of_multiline_codec_configuration]

The examples in this section cover the following use cases:

* Combining a Java stack trace into a single event
* Combining C-style line continuations into a single event
* Combining multiple lines from time-stamped events

### Java Stack Traces [_java_stack_traces]

Java stack traces consist of multiple lines, with each line after the initial line beginning with whitespace, as in this example:

```java
Exception in thread "main" java.lang.NullPointerException
        at com.example.myproject.Book.getTitle(Book.java:16)
        at com.example.myproject.Author.getBookTitles(Author.java:25)
        at com.example.myproject.Bootstrap.main(Bootstrap.java:14)
```

To consolidate these lines into a single event in Logstash, use the following configuration for the multiline codec:

```json
input {
  stdin {
    codec => multiline {
      pattern => "^\s"
      what => "previous"
    }
  }
}
```

This configuration merges any line that begins with whitespace up to the previous line.


### Line Continuations [_line_continuations]

Several programming languages use the `\` character at the end of a line to denote that the line continues, as in this example:

```c
printf ("%10.10ld  \t %10.10ld \t %s\
  %f", w, x, y, z );
```

To consolidate these lines into a single event in Logstash, use the following configuration for the multiline codec:

```json
input {
  stdin {
    codec => multiline {
      pattern => "\\$"
      what => "next"
    }
  }
}
```

This configuration merges any line that ends with the `\` character with the following line.


### Timestamps [_timestamps]

Activity logs from services such as Elasticsearch typically begin with a timestamp, followed by information on the specific activity, as in this example:

```shell
[2015-08-24 11:49:14,389][INFO ][env                      ] [Letha] using [1] data paths, mounts [[/
(/dev/disk1)]], net usable_space [34.5gb], net total_space [118.9gb], types [hfs]
```

To consolidate these lines into a single event in Logstash, use the following configuration for the multiline codec:

```json
input {
  file {
    path => "/var/log/someapp.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601} "
      negate => true
      what => previous
    }
  }
}
```

This configuration uses the `negate` option to specify that any line that does not begin with a timestamp belongs to the previous line.



