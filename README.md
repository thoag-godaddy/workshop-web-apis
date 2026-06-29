# Web Based API Design and Fundamentals
## Introduction
This is a companion resource to Web Based APIs workshop. This is not required for that, but if you'd like to follow along with a hands on approach please use this resource.

## Tools and Resources
### Tools used
* curl - Probably already on your system. All you need.
* telnet - Optional - might have to install (Mac) or enable (Windows)
* jq - Optional. Just formats and prints things nicer
* Postman - Not used today, but great HTTP client with a GUI and developer tooling

### APIs used
* https://www.weather.gov/documentation/services-web-api
* https://icanhazdadjoke.com/api
* Other fun & free APIs:
  * NASA, Spotify, Youtube, oMDB, Twitter, GitHub
  * https://github.com/public-apis/public-apis
  * https://apilist.fun/

### API design links
* [REST white paper](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
* https://restfulapi.net/
* [Status Codes](https://www.restapitutorial.com/httpstatuscodes.html) or [With Cats](https://http.cat/)
* [OpenAPI Spec](https://swagger.io/specification/)
* [Swagger API Editor](https://editor.swagger.io/)

## HTTP
#### Telnet HTTP
Use telnet to establish a basic network connection on HTTP port 80.
```bash
telnet google.com 80
GET /
```
Google is forgiving, but most APIs require headers
```bash
telnet icanhazdadjoke.com 80
GET /
```
results in the server terminating the session

#### cURL and HTTPS
Let's switch to curl, the most common HTTP client there is. It is probably already installed on your command line. Let's request a dad joke with curl and we will tell curl to be verbose so we can see the headers and more.
```bash
curl --verbose https://icanhazdadjoke.com/
```
Notice the default header of `accept: */*` which means we don't care what kind of response media we get. The server decided to send `Content-type: text/plain` as a result. Let's try some other options. Also instead of verbose let's use `-i` to get just the response body and headers.
```bash
curl -i -H "Accept: text/html" https://icanhazdadjoke.com/
curl -i -H "Accept: application/json" https://icanhazdadjoke.com/
```

## Media
Previously we negotiated for the content-type we wanted using the accept header. If we ask for structured JSON data we can use `|` ("pipe") to pass data to a program like `jq`, which can easily process the data returned.
```bash
curl -H "Accept: application/json" https://icanhazdadjoke.com/ | jq .
curl -H "Accept: application/json" https://icanhazdadjoke.com/ | jq .id
```
Imagine a program that saved your favorite dad jokes. You'd want to easily be able to process their ids to save. Extracting the ID seems trivial but most api data is much more complex. For example, I can ask weather.gov for my local forecast. It returns a lot of complex data but because it is structured I can easily extract the current temperature.
```bash
curl https://api.weather.gov/gridpoints/PSR/167,51/forecast
curl https://api.weather.gov/gridpoints/PSR/167,51/forecast | jq .properties.periods[0].temperature
```
The web is filled with all types of media, so content negotiation is critical. Let's look at a joke I saved earlier. The [documentation](https://icanhazdadjoke.com/api) says I can fetch images of jokes (and more!), but if I try to do that with a program like curl that expects text, bad things happen.
```bash
curl https://icanhazdadjoke.com/j/LuciNmbaMmb
curl -i --output - https://icanhazdadjoke.com/j/LuciNmbaMmb.png
```
Remember HTTP just sends bytes; it is the client's job to process the data. cURL can't do that, but your browser can.
## API Design

### History
Making a curl request using SOAP xml.
```bash
curl --location 'https://www.dataaccess.com/webservicesserver/NumberConversion.wso' \
--header 'Content-Type: text/xml; charset=utf-8' \
--data '<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <NumberToWords xmlns="http://www.dataaccess.com/webservicesserver/">
      <ubiNum>500</ubiNum>
    </NumberToWords>
  </soap:Body>
</soap:Envelope>'
```

### CRUD Example

#### Setup
To illustrate a RESTful API, we will use a simple To Do List example with todoist.com. You can follow along if you like, but you'll need an API key so you can perform token-authentication when making requests. Obtaining one for simple use is free:
1. [Sign up for an account](https://todoist.com/auth/signup)
2. Verify your email and sign in
3. [Navigate to their Developer section](https://todoist.com/app/settings/integrations/developer)
    - You may get a popup asking if you would like to pay for Pro. Exit out of this and do not pay
4. Select "Copy API Token"
5. In your (*nix) terminal run `export MYAPITOKEN=PASTE_YOUR_TOKEN_HERE`. 

***Checkpoint:*** Run:
```bash
echo $MYAPITOKEN
```
You should get back a randomized string.

#### GETting Resources

Let's build a GET request based on the Todoist API. First, look at the [Get Projects](https://developer.todoist.com/api/v1/#tag/Projects/operation/get_projects_api_v1_projects_get) documentation.

Let's try to get a list of our projects.

```bash
curl -i -X GET "https://api.todoist.com/api/v1/projects"
```

ToDoist didn't seem to like that. We used `-i`, so we can see the headers we sent. Based on the 401 response code and the lack of an `Authorization` header (and the fact that we didn't send any auth), that is the most likely culprit.

Let's add authentication:

```bash
curl -i -X GET https://api.todoist.com/api/v1/projects -H "Authorization: Bearer ${MYAPITOKEN}"
```

Much better. Because I got a 200 response I know the server accepted my AuthN via the Authorization header. It is common for the header to start with the authentication scheme which in this case is Bearer auth. Bearer auth is typically just a way to say token auth.

The response body above was a collection of project resources. We should be able to request the specific resource by ID. [Documentation](https://developer.todoist.com/api/v1/#tag/Projects/operation/get_project_api_v1_projects__project_id__get)

```bash
# Store a project ID in an environment variable
MYPROJECT=<your_project_id>

# Fetch the info for the project
curl -i -X GET "https://api.todoist.com/api/v1/projects/${MYPROJECT}" -H "Authorization: Bearer ${MYAPITOKEN}"
```
However, if your ID is wrong you will get the HTTP 404 not found status. 

***Checkpoint:*** Run:
```
curl -i -X GET "https://api.todoist.com/api/v1/projects/${MYPROJECT}" -H "Authorization: Bearer ${MYAPITOKEN}"
```
You should get back a dictionary representing a ToDoist project.

#### POSTing Resources

Next let's CREATE a new project using POST. Based on the [documentation](https://developer.todoist.com/api/v1/#tag/Projects/operation/create_project_api_v1_projects_post) the route should be the same as our GET collection, just using the POST method:

```bash
curl -i -X POST https://api.todoist.com/api/v1/projects -H "Authorization: Bearer ${MYAPITOKEN}"
```
We missed some important information. Let's try again with the required arguments. We will add:
- A request body including the required paramters, using the `--data` argument
- A `Content-Type` header to tell the API *how* we are sending that information: `application/json`

```bash
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/api/v1/projects"  --data '{"name": "API Skill Goals"}'
```

We should see a 201 response code telling us a resource has been created in our collection of projects. We will use this project ID going forward. We are going to set a bash terminal variable with it, just like we did above. Sometimes you want to see the environment variables you have set. You can list them with the env command and use grep to filter for the ones you want.

```bash
export MYPROJECT=ID_FROM_ABOVE
env | grep MY
```

***Checkpoint:*** Run:
```
curl -i -X GET "https://api.todoist.com/api/v1/projects/${MYPROJECT}" -H "Authorization: Bearer ${MYAPITOKEN}"
```
You should get back a dictionary representing a ToDoist project with name `API Skill Goals`.

#### Consuming Resource Data

Next let's go look at the documentation for tasks: https://developer.todoist.com/api/v1/#tag/Tasks. 

We can see all the properties for a task and how to interact with the API. We can add some tasks to that project and then list our collection of tasks. 

```bash
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/api/v1/tasks"  --data '{"content": "Consume APIs", "project_id": "'"$MYPROJECT"'"}'

curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/api/v1/tasks"  --data '{"content": "Build APIs", "project_id": "'"$MYPROJECT"'"}'

curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/api/v1/tasks"  --data '{"content": "Profit", "project_id": "'"$MYPROJECT"'"}'

curl -i -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/api/v1/tasks
```

This gives us a list of tasks. Let's retrieve the information for just one of them using the [Get Tasks By Filter](https://developer.todoist.com/api/v1/#tag/Tasks/operation/get_tasks_by_filter_api_v1_tasks_filter_get) endpoint.

```bash
export CONSUME=$(curl -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/api/v1/tasks/filter?query="search:Consume" | jq -r '.results.[0].id')
```

This is not a REST-compliant endpoint. `filter` is an intent, not a resource or collection. However, it is useful.

Let's do this for each task, and store the result in an environment variable. The `$(...)` means: "execute this command and return the value":

```bash
export CONSUME=$(curl -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/api/v1/tasks/filter?query="search:Consume" | jq -r '.results.[0].id')

export PROFIT=$(curl -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/api/v1/tasks/filter?query="search:Profit" | jq -r '.results.[0].id')

export BUILD=$(curl -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/api/v1/tasks/filter?query="search:Build" | jq -r '.results.[0].id')
```

***Checkpoint:*** Run:
```bash
curl -i -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/api/v1/tasks/${CONSUME}
```
You should get back a dictionary representing a ToDoist task with the name `Consume APIs`.

#### Modifying Resources

Let's try updating a task and see our change take effect. A REST-compliant API would likely use a `PATCH` or `PUT` to modify a resource, but reading the [documentation](https://developer.todoist.com/api/v1/#tag/Tasks/operation/update_task_api_v1_tasks__task_id__post) we can see that we will actually use a POST for our update.

```bash
# UPDATE (not really very RESTful)
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/api/v1/tasks/$PROFIT"  --data '{"priority": 4}'

# GET (read) resource
curl -i -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/api/v1/tasks/$PROFIT"
```

You can see the changes happening in your [project](https://app.todoist.com/app/projects/active) and also using the API itself. We can also delete and mark a task done with the close API.

```bash
# DELETE method/verb
curl -i -H "Authorization: Bearer ${MYAPITOKEN}"  -X DELETE "https://api.todoist.com/api/v1/tasks/$BUILD"

# Not RESTful - Intent or action
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}" "https://api.todoist.com/api/v1/tasks/$CONSUME/close"
```

***Checkpoint:*** Run:
```bash
curl -i -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/api/v1/tasks
```
Here, you will see the Consume APIs task is closed. The Build APIs task is deleted. And the Profit task is set to priority=4.

We have successfully tested the CRUD endpoints of this API, but the interface wasn't quite as expected. REST is a pattern which you should know and do your best to follow, but it's not a rigid requirement.

