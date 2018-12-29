# twitter-graph
Sample code of an example graph database implementation of Twitter using the [The-Don-Himself/gremlin-ogm](https://github.com/The-Don-Himself/gremlin-ogm) library.

## Create A Schema

Note: Not all gremlin compatible databases support schemas, in the event you are using one just skip this part.

First we deal with vertexes (a few places I called them vertices, similar to indices and indexes)

### Vertexes

````
<?php

namespace TheDonHimself\TwitterGraph\Graph\Vertices;

use JMS\Serializer\Annotation as Serializer;
use TheDonHimself\GremlinOGM\Annotation as Graph;

/**
 *  @Serializer\ExclusionPolicy("all")
 *  @Graph\Vertex(
 *      label="tweets",
 *      indexes={
 *          @Graph\Index(
 *              name="byTweetsIdComposite",
 *              type="Composite",
 *              unique=true,
 *              label_constraint=true,
 *              keys={
 *                  "tweets_id"
 *              }
 *          ),
 *          @Graph\Index(
 *              name="tweetsMixed",
 *              type="Mixed",
 *              label_constraint=true,
 *              keys={
 *                  "tweets_id"       : "DEFAULT",
 *                  "text"            : "TEXT",
 *                  "retweet_count"   : "DEFAULT",
 *                  "created_at"      : "DEFAULT",
 *                  "favorited"       : "DEFAULT",
 *                  "retweeted"       : "DEFAULT",
 *                  "source"          : "STRING"
 *              }
 *          )
 *      }
 *  )
 */
class Tweets
{
    /**
     * @Serializer\Type("integer")
     * @Serializer\Expose
     * @Serializer\Groups({"Default"})
     */
    public $id;

    /**
     * @Serializer\VirtualProperty
     * @Serializer\Expose
     * @Serializer\Type("integer")
     * @Serializer\Groups({"Graph"})
     * @Serializer\SerializedName("tweets_id")
     * @Graph\Id
     * @Graph\PropertyName("tweets_id")
     * @Graph\PropertyType("Long")
     * @Graph\PropertyCardinality("SINGLE")
     */
    public function getVirtualId()
    {
        return self::getId();
    }

    /**
     * @Serializer\Type("string")
     * @Serializer\Expose
     * @Serializer\Groups({"Default", "Graph"})
     * @Graph\PropertyName("text")
     * @Graph\PropertyType("String")
     * @Graph\PropertyCardinality("SINGLE")
     */
    public $text;

    /**
     * @Serializer\Type("integer")
     * @Serializer\Expose
     * @Serializer\Groups({"Default", "Graph"})
     * @Graph\PropertyName("retweet_count")
     * @Graph\PropertyType("Integer")
     * @Graph\PropertyCardinality("SINGLE")
     */
    public $retweet_count;

    /**
     * @Serializer\Type("boolean")
     * @Serializer\Expose
     * @Serializer\Groups({"Default", "Graph"})
     * @Graph\PropertyName("favorited")
     * @Graph\PropertyType("Boolean")
     * @Graph\PropertyCardinality("SINGLE")
     */
    public $favorited;

    /**
     * @Serializer\Type("boolean")
     * @Serializer\Expose
     * @Serializer\Groups({"Default", "Graph"})
     * @Graph\PropertyName("retweeted")
     * @Graph\PropertyType("Boolean")
     * @Graph\PropertyCardinality("SINGLE")
     */
    public $retweeted;

    /**
     * @Serializer\Type("DateTime<'', '', 'D M d H:i:s P Y'>")
     * @Serializer\Expose
     * @Serializer\Groups({"Default", "Graph"})
     * @Graph\PropertyName("created_at")
     * @Graph\PropertyType("Date")
     * @Graph\PropertyCardinality("SINGLE")
     */
    public $created_at;

    /**
     * @Serializer\Type("string")
     * @Serializer\Expose
     * @Serializer\Groups({"Default", "Graph"})
     * @Graph\PropertyName("source")
     * @Graph\PropertyType("String")
     * @Graph\PropertyCardinality("SINGLE")
     */
    public $source;

    /**
     * @Serializer\Type("TheDonHimself\TwitterGraph\Graph\Vertices\Users")
     * @Serializer\Expose
     * @Serializer\Groups({"Default"})
     */
    public $user;

    /**
     * @Serializer\Type("TheDonHimself\TwitterGraph\Graph\Vertices\Tweets")
     * @Serializer\Expose
     * @Serializer\Groups({"Default"})
     */
    public $retweeted_status;

    /**
     * Get id.
     *
     * @return int
     */
    public function getId()
    {
        return $this->id;
    }
}
````

### Edges

And an edge

````
<?php

namespace TheDonHimself\TwitterGraph\Graph\Edges;

use JMS\Serializer\Annotation as Serializer;
use TheDonHimself\GremlinOGM\Annotation as Graph;

/**
 *  @Serializer\ExclusionPolicy("all")
 *  @Graph\Edge(
 *      label="follows",
 *      multiplicity="MULTI"
 *  )
 */
class Follows
{
    /**
     *  @Graph\AddEdgeFromVertex(
     *      targetVertex="users",
     *      uniquePropertyKey="users_id",
     *      methodsForKeyValue={"getUserVertex1Id"}
     *  )
     */
    protected $userVertex1Id;

    /**
     *  @Graph\AddEdgeToVertex(
     *      targetVertex="users",
     *      uniquePropertyKey="users_id",
     *      methodsForKeyValue={"getUserVertex2Id"}
     *  )
     */
    protected $userVertex2Id;

    public function __construct($user1_vertex_id, $user2_vertex_id)
    {
        $this->userVertex1Id = $user1_vertex_id;
        $this->userVertex2Id = $user2_vertex_id;
    }

    /**
     * Get User 1 Vertex ID.
     *
     *
     * @return int
     */
    public function getUserVertex1Id()
    {
        return $this->userVertex1Id;
    }

    /**
     * Get User 2 Vertex ID.
     *
     *
     * @return int
     */
    public function getUserVertex2Id()
    {
        return $this->userVertex2Id;
    }
}
````

The beauty of this library is that it only helps you write gremlin commands but does not stop you from interfacing with Gremlin directly, for example as in the case of the Follows Edge above, the library will produce gremlin commands to create an edge between two vertexes if you can pass to it a unique identifier such as user_id, house_id, taxi_id, etc. If you want to added an edge by other ways you can simply write a gremlin command and submit it directly through $graph_connection->send(' my awesome gremlin command ;');

The follows edges class is really simple, in that it simply creates an edge linking two vertexes by user_id, in real life examples you'd probably create the edge but have added properties like followed_on, via_app, introduced_by and so on. Just add those properties to the class and let the library serialize them for you.


### Create Schema

When creating vertex and edge classes, look at the code from \TheDonHimself\TwitterGraph\Commands,
They include;

SchemaCheckCommand;
SchemaCreateCommand;
PopulateCommand;
VertexesCountCommand;
VertexesDropCommand;
EdgesCountCommand;
EdgesDropCommand;
GremlinTraversalCommand;

SchemaCheckCommand runs some checks to ensure that you did not duplicate names of properties and labels or indexes while SchemaCreateCommand actually iterates through you graph classes and send gremlin commands to create them. PopulateCommand populates the graph with data either from an API as with the case of the sample TwitterGraph or from a databases if you use Doctrine ORM (RDBMS) and/or Doctrine ODM (MongoDB). GremlinTraversalCommand let you send a gremlin command through the CLI e.g php bin/graph twittergraph:gremlin:traversal --traversal="g.V().count()".

### Traverse The Graph

The library is almost a seamless transition from the Gremlin API. The most important thing here is the TraversalBuilder from \TheDonHimself\Traversal\TraversalBuilder which returns ready to execute gremlin commands, for example to get back a users vertex from Twitter you can build a Traversal as follows

````
use TheDonHimself\GremlinOGM\Traversal\TraversalBuilder;
....

$user_id = 12345;

$traversalBuilder = new TraversalBuilder();

$command = $traversalBuilder
  ->g()
  ->V()
  ->hasLabel("'users'")
  ->has("'users_id'", "$user_id")
  ->getTraversal();

return $command;
````

Take special note of the single and double quotes

Echoing this command will show you this
````
g.V().hasLabel('users').has('users_id', 12345)
````

If you want to use bindings in the case of script parameterization (highly recommended) you can do this.


````
use TheDonHimself\GremlinOGM\Traversal\TraversalBuilder;
....

$user_id = 12345;

$traversalBuilder = new TraversalBuilder();

$command = $traversalBuilder
  ->raw('def b = new Bindings(); ')
  ->g()
  ->V()
  ->hasLabel("'users'")
  ->has("'users_id'", "b.of('user_id', $user_id)")
  ->getTraversal();

return $command;
````

Again please take special note of the single and double quotes

Echoing this command will show you this
````
def b = new Bindings(); g.V().hasLabel('users').has('users_id', b.of('user_id', 12345))
````

Re: check possible traversal steps in at the code from \TheDonHimself\Traversal\Step,


Let' get a little bit more complex now, fetching a user's feed

````
$screen_name = 'my_username';

$traversalBuilder = new TraversalBuilder();

$command = $traversalBuilder
  ->g()
  ->V()
  ->hasLabel("'users'")
  ->has("'screen_name'", "'$screen_name'")
  ->union(
    (new TraversalBuilder())->out("'tweeted'")->getTraversal(),
    (new TraversalBuilder())->out("'follows'")->out("'tweeted'")->getTraversal()
  )
  ->order()
  ->by("'created_at'", 'decr')
  ->limit(10)
  ->getTraversal();

return $command;
````

Echoing this command will show you this
````
g.V().hasLabel('users').has('screen_name', 'my_username').union(out('tweeted'), out('follows').out('tweeted')).order().by('created_at', decr).limit(10)
````

That's it for now, there is so much more that this simple library can do, please look in the sample TwitterGraph folder to quickly get started with a sample graph of your Twitter friends, followers, likes, tweets and retweets by running this command. The library comes with a preconfigured readonly Twitter App for this.


## Tests

Currently, I've not written any test suites but you can test the library by using a sample Twitter Graph that comes preconfigured with this library. Only the following Graph Databases have been tested to work though will test more when I get the time/resources

- [x] Azure Cosmos DB
- [x] JanusGraph on Compose
- [x] JanusGraph Self-Hosted

Simple configure any of them in their respective yaml files in the root folder then execute the following

````
php bin/graph twittergraph:schema:create
````

then 

````
php bin/graph twittergraph:populate
````

**Azure Cosmos DB**

Please Note: Schema create command not applicable for CosmosDB

example:

````
php bin/graph twittergraph:populate

Populate the Twitter Graph with Data
====================================

 Enter the path to a yaml configuration file or use defaults (JanusGraph, 127.0.0.1:8182 with ssl, no username or password):
 > \path\to\azure-cosmosdb.yaml

 The Twitter Username to Populate:
 > The_Don_Himself

 Perform a Dry Run [false]:
 >

Twitter User @The_Don_Himself Found
Twitter ID : 622225192
Creating Vertexes...
Done! 338 Vertexes Created
Creating Edges...
Done! 367 Edges Created
Graph Populated Successfully!

````


**JanusGraph on Compose**

example:

````
php bin/graph twittergraph:populate

Populate the Twitter Graph with Data
====================================

 Enter the path to a yaml configuration file or use defaults (JanusGraph, 127.0.0.1:8182 with ssl, no username or password):
 > \path\to\janusgraph-compose.yaml

 The Twitter Username to Populate:
 > The_Don_Himself

 Perform a Dry Run [false]:
 >

Twitter User @The_Don_Himself Found
Twitter ID : 622225192
Creating Vertexes...
Done! 338 Vertexes Created
Creating Edges...
Done! 367 Edges Created
Graph Populated Successfully!

````


**JanusGraph Self-Hosted**

example:

````
php bin/graph twittergraph:populate

Populate the Twitter Graph with Data
====================================

 Enter the path to a yaml configuration file or use defaults (JanusGraph, 127.0.0.1:8182 with ssl, no username or password):
 > \path\to\janusgraph.yaml

 The Twitter Username to Populate:
 > The_Don_Himself

 Perform a Dry Run [false]:
 >

Twitter User @The_Don_Himself Found
Twitter ID : 622225192
Creating Vertexes...
Done! 338 Vertexes Created
Creating Edges...
Done! 367 Edges Created
Graph Populated Successfully!

````

## GraphQL

You might also be interested in [graphql2gremlin](https://github.com/The-Don-Himself/graphql2gremlin), an attempt to create a standard around transforming GraphQL queries to Gremlin Traversals.
