```python
from fastapi import FastAPI

import requests, os, json, statistics, re, logging, sys, random

logging_dir = os.environ["LOGGING_DIR"]

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)
formatter = logging.Formatter("%(asctime)s - %(levelname)s %(message)s")

file_handler = logging.FileHandler(filename="{0}/sla.log".format(logging_dir),mode="a")
file_handler.setFormatter(formatter)
logger.addHandler(file_handler)

app = FastAPI()

class AvailabilityPart:
  def __init__(self, parser, application, name, nrql):
    self.parser = parser
    self.application = application
    self.name = name
    self.nrql = nrql
    self.value = -999999

class Availability:
  def __init__(self, application, name, metrics, value):
    self.application = application
    self.name = name
    self.metrics = metrics
    self.value = value

def request(method, uri, headers, data=None):
  if method == "GET":
    return requests.get(uri, headers = headers).text
  elif method == "POST":
    requests.post(uri, headers = headers, data=json.dumps(data))

def query(api_key, account_id, data):
  path = "http://insights-api.eu.newrelic.com"
  uri = "{0}/v1/accounts/{1}/query?nrql={2}".format(path, account_id, data)

  headers = {
    "Accept": "application/json",
    "X-Query-Key": api_key
  }

  return request("GET", uri, headers)

def insert(api_key, account_id, data):
  path = "https://insights-collector.eu01.nr-data.net"
  uri = "{0}/v1/accounts/{1}/events".format(path, account_id)

  headers = {
    "Content-Type": "application/json",
    "X-Insert-Key": api_key
  }

  request("POST", uri, headers, data)

def obj_dict(obj):
  return obj.__dict__

class ServiceLevelAgreement:
  def __init__(self, application, filename):
    self.application = application
    self.filename = filename

    if self.application == "Aaa Application":
      self.nr_insert_api_key = os.environ["NR_INSERT_API_KEY_EC"]
      self.nr_query_api_key = os.environ["NR_QUERY_API_KEY_EC"]
      self.nr_account_id = os.environ["NR_ACCOUNT_ID_EC"]

  def parse_trues(self, raw_json):
    parsed_json = json.loads(raw_json)

    results = []

    for i in parsed_json['current']['timeSeries']:
      for j in i['results']:
        results.append(j['result'])

    for i in parsed_json['previous']['timeSeries']:
      for j in i['results']:
        results.append(j['result'])

    return (sum(bool(x) for x in results)/len(results))*100

  def parse_single(self, raw_json):
    return json.loads(raw_json)['results'][0]['result']

  def calculate(self):
    monitors = []

    with open("./entry/{0}".format(self.filename)) as file:
      for line in file:
        splited = re.split("\t+", line)
        monitors.append(AvailabilityPart(splited[0], splited[1], splited[2], splited[3]))

    if monitors:
      for i in monitors:
        if i.parser == "single":
          i.value =  self.parse_single(query(self.nr_query_api_key, self.nr_account_id, i.nrql))
        elif i.parser == "trues":
          i.value = self.parse_trues(query(self.nr_query_api_key, self.nr_account_id, i.nrql))

      availability = round(statistics.mean(m.value for m in monitors), 5)
    else:
      availability = random.uniform(0, 100)

    data = { "eventType": "InsightSla", "slaAvailability": availability }

    print(self.nr_insert_api_key)

    insert(self.nr_insert_api_key, self.nr_account_id, data)

    logger.info("SLA {0} {1}".format(self.application, availability))
    monitors.append(Availability(self.application, "SLA", len(monitors), availability))

    return json.dumps(monitors, default=obj_dict)

@app.get("/api/aaa-application")
def api_ecommerce():
  availability = ServiceLevelAgreement("Aaa Application", "aaa-application.csv")
  return availability.calculate()
```
