+++
title="Graph"
date=2021-01-18

[taxonomies]
categories = ["Data Structure"]
tags = ["post","graph"]
+++


an entity that hold data is called **node** and a line connect two nodes is called **edge**.  
### Category

##### Undirected Graph: no directions  
> example: facebook, linkedin  

##### Directed Graph: asymmetric relationship  
> example: teacher and student, employer and employee  

##### Weighted Graph: edges represent values  
> example: distance, cost, similarity  

### Representing Graph
##### Adjacency matrix:  
    implementation: multi-dimension array
    pros: easy to implement
    cons: take long time to check all the edges that conect two node.
##### Adjacency list:  
    implementation: list of vector/liked list 
    pros: it only stores the value of connected nodes. number of elements of a vector are equal to number of edges
    cons: while adjacency matrix takes O(1) time complexity to check wheather two node has relationship, adjacency list takes O(n) time complexity. n is number of elements a vector has.





Reference  
https://sarah950716.tistory.com/12  
https://www.mygreatlearning.com/blog/representing-graphs-in-data-structures/