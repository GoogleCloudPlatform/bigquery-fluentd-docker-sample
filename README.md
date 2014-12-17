# Fluentd + Google BigQuery Getting Started Sample

This sample explains how to set up a [Fluentd](http://www.fluentd.org/) + [Google BigQuery](https://cloud.google.com/bigquery/) integration in a [Docker](https://www.docker.com/) container that sends [nginx](http://nginx.org/en/) web server access log to the BigQuery in real time with [fluent-plugin-bigquery](https://github.com/kaizenplatform/fluent-plugin-bigquery). The whole process may take only 20 - 30 minutes with the following steps:

- Sign Up for Google Cloud Platform and BigQuery Service
- Creating a dataset and table on Google BigQuery
- Run nginx + Fluentd on Google Compute Engine (GCE) in a Docker container
- Execute BigQuery query
- Using BigQuery Dashboard built with Google Sheets

## Sign Up for Google Cloud Platform

(You can skip this section if you have already set up a Google Cloud Project)

- If you don't already have one, sign up for a [Google account](https://accounts.google.com/SignUp).
- Go to the [Google Developers Console](https://console.developers.google.com/project).
- Select your target project. If you want to create a new project, click on Create Project.
- BigQuery is automatically enabled in new projects. To activate BigQuery in a pre-existing project, click APIS & AUTH in the left navigation, then click APIs. Navigate to BigQuery API. If the status indicator says OFF, click the indicator once to switch it to ON.
- Set up billing. BigQuery offers a free tier for queries, but other operations require billing to be set up before you can use the service.
- Open BigQuery Browser Tool (linked under the Big Data section of your project console)
- Click `COMPOSE QUERY` button at top left and execute the following sample query with the tool to check you can access BigQuery.

```
SELECT title FROM [publicdata:samples.wikipedia] WHERE REGEXP_MATCH(title, r'.*Query.*') LIMIT 100
```

## Creating a dataset and table on Google BigQuery

To create a dataset and table, you need to install `bq` command tool included in Cloud SDK. 

- Download and install the [Cloud SDK](https://cloud.google.com/sdk/).
- Authenticate your client by running:

```
$ gcloud auth login
```

- Set the project you are working on with the project ID noted earlier

```
$ gcloud config set project <YOUR PROJECT ID>
```

- Create a dataset `bq-test` by executing the following command:

```
$ bq mk bq_test
```

- `cd` into the directory for this repository if you are not already.

```
cd bigquery-fluentd-docker-sample
```

- Execute the following command to create the table `access_log`.

```
$ bq mk -t bq_test.access_log ./schema.json
```

- Reload the BigQuery Browser Tool page, select your project, `bq_test` dataset and `access_log` table. Confirm that the table has been created with the specified schema correctly.

## Creating a Google Compute Engine instance

- First you need to enable the "Google Compute Engine" API under the APIs
  section of the APIs & Auth area of the project console.

- Run the following command to create a GCE instance named `bq-test`. This will take around 30 secs.

For more information about the features of GCE instances and their features,
see the [product documentation](https://cloud.google.com/compute/docs/instances)

```
$ gcloud compute instances create "bq-test" \
--zone "us-central1-a"  \
--machine-type "n1-standard-1"  \
--network "default" \
--maintenance-policy "MIGRATE"  \
--scopes storage-ro bigquery \
--image container-vm-v20140929 \
--image-project google-containers
```

## Run nginx + Fluentd with a Docker container

- Enter the following command to log in to the GCE instance.

``` 
$ gcloud compute ssh bq-test --zone=us-central1-a
```

- In the GCE instance, run the following command (replace `YOUR_PROJECT_ID` with your project id). This will start downloading a Docker image `kazunori279/fluentd-bigquery-sample` which contains nginx web server with Fluentd.

```
$ sudo docker run -e GCP_PROJECT="YOUR_PROJECT_ID" -p 80:80 -t -i -d kazunori279/fluentd-bigquery-sample
```

This will launch and run a docker container preconfigured with nginx and fluentd. The contents of this Docker container are described in more detail below. We now want to generate some page views so that we can verify that fluentd is sending data to BigQuery.

- Open [Google Developers Console](https://console.developers.google.com/project) in a browser, choose your project and select `Compute` - `Compute Engine` - `VM instances`.

- Find `bq-test` GCE instance and click it's external IP link. On the dialog, select `Allow HTTP traffic` and click `Apply` to add the firewall rule. There will be an Activities dialog shown in the bottom right of the window with a message `Updating instance tags for "bq-test"`. (tags are used to associate firewall rules)

- After updating, click the external IP link again to direct your browser to hit the nginx server on the instance. It will show a blank web page titled "Welcome to nginx!". Click reload button several times.


## Execute BigQuery query

- Open BigQuery Browser Tool, click `COMPOSE QUERY` and execute the following query. You will see the requests from browser are recorded on access_log table (it may take a few minutes to receive the very first log entries from fluentd).

```
SELECT * FROM [bq_test.access_log] LIMIT 1000
```

*Note: If you are moving quickly, the first query results may be empty.
A BigQuery table has a warm-up time for the very first inserts to appear. Once this is
done, subsequent inserts appear in results very quickly.*

That's it! You've just confirmed that nginx access log events are collected by Fluentd, imported into BigQuery and visible in the Browser Tool. You may use Apache Bench tool or etc to hit the web page with more traffic to see how Fluentd + BigQuery can handle high volume logs in real time. It can support up to 10K rows/sec by default (and you can extend it to 100K rows/sec by requesting).

## Using BigQuery Dashboard

Using Google Sheets and the BigQuery connector, you can create a BigQuery Dashboard which lets you easily store and visualize queries and have them update periodically automatically (e.g. every minute, hour, or day).

### Features

- The dashboard is a **Google Spreadsheet**: hosted for free by [Google Sheets](http://www.google.com/sheets/about/)
- It is easy to customize and integrate with your business process even non-programmers. 
- It is just a matter of copying the spreadsheet, click some buttons as described in Getting Started, and then it's ready to use.

- Easy **Big Data** analytics with BigQuery: you can execute BigQuery query just by entering a SQL on a sheet. The Dashboard will automatically execute it every minute/hour and draw a chart from the result. 
  
![gsod_graph.png](images/gsod_graph.png)

![gsod_query.png](images/gsod_query.png)

### Setup

To start using BigQuery Dashboard, follow the instruction below.

1. Open [this spreadsheet](https://docs.google.com/spreadsheets/d/1Xwk2icyXH2DmVIZC33SAs5bs012ZIt0-goyX0dZZu7s/edit) and select `File` - `Make a copy` menu to make a copy of it
2. Copy the URL of the copied spreadsheet to clipboard
3. Select `Tools` - `Script editor...` menu
4. On the Script editor, open `bq_query.gs`. Paste the copied URL on the place of `<<PLEASE PUT YOUR SPREADSHEET URL HERE>>`. Select `File` - `Save` menu to save the file
5. Paste Project ID of your Google Cloud Platform project on the place of `<<PLEASE PUT YOUR PROJECT ID HERE>>`. Select `File` - `Save` menu to save the file
6. Select `Resources` - `Advanced Google services` menu and turn on `BigQuery API`
7. Click `Google Developers Console` link on the dialog. This will show a list of APIs. Find `BigQuery API` and toggle permissions widget from `OFF` to `ON` to enable access. You should see `BigQuery API` on the `Enabled APIs` list on the top of the page, if not enable that here.
8. Close the Console, click `OK` button in the dialog

### Execute a sample query:

Now it's ready to execute BigQuery queries from the spreadsheet. Use the following instructions to try executing a sample query.

1. Open the spreadsheet and open `BQ Queries` sheet. The sheet has a sample BQ query named `gsod_temparature_LINE` which aggregates temparature data of each year from the public GSOD dataset available on BigQuery
2. Select `Dashboard` - `Run All BQ Queries` menu. When run the first time, it will show a dialog `Authorization Required`. Click `Continue` button and then `Accept` button
3. There will be a `gsod_template` sheet added. Open the sheet and check there are the query results
4. Open the `BigQuery Dashboard` sheet and check there is a graph added for the query results.

### Query the fluentd data

- Add the following query on `BQ Queries` sheet with a query name `access_log_LINE` and interval `1` min.

```
SELECT
  STRFTIME_UTC_USEC(time * 1000000, "%Y-%m-%d %H:%M:%S") as tstamp, 
  count(*) as rps
FROM bq_test.access_log
GROUP BY tstamp ORDER BY tstamp DESC;
```

*Note: paste this query into the top data entry box, not the cell, otherwise the
lines of the Query may paste each into their own cell*

### Automatic query execution:


If you want to execute the queries periodically, use the following instructions.

1. Open the Tools > Script editor and select `Resources` - `Current project's triggers`
2. Click `Click here to add one now`
3. Select `runQueries` for `Run` menu, and select `Time-driven` `Minutes timer` `Every minute` for `Events`, and click `Save`
4. Go back to `BQ Queries` sheet, set `1` to the `interval (min)` column of the `gsod_temperature_LINE` query

With this setting, the queries will be executed once every minute. Set `0` to the `interval (min)` to disable the periodic execution.


### Simulating load

- (assuming you are using Mac OS or Linux) Open a local terminal and execute the following command to execute Apache Bench to hit the nginx server with simulated traffic. Replace `YOUR_EXTERNAL_IP` with the external IP of the GCE instance.

```
ab -c 100 -n 1000000 http://YOUR_EXTERNAL_IP/
```

- Open the `BigQuery Dashboard` and select `Dashboard` - `Run All BQ Queries` on the menu to see the graph `access_log` drawn on the dashboard sheet. If you followed the steps for automatic execution you will see this graph refresh every minute.

![access_log graph](images/access_log_graph.png)

- Stop the Apache Bench command by pressing `Ctrl+C`.

### Notes:

- When the spreadsheet executes a query with a new query name, it creates a new sheet with the query name
- If the query name has a suffix `_AREA`, `_BAR`, `_COLUMN`, `_LINE`, `_SCATTER`, or `_TABLE`, it will also create a new sheet with the specified chart
- If the query name has a suffix `_AREA_STACKED`, `_BAR_STACKED` or `_COLUMN_STACKED`, it will create a stacked chart
- Put `LIMIT 100` at the end of each query to limit the lines of query result to 100. Otherwise it may throw an error when the results exceed the limit
- The first field of the query results should be timestamp or date value to draw the chart chronologically

## Inside Dockerfile and td-agent.conf

If you take a look at the [Dockerfile](Dockerfile), you can learn how the Docker container has been configured. After preparing an Ubuntu image, it installs Fluentd, nginx and the [fluent-plugin-bigquery](https://github.com/kaizenplatform/fluent-plugin-bigquery).

```
FROM ubuntu:12.04
MAINTAINER kazunori279-at-gmail.com

# environment
ENV DEBIAN_FRONTEND noninteractive
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list

# update, curl, sudo
RUN apt-get update && apt-get -y upgrade
RUN apt-get -y install curl 
RUN apt-get install sudo

# fluentd
RUN curl -O http://packages.treasure-data.com/debian/RPM-GPG-KEY-td-agent && apt-key add RPM-GPG-KEY-td-agent && rm RPM-GPG-KEY-td-agent
RUN curl -L http://toolbelt.treasuredata.com/sh/install-ubuntu-precise-td-agent2.sh | sh 
ADD td-agent.conf /etc/td-agent/td-agent.conf

# nginx
RUN apt-get install -y nginx
ADD nginx.conf /etc/nginx/nginx.conf

# fluent-plugin-bigquery
RUN /usr/sbin/td-agent-gem install fluent-plugin-bigquery --no-ri --no-rdoc -V

# start fluentd and nginx
EXPOSE 80
ENTRYPOINT /etc/init.d/td-agent restart && /etc/init.d/nginx start && /bin/bash
```

In the [td-agent.conf](td-agent.conf) file, you can see how to configure it to forward Fluentd logs to fluent-plugin-bigquery. It's as simple as the following:

```
<match nginx.access>
  type bigquery
  auth_method compute_engine

  project "#{ENV['GCP_PROJECT']}"
  dataset bq_test
  tables access_log

  time_format %s
  time_field time
  fetch_schema true
  field_integer time
</match>
```

Since you are running the GCE instance within the same GCP project of BigQuery dataset, you don't have to copy any private key file to the GCE instance for OAuth2 authentication. ` "#{ENV['GCP_PROJECT']}"` refers to your project id passed through the environment variable you gave as an argument when starting the docker container.

You can also use fluentd to send data from compute instances running outside of
the project, or outside of GCE entirely. To authorize the fluentd BigQuery plugin, you will need the
private key and email for a Google API [Service Account](https://developers.google.com/accounts/docs/OAuth2ServiceAccount).

- Go to your project in the Developer Console and open the Credentials section
  of APIs & auth project console.
- Choose to "Create new Client ID" and choose "Service Account"
- Download the install the private key with fluentd on the host, and use the
  account email in the [fluentd plugin settings](https://github.com/kaizenplatform/fluent-plugin-bigquery#authentication).
- Note that this service account has access to the resources of the project, so
  it should only be distributed on trusted machines.

## Cleaning Up

- Execute the following command to delete GCE instance.

```
gcloud compute instances delete bq-test --zone=us-central1-a
```

- On BigQuery Browser Tool, click the drop down menu of `bq_test` dataset and select `Delete dataset`

## License

* See [LICENSE](LICENSE)
