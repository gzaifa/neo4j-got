# Exploring and analysing the Game Of Thrones dataset using visualisation powered by vis.js
Using information from epic fantasy series for graph exploration and analysis seems to be really popular with the data science community, with many posts on Game of Thrones, Lord of the Rings, and Star Wars. I thought it would be fun to follow some of the posts along with GoT data, even if the finale of GoT television series was disappointing to say the least.
The visualisation capabilities of [neovis.js](https://github.com/neo4j-contrib/neovis.js/) is certainly more impactful, flexible (configuration options), and performant than provided in Neo4j's desktop browser.  
![image](https://user-images.githubusercontent.com/830693/128681138-e215f79e-060b-4ce3-aa9a-a4aeb48126b3.png)
[HTML](https://github.com/gzaifa/neo4j-got/blob/main/got-community.html) available.
Compared to the same cypher query in Neo4j browser:
![image](https://user-images.githubusercontent.com/830693/128680880-1ad985e8-07c2-4692-b90d-095d910c2cc9.png)
  
Some of the features listed for neovis.js used in the visualisation:
1. Connect to Neo4j instance to get live data
2. User specified labels and property to be displayed
3. User specified Cypher query to populate
4. Specify edge property for edge thickness
5. Specify node property for community / clustering
6. Specify node property for node size

But first, as usual, we start by cleaning and importing the data to the database.

## Step 1: Import Data
We will be using data made public by GitHub user mathbeveridge, and presented by [Andrew Beveridge](https://networkofthrones.wordpress.com/) and Jie Shan in their article [Network of Thrones](http://www.maa.org/sites/default/files/pdf/Mathhorizons/NetworkofThrones%20%281%29.pdf) which was published in Math Horizons magazine.
As the data is very much smaller than the yelp dataset which I had explored recently, it should be relatively fast to use the [LOAD CSV](https://neo4j.com/developer/guide-import-csv/) cypher command directly from the browser command line.  
The files are in CSV format with header "Source,Target,Type,weight,book":
![image](https://user-images.githubusercontent.com/830693/128684189-ccf6e93e-3d4f-4019-bb6d-1fed2e229030.png)
Since the Type is all "undirected", we will ignore it. Before importing, it is a good practise to create any unique constraints first as we certainly do not want multiple nodes of the same person appearing in our graph and the corresponding index create will hopefully make the merge commands run faster.
<pre>CREATE CONSTRAINT ON (p:Person) ASSERT p.id IS UNIQUE</pre>

<pre>UNWIND ['1','2','3','45'] as book
LOAD CSV WITH HEADERS FROM 
'https://raw.githubusercontent.com/mathbeveridge/asoiaf/master/data/asoiaf-book' + book + '-edges.csv' as value
MERGE (source:Person{id:value.Source})
MERGE (target:Person{id:value.Target})
WITH source,target,value.weight as weight,book
CALL apoc.merge.relationship(
    source,
    'INTERACTS_' + book, 
    {}, 
    {weight:toFloat(weight)}, 
    target) YIELD relationship
RETURN distinct 'done'</pre>

## Step 2: Identify persons of interest
A node is said to be pivotal if it lies on all shortest paths between two other nodes in the network. We can find the top pivotal nodes using:
<pre>MATCH (a:Person), (b:Person) WHERE id(a) > id(b)
MATCH p=allShortestPaths((a)-[:INTERACTS_1*]-(b)) WITH collect(p) AS paths, a, b
MATCH (c:Person) WHERE all(x IN paths WHERE c IN nodes(x)) AND NOT c IN [a,b]
WITH collect(c.id) AS pivotal_nodes
UNWIND pivotal_nodes as node
RETURN node, COUNT(node) AS num ORDER BY num DESC
</pre>
The top results are the usual suspects in book 1 with Daenerys-Targaryen not very high on the list:
|"node"                           |"num"|
|:---|---: |
|"Eddard-Stark"                   |2731 |
|"Tyrion-Lannister"               |2549 |
|"Jon-Snow"                       |2157 |
|"Catelyn-Stark"                  |1628 |
|"Robert-Baratheon"               |1530 |
|"Drogo"                          |731  |
|"Daenerys-Targaryen"             |560  |
|"Walder-Frey"                    |549  |
|"Benjen-Stark"                   |382  |
  
In book 4 and 5, the Mother of Dragons (MoT) have shot to the top of the list with fan favorite Jon Snow.
|"node"                            |"num"|
|:---|---: |
|"Jon-Snow"                        |18688|
|"Daenerys-Targaryen"              |18038|
|"Cersei-Lannister"                |15859|
|"Stannis-Baratheon"               |14354|
|"Jaime-Lannister"                 |12309|
|"Tyrion-Lannister"                |9885 |
|"Asha-Greyjoy"                    |7466 |
|"Arya-Stark"                      |7419 |
|"Theon-Greyjoy"                   |6208 |
|"Victarion-Greyjoy"               |5860 |
  
[Degree](https://en.wikipedia.org/wiki/Degree_(graph_theory)) of a node is the number of connections (or edges) that it has in the network. In this graph, the edges are the interaction between the characters. We can derive the degree using the following cypher:
<pre>MATCH (c:Person)
RETURN c.id AS person, size( (c)-[:INTERACTS_1]-() ) AS degree ORDER BY degree DESC
</pre>
|"person"                                 |"degree"|
|:---|---: |
|"Eddard-Stark"                           |66      |
|"Robert-Baratheon"                       |50      |
|"Tyrion-Lannister"                       |46      |
|"Catelyn-Stark"                          |43      |
|"Jon-Snow"                               |37      |
|"Robb-Stark"                             |35      |
|"Sansa-Stark"                            |35      |
|"Bran-Stark"                             |32      |
|"Cersei-Lannister"                       |30      |  
  
Dear MoD does not even feature in the top 10 in book 1 and is overshadowed by the Lannisters in book 4 and 5:   
|"person"                                 |"degree"|
|:---|---: |
|"Jaime-Lannister"                        |67      |
|"Cersei-Lannister"                       |66      |
|"Jon-Snow"                               |65      |
|"Daenerys-Targaryen"                     |58      |
|"Stannis-Baratheon"                      |57      |
|"Tyrion-Lannister"                       |52      |
|"Theon-Greyjoy"                          |35      |
|"Brienne-of-Tarth"                       |29      |
|"Sansa-Stark"                            |26      |
|"Barristan-Selmy"                        |26      |
