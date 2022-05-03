### PERFORMANCE OF MODEL PREDICTIVE CONTROL ON QUAD-ROTOR

This repo contains the code associated to paper Data-Driven MPC for Quadrotors. 
The installation is based on the repository from https://github.com/uzh-rpg/data_driven_mpc.



## Installation

### Minimal Requirements

The code was tested with Ubuntu 18.04, Python 3.6 and ROS Melodic. 
We additionally provide python3.8 support tested with ROS Noetic in Ubuntu 20.04 in the branch `python3.8_support`.
Different OS and ROS versions are possible but not supported.

**Recommended**: Create a Python virtual environment for this package:
```
sudo pip3 install virtualenv
cd <PATH_TO_VENV_DIRECTORY>
virtualenv gp_mpc_venv --python=/usr/bin/python3.6
source gp_mpc_venv/bin/activate
```

**Installation of `acados` and its Python interface** : 
- Build and install the Acados C library. Please follow their [installation guide](https://docs.acados.org/installation/index.html). 
- Install the Python interface. Steps also provided in the [Python interface wiki](https://docs.acados.org/interfaces/index.html#installation).

### Initial setup

0. Source Python virtual environment if created.
   ```
   source <path_to_gp_mpc_venv>/bin/activate
   ```

1. Clone this repository into your catkin workspace.
   ```
   cd <CATKIN_WS_DIR>
   git clone https://github.com/uzh-rpg/data_driven_mpc.git
   ```
   
2. Install the rest of required Python libraries:
   ```
   cd data_driven_mpc
   python setup.py install
   ```
 
3. Build the catkin workspace:
   ```
   cd <CATKIN_WS_DIR>
   catkin build
   source devel/setup.bash
   ```

## Running the package in Simulation

First make sure to add to your Python path the main directory of this package. Also activate the virtual environment if created.
```
export PYTHONPATH=$PYTHONPATH:<CATKIN_WS_DIR>/src/data_driven_mpc/ros_gp_mpc
```

### First steps
To verify the correct installation of the package, execute first a test flight on the `Simplified Simulation`.
```
roscd ros_gp_mpc
python src/experiments/trajectory_test.py
```

After the simulation finishes, a correct installation should produce a result very similar to the following (`Mean optinization time` may vary).
```
:::::::::::::: SIMULATION SETUP ::::::::::::::

Simulation: Applied disturbances: 
{"noisy": true, "drag": true, "payload": false, "motor_noise": true}

Model: No regression model loaded

Reference: Executed trajectory `loop` with a peak axial velocity of 8 m/s, and a maximum speed of 8.273 m/s

::::::::::::: SIMULATION RESULTS :::::::::::::

Mean optimization time: 1.488 ms
Tracking RMSE: 0.2410 m
```

#### Further details
You may edit the configuration variables for the `Simplified Simulator` in the file `config/configuration_parameters.py` for better visualization.
Within the class `SimpleSimConfig`:
```
# Set to True to show a real-time Matplotlib animation of the experiments for the Simplified Simulator. Execution 
# will be slower if the GUI is turned on. Note: setting to True may require some further library installation work.
custom_sim_gui = True

# Set to True to display a plot describing the trajectory tracking results after the execution.
result_plots = True
```

Also, note that in this configuration file the disturbance settings of the `Simplified Simulation` are defined. 
Setting all of them to `False` reproduces the *Ideal* (as we call in our paper) scenario where the MPC has perfect knowledge of the quadrotor dynamics, and therefore will yield a much lower tracking error:
```
# Choice of disturbances modeled in our Simplified Simulator. For more details about the parameters used refer to 
# the script: src/quad_mpc/quad_3d.py.
simulation_disturbances = {
    "noisy": True,                       # Thrust and torque gaussian noises
    "drag": True,                        # 2nd order polynomial aerodynamic drag effect
    "payload": False,                    # Payload force in the Z axis
    "motor_noise": True                  # Asymmetric voltage noise in the motors
}
```

You may also vary the peak velocity and acceleration of the reference trajectory, or use the `lemniscate` trajectory instead of the `circle (loop)` one. All of these options can be specified in the script arguments. Further information can be displayed by typing: 
```
python src/experiments/trajectory_test.py --help
```

### Model Fitting Tutorial

Next, we collect a dataset for fitting GP and RDRv models in the `Simplified Simulator`.

#### Data collection

First, run the following script to collect a few minutes of flight samples.
```
python src/experiments/point_tracking_and_record.py --recording --dataset_name simplified_sim_dataset --simulation_time 300
```
After the simulation ends, you can verify that the collected data now appears at the directory `ros_gp_mpc/data/simplified_sim_dataset`. We can use this data to fit our regression models.

#### Fitting a GP model

First, edit the following variables of configuration file in `config/configuration_parameters.py` (class `ModelFitConfig`) so that the training script is referenced to the desired dataset. For redundancy, in order to identify the correct data file, we require to specify both the name of the dataset as well as the parameters used while acquiring the data.
In other words, you must input the simulator options used while running the previous python script. If you did not modify these variables earlier, you don't need to change anything this time as the default setting will work:
```
    # ## Dataset loading ## #
    ds_name = "simplified_sim_dataset"
    ds_metadata = {
        "noisy": True,
        "drag": True,
        "payload": False,
        "motor_noise": True
    }
```

In our approach, we train 3 independent GP's for every velocity dimension `v_x, v_y, v_z`, so we run three times the GP training script. To indicate that the model must map `v_x` to acceleration correction `a_x` (and similarly with `y` and `z`), run the following commands. Indices `7,8,9` correspond to `v_x, v_y, v_z` respectively in our data structures. The arguments `--x` and `--y` are used to specify the `X` and `Y` variables of the regression problem. 
We assign a name to the model for future referencing, e.g.: `simple_sim_gp`:
```
python src/model_fitting/gp_fitting.py --n_points 20 --model_name simple_sim_gp --x 7 --y 7
python src/model_fitting/gp_fitting.py --n_points 20 --model_name simple_sim_gp --x 8 --y 8
python src/model_fitting/gp_fitting.py --n_points 20 --model_name simple_sim_gp --x 9 --y 9
```
The models will be saved under the directory `ros_gp_mpc/results/model_fitting/<git_hash>/`. 

You can visualize the performance of the combined three models using the visualization script. Make sure to input the correct model version (git hash) and model name.
```
python src/model_fitting/gp_visualization.py --model_name simple_sim_gp --model_version <git_hash>
```

#### Fitting an RDRv model

Similarly, we train the RDRv model with the following one-line command. This script trains all three dimensions simultaneously and provides a plot of the fitting result. The model is similarly saved under the directory `ros_gp_mpc/results/model_fitting/<git_hash>/` with the given name (e.g.: `simple_sim_rdrv`). 
```
python src/model_fitting/rdrv_fitting.py --model_name simple_sim_rdrv --x 7 8 9
```

### Model comparison experiment

To compare the trained models, we provide an automatic script for the `Simplified Simulation`. Running the following command will compare the specified models with the "Ideal" and the "Nominal" scenarios by default, and produce several results plots in the directory: `results/images/`. Using the `--fast` argument will run the script faster with less velocity samples.

```
python src/experiments/comparative_experiment.py --model_version <git_hash_1 git_hash_2 ...> --model_name <name_1 name_2 ...> --model_type <type_1 type_2> --fast
```
For example:
```
python src/experiments/comparative_experiment.py --model_version 42b8650b 42b8650b --model_name simple_sim_gp simple_sim_rdrv --model_type gp rdrv --fast
```
