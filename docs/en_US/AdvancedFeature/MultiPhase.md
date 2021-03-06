## What is multi-phase experiment

Typically each trial job gets a single configuration (e.g., hyperparameters) from tuner, tries this configuration and reports result, then exits. But sometimes a trial job may wants to request multiple configurations from tuner. We find this is a very compelling feature. For example:

1. Job launch takes tens of seconds in some training platform. If a configuration takes only around a minute to finish, running only one configuration in a trial job would be very inefficient. An appealing way is that a trial job requests a configuration and finishes it, then requests another configuration and run. The extreme case is that a trial job can run infinite configurations. If you set concurrency to be for example 6, there would be 6 __long running__ jobs keeping trying different configurations.

2. Some types of models have to be trained phase by phase, the configuration of next phase depends on the results of previous phase(s). For example, to find the best quantization for a model, the training procedure is often as follows: the auto-quantization algorithm (i.e., tuner in NNI) chooses a size of bits (e.g., 16 bits), a trial job gets this configuration and trains the model for some epochs and reports result (e.g., accuracy). The algorithm receives this result and makes decision of changing 16 bits to 8 bits, or changing back to 32 bits. This process is repeated for a configured times.

The above cases can be supported by the same feature, i.e., multi-phase execution. To support those cases, basically a trial job should be able to request multiple configurations from tuner. Tuner is aware of whether two configuration requests are from the same trial job or different ones. Also in multi-phase a trial job can report multiple final results.

Note that, `nni.get_next_parameter()` and `nni.report_final_result()` should be called sequentially: __call the former one, then call the later one; and repeat this pattern__. If `nni.get_next_parameter()` is called multiple times consecutively, and then `nni.report_final_result()` is called once, the result is associated to the last configuration, which is retrieved from the last get_next_parameter call. So there is no result associated to previous get_next_parameter calls, and it may cause some multi-phase algorithm broken.

## Create multi-phase experiment

### Write trial code which leverages multi-phase:

__1. Update trial code__

It is pretty simple to use multi-phase in trial code, an example is shown below:

    ```python
    # ...
    for i in range(5):
        # get parameter from tuner
        tuner_param = nni.get_next_parameter()

        # consume the params
        # ...
        # report final result somewhere for the parameter retrieved above
        nni.report_final_result()
        # ...
    # ...
    ```

__2. Experiment configuration__

To enable multi-phase, you should also add `multiPhase: true` in your experiment YAML configure file. If this line is not added, `nni.get_next_parameter()` would always return the same configuration.

Multi-phase experiment configuration example:

```
authorName: default
experimentName: multiphase experiment
trialConcurrency: 2
maxExecDuration: 1h
maxTrialNum: 8
trainingServicePlatform: local
searchSpacePath: search_space.json
multiPhase: true
useAnnotation: false
tuner:
  builtinTunerName: TPE
  classArgs:
    optimize_mode: maximize
trial:
  command: python3 mytrial.py
  codeDir: .
  gpuNum: 0
```

### Write a tuner that leverages multi-phase:

Before writing a multi-phase tuner, we highly suggest you to go through  [Customize Tuner](https://nni.readthedocs.io/en/latest/Customize_Tuner.html). Same as writing a normal tuner, your tuner needs to inherit from `Tuner` class. When you enable multi-phase through configuration (set `multiPhase` to true), your tuner will get an additional parameter `trial_job_id` via tuner's following methods:
```
generate_parameters
generate_multiple_parameters
receive_trial_result
receive_customized_trial_result
trial_end
```
With this information, the tuner could know which trial is requesting a configuration, and which trial is reporting results. This information provides enough flexibility for your tuner to deal with different trials and different phases. For example, you may want to use the trial_job_id parameter of generate_parameters method to generate hyperparameters for a specific trial job.

### Tuners support multi-phase experiments:

[TPE](../Tuner/HyperoptTuner.md), [Random](../Tuner/HyperoptTuner.md), [Anneal](../Tuner/HyperoptTuner.md), [Evolution](../Tuner/EvolutionTuner.md), [SMAC](../Tuner/SmacTuner.md), [NetworkMorphism](../Tuner/NetworkmorphismTuner.md), [MetisTuner](../Tuner/MetisTuner.md), [BOHB](../Tuner/BohbAdvisor.md), [Hyperband](../Tuner/HyperbandAdvisor.md), [ENAS tuner](https://github.com/countif/enas_nni/blob/master/nni/examples/tuners/enas/nni_controller_ptb.py).

### Training services support multi-phase experiment:
[Local Machine](../TrainingService/LocalMode.md), [Remote Servers](../TrainingService/RemoteMachineMode.md), [OpenPAI](../TrainingService/PaiMode.md)
