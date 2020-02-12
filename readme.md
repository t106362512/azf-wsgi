# azf-wsgi - WSGI apps on Azure Functions

**DEPRECATION NOTICE**: The Azure Functions team has built [first-class support for WSGI](https://github.com/Azure/azure-functions-python-library/pull/45) on Azure Functions.
I recommend you switch to their implementation.

---

This was an adapter package to let you run WSGI apps (Django, Flask, etc.) on Azure Functions.

Example:
```python
import azure.functions as func

# note that the package is "azf-wsgi" but the import is "azf_wsgi"
from azf_wsgi import AzureFunctionsWsgi
# Django, for example, but works with any WSGI app
from my_django_app.wsgi import application


def main(req: func.HttpRequest, context: func.Context) -> func.HttpResponse:
    return AzureFunctionsWsgi(application).main(req, context)
```

## Usage

### Install Azure Functions
Follow the instructions [here](https://docs.microsoft.com/azure/azure-functions/functions-create-first-function-python) to get set up locally.
I created a Function called "DjangoTrigger", but you can call yours whatever.

### Install your WSGI app
I found it's easiest if you package your WSGI app using a setup.py script, then `pip install` it.
If you don't want to do that, you'll have to make sure your WSGI entrypoint is importable from the module where you define your Azure Function.
I'm no Python imports expert, so I just added `sys.path.insert(0, './my_proj')` right before I try to import the package.

### Install this package
`pip install azf-wsgi` - no need to put this in Django's `INSTALLED_APPS` or anything like that.
Be sure to update `requirements.txt` to include `azf-wsgi` as a requirement.

### Configure Azure Functions to hand off to your WSGI app
First, we want to delegate routing to your WSGI app. Edit your `function.json` to include a catch-all route called "{*route}":

```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": [
        "get",
        "post"
      ],
      "route": "app/{*route}"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
```

I also didn't want the default 'api/' path on all my routes, so I fixed my `host.json` to look like this:

```json
{
    "version":  "2.0",
    "extensions": {
        "http": {
            "routePrefix": ""
        }
    }
}
```

Without this configuration, the only paths your WSGI app would ever see would start with "api/\<FunctionName\>/".
That works, but it would require you to repeat those boilerplate prefixes on every route you configured.
**However**, you don't want to completely take over all routes (by having an empty `routePrefix` _and_ a catch-all route in your function) because this disables important Azure machinery.

Finally, setup your Function's `__init__.py` to delegate to the WSGI adapter:

```python
import azure.functions as func

from azf_wsgi import AzureFunctionsWsgi
from my_django_app.wsgi import application


def main(req: func.HttpRequest) -> func.HttpResponse:
    return AzureFunctionsWsgi(application).main(req)
```

The adapter optionally takes the `Context` object as well:

```python
def main(req: func.HttpRequest, context: func.Context) -> func.HttpResponse:
    return AzureFunctionsWsgi(application).main(req, context)
```


The adapter will stuff in the OS's environment block much like a CGI request. If for some reason you don't want that, you can pass `False` to `include_os_environ`:

```python
def main(req: func.HttpRequest, context: func.Context) -> func.HttpResponse:
    return AzureFunctionsWsgi(application, False).main(req, context)
```
