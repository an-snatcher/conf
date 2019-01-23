**crawler-conf.yaml**

```

config: 
  topology.workers: 1
  topology.message.timeout.secs: 300
  topology.max.spout.pending: 100
  topology.debug: false
  fetcher.threads.number: 50
  worker.heap.memory.mb: 2048
 
  topology.kryo.register:
    - com.digitalpebble.stormcrawler.Metadata

  metadata.persist:
   - _redirTo
   - error.cause
   - error.source
   - isSitemap
   - isFeed

  http.agent.name: "StormCrawler"
  http.agent.version: "1.0"
  http.agent.description: "crawler"
  http.agent.url: "https://test.edu/"
  http.agent.email: "noreply@test.edu"
 
  http.content.limit: -1
  jsoup.treat.non.html.as.error: false
  parsefilters.config.file: "parsefilters.json"
  urlfilters.config.file: "urlfilters.json"
  fetchInterval.default: 2880
  fetchInterval.fetch.error: 120
  fetchInterval.error: -1
  textextractor.include.pattern:
   - DIV[id="maincontent"]
   - DIV[itemprop="articleBody"]
   - ARTICLE

  textextractor.exclude.tags:
   - STYLE
   - SCRIPT
   - HEADER
   - FOOTER

  indexer.url.fieldname: "url"
  indexer.text.fieldname: "content"
  indexer.canonical.name: "canonical"
  indexer.md.mapping:
  - parse.title=title
  - parse.keywords=keywords
  - parse.description=description
  - domain=domain

  topology.metrics.consumer.register:
     - class: "org.apache.storm.metric.LoggingMetricsConsumer"
       parallelism.hint: 1

```



**es-injector.flux**

```
name: "injector"

includes:
    - resource: true
      file: "/crawler-default.yaml"
      override: false

    - resource: false
      file: "crawler-conf.yaml"
      override: true

    - resource: false
      file: "es-conf.yaml"
      override: true

spouts:
  - id: "spout"
    className: "com.digitalpebble.stormcrawler.spout.FileSpout"
    parallelism: 1
    constructorArgs:
      - "."
      - "seeds.txt"
      - true

bolts:
  - id: "status"
    className: "com.digitalpebble.stormcrawler.elasticsearch.persistence.StatusUpdaterBolt"
    parallelism: 1
  - id: "parser_bolt"
    className: "com.digitalpebble.stormcrawler.tika.ParserBolt"
    parallelism: 1

streams:
  - from: "spout"
    to: "status"
    grouping:
      type: CUSTOM
      customClass:
        className: "com.digitalpebble.stormcrawler.util.URLStreamGrouping"
        constructorArgs:
          - "byHost"
      streamId: "status"
```



**es-crawler.flux**

```
name: "crawler"

includes:
    - resource: true
      file: "/crawler-default.yaml"
      override: false

    - resource: false
      file: "crawler-conf.yaml"
      override: true

    - resource: false
      file: "es-conf.yaml"
      override: true

spouts:
  - id: "spout"
    className: "com.digitalpebble.stormcrawler.elasticsearch.persistence.AggregationSpout"
    parallelism: 10

bolts:
  - id: "partitioner"
    className: "com.digitalpebble.stormcrawler.bolt.URLPartitionerBolt"
    parallelism: 1
  - id: "fetcher"
    className: "com.digitalpebble.stormcrawler.bolt.FetcherBolt"
    parallelism: 1
  - id: "sitemap"
    className: "com.digitalpebble.stormcrawler.bolt.SiteMapParserBolt"
    parallelism: 1
  - id: "parse"
    className: "com.digitalpebble.stormcrawler.bolt.JSoupParserBolt"
    parallelism: 1
  - id: "index"
    className: "com.digitalpebble.stormcrawler.elasticsearch.bolt.IndexerBolt"
    parallelism: 1
  - id: "status"
    className: "com.digitalpebble.stormcrawler.elasticsearch.persistence.StatusUpdaterBolt"
    parallelism: 1
  - id: "status_metrics"
    className: "com.digitalpebble.stormcrawler.elasticsearch.metrics.StatusMetricsBolt"
    parallelism: 1
  - id: "redirection_bolt"
    className: "com.digitalpebble.stormcrawler.tika.RedirectionBolt"
    parallelism: 1
  - id: "parser_bolt"
    className: "com.digitalpebble.stormcrawler.tika.ParserBolt"
    parallelism: 1 

streams:
  - from: "spout"
    to: "partitioner"
    grouping:
      type: SHUFFLE
      
  - from: "spout"
    to: "status_metrics"
    grouping:
      type: SHUFFLE     

  - from: "partitioner"
    to: "fetcher"
    grouping:
      type: FIELDS
      args: ["key"]

  - from: "fetcher"
    to: "sitemap"
    grouping:
      type: LOCAL_OR_SHUFFLE

  - from: "sitemap"
    to: "parse"
    grouping:
      type: LOCAL_OR_SHUFFLE

  - from: "parse"
    to: "index"
    grouping:
      type: LOCAL_OR_SHUFFLE

  - from: "fetcher"
    to: "status"
    grouping:
      type: FIELDS
      args: ["url"]
      streamId: "status"

  - from: "sitemap"
    to: "status"
    grouping:
      type: FIELDS
      args: ["url"]
      streamId: "status"

  - from: "parse"
    to: "status"
    grouping:
      type: FIELDS
      args: ["url"]
      streamId: "status"

  - from: "index"
    to: "status"
    grouping:
      type: FIELDS
      args: ["url"]
      streamId: "status"
      
  - from: "parse"
    to: "redirection_bolt"
    grouping:
      type: LOCAL_OR_SHUFFLE

  - from: "redirection_bolt"
    to: "parser_bolt"
    grouping:
      type: LOCAL_OR_SHUFFLE

  - from: "redirection_bolt"
    to: "index"
    grouping:
      type: LOCAL_OR_SHUFFLE

  - from: "parser_bolt"
    to: "index"
    grouping:
      type: LOCAL_OR_SHUFFLE

  - from: "redirection_bolt"
    to: "parser_bolt"
    grouping:
      type: LOCAL_OR_SHUFFLE
      streamId: "tika"
```



**es-conf.yaml**

```
config:
  es.indexer.addresses: "http://localhost:9200"
  es.indexer.index.name: "www-test-index"

  es.indexer.doc.type: "doc"
  es.indexer.create: false
  es.indexer.settings:
    cluster.name: "elasticsearch"

  es.metrics.addresses: "http://localhost:9200"
  es.metrics.index.name: "www-test-metrics"
  es.metrics.doc.type: "datapoint"
  es.metrics.settings:
    cluster.name: "elasticsearch"
  
  es.status.addresses: "http://localhost:9200"
  es.status.index.name: "www-test-status"

  es.status.doc.type: "status"
  es.status.routing: true
  es.status.routing.fieldname: "metadata.hostname"
  es.status.bulkActions: 500
  es.status.flushInterval: "5s"
  es.status.concurrentRequests: 1
  #### Change the Cluster Name ####
  es.status.settings:
    cluster.name: "elasticsearch"
  
  spout.ttl.purgatory: 30
  spout.min.delay.queries: 500
  spout.reset.fetchdate.after: 120
  es.status.max.buckets: 50
  es.status.max.urls.per.bucket: 15
  es.status.bucket.field: "metadata.hostname"
  es.status.bucket.sort.field: "nextFetchDate"
  es.status.global.sort.field: "nextFetchDate"
  es.status.max.start.offset: 500
   es.status.sample: false

  es.status.recentDate.increase: -1
  es.status.recentDate.min.gap: -1
  topology.metrics.consumer.register:
       - class: "com.digitalpebble.stormcrawler.elasticsearch.metrics.MetricsConsumer"
         parallelism.hint: 1
 
```



**pom.xml**

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">

	<modelVersion>4.0.0</modelVersion>
	<groupId>www.colleges.edu</groupId>
	<artifactId>www-colleges</artifactId>
	<version>1.0-SNAPSHOT</version>
	<packaging>jar</packaging>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<stormcrawler.version>1.13</stormcrawler.version>
	</properties>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.2</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.codehaus.mojo</groupId>
				<artifactId>exec-maven-plugin</artifactId>
				<version>1.3.2</version>
				<executions>
					<execution>
						<goals>
							<goal>exec</goal>
						</goals>
					</execution>
				</executions>
				<configuration>
					<executable>java</executable>
					<includeProjectDependencies>true</includeProjectDependencies>
					<includePluginDependencies>false</includePluginDependencies>
					<classpathScope>compile</classpathScope>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-shade-plugin</artifactId>
				<version>1.3.3</version>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>shade</goal>
						</goals>
						<configuration>
							<createDependencyReducedPom>false</createDependencyReducedPom>
							<transformers>
								<transformer
									implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer" />
								<transformer
									implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
									<mainClass>org.apache.storm.flux.Flux</mainClass>
									<manifestEntries>
										<Change></Change>
										<Build-Date></Build-Date>
									</manifestEntries>
								</transformer>
							</transformers>
							<!-- The filters below are necessary if you want to include the Tika 
								module -->
							<filters>
								<filter>
									<artifact>*:*</artifact>
									<excludes>
										<exclude>META-INF/*.SF</exclude>
										<exclude>META-INF/*.DSA</exclude>
										<exclude>META-INF/*.RSA</exclude>
									</excludes>
								</filter>
								<filter>
									<!-- https://issues.apache.org/jira/browse/STORM-2428 -->
									<artifact>org.apache.storm:flux-core</artifact>
									<excludes>
										<exclude>org/apache/commons/**</exclude>
										<exclude>org/apache/http/**</exclude>
										<exclude>org/yaml/**</exclude>
									</excludes>
								</filter>
							</filters>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

	<dependencies>
		<dependency>
			<groupId>com.digitalpebble.stormcrawler</groupId>
			<artifactId>storm-crawler-core</artifactId>
			<version>${stormcrawler.version}</version>
		</dependency>
		<dependency>
			<groupId>com.digitalpebble.stormcrawler</groupId>
			<artifactId>storm-crawler-elasticsearch</artifactId>
			<version>${stormcrawler.version}</version>
		</dependency>
		<dependency>
			<groupId>com.digitalpebble.stormcrawler</groupId>
			<artifactId>storm-crawler-tika</artifactId>
			<version>${stormcrawler.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.storm</groupId>
			<artifactId>storm-core</artifactId>
			<version>1.2.2</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>org.apache.storm</groupId>
			<artifactId>flux-core</artifactId>
			<version>1.2.2</version>
		</dependency>
	</dependencies>
</project>

```



**CrawlTopology.java**

```
package www.test.com;

import org.apache.storm.metric.LoggingMetricsConsumer;
import org.apache.storm.topology.TopologyBuilder;
import org.apache.storm.tuple.Fields;

import com.digitalpebble.stormcrawler.ConfigurableTopology;
import com.digitalpebble.stormcrawler.Constants;
import com.digitalpebble.stormcrawler.bolt.FetcherBolt;
import com.digitalpebble.stormcrawler.bolt.JSoupParserBolt;
import com.digitalpebble.stormcrawler.bolt.SiteMapParserBolt;
import com.digitalpebble.stormcrawler.bolt.URLPartitionerBolt;
import com.digitalpebble.stormcrawler.elasticsearch.bolt.DeletionBolt;
import com.digitalpebble.stormcrawler.elasticsearch.bolt.IndexerBolt;
import com.digitalpebble.stormcrawler.elasticsearch.metrics.MetricsConsumer;
import com.digitalpebble.stormcrawler.elasticsearch.persistence.CollapsingSpout;
import com.digitalpebble.stormcrawler.elasticsearch.metrics.StatusMetricsBolt;
import com.digitalpebble.stormcrawler.elasticsearch.persistence.StatusUpdaterBolt;
import com.digitalpebble.stormcrawler.tika.RedirectionBolt;
import com.digitalpebble.stormcrawler.tika.ParserBolt;


import com.digitalpebble.stormcrawler.util.ConfUtils;

public class CrawlTopology extends ConfigurableTopology {

    public static void main(String[] args) throws Exception {
        ConfigurableTopology.start(new CrawlTopology(), args);
    }

    @Override
    protected int run(String[] args) {
        TopologyBuilder builder = new TopologyBuilder();

        int numWorkers = ConfUtils.getInt(getConf(), "topology.workers", 1);

        // set to the real number of shards ONLY if es.status.routing is set to
        // true in the configuration
        int numShards = 1;

        builder.setSpout("spout", new CollapsingSpout(), numShards);

        builder.setBolt("status_metrics", new StatusMetricsBolt())
                .shuffleGrouping("spout");

        builder.setBolt("partitioner", new URLPartitionerBolt(), numWorkers)
                .shuffleGrouping("spout");

        builder.setBolt("fetch", new FetcherBolt(), numWorkers).fieldsGrouping(
                "partitioner", new Fields("key"));

        builder.setBolt("sitemap", new SiteMapParserBolt(), numWorkers)
                .localOrShuffleGrouping("fetch");

        builder.setBolt("parse", new JSoupParserBolt(), numWorkers)
                .localOrShuffleGrouping("sitemap");

        builder.setBolt("indexer", new IndexerBolt(), numWorkers)
                .localOrShuffleGrouping("parse");
				
		 builder.setBolt("jsoup", new JSoupParserBolt())
				.localOrShuffleGrouping(
          "sitemap");
  
		builder.setBolt("shunt", new RedirectionBolt()).localOrShuffleGrouping("jsoup");	
  
		builder.setBolt("tika", new ParserBolt()).localOrShuffleGrouping("shunt",
          "tika");
        Fields furl = new Fields("url");
        builder.setBolt("status", new StatusUpdaterBolt(), numWorkers)
                .fieldsGrouping("fetch", Constants.StatusStreamName, furl)
                .fieldsGrouping("sitemap", Constants.StatusStreamName, furl)
                .fieldsGrouping("parse", Constants.StatusStreamName, furl)
                .fieldsGrouping("indexer", Constants.StatusStreamName, furl)
                .fieldsGrouping("jsoup", Constants.StatusStreamName, furl)				            .fieldsGrouping("shunt", Constants.StatusStreamName, furl)
				.fieldsGrouping("tika", Constants.StatusStreamName, furl);
        builder.setBolt("deleter", new DeletionBolt(), numWorkers)
                .localOrShuffleGrouping("status",
                        Constants.DELETION_STREAM_NAME);

        conf.registerMetricsConsumer(MetricsConsumer.class);
        conf.registerMetricsConsumer(LoggingMetricsConsumer.class);

        return submit("crawl", conf, builder);
    }
}

```

