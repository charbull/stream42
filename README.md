<!-- This README can be viewed at https://github.com/joshsh/stream42/wiki -->

## Stream42

![Stream42 logo|width=94px|height=65px](https://github.com/joshsh/stream42/wiki/graphics/sesamestream-logo-small.png)

Stream42 is an in-memory tuple stream processor and continuous SPARQL query engine built with the [Sesame](http://rdf4j.org/) RDF framework.  It implements an open-world subset (see below) of the [SPARQL](http://www.w3.org/TR/sparql11-query/) query language and uses an incremental technique based on the [symmetric hash join](http://en.wikipedia.org/wiki/Symmetric_Hash_Join), producing solutions to input tuples as soon as they are computable, and discarding irrelevant inputs.  Stream42 uses [time-to-live](http://en.wikipedia.org/wiki/Time_to_live) to process unbounded data streams: queries time out and are removed unless renewed, while intermediate solutions to queries exist in the stream processor only as long as their shortest-lived tuple, making room for fresh data as they expire.  Stream42 integrates with [LinkedDataSail](https://github.com/joshsh/ripple/wiki/LinkedDataSail), following links in response to join operations.

Below is a usage example in Java.  See the [source code](https://github.com/joshsh/stream42/blob/master/stream42-examples/src/main/java/net/fortytwo/stream/examples/BasicExample.java) for the full example.

```java
// A query for things written by Douglas Adams which are referenced with a pointing gesture
String query = "PREFIX activity: <http://fortytwo.net/2015/extendo/activity#>\n" +
        "PREFIX dbo: <http://dbpedia.org/ontology/>\n" +
        "PREFIX dbr: <http://dbpedia.org/resource/>\n" +
        "PREFIX foaf: <http://xmlns.com/foaf/0.1/>\n" +
        "SELECT ?actor ?indicated WHERE {\n" +
        "?a activity:thingIndicated ?indicated .\n" +
        "?a activity:actor ?actor .\n" +
        "?indicated dbo:author dbr:Douglas_Adams .\n" +
        "}";

// An RDF graph representing an event. Normally, this would come from a dynamic data source.
// The example is from the Typeatron keyer (see http://github.com/joshsh/extendo).
String eventData = "@prefix activity: <http://fortytwo.net/2015/extendo/activity#> .\n" +
        "@prefix dbr: <http://dbpedia.org/resource/> .\n" +
        "@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .\n" +
        "@prefix tl: <http://purl.org/NET/c4dm/timeline.owl#> .\n" +
        "@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .\n" +
        "\n" +
        "<urn:uuid:e6f4c759-712c-448c-96f0-c2ecee2ccb97> a activity:Point ;\n" +
        "    activity:actor <http://fortytwo.net/josh/things/JdGwZ4n> ;\n" +
        "    activity:thingIndicated dbr:The_Meaning_of_Liff ;\n" +
        "    activity:recognitionTime <urn:uuid:a4a2fd8c-ea0d-43bb-bcad-6510f4c9b55a> .\n" +
        "\n" +
        "<urn:uuid:a4a2fd8c-ea0d-43bb-bcad-6510f4c9b55a> a tl:Instant ;\n" +
        "    tl:at \"2015-02-13T21:00:12-05:00\"^^xsd:dateTime .";

// Instantiate the query engine.
SparqlStreamProcessor streamProcessor = new SHJSparqlStreamProcessor();

// Define a time-to-live for the query. It will expire after this many seconds,
// freeing up resources and ceasing to match statements.
int queryTtl = 10 * 60;

// Define a handler for answers to the query.
BiConsumer<BindingSet, Long> handler = new BiConsumer<BindingSet, Long>() {
    @Override
    public void accept(final BindingSet answer, Long expirationTime) {
        System.out.println("found an answer to the query: " + answer + ", expires at " + expirationTime);
    }
};

// Submit the query to the query engine to obtain a subscription.
Subscription sub = streamProcessor.addQuery(queryTtl, query, handler);

// Create subscriptions for additional queries at any time; queries match in parallel

// Add some data with infinite (= 0) time-to-live.
// Results derived from this data will never expire.
int staticTtl = 0;

// Add some static background knowledge.  Alternatively, let Stream42 discover this
// information as Linked Data (see LinkedDataExample.java).
Statement st = new StatementImpl(
        new URIImpl("http://dbpedia.org/resource/The_Meaning_of_Liff"),
        new URIImpl("http://dbpedia.org/ontology/author"),
        new URIImpl("http://dbpedia.org/resource/Douglas_Adams"));
streamProcessor.addInputs(staticTtl, st);

// Now define a finite time-to-live of 30 seconds.
// This will be used for the short-lived data of gesture events.
int eventTtl = 30;

RDFFormat format = RDFFormat.TURTLE;
RDFParser parser = Rio.createParser(format);
parser.setRDFHandler(streamProcessor.createRDFHandler(eventTtl));
// As new statements are added, computed query answers will be pushed to the BindingSetHandler.
parser.parse(new ByteArrayInputStream(eventData.getBytes()), "");

// Cancel the query subscription at any time;
// no further answers will be computed/produced for the corresponding query.
sub.cancel();

// Alternatively, renew the subscription for another 10 minutes.
sub.renew(10 * 60);
```

See also the [Linked Data example](https://github.com/joshsh/stream42/blob/master/stream42-examples/src/main/java/net/fortytwo/stream/examples/LinkedDataExample.java); here, we replace the above "hard-coded" background semantics with discovered information which the query engine proactively [fetches from the Web](https://github.com/joshsh/ripple/wiki/LinkedDataSail):

```java
// Create a Linked Data client.  The Sesame triple store will be used for
// managing caching metadata, while the retrieved Linked Data will be fed into the continuous
// query engine, which will trigger the dereferencing of URIs in response to join operations.
MemoryStore sail = new MemoryStore();
sail.initialize();
LinkedDataCache cache = LinkedDataCache.createDefault(sail);
queryEngine.setLinkedDataCache(cache);
```

For projects which use Maven, Stream42 snapshots and release packages can be imported by adding configuration like the following to the project's POM:

```xml
    <dependency>
        <groupId>net.fortytwo.stream</groupId>
        <artifactId>stream42-sparql</artifactId>
        <version>1.1</version>
    </dependency>
```

The latest Maven packages can be browsed [here](http://search.maven.org/#search%7Cga%7C1%7Csesamestream).
See also:
* [Stream42 API](http://fortytwo.net/projects/sesamestream/api/latest/index.html)

Send questions or comments to:

![Josh email](http://fortytwo.net/Home_files/josh_email.jpg)


### Syntax reference

SPARQL syntax currently supported by Stream42 includes:
* [SELECT](http://www.w3.org/TR/sparql11-query/#select) queries.  SELECT subscriptions in Stream42 produce query answers indefinitely unless cancelled.
* [ASK](http://www.w3.org/TR/sparql11-query/#ask) queries.  ASK subscriptions produce at most one query answer (indicating a result of **true**) and then are cancelled automatically, similarly to a SELECT query with a LIMIT of 1.
* [CONSTRUCT](http://www.w3.org/TR/sparql11-query/#construct) queries.  Each query answer contains "subject", "predicate", and "object" bindings which may be turned into an RDF statement.
* [basic graph patterns](http://www.w3.org/TR/sparql11-query/#BasicGraphPatterns)
* [variable projection](http://www.w3.org/TR/sparql11-query/#modProjection)
* all [RDF Term syntax](http://www.w3.org/TR/sparql11-query/#syntaxTerms) and [triple pattern](http://www.w3.org/TR/sparql11-query/#QSynTriples) syntax via Sesame
* [FILTER](http://www.w3.org/TR/sparql11-query/#tests) constraints, with all SPARQL [operator functions](http://www.w3.org/TR/sparql11-query/#SparqlOps) supported via Sesame **except for** [EXISTS](http://www.w3.org/TR/sparql11-query/#func-filter-exists)
* [DISTINCT](http://www.w3.org/TR/sparql11-query/#modDuplicates) modifier.  Use with care if the streaming data source may produce an unlimited number of solutions.
* [REDUCED](http://www.w3.org/TR/sparql11-query/#modDuplicates) modifier.  Similar to DISTINCT, but safe for long streams.  Each subscription maintains a solution set which begins to recycle after it reaches a certain size, configurable with `SparqlStreamProcessor.setReducedModifierCapacity()`.
* [LIMIT](http://www.w3.org/TR/sparql11-query/#modResultLimit) clause.  Once LIMIT number of answers have been produced, the subscription is cancelled.
* [OFFSET](http://www.w3.org/TR/sparql11-query/#modOffset) clause.  Since query answers roughly follow the order in which input statements are received, OFFSET can be practically useful even without ORDER BY (see below)

Syntax explicitly not supported:
* [ORDER BY](http://www.w3.org/TR/sparql11-query/#modOrderBy).  This is a closed-world operation which requires a finite data set or window; Stream42 queries over a stream of data and an infinite window.
* SPARQL 1.1 [aggregates](http://www.w3.org/TR/sparql11-query/#aggregates).  See above

Syntax not yet supported:
* [DESCRIBE](http://www.w3.org/TR/sparql11-query/#describe) query form
* [OPTIONAL](http://www.w3.org/TR/sparql11-query/#optionals) and [UNION](http://www.w3.org/TR/sparql11-query/#alternatives) patterns, [group graph patterns](http://www.w3.org/TR/sparql11-query/#GroupPatterns)
* [RDF Dataset syntax](http://www.w3.org/TR/sparql11-query/#rdfDataset), i.e. the FROM, FROM NAMED, and GRAPH keywords
* SPARQL 1.1's [NOT](http://www.w3.org/TR/sparql11-query/#negation), [Property Paths](http://www.w3.org/TR/sparql11-query/#propertypaths), [assignment](http://www.w3.org/TR/sparql11-query/#assignment) (BIND / AS / VALUES), [subqueries](http://www.w3.org/TR/sparql11-query/#subqueries)
* SPARQL 1.1 [Federated Query](http://www.w3.org/TR/sparql11-federated-query/) syntax

