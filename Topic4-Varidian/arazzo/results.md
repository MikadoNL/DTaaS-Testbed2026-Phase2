# Arazzo Workflow for OGC Process Recipes

## What is Arazzo?
Arazzo is an open standard for describing API workflows.

In an Arazzo file, you can describe, among other things:
 - Which steps the workflow must go through, and in what order.
 - What should happen when a step fails.
 - What should happen when a step succeeds.
 - How the output of a step can serve as input for the next step in the process.

Each step describes an API call or a call to another workflow.

## Why is Arazzo a good candidate for modeling OGC process recipes?
- Both operate based on API calls via HTTP.
- Arazzo is an open standard, with existing tools that can read a workflow file. This makes it easier to visualize or execute workflows without having to write separate tools.
- Because Arazzo operates at the HTTP level, you are not limited to calling OGC process endpoints. You can combine it with calls to any other HTTP endpoints.

## What are the disadvantages of Arazzo (in the context of OGC processes)?
Arazzo “communicates” in terms of HTTP requests, whereas within the domain of OGC processes, one speaks in terms of processes and jobs. As a result, the two domains do not have a one-to-one mapping:
- Since starting a process, retrieving the status of a job, and retrieving the results of a job are three different endpoints, they are also three separate steps in a workflow.
   - However, this can be abstracted away by defining a separate workflow that executes these steps, with the process ID and the inputs of the process to be invoked (as an untyped JSON object) as its inputs. The result of the process can be defined as the output of the workflow. See also the following chapter.
- Mapping outputs to inputs becomes impractical when it gets too complex. For example, when one process uses a different coordinate system than another process.
   - Within Arazzo, you can refer to the outputs of other processes using _runtime expressions_. However, these expressions are not expressive enough to define complex mapping steps.

Arazzo is a standard that is still young and under development.
- The standard itself does not yet feel fully mature. The first version (1.0.0) dates back to 2025, and the most recent version is from August 2026.
- As a result, the tools that use an Arazzo specification are also still in their early stages.
   - Some runners still have some teething problems:
      - [The Python module `arazzo-runner`](https://pypi.org/project/arazzo-runner/) appears to proceed to the next step even when the previous step failed.
      - [Redocly Respect](https://redocly.com/learn/arazzo/testing-arazzo-workflows) has a bug where importing workflows from other files causes a stack overflow.

## Invoking an OGC Process via a Workflow
At the heart is a workflow designed to trigger an OGC process and to make the results available as a workflow output.

This allows us to reuse the workflow for different processes, since the core functionality (_execute_, polling until the job is complete, retrieving the results) remains the same.

The specification for this workflow can be found in [the _spec_ subfolder](./spec/execute-process.yml).

The workflow specified in this Arazzo file follows these steps:
1. Trigger a process by calling the `/processes/{processId}/execute` endpoint. The `processId` is a workflow input. Save the job ID as output.
   - Use the inputs provided as workflow inputs as inputs for the process.
2. Call the job-status endpoint to retrieve the current status of the job, using the job ID from step 1.

   The next step depends on the status of the job (which can be determined from the body of the response):
   | Status  | Handler Type | Action |
   |---------|------------ --|-------|
   | _successful_ | Success | Proceed to the next step in the workflow (retrieve the results of the process) |
   | _running_ | Failure | Try again (up to _N_ times) |
   | _failed_ | Failure | End the workflow.
   | _dismissed_ | Failure | End the workflow.

3. Retrieve the job results by calling the `/jobs/{jobId}/results` endpoint. Refer to these results in the workflow's output description.
