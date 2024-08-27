# Function

Controls the smart agent and enables complex movement through processing, decision making, and movement execution.

## SmartAgentController.cs

### InitializeAgent()

Initializes the Agent, its attributes, and the interactions needed for it move. Sets the variables needed for state management and agent decision making. Creates trackers to record visual data and location and then initalizes movement.

    public void initializeAgent()
    {
        sm = GetComponent<StateMachine>();
        target = new GameObject("target").transform;
        subtarget = new GameObject("subtarget").transform;

        graphs = FindObjectsOfType<NavigationalGraph>();
        dps = FindObjectsOfType<DecisionPoint>();
        visitedDps = new List<DecisionPoint>();

        seenSigns = new List<Transform>();

        speed = Random.Range(1.1f, 1.2f);
        fov = GetComponent<FieldOfView>();

        stairsDoors = GameObject.FindGameObjectsWithTag("StairsDoor");

        lastSeenRoom = 0;
        beforeLastSeenRoom = 0;

        timeSinceSign = 0;

        coords = new List<TimeStampedCoords>();
        tracker = StartCoroutine(TrackAgent());

        destDoor.parseRoomNumber();
        fov.initializeValues();
        GetComponent<MoveTo>().Initialize(speed / 2);
        initializeMemory();
        if (GetComponent<PathWriter>() != null)
            GetComponent<PathWriter>().startWritePath();
    }

### Start()

Calls Initialize() to start the process.

    public void Start()
        {
            initializeAgent();
        }

### initializeMemory()

Initializes path memory by creating two dictionaries that log decisions made at decision points (dpMemory) and graph edges passed (stMemory).

    private void initializeMemory()
        {
            foreach (var graph in graphs)
            {
                foreach (var node in graph.graph.Vertices)
                {
                    stMemory[node + "-" + node] = new Dictionary<string, int> { [dest] = 0 };
                }
                foreach (var edge in graph.graph.Edges)
                {
                    stMemory[edge.Source + "-" + edge.Destination] = new Dictionary<string, int> { [dest] = 0 };
                    stMemory[edge.Destination + "-" + edge.Source] = new Dictionary<string, int> { [dest] = 0 };
                }
            }
            foreach (var dp in dps)
            {
                dpMemory[dp.id] = new Dictionary<string, int>();
                foreach (var sub in dp.outgoingVectors)
                {
                    dpMemory[dp.id].Add(sub.name, 0);
                }
            }

        }

### SetCentroid()

Sets the mid point for the agent.

    public void SetCentroid(GameObject centroid)
    {
        midPt = centroid;
    }

### scanEnvironmentWithDelay()

Sets the delay and then calls environment and sign scanning.

    private IEnumerator scanEnvironmentWithDelay(float delay)
        {
            while (true)
            {
                yield return new WaitForSeconds(delay);
                scanEnvironment();
                scanSignsTime();
            }
        }

### scanEnvironment()

Scans the environment comprehensively for decision making. Will first scan the floor if it has not been checked yet and then checks if stairs are visible and aborts action if floor must be changed. If destination is not visible, it will abort action. Will also override previous decision not based on signs if a useful sign is scanned.

        private void scanEnvironment()
            {
                if (!sm.floorChecked)
                    checkFloor();

                if (sm.floorShift != 0)
                {
                    if (checkVisibleStairs())
                    {
                        GetComponent<MoveTo>().abort();
                        sm.routeSelected = false;
                        sm.stairsVisible = true;
                    }
                    return;
                }

                if (!sm.destVisible && checkVisibleDestinations())
                {
                    GetComponent<MoveTo>().abort();
                    sm.isInDP = false;
                    sm.destVisible = true;
                    return;
                }

                if (fov.visibleSigns.Count > 0)
                {
                    foreach (var sign in fov.visibleSigns)
                    {
                        if (!seenSigns.Contains(sign))
                        {
                            seenSigns.Add(sign);
                            if (sm.randomDecision)
                            {
                                Tuple<Transform, Transform> result = processSeenSigns(); 
                                if (result.Item2 != null)
                                {
                                    GetComponent<MoveTo>().abort();
                                    target.position = result.Item2.position;
                                    target.name = result.Item2.name;
                                    sm.routeSelected = true;
                                    sm.randomDecision = false;
                                    Debug.Log("Random decision aborted. Sign " + result.Item1.GetComponent<Sign>().id + " indicates " + result.Item2.name);
                                }
                            }
                        }
                    }
                }
            }

### explore()

Moves agent following isovist and checks if inside decision nodes. Sets rotation ability and controls agent rotation when it becomes stuck.

    public void explore()
    {
        if (midPt == null) return;
        transform.rotation = Quaternion.RotateTowards(transform.rotation,
                Quaternion.LookRotation((midPt.transform.position - transform.position).normalized), maxDegreesDelta);
        transform.Translate(speed * Time.deltaTime * Vector3.forward);

        foreach (var dp in dps)
        {
            if (Vector3.Distance(transform.position, dp.transform.position) <= dp.radius && Math.Abs(dp.transform.position.y - transform.position.y) <= floorDiffThres)
            {
                sm.isInDP = true;
                sm.current_dp = dp;
                target.position = dp.transform.position;
                target.name = dp.transform.name;
                var sub_dps = dp.GetComponentsInChildren<Transform>();
                closestDir = sub_dps.OrderBy(x => Vector3.Distance(transform.position, x.position)).FirstOrDefault(x => x != sub_dps[0]);
                dpMemory[dp.id][closestDir.name]++;
            }
        }

        if (isov.Area < framesStuckArea)
        {
            framesStuck++;
            if (framesStuck > framesStuckMax)
            {
                transform.rotation = transform.rotation * Quaternion.AngleAxis(180, Vector3.up);
                framesStuck = 0;
            }
        }
        else
        {
            framesStuck = 0;
        }
    }

### checkFloor()

Checks if the current floor must be changed. If the destination door’s floor is unknown, it opens access to stairs. If the destination door is on a different floor and no floor shift has been set yet, it calculates and stores the floor shift needed to reach the target floor.

    public void checkFloor()
    {
        sm.floorChecked = true;

        if (destDoor.GetComponent<Door>().room.floor == "")
        {
            foreach (GameObject door in stairsDoors)
            {
                door.SetActive(false);
            }
            return;
        }
        if (sm.floorShift == 0 && Floors[destDoor.GetComponent<Door>().room.floor] != curr_floor)
        {

            sm.floorShift = Floors[destDoor.GetComponent<Door>().room.floor] - curr_floor;
        }
        return;

    }

### checkVisibleDestinations()

Checks rooms to decide destination visibility and if agent needs to turn around. Iterates through existing doors to find the destination door number. Returns true if there is a visible match, otherwise the agent prepares to look at the nearest door and returns false.

    public bool checkVisibleDestinations()
    {
        Transform closestDoor = null;
        if (fov.visibleDestinations.Count > 0)
        {
            float minDist = Vector3.Distance(fov.visibleDestinations[0].position, transform.position);
            foreach (var door in fov.visibleDestinations)
            {
                Vector3 doorBox = door.GetComponent<BoxCollider>().center;
                Vector3 colPos = new Vector3(doorBox.x, doorBox.z, -doorBox.y); // Door position in Terrace is retrieved from its box collider position 

                if (door.name.Contains(dest))
                {
                    // not Terrace
                    if (doorBox == new Vector3(0, 0, 0))
                    {
                        target.position = door.position;
                    }
                    //Terrace
                    else
                    {
                        target.position = colPos;
                    }
                    target.name = door.name;
                    return true;
                }

                // not Terrace
                if (doorBox == new Vector3(0, 0, 0))
                {
                    if (Vector3.Distance(door.position, transform.position) < doorDistThres && Vector3.Distance(door.position, transform.position) < minDist)
                    {
                        minDist = Vector3.Distance(door.position, transform.position);
                        closestDoor = door;
                    }
                }
                //Terrace
                else
                {
                    if (Vector3.Distance(colPos, transform.position) < doorDistThres && Vector3.Distance(colPos, transform.position) < minDist)
                    {
                        minDist = Vector3.Distance(colPos, transform.position);
                        closestDoor = door;
                    }
                }
            }
        }
        return false;
    }

### checkVisibleStairs()

Checks if stairs are are already flagged as visible through the state machine and if not, checks if there are any stairs visible in the agent's fov. If the agent does detect stairs, it opens access to the stairs, sets a target to the stairs' position, and tries to determine a subtarget on the next floor. If successful it returns true, otherwise it returns false.

    public bool checkVisibleStairs()
    {
        if (!sm.stairsVisible)
        {
            if (fov.visibleStairs.Count > 0)
            {
                foreach (var stairs in fov.visibleStairs)
                {
                    if (Math.Abs(stairs.position.y - transform.position.y) > floorDiffThres)
                        continue;

                    // Make stairs accessible
                    foreach (GameObject door in stairsDoors)
                    {
                        door.SetActive(false);
                    }

                    target.position = stairs.position;
                    target.name = stairs.name;

                    string subtarget_name = (curr_floor + sm.floorShift).ToString();

                    for (int i = 0; i < stairs.parent.childCount; i++)
                    {
                        if (stairs.parent.GetChild(i).name == subtarget_name)
                        {
                            subtarget.position = stairs.parent.GetChild(i).position;
                            subtarget.name = subtarget_name;
                            return true;
                        }
                    }
                }
            }
        }
        return false;
    }

### climbStairs()

Sets target to climb stairs.

    public void climbStairs()
    {
        target.position = subtarget.position;
        target.name = subtarget.name;
    }

### isTargetReached()

Checks if agent reached target. If agent distance is less than designated threshold, return true; otherwise return false.

    public bool isTargetReached()
    {
        if (sm.destVisible) {
            if (Vector3.Distance(transform.position, target.position) < 2)
            {
                return true;
            }
        }
        if (Vector3.Distance(transform.position, target.position) < destThreshold)
        {
            return true;
        }
        return false;
    }

### processDecisionNode()

Takes a decision when inside a decision node. If the sign is helpful, the memory will be updated and the new target will be set for the agent. If the sign processed is not helpful, then it will go through the visted nodes and pick the least frequently visited sub dps with the longest line of sight. Once either decision point is made, the seen signs will clear and there will be an existing selected route for the agent to take.  

    public void processDecisionNode()
    {
        sm.randomDecision = false;
        visitedDps.Add(sm.current_dp);
        Debug.Log("Agent has seen " + seenSigns.Count + " signs.");

        Tuple<Transform, Transform> result = processSeenSigns();
        Transform decision_sub_dp = result.Item2;
        Transform chosenSign = result.Item1;

        if (decision_sub_dp == null)
        {
            var sub_dps = sm.current_dp.GetComponentsInChildren<Transform>();
            List<Transform> min_sub_dps = new List<Transform>();

            List<string> min_subs = new List<string>();
            var min = dpMemory[sm.current_dp.id].Aggregate((l, r) => (l.Value < r.Value) ? l : r).Value;
            foreach (var sub in dpMemory[sm.current_dp.id])
            {
                if (sub.Value == min && sub.Key != closestDir.name)
                {
                    min_subs.Add(sub.Key);
                }
            }
            if (min_subs.Count > 1)
            {
                for (int i = 1; i < sub_dps.Length; i++)
                {
                    if (min_subs.Contains(sub_dps[i].name))
                    {
                        min_sub_dps.Add(sub_dps[i]);
                    }
                }
                double max = 0;
                for (int i = 0; i < min_sub_dps.Count; i++)
                {
                    RaycastHit hit;
                    Ray downRay = new Ray(sub_dps[0].position, min_sub_dps[i].position - sub_dps[0].position);
                    Physics.Raycast(downRay, out hit);

                    if (hit.distance > max)
                    {
                        max = hit.distance;
                        decision_sub_dp = min_sub_dps[i];
                    }
                }

                Debug.Log("Agent decided to go " + decision_sub_dp.name);
            }
            else
            {
                for (int i = 1; i < sub_dps.Length; i++)
                {
                    if (sub_dps[i].name == min_subs[0])
                    {
                        decision_sub_dp = sub_dps[i];
                    }
                }
            }
            sm.randomDecision = true;
        }
        else
        {
            Debug.Log("Agent chose to follow sign " + chosenSign.GetComponent<Sign>().id + " and deciced for " + decision_sub_dp);
        }
        dpMemory[sm.current_dp.id][decision_sub_dp.name]++;
        target.position = decision_sub_dp.position;
        target.name = decision_sub_dp.name;

        seenSigns.Clear();

        sm.routeSelected = true;
    }

### processSeenSigns()

Computes direction given by signs viewed by the agent. Takes sign data in a certain direction and creates an tuple that contains a chosen sign and the sub dps associated with it.

     // Computes direction given by seen signs 
    private Tuple<Transform, Transform> processSeenSigns()
    {
        Transform decision_sub_dp = null;
        Transform chosenSign = null;
        if (seenSigns.Count > 0)
        {
            float dot = -2;
            foreach (var sign in seenSigns)
            {
                var direction = sign.GetComponent<Sign>().ProcessSign(dest);
                if (direction != null)
                {
                    foreach (var subsign in sign.GetComponentsInChildren<Transform>())
                    {
                        if (direction == subsign.name)
                        {
                            foreach (var sub_dp in sm.current_dp.GetComponentsInChildren<Transform>())
                            {
                                if (sub_dp.name != sm.current_dp.name)
                                {
                                    var temp_dot = Vector3.Dot((sub_dp.position - sm.current_dp.transform.position).normalized,
                                        (new Vector3(subsign.position.x, 0.0f, subsign.position.z) - new Vector3(sign.transform.position.x, 0.0f, sign.transform.position.z)).normalized);
                                    if (temp_dot > dot)
                                    {
                                        dot = temp_dot;
                                        decision_sub_dp = sub_dp;
                                        chosenSign = sign;
                                    }
                                }
                            }
                            break;
                        }
                    }
                }
                else
                {
                    continue;
                }
            }
        }
        return Tuple.Create(chosenSign, decision_sub_dp);
    }

### updateMemory()

Updates stMemory at every decision node.

    public void updateMemory()
    {
        if (visitedDps.Count > 1)
        {
            string edge = visitedDps[visitedDps.Count - 2].id + "-" + visitedDps[visitedDps.Count - 1].id;
            stMemory[edge][dest]++;
            Debug.Log("Graph edge " + edge + " in destination " + dest + " has been passed " + stMemory[edge][dest] + " times");
        }
    }

### Update()

Scans the environment and updates the time when the last sign has been seen while guiding the agent's decision-making process through the state machine. The agent checks its surroundings, determines its current floor, processes signs, and then executes actions based on its current state. Also measures and flags uncertainty while inside of decision points.

    protected virtual void Update()
    {
        scanEnvironment();
        scanSignsTime();

        //Uncertainty
        if (sm.isInDP)
        {
            uncertainty.withinDecisionZoneFlag = 1;
            uncertainty.numberOfRoutesAtIntersection = sm.current_dp.outgoingVectors.Length;
        }
        else {
            uncertainty.withinDecisionZoneFlag = 0;
            uncertainty.numberOfRoutesAtIntersection = 0;
        }
        uncertainty.numberOfVisibleHelpfulSign = 0;
        if (seenSigns.Count > 0)
        {
            foreach (var sign in seenSigns)
            {
                var direction = sign.GetComponent<Sign>().ProcessSign(dest);
                if (direction != null)
                { uncertainty.numberOfVisibleHelpfulSign++; }
            }
        }
        uncertainty.timeGapBetweenTwoHelpfulSigns = timeSinceSign;


        switch (transform.position.y)
        {
            case float n when n < 7.0f:
                curr_floor = -1;
                break;
            case float n when n < 13.0f:
                curr_floor = 0;
                break;
            default:
                curr_floor = 1;
                break;
        }
        if (GetComponent<MoveTo>().Pathfinding) return;
        if (midPt == null) return;

        var state = GetComponent<StateMachine>()._state;
        switch (state)
        {
            case StateMachine.State.Init:
                sm.stateInitialize();
                break;
            case StateMachine.State.Execute:
                sm.stateExecute();
                break;
            case StateMachine.State.Explore:
                sm.stateExplore();
                break;
            case StateMachine.State.DecisionNode:
                sm.stateDecisionNode();
                break;
            case StateMachine.State.Subgoal:
                sm.stateSubGoal();
                break;
            case StateMachine.State.Final:
                sm.stateFinal();
                break;
            case StateMachine.State.Lost1:
                sm.stateLost1();
                break;
            case StateMachine.State.Lost2:
                sm.stateLost2();
                break;
            case StateMachine.State.Lost3:
                sm.stateLost3();
                break;
            default:
                Debug.Log("No state defined !");
                break;
        }
    }

### scansSignsTime()

Updates the time when the last sign has been seen.

    private void scanSignsTime()
    {
        if (sm.current_dp != null)
        {
            if (fov.visibleSigns.Count > 0)
            {
                foreach (var sign in fov.visibleSigns)
                {
                    Tuple<Transform, Transform> result = processSeenSigns(); 
                    if (result.Item2 != null)
                    {
                        timeSinceSign = Time.time;
                    }
                }
            }
        }
    }

### TrackAgent()

Tracks the agent's position.

    private IEnumerator TrackAgent()
        {
            while (true)
            {
                yield return new WaitForSeconds(trackingTimeResolution);
                coords.Add(new TimeStampedCoords(Time.time, transform.position.x, transform.position.z));
            }
        }

### TimeStampedCoords()

Provides a readonly construct of the agent's timestamped coordinates.

    private readonly struct TimeStampedCoords
    {
        public TimeStampedCoords(float time, float x, float z)
        {
            Time = time;
            X = x;
            Z = z;
        }

        public float Time { get; }
        public float X { get; }
        public float Z { get; }
    }

### onDestroy()

Stops coroutines, constructs a csv file, and constantly writes to it with frame data containing timestamped locations.

    public void OnDestroy()
    {
        if (id == 0) return; 
        StopCoroutine(tracker);
        var fileName = rootPath + id + ".csv";
        File.Create(fileName).Dispose();
        var writer = new StreamWriter(fileName);
        writer.WriteLine("Time,XPos,ZPos");
        foreach (var frame in coords)
        {
            writer.WriteLine(frame.Time + "," + frame.X + "," + frame.Z);
        }
        writer.Close();
    }