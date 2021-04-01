![build+test](https://github.com/izar/pytm/workflows/build%2Btest/badge.svg)


# pytm: A Pythonic framework for threat modeling

![pytm logo](docs/pytm-logo.svg)

## Introduction

Traditional threat modeling too often comes late to the party, or sometimes not at all. In addition, creating manual data flows and reports can be extremely time-consuming. The goal of pytm is to shift threat modeling to the left, making threat modeling more automated and developer-centric.

## Features

Based on your input and definition of the architectural design, pytm can automatically generate the following items:
- Data Flow Diagram (DFD)
- Sequence Diagram
- Relevant threats to your system

## Requirements

* Linux/MacOS
* Python 3.x
* Graphviz package
* Java (OpenJDK 10 or 11)
* [plantuml.jar](http://sourceforge.net/projects/plantuml/files/plantuml.jar/download)

## Getting Started

The `tm.py` is an example model. You can run it to generate the report and diagram image files that it references:

```
mkdir -p tm
./tm.py --report docs/template.md | pandoc -f markdown -t html > tm/report.html
./tm.py --dfd | dot -Tpng -o tm/dfd.png
./tm.py --seq | java -Djava.awt.headless=true -jar $PLANTUML_PATH -tpng -pipe > tm/seq.png
```

There's also an example `Makefile` that wraps all these into targets that can be easily shared for multiple models. If you have [GNU make](https://www.gnu.org/software/make/) installed (available by default on Linux distros but not on OSX), simply run:

```
make
```

To avoid installing all the dependencies, like `pandoc` or `Java`, the script can be run inside a container:

```
# do this only once
export USE_DOCKER=true
make image

# call this after every change in your model
make
```


## Usage

All available arguments:

```text
tm.py [-h] [--debug] [--json] [--dfd] [--report REPORT] [--exclude EXCLUDE] [--seq] [--list] [--describe DESCRIBE] [--sqldump DBNAME]

optional arguments:
  -h, --help           show this help message and exit
  --debug              print debug messages
  --dfd                output DFD (default)
  --report REPORT      output report using the named template file (sample template file is under docs/template.md)
  --exclude EXCLUDE    specify threat IDs to be ignored
  --seq                output sequential diagram
  --list               list all available threats
  --describe DESCRIBE  describe the properties available for a given element
  --sqldump DBNAME     dumps all threat model elements and findings into the named sqlite file (erased if exists)
  --json               output a JSON file

```

Currently available elements are: TM, Element, Server, ExternalEntity, Datastore, Actor, Process, SetOfProcesses, Dataflow, Boundary and Lambda.

The available properties of an element can be listed by using `--describe` followed by the name of an element:

```text

(pytm) ➜  pytm git:(master) ✗ ./tm.py --describe Element
Element class attributes:
  OS
  definesConnectionTimeout        default: False
  description
  handlesResources                default: False
  implementsAuthenticationScheme  default: False
  implementsNonce                 default: False
  inBoundary
  inScope                         Is the element in scope of the threat model, default: True
  isAdmin                         default: False
  isHardened                      default: False
  name                            required
  onAWS                           default: False

```

## Creating a Threat Model

The following is a sample `tm.py` file that describes a simple application where a User logs into the application
and posts comments on the app. The app server stores those comments into the database. There is an AWS Lambda
that periodically cleans the Database.

```python

#!/usr/bin/env python3

from pytm.pytm import TM, Server, Datastore, Dataflow, Boundary, Actor, Lambda

tm = TM("my test tm")
tm.description = "another test tm"
tm.isOrdered = True

User_Web = Boundary("User/Web")
Web_DB = Boundary("Web/DB")

user = Actor("User")
user.inBoundary = User_Web

web = Server("Web Server")
web.OS = "CloudOS"
web.isHardened = True

db = Datastore("SQL Database (*)")
db.OS = "CentOS"
db.isHardened = False
db.inBoundary = Web_DB
db.isSql = True
db.inScope = False

my_lambda = Lambda("cleanDBevery6hours")
my_lambda.hasAccessControl = True
my_lambda.inBoundary = Web_DB

my_lambda_to_db = Dataflow(my_lambda, db, "(&lambda;)Periodically cleans DB")
my_lambda_to_db.protocol = "SQL"
my_lambda_to_db.dstPort = 3306

user_to_web = Dataflow(user, web, "User enters comments (*)")
user_to_web.protocol = "HTTP"
user_to_web.dstPort = 80
user_to_web.data = 'Comments in HTML or Markdown'

web_to_user = Dataflow(web, user, "Comments saved (*)")
web_to_user.protocol = "HTTP"
web_to_user.data = 'Ack of saving or error message, in JSON'

web_to_db = Dataflow(web, db, "Insert query with comments")
web_to_db.protocol = "MySQL"
web_to_db.dstPort = 3306
web_to_db.data = 'MySQL insert statement, all literals'

db_to_web = Dataflow(db, web, "Comments contents")
db_to_web.protocol = "MySQL"
db_to_web.data = 'Results of insert op'

tm.process()

```

### Generating Diagrams

Diagrams are output as [Dot](https://graphviz.gitlab.io/) and [PlantUML](https://plantuml.com/).

When `--dfd` argument is passed to the above `tm.py` file it generates output to stdout, which is fed to Graphviz's dot to generate the Data Flow Diagram:

```bash

tm.py --dfd | dot -Tpng -o sample.png

```

Generates this diagram:

![dfd.png](.gitbook/assets/dfd.png)

Adding ".levels = [1,2]" attributes to an element will cause it (and its associated Dataflows if both flow endings are in the same DFD level) to render (or not) depending on the command argument "--levels 1 2".

The following command generates a Sequence diagram.

```bash

tm.py --seq | java -Djava.awt.headless=true -jar plantuml.jar -tpng -pipe > seq.png

```

Generates this diagram:

![seq.png](.gitbook/assets/seq.png)

### Creating a Report

The diagrams and findings can be included in the template to create a final report:

```bash

tm.py --report docs/template.md | pandoc -f markdown -t html > report.html

```
The templating format used in the report template is very simple:

```text

# Threat Model Sample
***

## System Description

{tm.description}

## Dataflow Diagram

![Level 0 DFD](dfd.png)

## Dataflows

Name|From|To |Data|Protocol|Port
----|----|---|----|--------|----
{dataflows:repeat:{{item.name}}|{{item.source.name}}|{{item.sink.name}}|{{item.data}}|{{item.protocol}}|{{item.dstPort}}
}

## Findings

{findings:repeat:* {{item.description}} on element "{{item.target}}"
}

```

To group findings by elements, use a more advanced, nested loop:

```text
## Findings

{elements:repeat:{{item.findings:if:
### {{item.name}}

{{item.findings:repeat:
**Threat**: {{{{item.id}}}} - {{{{item.description}}}}

**Severity**: {{{{item.severity}}}}

**Mitigations**: {{{{item.mitigations}}}}

**References**: {{{{item.references}}}}

}}}}}
```

All items inside a loop must be escaped, doubling the braces, so `{item.name}` becomes `{{item.name}}`.
The example above uses two nested loops, so items in the inner loop must be escaped twice, that's why they're using four braces.

### Overrides

You can override attributes of findings (threats matching the model assets and/or dataflows), for example to set a custom CVSS score and/or response text:

```python
user_to_web = Dataflow(user, web, "User enters comments (*)", protocol="HTTP", dstPort="80")
user_to_web.overrides = [
    Finding(
        # Overflow Buffers
        id="INP02",
        CVSS="9.3",
        response="""**To Mitigate**: run a memory sanitizer to validate the binary""",
    )
]
```

## Threats database

For the security practitioner, you may supply your own threats file by setting `TM.threatsFile`. It should contain entries like:

```json
{
   "SID":"INP01",
   "target": ["Lambda","Process"],
   "description": "Buffer Overflow via Environment Variables",
   "details": "This attack pattern involves causing a buffer overflow through manipulation of environment variables. Once the attacker finds that they can modify an environment variable, they may try to overflow associated buffers. This attack leverages implicit trust often placed in environment variables.",
   "Likelihood Of Attack": "High",
   "severity": "High",
   "condition": "target.usesEnvironmentVariables is True and target.sanitizesInput is False and target.checksInputBounds is False",
   "prerequisites": "The application uses environment variables.An environment variable exposed to the user is vulnerable to a buffer overflow.The vulnerable environment variable uses untrusted data.Tainted data used in the environment variables is not properly validated. For instance boundary checking is not done before copying the input data to a buffer.",
   "mitigations": "Do not expose environment variable to the user.Do not use untrusted data in your environment variables. Use a language or compiler that performs automatic bounds checking. There are tools such as Sharefuzz [R.10.3] which is an environment variable fuzzer for Unix that support loading a shared library. You can use Sharefuzz to determine if you are exposing an environment variable vulnerable to buffer overflow.",
   "example": "Attack Example: Buffer Overflow in $HOME A buffer overflow in sccw allows local users to gain root access via the $HOME environmental variable. Attack Example: Buffer Overflow in TERM A buffer overflow in the rlogin program involves its consumption of the TERM environmental variable.",
   "references": "https://capec.mitre.org/data/definitions/10.html, CVE-1999-0906, CVE-1999-0046, http://cwe.mitre.org/data/definitions/120.html, http://cwe.mitre.org/data/definitions/119.html, http://cwe.mitre.org/data/definitions/680.html"
 }
```

The `target` field lists classes of model elements to match this threat against.
Those can be assets, like: Actor, Datastore, Server, Process, SetOfProcesses, ExternalEntity,
Lambda or Element, which is the base class and matches any. It can also be a Dataflow that connects two assets.

All other fields (except `condition`) are available for display and can be used in the template
to list findings in the final [report](#report).

> **WARNING**
>
> The `threats.json` file contains strings that run through `eval()`. Make sure the file has correct permissions
> or risk having an attacker change the strings and cause you to run code on their behalf.

The logic lives in the `condition`, where members of `target` can be logically evaluated.
Returning a true means the rule generates a finding, otherwise, it is not a finding.
Condition may compare attributes of `target` and also call one of these methods:

* `target.oneOf(class, ...)` where `class` is one or more: Actor, Datastore, Server, Process, SetOfProcesses, ExternalEntity, Lambda or Dataflow,
* `target.crosses(Boundary)`,
* `target.enters(Boundary)`,
* `target.exits(Boundary)`,
* `target.inside(Boundary)`.

If `target` is a Dataflow, remember you can access `target.source` and/or `target.sink` along with other attributes.

Conditions on assets can analyze all incoming and outgoing Dataflows by inspecting
the `target.input` and `target.output` attributes. For example, to match a threat only against
servers with incoming traffic, use `any(target.inputs)`. A more advanced example,
matching elements connecting to SQL datastores, would be `any(f.sink.oneOf(Datastore) and f.sink.isSQL for f in target.outputs)`.

## Currently supported threats

```text
UART01 - External Entity Unauthenticated User Potentially Denies Receiving Data
UART02 - Spoofing of the Unauthenticated User External Destination Entity
WIFI01 - Spoofing of the Attacker External Destination Entity
WIFI02 - External Entity Attacker Potentially Denies Receiving Data
CLOUD01 - Data Logs from an Unknown Source
CLOUD02 - Lower Trusted Subject Updates Logs
CLOUD03 - Data Store Inaccessible
CLOUD04 - Data Flow Generic Data Flow Is Potentially Interrupted
CLOUD05 - Data Store Denies Cloud Server Potentially Writing Data
CLOUD06 - The Cloud Server Data Store Could Be Corrupted
CLOUD07 - Spoofing of Destination Data Store Cloud Server
PC01 - Data Logs from an Unknown Source
PC02 - Lower Trusted Subject Updates Logs
PC03 - Authenticated Data Flow Compromised
PC04 - Data Store Inaccessible
PC05 - Data Flow Generic Data Flow Is Potentially Interrupted
PC06 - Data Store Denies PC Potentially Writing Data
PC07 - The PC Data Store Could Be Corrupted
PC08 - Spoofing of Destination Data Store PC
ICSP01 - Spoofing of the Unauthenticated User External Destination Entity




```
