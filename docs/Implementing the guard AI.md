In this tutorial, we will create a basic AI for a guard. By the end of the session, it will be able to patrol between a set of points when idle, keep an eye out for monsters and go to them if one is nearby and finally attack until the monster is dead:

About the 3D scene
In this tutorial, I’ll be using various free assets from the Asset Store or from online texture databases. Here are the links if you want more info, or if you want to download them yourself:
the guard model (+ animations): https://assetstore.unity.com/packages/3d/characters/toon-rts-units-demo-69687
the skeleton model (+ animations): https://assetstore.unity.com/packages/3d/characters/creatures/dungeon-skeletons-demo-71087
the ground tiles texture: https://ambientcg.com/view?id=Tiles089
the wood crate: https://opengameart.org/content/crate-2

<img width="1200" height="672" alt="image" src="https://github.com/user-attachments/assets/c4dc8550-21fa-4dfe-bd08-e670bf34d28e" />

The guard has a basic animator that has three states: idle, walking and attacking. Each state has its own animation and the transitions between those animator states are controlled by two boolean variables: “Walking” and “Attacking”:

<img width="1200" height="393" alt="image" src="https://github.com/user-attachments/assets/7ab75192-684c-49ae-9a05-f9c139f56fad" />

The skeleton just has a single animation in its animator, it has a BoxCollider and it is on a user-custom layer called “Enemy”:

<img width="400" height="743" alt="image" src="https://github.com/user-attachments/assets/5d0dda76-96cb-4822-8dca-8cf69df18ec2" />

This physics layer will help optimise the checks for nearby enemy units in our code ;)

<img width="800" height="456" alt="image" src="https://github.com/user-attachments/assets/2f3e5919-9e04-4e64-8c4e-6446f4d61dd4" />

Now, it’s time to dive into the programming of our behaviour tree! ;)

About the behaviour tree
To implement the behaviour tree, we will work in two phases:

first, we’ll define some generic architecture that could be used by any behaviour tree, along with a few composite nodes
then, we’ll focus more specifically on our guard and design its particular behaviour tree structure and the nodes inside it
Preparing a generic behaviour tree architecture
First, let’s prepare our generic behaviour tree architecture.

The Node class

To begin with, we’ll work on our atomic element: the Node. Let’s create a new script folder called BehaviorTree/ and, inside it, create a new C# script: Node.cs - we'll open it and remove the default Unity Start(), Update() and MonoBehaviour. Also, to make this tool more re-usable, we can declare the generic classes inside a BehaviorTree namespace:

using System.Collections;
using System.Collections.Generic;

namespace BehaviorTree
{

    public class Node
    {
    }

}

Our Node class will be a basic C# class that represents a single element in the tree and can access both its children and its parent, has a node state that uses an enum and can store, retrieve or clear shared data.

Let’s start with the state. We need an enum with 3 possible values: RUNNING, SUCCESS or FAILURE. Then, in the class, we’ll declare a state for the node using this enum. We’ll make it protected so that classes that derive from this Node class can access and modify this value:


using System.Collections;
using System.Collections.Generic;

namespace BehaviorTree
{
    public enum NodeState
    {
        RUNNING,
        SUCCESS,
        FAILURE
    }

    public class Node
    {
        protected NodeState state;
    }

}

Similarly, we can make a public parent variable and a protected list of children. Having a link in both directions is interesting because it will make it easier to create our composite nodes (by looking at the children) and to have shared data (by looking at the parent and backtracking in the branch):

public class Node
{
    protected NodeState state;
    
    public Node parent;
    protected List<Node> children = new List<Node>();

    These children will be assigned in the constructor of the class, or else they’ll just be empty; by default, the empty constructor will simply assign a null parent node. To be sure we properly link the parent field when we create our tree, we’ll rely on a util method called _Attach() that properly creates the edge between a node and its new child, and call it from the Node constructor, like this:

    public class Node
{
    protected NodeState state;
    
    public Node parent;
    protected List<Node> children = new List<Node>();
    
    public Node()
    {
        parent = null;
    }
    public Node(List<Node> children)
    {
        foreach (Node child in children)
            _Attach(child);
    }

    private void _Attach(Node node)
    {
        node.parent = this;
        children.Add(node);
    }
}

Now, we can prepare the prototype of the Evaluate() function – it will be virtual so that each derived-Node class can implement its own evaluation function and have a unique role in the behaviour tree:

public class Node
{
    // ...
    
    public virtual NodeState Evaluate() => NodeState.FAILURE;
}

To finish up our class, we need to take care of the shared data. This data will be stored in a dictionary of string-object pairs: by using the C# object lazy type, we essentially have a mapping of named variables that can be of any type and that is therefore pretty agnostic to the data you want to store:

public class Node
{
    // ...
    
    private Dictionary<string, object> _dataContext =
        new Dictionary<string, object>();
}

To set the data, we just need to add a key in the dict:

public class Node
{
    // ...
    
    public void SetData(string key, object value)
    {
        _dataContext[key] = value;
    }
}

To get it back, it’s a bit more complex because we want to check if it’s defined somewhere in our branch — not just in this particular node. This will make it easier to access and use shared data in our behaviour tree. To do this we’ll make the GetData() function recursive and continue working up the branch until we’ve either found the key we were looking for or reached the root of the tree:


public class Node
{
    // ...
    
    public object GetData(string key)
    {
        object val = null;
        if (_data.TryGetValue(key, out val))
            return val;

        Node node = _parent;
        if (node != null)
            val = node.GetData(key);
        return val;
    }
}

To clear data, the process is pretty much the same: we recursively search for the key and, if we find it, then we remove it from the dict. Else if we reach the root we just ignore the request.

public class Node
{
    // ...
    
    public bool ClearData(string key)
    {
        bool cleared = false;
        if (_data.ContainsKey(key))
        {
            _data.Remove(key);
            return true;
        }

        Node node = _parent;
        if (node != null)
            cleared = node.ClearData(key);
        return cleared;
    }
}

The Tree class

Now that we have a Node class, we can go ahead and implement the second core object of our package: the Tree class. This other C# script will be a MonoBehaviour and it will contain a reference to a root node that, itself, recursively contains the entire tree.

It just needs to do two things: first, upon Start(), the Tree class will build the behaviour tree according to the SetupTree() function we defined; then, in the Update(), if it has a tree, it will evaluate it continuously:

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace BehaviorTree
{
    public abstract class Tree : MonoBehaviour
    {

        private Node _root = null;

        protected void Start()
        {
            _root = SetupTree();
        }

        private void Update()
        {
            if (_root != null)
                _root.Evaluate();
        }

        protected abstract Node SetupTree();

    }

}

Preparing util nodes

Finally, to end this first part on the implementation of a generic behaviour tree, let’s prepare two composite nodes: the Sequence and the Selector.

The Sequence is a composite that acts like an "and" logic gate: only if all child nodes succeed will this node succeed itself.

To implement it, we just have to derive from the Node class and override its Evaluate() method. In our overridden evaluation function, we will iterate through the children and check their state after evaluation. If any child fails, then we can stop there and return a failed state too. Else, we need to keep on processing the children, and eventually check if some are running (and therefore block us in the running state too) or if all have succeeded:


using System.Collections.Generic;

namespace BehaviorTree
{
    public class Sequence : Node
    {
        public Sequence() : base() { }
        public Sequence(List<Node> children) : base(children) { }

        public override NodeState Evaluate()
        {
            bool anyChildIsRunning = false;

            foreach (Node node in children)
            {
                switch (node.Evaluate())
                {
                    case NodeState.FAILURE:
                        state = NodeState.FAILURE;
                        return state;
                    case NodeState.SUCCESS:
                        continue;
                    case NodeState.RUNNING:
                        anyChildIsRunning = true;
                        continue;
                    default:
                        state = NodeState.SUCCESS;
                        return state;
                }
            }

            state = anyChildIsRunning ? NodeState.RUNNING : NodeState.SUCCESS;
            return state;
        }

    }

}

The Selector node, on the other hand, is like an "or" logic gate. It’s almost the same code as the Sequence, except that we return early when a child has succeeded or is running:

using System.Collections.Generic;

namespace BehaviorTree
{
    public class Selector : Node
    {
        public Selector() : base() { }
        public Selector(List<Node> children) : base(children) { }

        public override NodeState Evaluate()
        {
            foreach (Node node in children)
            {
                switch (node.Evaluate())
                {
                    case NodeState.FAILURE:
                        continue;
                    case NodeState.SUCCESS:
                        state = NodeState.SUCCESS;
                        return state;
                    case NodeState.RUNNING:
                        state = NodeState.RUNNING;
                        return state;
                    default:
                        continue;
                }
            }

            state = NodeState.FAILURE;
            return state;
        }

    }

}

