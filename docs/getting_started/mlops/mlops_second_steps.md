---
title: Next Steps
---

Once Tasks are defined and in the ClearML system, they can be chained together to create Pipelines.
Pipelines provide users with a greater level of abstraction and automation, with Tasks running one after the other.<br/>
Tasks can interface with other Tasks in the pipeline and leverage other Tasks' work products.<br/> 
We'll go through a scenario where users create a Dataset, process the data then consume it with another task, all running as a pipeline.


## Building Tasks
### Dataset Creation

Let's assume we have some code that extracts data from a production Database into a local folder.
Our goal is to create an immutable copy of the data to be used by further steps:

```bash
clearml-data create --project data --name dataset
clearml-data sync --folder ./from_production 
```

We could also add a Tag `latest` to the Dataset, marking it as the latest version.

### Preprocessing Data
The second step is to preprocess the date. First we need to access it, then we want to modify it
and lastly we want to create a new version of the data.

```python
# create a task for the data processing part
task = Task.init(project_name='data', task_name='ingest', task_type='data_processing')

# get the v1 dataset
dataset = Dataset.get(dataset_project='data', dataset_name='dataset_v1')

# get a local mutable copy of the dataset
dataset_folder = dataset.get_mutable_local_copy(target_folder='work_dataset', overwrite=True)
# change some files in the `./work_dataset` folder
...
# create a new version of the dataset with the pickle file
new_dataset = Dataset.create(
    dataset_project='data', dataset_name='dataset_v2', 
    parent_datasets=[dataset], 
    use_current_task=True,  # this will make sure we have the creation code and the actual dataset artifacts on the same Task
)
new_dataset.sync_folder(local_path=dataset_folder)
new_dataset.upload()
new_dataset.finalize()
# now let's remove the previous dataset tag
dataset.tags = []
new_dataset.tags = ['latest']
```

We passed the `parents` argument when we created v2 of the Dataset, this inherits all the parent's version content.
This will not only help us in tracing back dataset changes with full genealogy, but will also make our storage more efficient,
as it will only store the files that were changed \ added from the parent versions.
When we will later need access to the Dataset it will automatically merge the files from all parent versions 
in a fully automatic and transparent process, as if they were always part of the requested Dataset.

### Training
We can now train our model with the **latest** Dataset we have in the system.
We will do that by getting the instance of the Dataset based on the `latest` tag 
(if by any chance we have two Datasets with the same tag we will get the newest).
Once we have the dataset we can request a local copy of the data. All local copy requests are cached,
which means that if we are accessing the same dataset multiple times we will not have any unnecessary downloads.

```python
# create a task for the model training
task = Task.init(project_name='data', task_name='ingest', task_type='training')

# get the latest dataset with the tag `latest`
dataset = Dataset.get(dataset_tags='latest')

# get a cached copy of the Dataset files 
dataset_folder = dataset.get_local_copy()

# train our model here
```

## Building the Pipeline

Now that we have the data creation step, and the data training step, let's create a pipeline that when executed,
will first run the first and then run the second.
It is important to remember that pipelines are Tasks by themselves and can also be automated by other pipelines (i.e. pipelines of pipelines).

```python
pipe = PipelineController(
    always_create_task=True,
    pipeline_project='data', pipeline_name='pipeline demo',
)

pipe.add_step(
    name='step 1 data',
    base_task_id='cbc84a74288e459c874b54998d650214',  # Put the task ID here
)
pipe.add_step(
    name='step 2 train', 
    parents=['step 1 data', ],
    base_task_id='cbc84a74288e459c874b54998d650214',  # Put the task ID here
)
```

We could also pass the parameters from one step to the other (for example `Task.id`).
See more in the full pipeline documentation [here](../../fundamentals/pipelines.md).