# CarND-Path-Planning-Project
Self-Driving Car Engineer Nanodegree Program
   
### Simulator.
You can download the Term3 Simulator which contains the Path Planning Project from the [releases tab (https://github.com/udacity/self-driving-car-sim/releases/tag/T3_v1.2).  

To run the simulator on Mac/Linux, first make the binary file executable with the following command:
```shell
sudo chmod u+x {simulator_file_name}
```

### Goals
In this project your goal is to safely navigate around a virtual highway with other traffic that is driving +-10 MPH of the 50 MPH speed limit. You will be provided the car's localization and sensor fusion data, there is also a sparse map list of waypoints around the highway. The car should try to go as close as possible to the 50 MPH speed limit, which means passing slower traffic when possible, note that other cars will try to change lanes too. The car should avoid hitting other cars at all cost as well as driving inside of the marked road lanes at all times, unless going from one lane to another. The car should be able to make one complete loop around the 6946m highway. Since the car is trying to go 50 MPH, it should take a little over 5 minutes to complete 1 loop. Also the car should not experience total acceleration over 10 m/s^2 and jerk that is greater than 10 m/s^3.

#### The map of the highway is in data/highway_map.txt
Each waypoint in the list contains  [x,y,s,dx,dy] values. x and y are the waypoint's map coordinate position, the s value is the distance along the road to get to that waypoint in meters, the dx and dy values define the unit normal vector pointing outward of the highway loop.

The highway's waypoints loop around so the frenet s value, distance along the road, goes from 0 to 6945.554.

## Basic Build Instructions

1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./path_planning`.

Here is the data provided from the Simulator to the C++ Program

#### Main car's localization Data (No Noise)

["x"] The car's x position in map coordinates

["y"] The car's y position in map coordinates

["s"] The car's s position in frenet coordinates

["d"] The car's d position in frenet coordinates

["yaw"] The car's yaw angle in the map

["speed"] The car's speed in MPH

#### Previous path data given to the Planner

//Note: Return the previous list but with processed points removed, can be a nice tool to show how far along
the path has processed since last time. 

["previous_path_x"] The previous list of x points previously given to the simulator

["previous_path_y"] The previous list of y points previously given to the simulator

#### Previous path's end s and d values 

["end_path_s"] The previous list's last point's frenet s value

["end_path_d"] The previous list's last point's frenet d value

#### Sensor Fusion Data, a list of all other car's attributes on the same side of the road. (No Noise)

["sensor_fusion"] A 2d vector of cars and then that car's [car's unique ID, car's x position in map coordinates, car's y position in map coordinates, car's x velocity in m/s, car's y velocity in m/s, car's s position in frenet coordinates, car's d position in frenet coordinates. 

## Details

1. The car uses a perfect controller and will visit every (x,y) point it recieves in the list every .02 seconds. The units for the (x,y) points are in meters and the spacing of the points determines the speed of the car. The vector going from a point to the next point in the list dictates the angle of the car. Acceleration both in the tangential and normal directions is measured along with the jerk, the rate of change of total Acceleration. The (x,y) point paths that the planner recieves should not have a total acceleration that goes over 10 m/s^2, also the jerk should not go over 50 m/s^3. (NOTE: As this is BETA, these requirements might change. Also currently jerk is over a .02 second interval, it would probably be better to average total acceleration over 1 second and measure jerk from that.

2. There will be some latency between the simulator running and the path planner returning a path, with optimized code usually its not very long maybe just 1-3 time steps. During this delay the simulator will continue using points that it was last given, because of this its a good idea to store the last points you have used so you can have a smooth transition. previous_path_x, and previous_path_y can be helpful for this transition since they show the last points given to the simulator controller with the processed points already removed. You would either return a path that extends this previous path or make sure to create a new path that has a smooth transition with this last path.
**Path Planning**

This projects aims to develop a path planning algorithm for autonomous driving in a simulated highway.

### 1. Environment.
Simulator is based in Unity and mimics a car for autonomous driving in a highway. The vehicle is controlled from a c++ script using web sockets where the script specifies whih will be the nest positions in x and y coordinates.

The main goal of this project is to drive safely sorting the other vehicles when possible without exceeding the maximum allowed speed and jerk for vehicle's user comfort.

### 2. Trajectory generation.
In order to generate the trajectories for the vehicle the library spline is used. This tool allows to obtain a smooth trajectory using reference points. Fisrt step to generate the trajectory is to define 2 intial points based either on the previous path data given to the planer or the actual vehicle position if previous data has been already executed.

Next step is to define 3 points at 30, 60 and 90m ahead in s position in Frenet coordinates acoording to our current s and d (frenet coordinates) that is given in terms of the actual lane. This lane term will help us to move between lanes since the spline function will generate acoording to this, for instance, lane 0 corresponds to the left most lane, 1 to the center one and 2 is the right most lane.

```
vector<double> next_wp0 = getXY(car_s+30,(2+4*lane),map_waypoints_s,map_waypoints_x,map_waypoints_y);
vector<double> next_wp1 = getXY(car_s+60,(2+4*lane),map_waypoints_s,map_waypoints_x,map_waypoints_y);
vector<double> next_wp2 = getXY(car_s+90,(2+4*lane),map_waypoints_s,map_waypoints_x,map_waypoints_y);
```

These 5 points in cartesian coordinates are referenced according to the previous final position and heading.

```
for(int i = 0; i < ptsx.size(); i++){
  // Shift car reference angle to 0 degrees
  double shift_x = ptsx[i]-ref_x;
  double shift_y = ptsy[i]-ref_y;

  ptsx[i] = (shift_x*cos(0-ref_yaw)-shift_y*sin(0-ref_yaw));
  ptsy[i] = (shift_x*sin(0-ref_yaw)+shift_y*cos(0-ref_yaw));
}
```
With these points a spline function is generated. This function acts as an equation where according to a given x position the function returns the y coordinate  of the trajectory. Using the reference velocity at which we want to travel and the delta time of our simulator (0.02s) we generate the rest of the points of the trajectory which will always be of 50. 

```
// Calculate how to break up spline points so that we travel at our desired reference velocity
double target_x = 30.0;
double target_y = s(target_x);
double target_dist = sqrt((target_x*target_x)+(target_y*target_y));

double x_add_on = 0;

// Fill up the rest of our path planner after filling it with previous points, here we will always output 50 points
for(int i =0; i <= 50-previous_path_x.size(); i++){
  double N = (target_dist/(0.02*ref_vel/2.24));
  double x_point = x_add_on+target_x/N;
  double y_point = s(x_point);
  x_add_on = x_point;

  double x_ref = x_point;
  double y_ref = y_point;

  // Rotate back to normal after rotating it earlier
  x_point = (x_ref*cos(ref_yaw)-y_ref*sin(ref_yaw));
  y_point = (x_ref*sin(ref_yaw)+y_ref*cos(ref_yaw));

  x_point += ref_x;
  y_point += ref_y;

  next_x_vals.push_back(x_point);
  next_y_vals.push_back(y_point);
}
```

The last step is to rotated back to its original orientation and feed to the simulator the path.


### 3. Sensor fusion.
The simulator give us all relevant information about surrounding vehicles:
* s position.
* d position.
* velocity in x.
* vlocity in y.

Using this information the planner will decide in which lane to drive and at what velocity to do it. The approach for driving safely according to the other vehicles will be the following:

Frist step is to determine in which lane is every vehicle using the d position.

```
// Check in wich lane is the car
int car_lane;
if(d >= 0 && d < 4){
  car_lane = 0;
}
else if(d >= 4 && d < 8){
  car_lane = 1;
}
else if(d >= 8 && d < 12){
  car_lane = 2;
}
```

Next thing is to determine which is the closest vehicle within 30m ahead or behind in s position in each of the surrounding anc actual lanes and gather their speeds. 

```
// velocity of the closest one
if(check_car_s > car_s - 30 && check_car_s <= car_s){
  if(car_lane == lane - 1){
    left_behind_flag = true;
    if(car_s - check_car_s < closest_left_behind){
      closest_left_behind = car_s - check_car_s;
      left_behind_vel = check_speed;
    }
  }
  // Check if it's on the right 
  else if(car_lane == lane + 1){
    right_behind_flag = true;
    if(car_s - check_car_s < closest_right_behind){
      closest_right_behind = car_s - check_car_s;
      right_behind_vel = check_speed;
    }
  }
}
// Check cars ahead
else if(check_car_s > car_s && check_car_s <= car_s + 30){
  if(car_lane == lane){
    ahead_flag = true;
  }
  // Check if it's on the left 
  else if(car_lane == lane - 1){
    left_ahead_flag = true;
    if(car_s - check_car_s < closest_left_ahead){
      closest_left_ahead = check_car_s - car_s;
      left_ahead_vel = check_speed;
    }
  }
  // Check if it's on the right 
  else if(car_lane == lane + 1){
    right_ahead_flag = true;
    if(car_s - check_car_s < closest_right_ahead){
      closest_right_ahead = check_car_s - car_s;
      right_ahead_vel = check_speed;
    }
  }
}
```

With this information we will determine if either we want to move to our right, to our left or simply stay in the same lane and also if we want to speed up or down incrementally.

```

if(ahead_flag){
  if(closest_left_behind >= 20 && closest_left_ahead >= 20 && 
     left_behind_vel - ref_vel < 15 && ref_vel - left_ahead_vel < 15 &&
     lane > 0){
    lane--;
  }
  else if(closest_right_behind >= 20 && closest_right_ahead >= 20 &&
          right_behind_vel - ref_vel < 15 && ref_vel - right_ahead_vel < 15
          && lane != 2){
    lane++;
  }
  else{
    ref_vel -= 0.224;
  }
}
else{
  if(ref_vel < 49.5){
    ref_vel += 0.224;
  }
} 
```

Simply if there is a car ahead us at less than 30m away we will look for a 40m gap on any lane where if there is a car behind us in that lane, this vehicle is not going 15mph faster than us and if there is a vehicle ahead in that lane, this is not going 15mph slower than us. If we can not merge we will we will reduce the speed 1mph. And if there is not a car ahead us we will speed up 1mph at a time, but being below the speed limit. 


### 4. Results.
The vehicle is able to navigate succesfuly for several miles and does not surpass the established limits. 

## Tips

A really helpful resource for doing this project and creating smooth trajectories was using http://kluge.in-chemnitz.de/opensource/spline/, the spline function is in a single hearder file is really easy to use.

---

## Dependencies

* cmake >= 3.5
  * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets 
    cd uWebSockets
    git checkout e94b6e1
    ```

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!


## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./

## How to write a README
A well written README file can enhance your project and portfolio.  Develop your abilities to create professional README files by completing [this free course](https://www.udacity.com/course/writing-readmes--ud777).
