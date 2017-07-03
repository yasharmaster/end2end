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
private static void createReturnClause(Query query, PathTree pathTree, PathQuery pathQuery) {
    for (String path : pathQuery.getView()) {
        TreeNode treeNode = pathTree.getTreeNode(path);
        if (treeNode.getTreeNodeType() == TreeNodeType.PROPERTY) {
            // Return ONLY IF a property is queried !!
            query.addToReturn(treeNode.getParent().getVariableName() + 
        					"." + 
        					treeNode.getGraphicalName());
        }
    }
}
{% endhighlight %}

#### ORDER BY

The order by clause always starts with the *ORDER BY* keyword. After that, we simply append the variable name and the respective graphical name for each `sortOrder's` path of the PathQuery. The sort type (Ascending/Descending) is also added.

{% highlight java %}
// The method that creates the order by clause
private static void createOrderByClause(Query query, PathTree pathTree, PathQuery pathQuery) {
    List<OrderElement> orderElements = pathQuery.getOrderBy();
    for (OrderElement orderElement : orderElements) {
        Order order = new Order(orderElement, pathTree);
        query.addToOrderBy(order.toString());
    }
}

// Constructor of the order class
Order(OrderElement orderElement, PathTree pathTree){
    TreeNode treeNode = pathTree.getTreeNode(orderElement.getOrderPath());
    propertyKey = treeNode.getGraphicalName();
    // Store variableName of parent TreeNode
    variableName = treeNode.getParent().getVariableName();
    direction = orderElement.getDirection().toString();
}

// String representation of the order class object
public String toString() {
    return variableName + "." +
            propertyKey + " " +
            direction;
}
{% endhighlight %}

#### Match

Creation of Match clause is rather complex. The match clause always starts with the *MATCH* keyword. After that, for each edge in the PathTree we write two nodes and the relationship between them. If two adjacent TreeNodes represent Graph Nodes, then we use a dummy relationship to join them. Otherwise we add the relationship represented by the TreeNode in between.

The `createMatchClause()` recursive method takes in the `Query` object and the root `TreeNode` as parameters. The following code snippet shows the method in action.

{% highlight java %}
private static void createMatchClause(Query query, TreeNode treeNode) {
    if (treeNode == null) {
        return;
    }
    else if (treeNode.getParent() == null) {
        // Root TreeNode is always a Graph Node
        query.addToMatch("(" + treeNode.getVariableName() +
                         " :" + treeNode.getGraphicalName() + ")");
    }
    else if (treeNode.getTreeNodeType() == TreeNodeType.NODE) {
        if (treeNode.getParent().getTreeNodeType() == TreeNodeType.NODE) {
            // If current TreeNode is a Graph Node and its parent is also a Graph Node,
            // then add a dummy relationship.
            query.addToMatch("(" + treeNode.getParent().getVariableName() + ")" +
                            "-[]-(" + treeNode.getVariableName() +
                            " :" + treeNode.getGraphicalName() + ")");
        }
        else if (treeNode.getParent().getTreeNodeType() == TreeNodeType.RELATIONSHIP) {
            // If current TreeNode is a Graph Node and its parent is a Graph Relationship,
            // then match an actual relationship of the current node with its grand parent node.
            query.addToMatch("(" + treeNode.getParent().getParent().getVariableName() + ")" +
                            "-[" + treeNode.getParent().getVariableName() +
                            ":" + treeNode.getParent().getGraphicalName() + "]" +
                            "-(" + treeNode.getVariableName() +
                            " :" + treeNode.getGraphicalName() + ")");
        }
    }
    // If current TreeNode represents a Graphical Relationship, then Do nothing.
    // We will match this relationship when recursion reaches its children.

    // Add all children to Match clause
    for (String key : treeNode.getChildrenKeys()) {
        createMatchClause(query, treeNode.getChild(key));
    }
}
{% endhighlight %}

In the next post, the generation of [Where](http://neo4j.com/docs/developer-manual/current/cypher/clauses/where/) clause will be covered. It is most complex of all because it involves converting around 30+ PathQuery contraints into their equivalend Cypher expressions.
