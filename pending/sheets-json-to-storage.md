# Serving JSON from Google Sheets with Cloud Storage

Google Sheets is a convenient way for keeping track and editing light
weight data.  It is extensible with Apps Script, which is basically
JavaScript hosted on Google's servers.  It offers many ready libraries
for more user oriented Google services and the rest can be accessed
with OAuth2.  I'll work a simple example of how Sheets can be used
with Cloud Storage, to generate some JSON data that can be accessed
publicly via HTTPS.

I've set up a [simple example project]() that you can use to follow
along with this blog post.  Apps Script offers its own online editor
but I recommend using [clasp](https://github.com/google/clasp).  A
sheet's Apps Script editor can be opened in Tools > Script editor in
its menu bar.  Check your script id from project settings tab and with
it you can upload the example script with `clasp push your_id`.  First
off, let's define a simple function that grabs the contents of a
sheet.

```
function readSheet() {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var sheet = ss.getActiveSheet();
    var range = sheet.getDataRange();
    var values = range.getValues();
    var header = [];
    var output = [];

    for (var j = 0; j < values[0].length; ++j) {
	header.push(values[0][j]);
    }
    for (var i = 1; i < values.length; ++i) {
	row = {};
	for (var j = 0; j < values[i].length; ++j) {
	    if (!values[i][j]) {
		row = false;
		break;
	    }
	    row[header[j]] = values[i][j];
	}
	if (row) {
	    output.push(row);
	}
    }

    return output;
}
```

The first row of the sheet defines the record names for the generated
JSON, use something like `date` and `msg` for them.  So far so good.
What we'll need is a trigger that gets this data and stores it to a
bucket.  For this, I'm using a [service
account](https://cloud.google.com/iam/docs/service-accounts).  With
it, the trigger can use the minimal amount of permissions to access
the bucket.  I'll go through the steps in Cloud Console's IAM page to
define one.  First off, create a new role with
`storage.objects.create` and `storage.objects.delete` permissions.
IAM's "add permissions" interface doesn't offer those directly but
select "Storage Object Admin" and it should list the roles then.

Create a service account on the service account tab and create and
download a key for it.  On the IAM tab, give the service account the
role you created and add a condition for it:

```
resource.type == "storage.googleapis.com/Object" &&
resource.name == "projects/_/buckets/YOUR_BUCKET/objects/FILE"
```

Where `YOUR_BUCKET` and `FILE` depend on your environment.  For
reference, [the format for resource names is documented
here](https://cloud.google.com/iam/docs/conditions-resource-attributes)
and the CEL linter won't validate it for you.

Let's take what we have and see how to put it in use in the script.
I'm using [OAuth2 for Apps
Script](https://github.com/googleworkspace/apps-script-oauth2) library
for this.

```
var PRIVATE_KEY = "-----BEGIN PRIVATE KEY-----\n ...";
var CLIENT_EMAIL "your-service-account@your-project.iam.gserviceaccount.com";
var BUCKET = "your-bucket";
var FILE = "your-file.json"

function getService() {
    return OAuth2.createService("CloudStorate")
        .setTokenUrl('https://oauth2.googleapis.com/token')
	.setPrivateKey(PRIVATE_KEY)
	.setIssuer(CLIENT_EMAIL)
        .setPropertyStore(PropertiesService.getScriptProperties())
	.setScope("https://www.googleapis.com/auth/devstorage.read_write");
}
```

Finally, let's define the actual trigger function which creates the
JSON data and sends it to the bucket.  Running `clasp` won't set this
trigger for you but you'll need to create it yourself on the Apps
Script editor page.  It'll ask permissions for the Apps Script to use
`UrlFetchApp`.

```
function editTrigger(e) {
    var output = readSheet();
    var outputStr = "--boundary\n" +
	"Content-Type: application/json; charset=UTF-8\n\n" +
	JSON.stringify({'name':FILE, 'cacheControl': 'no-store'})+ "\n" +
	"--boundary\n" +
	"Content-Type: application/json; charset=UTF-8\n\n" +
	JSON.stringify(output) +
	"\n--boundary--\n";
    var service = getService();
    var url = "https://storage.googleapis.com/upload/storage/v1/b/"+BUCKET+"/o?name="+FILE+"&uploadType=multipart";
    var options = {
	'method': 'post',
	'contentType': 'multipart/related; boundary=boundary',
	'headers': {
	    'Authorization': 'Bearer ' + service.getAccessToken()
	},
	'payload': outputStr
    };
    var response = UrlFetchApp.fetch(url, options);
    Logger.log("Response status "+response.getResponseCode());
}
```

This goes a bit beyond basics.  The default caching period is 60
minutes and for the sake of making testing this a bit more convenient
I've disabled caching with `cacheControl`.  For reference, [here's the
documentation on cloud storage's REST
API](https://cloud.google.com/storage/docs/json_api/v1/objects/insert).

The final bit of making this work is the `appsscript.json` file.
`clasp` should be taking care of this for you.

```
{
  "timeZone": "Europe/Helsinki",
  "dependencies": {
    "libraries": [
      {
        "userSymbol": "OAuth2",
        "version": "40",
        "libraryId": "1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF"
      }
    ]
  },
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8"
}
```
