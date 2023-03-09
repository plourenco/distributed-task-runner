# Mailchimp data importer

Build a system to run thousands of import jobs that will:

1. Retrieve contact data from Mailchimp for a client account.
2. Convert the contact data into the format accepted by Ometria’s API and import it to Ometria under the client’s account for processing/storage.

## How to run

There is a docker compose file with three services:

- rabbitmq: the message queue (may take a while to boot as it depends on a health-check)
- celery_worker: the service workers (with 5 replicas)
- celery_beat: the task scheduler

Running `docker compose up -d` will bring up all services and start scheduling the task.

![Untitled](https://user-images.githubusercontent.com/12183954/224170447-894f94f9-6b83-403a-874b-eb7b9a7e39f0.png)

A manual import can also be run using the python CLI:

```python
list_key = '1a2d7ebf82'

from app import run
run(list_key)

# or to avoid a long running import (from now onwards)
from datetime import datetime
run(list_key, datetime.now())
```

![Untitled-1](https://user-images.githubusercontent.com/12183954/224170484-3abedba8-c3e9-47e4-8865-a397fb4b4e0b.png)

For the sake of simplicity, as the external API seems to have no concept of per-client contact data but instead a heterogenous list of records, I simplified the implementation to import a list at a time. However, if each record or message contained a notion of `list_key`, supporting multiple lists should become trivial.

## Inspecting the queue

While the service is running, the RabbitMQ image in use contains a management UI on http://localhost:15672/#/queues useful to inspect the queue status.

![imagem](https://user-images.githubusercontent.com/12183954/224170797-15b13dd9-67b1-47d9-8113-1ca7c079cc91.png)

![imagem](https://user-images.githubusercontent.com/12183954/224170857-cfe1e5ee-baf5-4d69-af00-fccd7ddf3083.png)

At the time of writing, I unfortunately got blocked by Mailchimp API so testing a full sync was limited:

```html
<p>We blocked your request because the IP address you’re using looks suspicious. This issue will usually 
  resolve itself after a short period of time, and you can try your request again. You can also try
  using a different IP address to see if that resolves the issue. <br><br>
  If you need additional help, you can try one of these <a
  href="https://mailchimp.com/help/mailchimp-support-options/#How_to_contact_technical_support">support
		options</a>.
</p>
```

## Architecture and implementation

The core architecture consists on a service defined as `importer_task_wrapper` responsible for retrieving a rolling window of records to import and exporting into an API. The service expects to consume a queue message containing:

```json
{
 "list_key": "1a2d7ebf82",
  "offset": 0,
  "count": 1000,
  "since_last_changed": date
}
```

The import is scheduled as a period task to run every hour (the container `celery_beat` is responsible for running scheduled tasks). When an import starts, the task will retrieve the last occurrence from a database conceived to log import runs, this will determine if we should begin a full-sync or an incremental from the last sync-point onwards.

A set of messages are then created (or, in other words, tasks) to import a window (from `offset` and `offset + count`) of records from Mailchimp into the internal API. As Mailchimp is limited to 10 concurrent requests and 1000 records per request, horizontal scaling is limited.

Celery, the distributed task queue in use, groups the tasks and logs their status in a database backend which is used to determine when all records have been processed and the run can be logged as completed.

![Diagrama sem nome drawio(1)](https://user-images.githubusercontent.com/12183954/224170294-36e894bd-7cdf-4e0f-82e4-9f97e1752209.png)

## Error handling

In order to increase the resilience of a single-run in case a sub-set of the records to import fail and we end up in an inconsistent state, both retries and a dead letter queue have been implemented.

The retry mechanism retries the same task 3 times with an interval of 5 seconds, after which, if no acknowledgement (`ACK`) is given by the service, the message will be sent into a dead letter queue for later inspection or ingestion into another service.

In order to be able to reject messages after a certain amount of retries, the services have been configured to give a *late `ACK`* , that is, after completion. When a service starts processing a message, it will become hidden from the queue until `ACK` is given. If no `ACK` is given for a period of 30 minutes (by default in RabbitMQ - but also a sensible threshold in comparison with the retry interval or processing time), the message will become visible again.