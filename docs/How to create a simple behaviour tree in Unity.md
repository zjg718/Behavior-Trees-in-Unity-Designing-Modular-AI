How to create a simple behaviour tree in Unity/C#
What are behaviour trees?
Another common AI pattern that is way more flexible is the behaviour tree.

Here, rather than defining a finite set of states that your character can transition between, you define a tree of nodes that, altogether, create branches with various behaviours.
<img width="800" height="312" alt="image" src="https://github.com/user-attachments/assets/1d1ed80d-53ab-4241-936a-ba22c425adc2" />
Each node has an execution state that determines whether it’s currently running, if it’s succeeded or if it’s failed. This state is computed by “evaluating” the node thanks to a function naturally called Evaluate(). The state of the branches is then computed from the state of the children nodes and aggregated in various ways, depending on the parent node type.
<img width="400" height="577" alt="image" src="https://github.com/user-attachments/assets/107258e8-2049-40b4-b71f-938710c35864" />
The nodes at the very bottom that have no children are called leaves. Those leaves are going to contain the actual checks and tasks performed by our AI while the rest of the tree is the logic structure that’s going to essentially “switch” from one behaviour branch to another. In other words: the inner nodes control the flow and the leaves ultimately implement the actions.

<img width="600" height="529" alt="image" src="https://github.com/user-attachments/assets/db9188db-7ae1-4237-a9b9-f326341f3cd4" />
Flow-control nodes

The “flow-control” nodes in the middle define how the tree is processed; they can be of two types:

composites: those nodes are sequencers that have a bunch of child nodes that they run in predetermined or random order, one after the other or in parallel. Usually, they act like an “and” or an “or” logical gate, for example.
decorators: in that case, they have a single child and they may impact its processing or modify its result before returning from the branch (like reversing the failure/success state, delaying the execution, and so on)
Action nodes

Action nodes are more diverse but you can usually group them in two categories: the checks and the tasks. Checks are simple if-like nodes that condition whether or not this branch is valid. Tasks are more complex nodes that perform a real update of the character or its environment.

Why use behaviour trees?
Something really nice with behaviour trees is that prioritising actions is easy: because the success of the child nodes conditions how the branch and therefore the tree runs, combining composite “and”s and “or”s can directly give you a set of prioritised features. They natively handle sequences, interruptions and fallbacks.

Also, as you can see, this structure is pretty easy to read and visualise — you just follow the branches by priority and, at each node, check whether it succeeds or not to know if you can continue on this branch or must go to the next one.

But the biggest advantage of behaviour trees is that they are highly modular and scalable: by removing or adding a branch, you essentially add or remove a feature to your character; similarly, it’s pretty simple to copy a sub-branch and paste it somewhere else to copy-paste part of your behaviour. Some sub-trees can even be completely self-contained autonomous bits of logic that you can then re-import and re-use in another character’s behaviour tree to quickly implement chunks of a system (like following a target, patrolling between a set of points, et caetera).

Moreover, this modularity means that you can build your tree piece by piece and have an incremental design strategy.

Finally, note that contrary to finite state machines where, by definition, states are as isolated from each other as possible, behaviour trees are far more flexible to leaking information between nodes. Even though each node is executed on its own and lives its own life, it is pretty common to have some shared data state that can be accessed from various points in your tree to have global (although context-specific) variables.


