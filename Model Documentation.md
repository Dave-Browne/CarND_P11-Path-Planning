# Path Planning Project

### Goal
The goal of this project is to create a path planner that creates smooth, safe trajectories for a vehicle on the Udacity Simulator highway track. The following requirements must be met:
- Complete atleast 1 lap around the 4.32miles track without incident
- Stay under the speed limit of 50mph, but don't drive too slowly, unless stuck behind traffic
- Do not exceed a total acceleration of 10 m/s^2 and a jerk of 10 m/s^3
- No collisions with other cars on the road
- Make smooth lane changes when necessary

### Code Structure
The following 5 code files and 1 cmake file were changed:
- main.cpp - this contains the main program
- vehicle.h
- vehicle.cpp - class vehicle
- helper.h
- helper.cpp - math and geometry functions to assist in the vehicle class
- CMakeLists.txt - contains the names of the cpp files to be used

### Given
- Waypoints to direct the vehicle around the track
- Sensor fusion data of other vehicles travelling in the same direction on the highway `[id, x, y, vx, vy, s, d]`
- Vehicle data `[x (m), y (m), s (m), d (m), yaw (rad), speed (mph)]`
- Previous trajectory points less those already passed

### Strategy
There are 3 components to creating a Path Planner. A Path Planner takes as input localisation and sensor fusion data and produces an output trajectory for the vehicle to follow.
1) Prediction - Predict where nearby vehicles/objects are going
2) Behaviour - Decide what the vehicle should do based on nearby vehicles/objects movements
3) Trajectory - Produce a jerk minimising trajectory for the vehicle to follow

### 1) Prediction
See `Vehicle::bot_predictions` function.

There are 3 ways of solving the prediction problem:
- Data-driven approach. Requires lots of input data to train a model
- Model-driven approach. Requires mathematical models of different maneuvers
- Hybrid approach. Process model + machine learning.

The Model-driven approach is used for this project as no training data is provided. The highway vehicle's locations and velocities are provided in sensor fusion data so their future states can be predicted using simple math. Most of the time the vehicles are travelling at a constant velocity. So a constant velocity model can be used and acceleration is assumed to be 0. This can be found in the `Vehicle::state_at` function.

```
double s = this->s + this->v * t;
double v = this->v;
```

### 2) Behaviour Planner
See `Vehicle::update_state()` function.

A Finite State Machine was used for the behavioural planner. This is possible as highway driving has relatively few maneuvers compared to urban driving.

![FSM](https://user-images.githubusercontent.com/22233694/29252229-5139a5a8-8063-11e7-800d-87fd6c427534.jpg "FSM")


Five states were used as seen in the figure above. A vehicle can often not change lanes immediately as there may be another vehicle next to it. It is therefore necessary to take some action until it is safe to turn or change lanes in the other direction. This is why there are Prepare Lane Change Left/Right (PLCL/R) states. Various costs are assigned (see `Vehicle::update_state()`) to different actions and can be summarised as follows:
1) `cost_kl` - 10 points. If stuck behind a slower vehicle in it's lane
2) `cost_d` - 1000 points. If the vehicle wants to turn off the highway lanes (ie into on-coming traffic or into the dirt)
3) `cost_collision` - 100 000 points. If the vehicle will collide with another vehicle
4) `cost_tie_breaker` - 0.2 points. The cost to move to PLCL/R states. This gives preference to staying in the same lane
5) `cost_tie_breaker` - 0.1 points. The cost to move from PLCL/R to LCL/R states. This gives preference to actually changing into the new lane
6) `cost_tie_breaker` - 0.1 points. Added if a collision is predicted in PLCL/R states. This gives preference to either PLCL or PLCR

The following are actions the car will take:
1) Increase speed if there are no vehicles in front
2) Slow down if a vehicle is detected in front and prepare to change lanes left/right
3) If a lane change cannot be done, reduce speed and attempt to change lanes again, otherwise keep going straight

Tricky situations
- If stuck behind a vehicle and another vehicle is in the next lane blocking the lane change. Reduce speed and wait until the lane is clear to change lanes
- If a car is approaching very fast in an adjacent lane, the planner may not detect it and change lanes into it. This is because only cars up to 20m behind are detected for collisions: `if ( (l_ == lane) && (s_ > (s-20) ) )`
- When other vehicles change lanes the model only detects the car in the new lane once it has arrived there. There is no lane changing prediction.

### 3) Trajectory
See `Vehicle::generate_vehicle_trajectory` function.

The Jerk Minimising Trajectory (JMT) is not used. A spline is created using 5 points in Frenet coordinates, namely:
1) previous position (20ms ago)
2) current position
3) 50m, 75m and 100m ahead
The 1st and 2nd points are used for continuity. Changing the value of the 3rd point (currently 50m) will change the jerk in a lane change. Following a spline that ventures 100m into the future creates a smooth lane change strategy but causes the vehicle to drift towards the inside of its lane on sharp corners.

The speed of the vehicle is determined by the distance between each (x, y) point sent to the simulator. The vehicle takes 20ms to travel from one point to the next. Speed is defined in the `Vehicle::update_state()` function. The speed starts at 0m and increases as per the behavioural planner logic. As the vehicle approaches the speed limit, the speed reduces according to:
```
double v_inc = dist_max/2 * delta_speed * 0.12;
if (v_inc > dist_max/2) v_inc = dist_max/2;
  ref_v += v_inc;
```
Here `v_inc` is the incremental distance, `dist_max` is the distance between points corresponding to the maximum acceleration and `delta_speed` is the difference between the current speed and speed limit. `ref_v` is the new reference velocity.

### Conclusion
The behavioural planner is good but not perfect, yet. It will not always complete a lap without an incident. More complex logic is required to handle the more complex situations. Due to time limitations for the project submission those will not be implemented now.

### Resources
- [Introduction to Robotics #4 Path-Planning](http://correll.cs.colorado.edu/?p=965)
- [Path Planning](http://www.coppeliarobotics.com/helpFiles/en/pathPlanningModule.htm)
- [The path planning problem in depth](https://www.cs.cmu.edu/afs/cs/project/jair/pub/volume9/mazer98a-html/node2.html)
- [Robo realm: Path Planning](http://www.roborealm.com/help/Path_Planning.php)
- [A discussion on What is the difference between path planning and motion planning?](https://robotics.stackexchange.com/questions/8302/what-is-the-difference-between-path-planning-and-motion-planning)
- [Excellent Tutorial on A* Robot Path Planning](http://robotshop.com/letsmakerobots/excellent-tutorial-a-robot-path-planning)
- [Path Planning in Environments of Different Complexity](https://www.mathworks.com/help/robotics/examples/path-planning-in-environments-of-different-complexity.html?requestedDomain=www.mathworks.com)
- [Introduction to robot motion: Robot Motion Planning](http://ais.informatik.uni-freiburg.de/teaching/ss11/robotics/slides/18-robot-motion-planning.pdf)
- [Introduction to robot motion: Path Planning and Collision Avoidance](http://ais.informatik.uni-freiburg.de/teaching/ss10/robotics/slides/16-pathplanning.pdf)
- [Sebastian Thrun - practical search techniques in path planning](http://ai.stanford.edu/~ddolgov/papers/dolgov_gpp_stair08.pdf)
- [Thesis: A path planning and obstacle avoidance algorithm for an autonomous robotic vehicle](https://webpages.uncc.edu/~jmconrad/GradStudents/Thesis_Ghangrekar.pdf)
