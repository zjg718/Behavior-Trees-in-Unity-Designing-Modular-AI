We now have a toolbox for our behaviour trees that will make it easier to create our own specific AI system for this guard.

We will build this custom tree gradually and add more actions as we progress.

Step 1: “Patrolling” — a single-node tree

So our very first behaviour tree will be real simple — it will have only one node: the TaskPatrol node. There won’t be any internal node for flow control, because for now the guard will always execute the same action: patrolling. 

<img width="350" height="242" alt="image" src="https://github.com/user-attachments/assets/a98c0f9f-76c9-4128-a781-565043b91306" />

First, in another scripts subfolder (GuardAI/), let’s create a new C# class called TaskPatrol, import our BehaviorTree package and have TaskPatrol inherit from the Node class.

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

using BehaviorTree;

public class TaskPatrol : Node
{
    
}

Again, we’ll want to override the Evaluate() function, but also the constructor.

That’s because our patrolling algorithm requires some additional info, like the waypoints to use, but also a reference to the transform of the agent performing this action. Since TaskPatrol is not a MonoBehaviour but a somewhat abstract blob of data, we need to pass it the transform to use.

We will store these variables in the class as private members and assign them in the constructor:

public class TaskPatrol : Node
{
    private Transform _transform;
    private Transform[] _waypoints;

    public TaskPatrol(Transform transform, Transform[] waypoints)
    {
        _transform = transform;
        _waypoints = waypoints;
    }

    public override NodeState Evaluate()
    {
    }

}

Of course, I need to update and return the state of my evaluation function. This node will just be running continuously, again and again, so I’ll just put this in the method:

public class TaskPatrol : Node
{
    // ...

    public override NodeState Evaluate()
    {
        state = NodeState.RUNNING;
        return state;
    }

}

Then, let’s simply copy back the code we had in the patrolling tutorial.

I need to replace the few missing variables with their new equivalent — so instead of the transform, I’ll use _transform; same thing for waypoints:


public class TaskPatrol : Node
{
    // ...
    private int _currentWaypointIndex = 0;

    private float _waitTime = 1f; // in seconds
    private float _waitCounter = 0f;
    private bool _waiting = false;
    
    // ...

    public override NodeState Evaluate()
    {
        if (_waiting)
        {
            _waitCounter += Time.deltaTime;
            if (_waitCounter >= _waitTime)
            {
                _waiting = false;
            }
        }
        else
        {
            Transform wp = _waypoints[_currentWaypointIndex];
            if (Vector3.Distance(_transform.position, wp.position) < 0.01f)
            {
                _transform.position = wp.position;
                _waitCounter = 0f;
                _waiting = true;

                _currentWaypointIndex = (_currentWaypointIndex + 1) % _waypoints.Length;
            }
            else
            {
                _transform.position = Vector3.MoveTowards(
                    _transform.position,
                    wp.position,
                    ? * Time.deltaTime);
                _transform.LookAt(wp.position);
            }
        }
    
        state = NodeState.RUNNING;
        return state;
    }

}

For the speed, I’d like this to be defined at a higher-level so that all the nodes in my tree that need this variable can find and share it easily. So I’ll just make it a public static float in my behaviour tree itself.

This behaviour tree will be defined in a GuardBT C# class that inherits from our Tree class.

This is where we’ll expose a public field for the waypoints (that we’ll then pass on to the TaskPatrol node) where we’ll set our global speed variable and where we’ll define the structure of our tree, in the SetupTree() method.

using System.Collections.Generic;
using BehaviorTree;

public class GuardBT : Tree
{
    public UnityEngine.Transform[] waypoints;

    public static float speed = 2f;

    protected override Node SetupTree()
    {}
}

Don’t forget we can now use the GuardBT.speed in our TaskNode class:

public class TaskPatrol : Node
{
    // ...

    public override NodeState Evaluate()
    {
        if (_waiting)
        { ... }
        else
        {
            Transform wp = _waypoints[_currentWaypointIndex];
            if (Vector3.Distance(_transform.position, wp.position) < 0.01f)
            { ... }
            else
            {
                _transform.position = Vector3.MoveTowards(
                    _transform.position,
                    wp.position,
                    GuardBT.speed * Time.deltaTime);
                _transform.LookAt(wp.position);
            }
        }
        // ...
    }

}

All that’s left to do is, back in our GuardBT, to design the tree structure. For now, this tree is just a single root that uses our new TaskPatrol class:


public class GuardBT : Tree
{
    // ...

    protected override Node SetupTree()
    {
        Node root = new TaskPatrol(transform, waypoints);
        return root;
    }
}

I can finally put this component on my guard and assign the waypoints:

<img width="500" height="588" alt="image" src="https://github.com/user-attachments/assets/fd39f9cd-377f-44e5-a327-809d7806fe13" />

Now, you see that when I run the game, the guard goes from one waypoint to the other with a little wait at each.
<img width="720" height="360" alt="image" src="https://github.com/user-attachments/assets/f4eb019b-92d6-4615-9649-3f789a6ced03" />

Of course, we’d like for the model to use its walk animation in-between the waypoints. So let’s go back to our TaskPatrol and, when we get the transform, also get a reference to the animator on the same object. Then, in the Evaluate() function, we’ll set the “Walking” boolean to true or false when we finish waiting or when we reach a new waypoint:

public class TaskPatrol : Node
{
    // ...
    private Animator _animator;

    public TaskPatrol(Transform transform, Transform[] waypoints)
    {
        // ...
        _animator = transform.GetComponent<Animator>();
    }

    public override NodeState Evaluate()
    {
        if (_waiting)
        {
            _waitCounter += Time.deltaTime;
            if (_waitCounter >= _waitTime)
            {
                _waiting = false;
                _animator.SetBool("Walking", true);
            }
        }
        else
        {
            Transform wp = _waypoints[_currentWaypointIndex];
            if (Vector3.Distance(_transform.position, wp.position) < 0.01f)
            {
                // ...
                _animator.SetBool("Walking", false);
            }
            else
            { ... }
        }

        // ...
    }

}

The guard now walks through its patrol path!

<img width="720" height="360" alt="image" src="https://github.com/user-attachments/assets/44af6689-8951-44d8-94f7-14c9883a15b9" />

Step 2: “Targeting” — checking for nearby colliders

Now, let’s ramp things up a little and add some behaviour to spot nearby enemies and go to a target once it’s been defined. This time, this action will require two nodes:

first, a CheckEnemyInFOVRange node that returns a success if there is at least one enemy close by, in the field of vision radius, or else a failure
second, a TaskGoToTarget node that moves the guard towards the target that has been found
These actions will be inside a Sequence because we only want to move to a target if we spotted one. Then, overall, we’ll want either this Sequence or the basic TaskPatrol to be used, so we’ll need a new Selector node as the root to link those two little branches together:

<img width="800" height="412" alt="image" src="https://github.com/user-attachments/assets/641ecf7c-ce52-432f-8422-30b2dd473f16" />

To keep a reference to this target enemy, we’ll use our shared data system: the CheckEnemyInFOVRange, if it finds an enemy, will store this target in the root of the tree so that the TaskGoToTarget (and perhaps other nodes later on...) can retrieve it and use it in its computation.

This new class will again inherit from the Node class and require the transform in its constructor.


using System.Collections;
using System.Collections.Generic;
using UnityEngine;

using BehaviorTree;

public class CheckEnemyInFOVRange : Node
{
    private Transform _transform;

    public CheckEnemyInFOVRange(Transform transform)
    {
        _transform = transform;
    }

    public override NodeState Evaluate()
    {}

}

In the overridden Evaluate() function, we’ll check to see if we already have a target, by using GetData(). If we have one, then we succeed automatically.

public class CheckEnemyInFOVRange : Node
{
    // ...

    public override NodeState Evaluate()
    {
        object t = GetData("target");
        if (t == null)
        {
            
        }

        state = NodeState.SUCCESS;
        return state;
    }

}

Else, we need to do a little check of our surroundings, using something like Unity’s Physics.OverlapSphere(), that returns all the colliders that are found inside a sphere around a given position. We’ll limit this search to just the enemy layer, take our position as the center of the sphere and define a new static variable in the tree, the fovRange, that we’ll use here:

public class GuardBT : Tree
{
    // ...
    public static float fovRange = 6f;
}

public class CheckEnemyInFOVRange : Node
{
    // ...
    private static int _enemyLayerMask = 1 << 6;
    
    // ...

    public override NodeState Evaluate()
    {
        object t = GetData("target");
        if (t == null)
        {
            Collider[] colliders = Physics.OverlapSphere(
                _transform.position, GuardBT.fovRange, _enemyLayerMask);
        }

        // ...
    }

}

Then, if we did find at least one collider, we’ll store the first collider in our list in the target slot of our shared data, in the root (which is two levels above, that's why I use parent.parent) and return a success.

Else, we’ll fail on this node and stop the processing of this branch there.

Just to be sure, we can also make sure that the walk animation is playing by getting a reference to our animator and setting the “Walking” boolean to true.

public class CheckEnemyInFOVRange : Node
{
    // ...
    private Animator _animator;
    
    public CheckEnemyInFOVRange(Transform transform)
    {
        // ...
        _animator = transform.GetComponent<Animator>();
    }

    public override NodeState Evaluate()
    {
        object t = GetData("target");
        if (t == null)
        {
            Collider[] colliders = Physics.OverlapSphere(
                _transform.position, GuardBT.fovRange, _enemyLayerMask);

            if (colliders.Length > 0)
            {
                parent.parent.SetData("target", colliders[0].transform);
                _animator.SetBool("Walking", true);
                state = NodeState.SUCCESS;
                return state;
            }

            state = NodeState.FAILURE;
            return state;
        }

        // ...
    }

}

The TaskGoToTarget class is pretty straight-forward to code, based on the previous nodes we prepared. We just get our transform, try and get a target from our target data slot and, if we have one, move towards it. Also, we return a continuous running state.

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

using BehaviorTree;

public class TaskGoToTarget : Node
{
    private Transform _transform;

    public TaskGoToTarget(Transform transform)
    {
        _transform = transform;
    }

    public override NodeState Evaluate()
    {
        Transform target = (Transform)GetData("target");

        if (Vector3.Distance(_transform.position, target.position) > 0.01f)
        {
            _transform.position = Vector3.MoveTowards(
                _transform.position,
                target.position,
                GuardBT.speed * Time.deltaTime);
            _transform.LookAt(target.position);
        }

        state = NodeState.RUNNING;
        return state;
    }

}

To finish this up, let’s use our new nodes in the GuardBT script. We’ll replace our root with a Selector and pass in two children: first, the Sequence with the check and the move task; and then our TaskPatrol node from before:

public class GuardBT : Tree
{
    // ...

    protected override Node SetupTree()
    {
        Node root = new Selector(new List<Node>
        {
            new Sequence(new List<Node>
            {
                new CheckEnemyInFOVRange(transform),
                new TaskGoToTarget(transform),
            }),
            new TaskPatrol(transform, waypoints),
        });

        return root;
    }
}

Remember that the order in here is important: this gives the list of priorities for your actions and, in our case, sets patrolling as the fallback.

If you run the game again, you’ll see that, now, when the guard gets close enough from the enemy, it interrupts its patrol and move towards the target!

<img width="720" height="360" alt="image" src="https://github.com/user-attachments/assets/51cc7248-6fca-4301-bc6e-03afcd6bbab0" />

Step 3: “Attacking” — one more branch!

The last thing we want to implement is a basic attack behaviour. When the guard gets really close to an enemy and this enemy enters its attack range, then the guard should switch to attack mode and regularly deal some damage to the enemy. Again, we won’t have the enemy react in any way in this tutorial but you see we could easily use the same techniques to create a behaviour tree for this unit too :)

The attack Sequence will be pretty similar to the targeting one: we’ll have a check node for the attack range and then a task node for actually attacking.

<img width="800" height="335" alt="image" src="https://github.com/user-attachments/assets/37e8e0ff-0ee1-4fbe-bde1-d88d3f70e032" />

However, we’ll assume that the target has already been found and filled in when we try to attack, so the CheckEnemyInAttackRange routine will not look for colliders – it will just check the target shared data slot.

If it doesn’t find a target, it fails. Else, if the enemy is close enough, it will succeed. We’ll also want to change the guard animation to attack, so I’ll take my animator and set the “Attacking” flag to true and the “Walking” flag to false.

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

using BehaviorTree;

public class CheckEnemyInAttackRange : Node
{
    private Transform _transform;
    private Animator _animator;

    public CheckEnemyInAttackRange(Transform transform)
    {
        _transform = transform;
        _animator = transform.GetComponent<Animator>();
    }

    public override NodeState Evaluate()
    {
        object t = GetData("target");
        if (t == null)
        {
            state = NodeState.FAILURE;
            return state;
        }

        Transform target = (Transform)t;
        if (Vector3.Distance(_transform.position, target.position) <= GuardBT.attackRange)
        {
            _animator.SetBool("Attacking", true);
            _animator.SetBool("Walking", false);

            state = NodeState.SUCCESS;
            return state;
        }

        state = NodeState.FAILURE;
        return state;
    }

}

Basically, the EnemyManager script defines an initial amount of healthpoints for the enemy and then has some methods to decrease this amount by a fixed amount or, when the healthpoints reach zero, have the enemy "die" and its game object disappear.

To get the reference to this script, we’ll use Unity’s GetComponent(). But because GetComponent() is not efficient, I will cache this result for as long as my target stays the same.

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

using BehaviorTree;

public class TaskAttack : Node
{
    private Transform _lastTarget;
    private EnemyManager _enemyManager;

    public override NodeState Evaluate()
    {
        Transform target = (Transform)GetData("target");
        if (target != _lastTarget)
        {
            _enemyManager = target.GetComponent<EnemyManager>();
            _lastTarget = target;
        }

        state = NodeState.RUNNING;
        return state;
    }

}

Then, I’ll just have a little counter, like in my TaskPatrol, to attack every second:



public class TaskAttack : Node
{
    // ...
    private float _attackTime = 1f;
    private float _attackCounter = 0f;

    public override NodeState Evaluate()
    {
        Transform target = (Transform)GetData("target");
        if (target != _lastTarget)
        { ... }
        
        _attackCounter += Time.deltaTime;
        if (_attackCounter >= _attackTime)
        {
            _enemyManager.TakeHit();
            _attackCounter = 0f;
        }

        state = NodeState.RUNNING;
        return state;
    }

}

The TakeHit() method from my EnemyManager script returns a boolean that tells me if this blow was the final one and if the enemy is now dead. In that case, I want to clear my target data slot and switch back to walking, to return to my patrol path:

public class TaskAttack : Node
{
    // ...
    
    private Animator _animator;
    
    public TaskAttack(Transform transform)
    {
        _animator = transform.GetComponent<Animator>();
    }

    public override NodeState Evaluate()
    {
        // ...
        
        _attackCounter += Time.deltaTime;
        if (_attackCounter >= _attackTime)
        {
            bool enemyIsDead = _enemyManager.TakeHit();
            if (enemyIsDead)
            {
                ClearData("target");
                _animator.SetBool("Attacking", false);
                _animator.SetBool("Walking", true);
            }
            else
            {
                _attackCounter = 0f;
            }
        }
        
        // ...
    }

}

We can add this new branch in our tree at the very beginning so that it has the highest priority:

public class GuardBT : Tree
{
    // ...
    public static float attackRange = 1f;

    protected override Node SetupTree()
    {
        Node root = new Selector(new List<Node>
        {
            new Sequence(new List<Node>
            {
                new CheckEnemyInAttackRange(transform),
                new TaskAttack(transform),
            }),
            new Sequence(new List<Node>
            {
                new CheckEnemyInFOVRange(transform),
                new TaskGoToTarget(transform),
            }),
            new TaskPatrol(transform, waypoints),
        });

        return root;
    }
}

And if we run this, we see that the guard now patrols, spots the enemy, moves towards it, attacks until the skeleton is down and finally resumes patrolling as its fallback behaviour!

<img width="720" height="405" alt="image" src="https://github.com/user-attachments/assets/a74af145-2a13-4d58-a1cd-2a9e75af8e23" />


