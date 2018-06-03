### Installing Datadog Agent
For this exercise, I have installed the the datadog-agent on Mac OS X using this command given in the [installation instructions](https://app.datadoghq.com/signup/agent#mac) after signing up for my datadog account:

~~~
DD_API_KEY=<MY-API-KEY> bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_mac_os.sh)"
~~~

Once installed you should be able to open the Datadog Agent through the system tray. (cmd + space > Search datadog agent)

![datadog agent](datadog-agent.png)
### Add tags in the Agent config file and show us a screenshot of your host and its tags on the Host Map page in Datadog.

There are multiple methods of applying tags to your host including the use of integrations, the Datadog UI, the Datadog API, and your host configuration file. In order to assign a tag using the configuration file you must:

1. Open your datadog.yaml file in your text editor
2. Add tags to your tag dictionary following the required syntax shown in the [Datadog Documentation](https://docs.datadoghq.com/getting_started/tagging/assigning_tags/). Here we are using key:value tags.

![configuration file tags](./screenshots/assigning-tags.png)
3. Reset your Agent from your terminal using the following lines
~~~
datadog-agent restart
~~~

*Before accessing the hostmap, you may be prompted to enable WebGl if you are using Google Chrome. You can enable WebGL by*:

1. Typing [chrome://flags/](chrome://flags/) in the adress bar and hitting enter
2. Click enable under _WebGL Draft Extensions_
![webgl image](./screenshots/webgl_enable.png)

On the [host map page](https://app.datadoghq.com/infrastructure/map), you will see the tags assigned to your host by clicking on the hexagon representing your host.
![hostmap](./screenshots/hostmap.png)

### Install a database on your machine (MongoDB, MySQL, or PostgreSQL) and then install the respective Datadog integration for that database.

For this question I decided to install the PostgreSQL integration as most of the applications utilize Postgres already.

1. Click on the integrations tab from the top navbar of the Datadog app and search for `postgresql` to find the PostgreSQL integration.

![PostgreSQL search](/screenshots/search-int.png)

2. Click __Install__ and follow the steps under the Configuration tab of the modal that pops-up.

![psql1](./screenshots/psql1.png)
 - Generate your password and and run `psql` in your terminal (if you run into any issues with your postgresql database, refer to [this link](https://www.digitalocean.com/community/tutorials/how-to-use-roles-and-manage-grant-permissions-in-postgresql-on-a-vps--2#how-to-delete-roles-in-postgresqlhttps://www.digitalocean.com/community/tutorials/how-to-use-roles-and-manage-grant-permissions-in-postgresql-on-a-vps--2#how-to-delete-roles-in-postgresql) as you may need to create your database or complete initial setup)
 - Once `psql` has been successfully run, run step 1 of the configuration in your terminal.
 ~~~
 create user datadog with password <GENERATED-PASSWORD>;
grant SELECT ON pg_stat_database to datadog;
 ~~~
 - You can also test that the user has been successfully created with
 ~~~
 psql -h localhost -U datadog postgres -c "select * from pg_stat_database LIMIT(1);" && \
echo -e "\e[0;32mPostgres connection - OK\e[0m" || \
echo -e "\e[0;31mCannot connect to Postgres\e[0m"
 ~~~
 You should be prompted for your password
 - create a `conf.yaml` file in your `postgres.d` directory and add the code below (copied from conf.yaml.example in the same folder)

  ~~~
   init_config:

  instances:
     -   host: localhost
         port: 5432
         username: datadog
         password: <GENERATED-PASSWORD>

   ~~~

- Restart your agent and run `datadog-agent status` to make sure your integration is successfully executing. You should see something like this:

  ![postgres-term](./screenshots/postgres-term.png)
### Create a custom Agent check that submits a metric named my_metric with a random value between 0 and 1000.

In order to create an agent check, we need to create a .yaml file in our conf.d directory and a .py file in our checks.d directory (both files need to share the same name). In this example, I have created `randomvalue.yaml` and `randomvalue.py`

First we should test if the agent check is running properly by having it submit a simple metric. Here I used the check provided in the Datadog documentation:

__randomvalue.py__
~~~
from checks import AgentCheck
class HelloCheck(AgentCheck):
    def check(self, instance):
        self.gauge('hello.world', 1)
~~~
__randomvalue.yaml__
~~~
init_config:

instances:
    [{}]
~~~

Now, if you run `datadog-agent check <check-name>` in your terminal, you should see something like this:

~~~
~/.datadog-agent $ datadog-agent check randomvalue
=== Series ===
{
  "series": [
    {
      "metric": "hello.world",
      "points": [
        [
          1527821909,
          1
        ]
      ],
      "tags": null,
      "host": "Jamess-MBP-2.home",
      "type": "gauge",
      "interval": 0,
      "source_type_name": "System"
    }
  ]
}
=========
Collector
=========

  Running Checks
  ==============
    randomvalue
    -----------
      Total Runs: 1
      Metrics: 1, Total Metrics: 1
      Events: 0, Total Events: 0
      Service Checks: 0, Total Service Checks: 0
      Average Execution Time : 0ms


  Config Errors
  ==============
    docker
    ------
      Configuration file contains no valid instances

  Loading Errors
  ==============
    apm
    ---
      Core Check Loader:
        Could not configure check APM Agent: APM agent disabled through main configuration file

      JMX Check Loader:
        check is not a jmx check, or unable to determine if it's so

      Python Check Loader:
        No module named apm


Check has run only once, if some metrics are missing you can try again with --check-rate to see any other metric if available.

~~~

Let's configure our check to pass a different metric name and incorporate an instance of a random number. I have replaced `self.gauge('hello.world', 1)` with `self.gauge('my_metric', instance['number'])` and added an instance in my randomvalue.yaml file to reference.

![agent-check](./agentcheck.png)


### Change your check's collection interval so that it only submits the metric once every 45 seconds.

The collection interval is configured on the instance level. For our instance of `number` we can add the line `min_collection_interval: 45` to set our collectin interval to 45 seconds.

~~~
init_config:

instances:
  - number: 123
    min_collection_interval: 45

~~~


### Utilize the Datadog API to create a Timeboard that contains:

#### - Your custom metric scoped over your host.
To create my timeboard using the API I referred to the [API Docs](https://docs.datadoghq.com/api/?lang=ruby#timeboards) and wrote my script in ruby. Adjusting the sample provided in the docs, I created a timeboard including a graph for my_metric. To adjust this for your metric, change your app-key, api-key, and your query (`"q" =>`) call your metric.

~~~
require 'rubygems'
require 'dogapi'

api_key = '<API-Key>'
app_key = '<APP-KEY>'

dog = Dogapi::Client.new(api_key, app_key)

# Create a timeboard for PostgreSQL commits.
title = 'My First Metrics'
description = 'And they are marvelous.'
graphs = [
{
    "definition" => {
        "events" => [],
        "requests" => [{
            "q" => "avg:my_metric{*} by {host}"
        }],
        "viz" => "timeseries"
    },
    "title" => "My Metric"
}]

template_variables = [{
    "name" => "host1",
    "prefix" => "host",
    "default" => "host:my-host"
}]

#Create anomaly function

tags = ['app:webserver', 'frontend']

dog.create_dashboard(title, description, graphs, template_variables)

~~~

#### - Any metric from the Integration on your Database with the anomaly function applied.

For our integration metric, I have selected the postgres.commits metric. We can add a second graph in our `graphs` array and change the title and query to visualize our database commit data.

~~~~
graphs = [
{
    "definition" => {
        "events" => [],
        "requests" => [{
            "q" => "avg:my_metric{*} by {host}"
        }],
        "viz" => "timeseries"
    },
    "title" => "My Metric"
},
{
    "definition" => {
        "events" => [],
        "requests" => [{
            "q" => "avg:postgresql.commits{*}"
        }],
        "viz" => "timeseries"
    },
    "title" => "PostgreSQL Commits"
}]
~~~~

In order to add the anomaly function to our metric, we need to add a monitor. Again, I copied the sample API request to post a monitor and modified the code to use the anomaly function. To create a monitor, we will also need to provide `options` as shown in the snippet below

~~~
#Create anomaly function
options = {
    'notify_no_data' => true,
    'no_data_timeframe' => 20
}
tags = ['app:webserver', 'frontend']
dog.monitor("metric alert", "avg(last_4h):anomalies(avg:postgresql.commits{*}, 'basic', 2, direction='both', alert_window='last_5m', interval=60, count_default_zero='true') >= 1", : name => "Postgres commits", :message => "More commits than usual", : tags => tags, : options => options)

~~~

__Note__: *The structure of the monitor:
`dog.monitor(type, query, name, tag, options)` and the structure of the query can differ depending on the type. [More info on queries in API requests](https://docs.datadoghq.com/api/?lang=ruby#create-a-monitor)*

#### - Your custom metric with the rollup function applied to sum up all the points for the past hour into one bucket

And we can create another monitor for our custom metric by requesting a similar monitor but changing the query to specify the sum of all the points for `my_metric` in the past hour
~~~
dog.monitor("metric alert", "avg(last_1h):anomalies(avg:my_metric{*} by {host}, 'basic', 2, direction='both', alert_window='last_5m', interval=60, count_default_zero='true').rollup(sum,60) >= 1", :name => "My_Metric", :message => "Sum of points", :tags => tags, :options => options)
~~~
####  *Please be sure, when submitting your hiring challenge, to include the script that you've used to create this Timemboard.*

#### Once this is created, access the Dashboard from your Dashboard List in the UI:

- Set the Timeboard's timeframe to the past 5 minutes

  *I will return to this question as I seem to be limited to setting the time frame to 1hour at the lowest*
- Take a snapshot of this graph and use the @ notation to send it to yourself.

  Clicking the camera icon will allow you to snapshot and send my graph to yourself.
![snapshot](snapshot.png)
![send](send.png)
- Bonus Question: What is the Anomaly graph displaying?
The Anomaly is displaying 
