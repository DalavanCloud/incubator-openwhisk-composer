# Compose Command

The `compose` command makes it possible to encode and deploy compositions.
```
compose
```
```
Usage:
  compose composition.js[on] command [flags]
Commands:
  --json                 output the json representation for the composition (default command)
  --deploy NAME          deploy the composition with name NAME
  --encode               output the conductor action code for the composition
Flags:
  --apihost HOST         API HOST
  -u, --auth KEY         authorization KEY
  -i, --insecure         bypass certificate checking
```
The `compose` command requires either a Javascript file that evaluates to a composition (for example [demo.js](../samples/demo.js)) or a JSON file that encodes a composition (for example [demo.json](../samples/demo.json)). The JSON format is documented in [FORMAT.md](FORMAT.md).

The `compose` command has three mode of operation:
- By default or when the `--json` option is specified, the command returns the composition encoded as a JSON dictionary.
- When the `--deploy` option is specified, the command deploys the composition with the desired name.
- When the `--encode` option is specified, the command returns the Javascript code for the [conductor action](https://github.com/apache/incubator-openwhisk/blob/master/docs/conductors.md) for the composition.

## Encoding

By default, the `compose` command returns the composition encoded as a JSON dictionary:
```
compose demo.js
```
```json
{
    "actions": [
        {
            "name": "/_/authenticate",
            "action": {
                "exec": {
                    "kind": "nodejs:default",
                    "code": "function main({ password }) { return { value: password === 'abc123' } }"
                }
            }
        },
        {
            "name": "/_/success",
            "action": {
                "exec": {
                    "kind": "nodejs:default",
                    "code": "function main() { return { message: 'success' } }"
                }
            }
        },
        {
            "name": "/_/failure",
            "action": {
                "exec": {
                    "kind": "nodejs:default",
                    "code": "function main() { return { message: 'failure' } }"
                }
            }
        }
    ],
    "composition": [
        {
            "type": "if",
            "test": [
                {
                    "type": "action",
                    "name": "/_/authenticate"
                }
            ],
            "consequent": [
                {
                    "type": "action",
                    "name": "/_/success"
                }
            ],
            "alternate": [
                {
                    "type": "action",
                    "name": "/_/failure"
                }
            ]
        }
    ]
}
```
The evaluation context for the Javascript code includes the `composer` object implicitly defined as:
```javascript
composer = require('@ibm-functions/composer')
```

## Deployment

The `--deploy` option makes it possible to deploy a composition (Javascript or JSON) given the desired name for the composition:
```
compose demo.js --deploy demo
```
```
ok: created actions /_/authenticate,/_/success,/_/failure,/_/demo
```
Or:
```
compose demo.js > demo.json
compose demo.json --deploy demo
```
```
ok: created actions /_/authenticate,/_/success,/_/failure,/_/demo
```
The `compose` command synthesizes and deploys a conductor action that implements the
composition with the given name. It also deploys the composed actions if
definitions are provided for them as part of the composition.

The `compose` command outputs the list of deployed actions or an error result. If an error occurs during deployment, the state of the various actions is unknown.

The `compose` command deletes the deployed actions before recreating them if necessary. As a result, default parameters, limits, and annotations on preexisting actions are lost.

### Configuration

Like the OpenWhisk CLI, the `compose` command supports the following flags for specifying the OpenWhisk deployment to use:
```
 --apihost HOST         API HOST
  -u, --auth KEY        authorization KEY
  -i, --insecure        bypass certificate checking
```
If the `--apihost` flag is absent, the environment variable `__OW_API_HOST` is used in its place. If neither is available, the `compose` command attempts to obtain the api host from the whisk property file for the current user.

If the `--auth` flag is absent, the environment variable `__OW_API_KEY` is used in its place. If neither is available, the `compose` command attempts to obtain the authorization key information from the whisk property file for the current user.

The default path for the whisk property file is `$HOME/.wskprops`. It can be altered by setting the `WSK_CONFIG_FILE` environment variable.

## Code generation

The `compose` command returns the code of the conductor action for the composition (Javascript or JSON) when invoked with the `--encode` option.
For instance, the conductor action code for the [demo.js](../samples/demo.js) composition is [demo-conductor.js](../samples/demo-conductor.js):
```
compose demo.js --encode > demo-conductor.js
```
This code may be deployed using the OpenWhisk CLI:
```
wsk action create demo demo-conductor.js -a conductor true
```
```
ok: created action demo
```
The conductor action code does not include definitions for nested actions or compositions.