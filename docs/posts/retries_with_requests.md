# Retrying HTTP requests with Python `requests` library

The Python `requests` library is among the most popular today. According to GitHub, **2.3** million users 😱 are using it. One of the primary reasons for its widespread adoption is its user-friendly API.

By default, the requests made with the `requests` library are not "**retrying**", this means that once a request is sent and it fails for any reason (like network issues, server timeouts, or temporary outages), the library won't automatically attempt to send the request again.

Sometimes we want to ensure reliable communication between systems. For example, in e-commerce platforms, when a customer places an order, it's essential that the order details are accurately communicated to inventory systems, payment gateways, and shipping providers. Any failure in these communications can result in missed shipments, incorrect billing, or overselling of inventory. In such critical scenarios, merely logging an error isn't enough; implementing *retries* becomes crucial to ensure that business operations run smoothly and customer trust is maintained.

## How can `requests` library help us implement *retries* ?

The first thing to know is that the `requests` library is built on top of [urllib3](https://urllib3.readthedocs.io/en/stable/). This foundational library, [urllib3](https://urllib3.readthedocs.io/en/stable/), has built-in support for connection pooling and ***retries***, which makes it a robust choice for web requests. Leveraging this underlying functionality, `requests` can be configured to implement retries with a big variety of features.

To provide the retrying capabilities to `requests` we have to import the `Retry` class from `urllib3` library like this:


```python
from urllib3.util import Retry # (1)!
```

1. If you installed `requests`, `urllib3` is already installed too.

now, let's explore what attributes `Retry` class accepts:

???+ note

    For a better description of `Retry` attributes, please go to https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html#urllib3.util.Retry.

```python
Retry(
    total=10,
    connect=None,
    read=None,
    redirect=None,
    status=None,
    other=None,
    allowed_methods=frozenset({'DELETE', 'GET', 'HEAD', 'OPTIONS', 'PUT', 'TRACE'}),
    status_forcelist=None,
    backoff_factor=0,
    backoff_max=120,
    raise_on_redirect=True,
    raise_on_status=True,
    history=None,
    respect_retry_after_header=True,
    remove_headers_on_redirect=frozenset({'Authorization'}),
    backoff_jitter=0.0
)
```

1. **total** (int): Total number of retries to allow.
2. **connect** (int): How many connection-related errors to retry on.
3. **read** (int): How many times to retry on read errors.
4. **redirect** (int): How many redirects to perform. Limit this to avoid infinite redirect loops.
5. **status** (int): How many times to retry after bad status codes.
6. **other** (int): How many times to retry on other types of errors.
7. **allowed_methods** (frozenset): Set of uppercased HTTP method verbs that we should retry on.
8. **status_forcelist** (list): A set of integer HTTP status codes that we should force a retry on.
9. **backoff_factor** (float): A backoff factor to apply between attempts. urllib3 will sleep for: ```{backoff factor} * (2 **({number of previous retries}))``` seconds. For instance, with `backoff_factor` set to `0.5`, the sleep times would be `[0s, 0.5s, 1s, 2s, 4s, …]`.
10. **backoff_max** (int): Maximum sleep time between retries.
11. **raise_on_redirect** (bool): Whether or not to raise an error on redirects.
12. **raise_on_status** (bool): Whether or not to raise an error on status.
13. **history** (tuple): The history of the request encountered during each call to `increment()`.
14. **respect_retry_after_header** (bool): Whether or not to respect the Retry-After header in responses.
15. **remove_headers_on_redirect** (Collection): Sequence of headers to remove from the request when a response indicating a redirect is returned before firing off the redirected request.
16. **backoff_jitter** (float): The amount of jitter (randomness) to apply to the delay time before the next retry, given as a fraction of the computed delay time.

Now that we know the `Retry` class from *urllib3* and gained a foundational understanding of its attributes, it's time to integrate it with the `requests` library. To achieve this, we'll utilize the `Session` class from `requests` library. After creating a session, we can then attach our retry logic by mounting an adapter (`HTTPAdapter`) to it. Here's how it's done:

```python
from urllib3.util import Retry

from requests import Session
from requests.adapters import HTTPAdapter


session = Session()
retries = Retry() # (1)!

session.mount('https://', HTTPAdapter(max_retries=retries)) # (2)!
```

1. If we leave out the Retry's arguments, by default, it'll retry the requests 10 times inmediately in case of connection or read errors.
2. If you want to make requests using the `http` protocol you need to mount another HTTPAdapter like this: `session.mount('http://', HTTPAdapter(max_retries=retries))`.

The above code means that when we make an HTTP request using the `session` variable, it will have the retry logic we set up previously in the `Retry` class.

### Retrying in case of gettting status codes

Consider a scenario where you're making an HTTP request to a service and you want to retry the request in case you get a **500**(indicating server errors) and **503**(often indicating service unavailability) status codes. For this scenario, you could set up a *retry* logic using `requests` like so:

```python linenums="1"
from urllib3.util import Retry
from urllib3 import add_stderr_logger

from requests import Session
from requests.adapters import HTTPAdapter

add_stderr_logger() # (1)!


session = Session()
retries = Retry(
    total=None,
    connect=False,
    read=False,
    status=3,
    backoff_factor=1,
    status_forcelist=[503, 500],
)
session.mount('https://', HTTPAdapter(max_retries=retries))

resp = session.get("https://my_sevice_url.com")
```

1. This function help us see the logs for each retry on the stderror.

The above code initiates an HTTP GET request to `https://my_service_url.com`. If the server responds with a status code found in `status_forcelist`, the request will be retried 3 times. The wait times between retries follow this pattern: `[0s, 2s, 4s]`. However, the delay won't exceed the value specified in `backoff_max` (you can change this value if needed). If the server's response header includes the `Retry-After` key, its value will override the calculated wait time based on the backoff factor (You can disable this behaviour setting `respect_retry_after_header` to `False`). In the above retry logic, it's important to note that if we exhaust all retries, we will receive a `RetryError` exception. Additionally, if for any reason we receive a status code during a retry that's not in the list `status_forcelist` (e.g., 403), the retries will stop immediately, and we will encounter the exception associated with that specific status code.


???+ warning

    You might be curious about why I set `connect` and `read` to `False`. The reason is that if these parameters are left at their default values, in the event of a connect or read error, the retries will be attempted indefinitely.I also set `total` to `None`, which causes the retry logic to fall back on the `read`, `connect`, or `status` counters.

### Retry HTTP POST method

By default, retryable HTTP methods include **HEAD**, **GET**, **PUT**, **DELETE**, **OPTIONS**, and **TRACE**. However, if you need to retry a **POST** request, you must update the `allowed_methods` parameter in the `Retry` configuration. See the example below for guidance.


```python

# ... previous imports

session = Session()
retries = Retry(
    total=None,
    connect=False,
    read=False,
    status=3,
    backoff_factor=1,
    status_forcelist=[503, 500],
)
session.mount('https://', HTTPAdapter(max_retries=retries))
resp = session.post("https://my_service_url.com", data={"key": "value"})

# ... more code
```

You can include any **HTTP** methods of your choice and specify the status codes you wish to retry on.


## To Sum up!

As demonstrated, it's straightforward to implement retry logic for HTTP requests using the `requests` library. This is made possible by the `Retry` class from urllib3, which offers extensive configuration options. Experiment with parameters not covered in this post and see how they can fit into your use cases.

Moreover, remember not to over-retry, as this can increase load on servers or even be perceived as a DDoS attack. Also, consider logging retries. When they occur, it's crucial for debugging.

There are other retry libraries out there that you can use to implement retries for your requests, I will drop a list of some that I know:

1. **Backoff** - https://github.com/litl/backoff
2. **Tenacity** - https://github.com/jd/tenacity
3. **Stamina** - https://github.com/hynek/stamina

All of them use the decorator strategy and have some nice features that you can try.