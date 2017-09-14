---
layout: post
title:  "Creating Match, Return, Order By clauses From a PathTree"
date:   2017-07-02 08:00:00
comments: true
categories: gsoc
tags:
    - intermine
    - neo4j
    - PathQuery
    - gsoc
    - Cypher
    - conversion
    - database
---

In this post I'll describe in detail, how I created Match, Return and Order By clauses of Cypher using a PathTree. PathTree was discussed in the [previous post](/blog/2017/path-query-cypher-puzzle-part-2/).

## Description of Clauses

### Match

Match is the very first clause in every Cypher query. It allows you to specify the patterns Neo4j will search for in the database. These patterns must be such that the data which matches them, must be the data you wish to operate (or query) on. Consider an example Match clause which matches three *Genes* such that two of them  *INTERACTS_WITH* the third one.

{% highlight cypher %}
MATCH (a:Gene)-[:INTERACTS_WITH]->(b:Gene)<-[:INTERACTS_WITH]-(c:Gene)
{% endhighlight %}

The pattern above is a 2-dimensional pattern which we often have to deal with, in Cypher. Although, it is easier to imagine the pattern in 2-D, we generally need to break them down into multiple 1-D patterns while writing cypher queries. Also, 1-D patterns would be easier to generate programmatically than their 2-D counterparts. For example, the following Match clause with two 1-D patterns is equivalent to the one shown above.

{% highlight cypher %}
MATCH (a:Gene)-[:INTERACTS_WITH]->(b:Gene),
(c:Gene)-[:INTERACTS_WITH]->(b)
{% endhighlight %}

Note that we could simply use the variable name `b` in the second pattern without specifying the Label *Gene*. It doesn't seem very significant here but it makes generating queries very convinient while dealing with some complicated paths. Suppose you need to specify a Path/Node/Relationship again and again in the query, wouldn't you be comfortable in just writing its variable name and get done with it?

### Return 

In the RETURN part of the cypher query, we define the parts of the pattern in which we are interested. It can be nodes, relationships, or properties on these.

{% highlight cypher %}
MATCH (a:Gene {primaryIdentifier:'WBGene00007063'})-[r:INTERACTS_WITH]->(b:Gene)
// Following are some possbile return statements,
RETURN b
RETURN r
RETURN b.primaryIdentifier
RETURN r.dataSet
{% endhighlight %}

### Order By 

ORDER BY is a sub-clause following RETURN, and it specifies that the output should be sorted and how. For example,

{% highlight cypher %}
MATCH (a:Gene {primaryIdentifier:'WBGene00007063'})-[r:INTERACTS_WITH]->(b:Gene)
RETURN b
ORDER BY b.symbol
{% endhighlight %}

For more information, please refer to [Neo4j Developer Manual](http://neo4j.com/docs/developer-manual/current/cypher/).

## Assigning variable names to TreeNodes

We see that variable names are crucial in a Cypher query. A PathTree represents all the paths in the corresponding PathQuery. Since we can have a constraint/view/order on any path, therefore we need to assign variable name to each node of the PathTree.

![A Path Tree](/images/PathTree.png)

We cannot just use the component name as the variable name. This is because a component can be repeated in the same path. For example, in `Gene.chromosome.gene.length`, `gene` appears twice. Having a unique variable name for each TreeNode is crucial to generating the Cypher query.

To create the variable names, I have simply separated components in the path using an underscore instead of a dot. Also, I have converted the path string to its lower case form. For example, the TreeNode for the path `Gene.homologues.dataSets.name` will have variable name `gene_homologues_datasets_name`. This approach is fine till we don't exceed the max length of variable name for a path.

#### TreeNode Description

A TreeNode for the path `Gene.chromosome.gene.length`, stores the following information.

- `type` : Whether this TreeNode represents a Graphical Node or Relationship or a Property on them.
- `name` : The last component of the path, i.e. length.
- `variable name` : The one we generated using approach shown above, i.e. gene_chromosome_gene_length.
- `graphical name` : This is the name by which we refer this entity/property in the InterMine Neo4j graph. Here we keep it same as `name`, i.e. length.

## Clause Creation

#### RETURN

The return clause always starts with the *RETURN* keyword. After that, for each view in the PathQuery, we add an expression separated by commas. The expression consists of the variable name and the respective graphical name.

{% highlight java %}
/**
 * Creates RETURN clause of the Cypher Query using the Path Query and Path Tree
 *
 * @param cypherQuery     the Cypher Query object
 * @param pathTree  the given PathTree
 * @param pathQuery the given PathQuery
 */
private static void createReturnClause(CypherQuery cypherQuery, PathTree pathTree, PathQuery pathQuery) {
    for (String path : pathQuery.getView()) {
        TreeNode treeNode = pathTree.getTreeNode(path);
        if (treeNode != null && treeNode.getTreeNodeType() == TreeNodeType.PROPERTY) {
            // Add to return clause ONLY IF the TreeNode represents a property !!
            // Nodes & Relationships cannot be returned.
            cypherQuery.addToReturn(treeNode.getParent().getVariableName()
                                    + "."
                                    + treeNode.getGraphicalName());
        }
    }
}
{% endhighlight %}

#### ORDER BY

The order by clause always starts with the *ORDER BY* keyword. After that, we simply append the variable name and the respective graphical name for each `sortOrder's` path of the PathQuery. The sort type (Ascending/Descending) is also added.

{% highlight java %}
/**
 * Generates Order objects from the Path Query and adds them to the Cypher Query.
 * These will be used to create the Order By clause within the CypherQuery class.
 *
 * @param cypherQuery     the Cypher Query object
 * @param pathTree  the given PathTree
 * @param pathQuery the given PathQuery
 */
private static void addOrdersToCypher(CypherQuery cypherQuery, PathTree pathTree, PathQuery pathQuery) {
    List<OrderElement> orderElements = pathQuery.getOrderBy();
    for (OrderElement orderElement : orderElements) {
        Order order = new Order(orderElement, pathTree);
        cypherQuery.addOrder(order);
    }
}
{% endhighlight %}

#### Match

Creation of Match clause is rather complex. The match clause always starts with the *MATCH* keyword. After that, for each edge in the PathTree we write two nodes and the relationship between them. If two adjacent TreeNodes represent Graph Nodes, then we use a dummy relationship to join them. Otherwise we add the relationship represented by the TreeNode in between.

The `createMatchClause()` recursive method takes in the `Query` object and the root `TreeNode` as parameters. The following code snippet shows the method in action.

{% highlight java %}
/**
 * Creates MATCH clause of the Cypher Query using the given PathTree
 *
 * @param cypherQuery    the Cypher Query object
 * @param treeNode the root node of the PathTree
 */
private static void createMatchClause(CypherQuery cypherQuery, TreeNode treeNode) throws IOException, ModelParserException, SAXException, ParserConfigurationException {
    if (treeNode == null) {
        throw new IllegalArgumentException("Root node of PathTree cannot be null.");
    }
    else if (treeNode.getTreeNodeType() == TreeNodeType.PROPERTY) {
        // Properties don't need to be matched in the Match statement of cypher.
        return;
    }
    else if (treeNode.getParent() == null) {
        // Root TreeNode is always a Graph Node
        cypherQuery.addToMatch("(" + treeNode.getVariableName() +
                         " :" + treeNode.getGraphicalName() + ")");
    }
    else {
        String match = null;
        if (treeNode.getTreeNodeType() == TreeNodeType.NODE) {

            Neo4jModelParser modelParser = new Neo4jModelParser();
            modelParser.process(new Neo4jLoaderProperties());
            TreeNode parentTreeNode = treeNode.getParent();

            if (parentTreeNode.getTreeNodeType() == TreeNodeType.NODE) {
                // If current TreeNode is a NODE and its parent is also a NODE, then use
                // ModelParser.getRelationshipType to fetch the Relationship Type from the
                // XML data model file.
                String className = parentTreeNode.getPath().getEndClassDescriptor().getSimpleName();
                String refName = treeNode.getName();

                String relationshipType = modelParser.getRelationshipType(className, refName);
                match = "(" +
                        parentTreeNode.getVariableName() +
                        ")-[:" +
                        relationshipType +
                        "]-(" +
                        treeNode.getVariableName() +
                        " :" +
                        treeNode.getGraphicalName() +
                        ")";
            }
            else if (parentTreeNode.getTreeNodeType() == TreeNodeType.RELATIONSHIP) {
                // If current TreeNode is a NODE and its parent is a RELATIONSHIP, then fetch
                // the grand parent from the PathTree. Use relationship described by the parent
                // TreeNode between the grand parent and the current TreeNode.
                match = "(" +
                        parentTreeNode.getParent().getVariableName() +
                        ")-[" +
                        parentTreeNode.getVariableName() +
                        ":" +
                        parentTreeNode.getGraphicalName() +
                        "]-(" +
                        treeNode.getVariableName() +
                        " :" +
                        treeNode.getGraphicalName() +
                        ")";
            }
        }
        else if (treeNode.getTreeNodeType() == TreeNodeType.RELATIONSHIP) {
            // If the current TreeNode is a RELATIONSHIP and it has no children, then
            // use this relationship between the NODE represented by the parent TreeNode
            // and an empty node. For example, (m:Gene)-[r:LOCATED_ON]-().
            if (treeNode.getChildrenKeys().isEmpty()) {
                TreeNode parentTreeNode = treeNode.getParent();
                match = "(" +
                        parentTreeNode.getVariableName() +
                        ")-[" +
                        treeNode.getVariableName() +
                        ":" +
                        treeNode.getGraphicalName() +
                        "]-()";
            }
        }
        if (match != null) {
            // If string is not null then any of the above 3 cases has occurred,
            // so we must add it to MATCH or OPTIONAL MATCH clause as required.
            if (treeNode.getOuterJoinStatus() == OuterJoinStatus.INNER) {
                cypherQuery.addToMatch(match);
            }
            else {
                cypherQuery.addToOptionalMatch(match);
            }
        }
    }

    // Recursively add all the children TreeNodes to the Match/Optional Match clause
    for (String key : treeNode.getChildrenKeys()) {
        createMatchClause(cypherQuery, treeNode.getChild(key));
    }
}
{% endhighlight %}

In the next post, the generation of [Where](http://neo4j.com/docs/developer-manual/current/cypher/clauses/where/) clause will be covered. It is most complex of all because it involves converting around 30+ PathQuery contraints into their equivalend Cypher expressions.
