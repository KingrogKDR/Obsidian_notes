
> Edge case: if root is null, we should return null, otherwise we will get null pointer exceptions
# Invert Binary Trees

- Three ways to solve this:

1. BFS :  _O(n)_ _O(n)_
	- Initialize a queue with the root
	- Then until the queue is empty:
		- Remove the front element from the queue
		- Swap its children
		- Add the (new) left and right children to the queue if they exist
	- Return the root
	
2. DFS (using recursive stack): _O(n)_ _O(n)_
	- Swap the children of the root
	- Then recursively do the same for the left and right subtrees
	
3. Iterative DFS: _O(n)_ _O(n)_
	- Initialize a stack with the root
	- Then until the queue is empty:
		- Pop the top element
		- Swap its children
		- Push the (new) left and right children to the stack if they exist
	- Return the root

## Real Usecases:

- Mirroring a UI or visual hierarchy like in visual editors, game/graphics scene tree, etc.
- Expression trees for commutative operations

# Maximum Depth of a Tree

- Three ways to solve this:

1. DFS: _O(n)_ _O(n)_
	- 
