# Automatic database update and restore with AWS Lambda functions

The road map to cloud adoption can be difficult to set and implement, but once it is completed it offers a lot of flexibility, a lot of space for continuous improvement and creativity for building new solutions.
Having automated processes helps your company to focus on what is important for the business and lets the developers experiment more and efficienlty optimize their work. 
One of the latest things I've been working on involves a task that needed to be continued after the client's solution was already moved to the cloud. What I liked about it was that it is not only taking the advantage of being in the cloud, it also involves serverless technology that I consider to be the next level of cloud solutions.

## Scenario
Our customer has 2 different AWS accounts:
 * Acceptance
 * Production

and all the changes on the databases in Production account are needed in Acceptance account every 2 weeks. The databases used are AWS DocumentDB and AWS Aurora.
The old way of doing this with the on-prem infrastructure was to manually run some commands and update the databases.
The process should be done in the maintenance window which implies working in the night/early morning when nobody is using the databases.
The big advantage of having the solution in the cloud is that it can easly be automated using services that bring low or no cost at all, and that this solution needs no human interaction.

## Solution overview
The solution we agreed on was to split the process in 2 main parts having 2 AWS Lambda functions:
 * first one, *db-update-latest-snapshot-id*, will mainly focus on preparing the environment:
    - copy the shared DocumentDB snapshot (because we can’t restore from it directly, see [this](https://docs.aws.amazon.com/documentdb/latest/developerguide/backup_restore-share_cluster_snapshots.html) article)
    - check the latest copied snapshot for DocumentDB and the latest shared snapshot for Aurora
    - update the latest snapshot id parameters in AWS SSM Parameter Store 
 * the second one, *db-restore-from-snapshot*, will mainly focus on triggering the update and restore process:
    - will copy the latest snapshot ID value to the current snapshot ID and then trigger the main pipeline to deploy the changes and restore the databasess from the latest snapshots
I took advantage of the already existing AWS Lambda functions that are sharing the database snapshots from one account to the other. The sharing part is not in the scope of this article but it was ilustrated for a better view of the solution.

In AWS SSM Parameter Store, I have created 5 parameters:

 * aurora_current_snapshot_id
 * aurora_latest_snapshot_id
 * documentdb_current_snapshot_id 
 * documentdb_latest_snapshot_id
 * main_cicd_name

The current value will be the one from which the databases are restored from and the latest value will be the value of the most recent snapshot that exists in the account for DocumentDB or Aurora.
We need to have 2 different parameters, one for the current and one for the latest snapshot, to be able to restore the databases from a specific snapshot ID outside of this process.
There are 2 pipeline in the account, so the name of the main pipeline, the one that is deploying the infrastructure was needed.

The entire solution was written using IaC in AWS CDK with Python.

![Architecture](db_autorestore)
