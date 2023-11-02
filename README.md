# Log Summarization Pipeline

## Overview
The Log summarization pipeline is a sample solution that demonstrates how to process a larger
log volume leveraging OpenAI model to summarize the content. The pipeline itself includes the following steps:
1. __Map step__ that reads a message from the input, and extracts keys from the message fields that are used
in the reduce step aggregation. The expected message format is as follows:
```
    {
        “ts”: 1692036005,
        “tid”: VADE0B248932
        “start_time”: 1692036005,
        “end_time”: 1692036005,
        “app_name”: “my-app”,
        “app_type”: “Deployment”,
        “summarization_type”: “pod”,
        “summarization_name”: “log-ingestor-56d57cdc95-vfjtr”
        “logs”: [
            “2023/08/14 21:07:03.515 INFO  c.i.o.o.generic.transform.ToJSONDoFn…”,
            “2023/08/14 21:07:03.211 INFO  c.i.o.o.g.t.CombineByKeyFn$PrintCombined…”,
                    …
        ]
    }
```
2. __Reduce step__ that aggregates the logs by the keys extracted in the map step. In addition to simple
message collection by key for a time period of one minute, the processing also compresses the content
by clustering the log items together and leaving just one log sample for each cluster, at the same time
counting the number of the similar log lines. The outcome is that the size of the log aggregation is
reduced, and can easily be included in an LLM prompt without hitting the token limit. The logic for 
the map and reduce is implemented in the [sampler.py](https://github.com/numaproj-labs/log-summarization/blob/main/src/udf/sampler/sampler.py) file.
3. __GenAI invocation step__ obtains the log aggregation from the previous step, makes the required 
substitutions in a prompt template, and invokes the OpenAI model to generate the summary. The logic 
for the above is implemented in the [processor.py](https://github.com/numaproj-labs/log-summarization/blob/main/src/udf/processor/processor.py) file.
4. __Redis sink__ is the last step writes the generated summary to the local Redis instance, where it can be queried.
The logic for the above is in the [redissink.py](https://github.com/numaproj-labs/log-summarization/blob/main/src/udf/customsink/redissink.py) file.

The Numaflow pipeline that links the above steps together in the right sequence is defined in the [log_summarization_pipeline.yaml](https://github.com/numaproj-labs/log-summarization/blob/main/log_summarization_pipeline.yaml) file.

## Building and deploying the pipeline
To make the OpenAI calls, you need to obtain an [API key](https://platform.openai.com/account/api-keys). 
The key then must be added to the Kubernetes namespace where the pipeline is deployed.
The command to deploy the secret is as follows:
```
kubectl create secret generic log-summarization-tokens --from-literal=openai-api-key='...'
```
Once this is done, you can deploy the pipeline using the following command:
```
kubectl apply -f log_summarization_pipeline.yaml
```
If you make a code change you have to rebuild the Docker image and push it to the Docker registry.
The commands to do that are as follows:
```
docker build -t <docker image name>:<image verison> --target udf .
docker push <docker image name>:<image verison>
```
Once the image is pushed, you can update the docker image references in the [log_summarization_pipeline.yaml](https://github.com/numaproj-labs/log-summarization/blob/main/log_summarization_pipeline.yaml)
and reapply the pipeline definition.
