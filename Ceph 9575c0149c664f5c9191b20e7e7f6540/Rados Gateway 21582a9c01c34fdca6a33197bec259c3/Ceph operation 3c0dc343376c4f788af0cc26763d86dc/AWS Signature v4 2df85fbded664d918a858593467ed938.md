# AWS Signature v4

# 1. Overview

Authentication with Signature v4 provide some or all of the following feature.

- Verification of identity of the request
- In-transist data protection
- Protect against reuse of the signed portions of the request : Sniffer who can access to a signed request can modify the unsigned portion of the request, and reuse it within 15 minutes of validity of the request.

## 1.1 Authentication methods

### 1.1.1 HTTP Authorization header

Add the Authorization field to the header in order to add authorization information.

### 1.1.2 Query string parameters

Add authorization to query string URL.

![Untitled](AWS%20Signature%20v4%202df85fbded664d918a858593467ed938/Untitled.png)

# 2. Example code

```bash
from botocore import crt
import requests
from botocore.awsrequest import AWSRequest
from botocore.credentials import Credentials
import botocore.session
import boto3
service = 's3'
region = '*'
method = 'GET'
url = 'http://192.168.0.142:8080/sld-bucket?list-type=2&encoding-type=url'

session = boto3.Session(aws_access_key_id='FA81SJJDTLFHL3EUDB4G', aws_secret_access_key='GuwnEpWJJrbUHjFBItTPA6FqBa5bbyXD7ZNki3xU')
signer = crt.auth.CrtS3SigV4Auth(session.get_credentials(), service, region)
request = AWSRequest(method='GET', url=url)
request.context["payload_signing_enabled"] = False
signer.add_auth(request)
prepped = request.prepare()
r = requests.get(prepped.url, headers=prepped.headers)
print(f'status_code: {r.status_code} \nobject text: {r.text}')
```

![Untitled](AWS%20Signature%20v4%202df85fbded664d918a858593467ed938/Untitled%201.png)

![Untitled](AWS%20Signature%20v4%202df85fbded664d918a858593467ed938/Untitled%202.png)

![Untitled](AWS%20Signature%20v4%202df85fbded664d918a858593467ed938/Untitled%203.png)

![Untitled](AWS%20Signature%20v4%202df85fbded664d918a858593467ed938/Untitled%204.png)

Try to convert it again

![Untitled](AWS%20Signature%20v4%202df85fbded664d918a858593467ed938/Untitled%205.png)