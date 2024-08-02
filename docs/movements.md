# Function

**Determines the target of the movement and the movement speed of the agent.**

## MoveTo.cs

### Initialize()

Initializes an Agent object and sets the movement speed using the parameter ```speed```. An initialized agent is not pathfinding or moving by default.

    public void Initialize(float speed)
    {
        agent = GetComponent<NavMeshAgent>();
        agent.avoidancePriority = Random.Range(30, 60);
        agent.speed = speed;
        Pathfinding = false;
    }

### Start()

Calls initializer and sets the movement speed of the Agent.

    private void Start()
    {
        Initialize(1.25f);
    }

### SetTarget()

Sets the target goal that the Agent will move to. 

Goal object is set through parameter ```goal``` and the Agent destination is mapped to the position of the goal object. The Agent then transitions to an active pathfinding state and will begin to move. The ```stairs``` parameter denotes if there are any stairs in the environment, and therefore dictates multi-level movement for the Update() function.

    public void SetTarget(Transform goal, bool stairs)
        {
            this.goal = goal;
            agent.destination = goal.position;
            agent.isStopped = false;
            Pathfinding = true;
            climbing = stairs;
        }

### Update()

Checks if the target position is reached while the Agent is in an active pathfinding state.

Using the x,z coordinates of the goal position and the agent position, we calculate the distance between the two objects. If the distance is less than 0.5 units, we stop the agent's movements and take it out of its active pathfinding state.

If there is a multi-level environment as specified in SetTarget(), we incorporate the y value into the distance calculation. With this calculation, if the distance is less than 0.7 units we cease the agent's movements and take it out of its active pathfinding state.

    private void Update()
    {
        if (Pathfinding)
        {
            var g = goal.position;
            var t = transform.position;
            if (climbing)
            {
                var temp = Vector3.Distance(new Vector3(g.x, g.y, g.z), new Vector3(t.x, t.y, t.z));
                if (Vector3.Distance(new Vector3(g.x, g.y, g.z), new Vector3(t.x, t.y, t.z)) < 0.7f)
                {
                    agent.isStopped = true;
                    Pathfinding = false;
                }
            }
            else
            {
                if (Vector3.Distance(new Vector3(g.x, 0, g.z), new Vector3(t.x, 0, t.z)) < 0.5f)
                {
                    agent.isStopped = true;
                    Pathfinding = false;
                }
            }
        }
    }

### abort()

Stops the Agent and pauses pathfinding efforts.

    public void abort()
        {
            agent.isStopped = true;
            Pathfinding = false;
        }
