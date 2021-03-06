# Use Neural Network Compression Framework (NNCF) as Standalone

This is a step-by-step tutorial on how to integrate the NNCF package into the existing project.
The use case implies that the user already has a training pipeline that reproduces training of the model in the floating  point precision and pretrained model.
The task is to prepare this model for accelerated inference by simulating the compression at train time.
The instructions below use certain "helper" functions of the NNCF which abstract away most of the framework specifics and make the integration easier in most cases.
As an alternative, you can always use the NNCF internal objects and methods as described in the [architectural overview](./NNCFArchitecture.md).


## Basic usage

#### Step 1: Create an NNCF configuration file

A JSON configuration file is used for easier setup of the parameters of compression to be applied to your model.
See [configuration file description](./ConfigFile.md) or the sample configuration files packaged with the [example scripts](../examples) for reference.

#### Step 2: Modify the training pipeline
NNCF enables compression-aware training by being integrated into the regular training pipelines.
The framework is designed so that the modifications to your original training code are minor.

 1. **Add** the imports required for NNCF:
    ```python
    import nncf
    from nncf import NNCFConfig, create_compressed_model, load_state
    ```
 2. Load the NNCF JSON configuration file that you prepared during Step 1:
    ```python
    nncf_config = NNCFConfig.from_json("nncf_config.json")  # Specify a path to your own NNCF configuration file in place of "nncf_config.json"
    ```
 3. (Optional) For certain algorithms such as quantization it is highly recommended to **initialize the algorithm** by  
 passing training data via `nncf_config` prior to starting the compression fine-tuning properly:
    ```python
    from nncf import register_default_init_args
    nncf_config = register_default_init_args(nncf_config, criterion, train_loader)
    ```
    Training data loaders should be attached to the NNCFConfig object as part of a library-defined structure. `register_default_init_args` is a helper
    method that registers the necessary structures for all available initializations (currently quantizer range and precision initialization) by taking criterion and data loader.  

    The initialization expects that the model is called with its first argument equal to the dataloader output.
    If your model has more complex input arguments you can create and pass an instance of
    `nncf.initialization.InitializingDataLoader` that overrides its `__next__` method to return
    a tuple of (_single model input_ , _the rest of the model inputs as a kwargs dict_).

 4. Right after you create an instance of the original model and load its weights, **wrap the model** by making the following call
    ```python
    compression_ctrl, compressed_model = create_compressed_model(model, nncf_config)
    ```
    The `create_compressed_model` function parses the loaded configuration file and returns two objects. `compression_ctrl` is a "controller" object that can be used during compressed model training to adjust certain parameters of the compression algorithm (according to a scheduler, for instance), or to gather statistics related to your compression algorithm (such as the current level of sparsity in your model).

 5. (Optional) Wrap your model with `DataParallel` or `DistributedDataParallel` classes for multi-GPU training. If you do so, add the following call afterwards:
   ```python
   compression_ctrl.distributed()
   ```

   in case the compression algorithms that you use need special adjustments to function in the distributed mode.


6. In the **training loop**, make the following changes:
     - After inferring the model, take a compression loss and add it (using the `+` operator) to the common loss, for example cross-entropy loss:
        ```python
        compression_loss = compression_ctrl.loss()
        loss = cross_entropy_loss + compression_loss
        ```
     - Call the scheduler `step()` after each training iteration:
        ```python
        compression_ctrl.scheduler.step()
        ```
     - Call the scheduler `epoch_step()` after each training epoch:
        ```python
        compression_ctrl.scheduler.epoch_step()
        ```

> **NOTE**: For a real-world example of how these changes should be introduced, take a look at the [examples](../examples) published in the NNCF repository.

#### Step 3: Run the training pipeline
At this point, the NNCF is fully integrated into your training pipeline.
You can run it as usual and monitor your original model's metrics and/or compression algorithm metrics and balance model metrics quality vs. level of compression.


Important points you should consider when training your networks with compression algorithms:
  - Turn off the `Dropout` layers (and similar ones like `DropConnect`) when training a network with quantization or sparsity
  - It is better to turn off additional regularization in the loss function (for example, L2 regularization via `weight_decay`) when training the network with RB sparsity, since it already imposes an L0 regularization term.

## Saving and loading compressed models in PyTorch
You can save the `compressed_model` object using `torch.save` as usual.
However, keep in mind that in order to load the resulting checkpoint file the `compressed_model` object should have the
same structure with regards to PyTorch module and parameters as it was when the checkpoint was saved.
In practice this means that you should use the same compression algorithms (i.e. the same NNCF configuration file) when loading a compressed model checkpoint.
Use the optional `resuming_checkpoint` argument of the `create_compressed_model` helper function to specify a PyTorch state dict to be loaded into your model once it is created.

Alternatively, you can use the `nncf.load_state` function.
It will attempt to load a PyTorch state dict into a model by first stripping the irrelevant prefixes, such as `module.` or `nncf_module.`, from both the checkpoint and the model layer identifiers, and then do the matching between the layers.
Depending on the value of the `is_resume` argument, it will then fail if an exact match could not be made (when `is_resume == True`), or load the matching layer parameters and print a warning listing the mismatches (when `is_resume == False`).
`is_resume=False` is most commonly used if you want to load the starting weights from an uncompressed model into a compressed model, and `is_resume=True` is used when you want to evaluate a compressed checkpoint or resume compressed checkpoint training without changing the compression algorithm parameters.
