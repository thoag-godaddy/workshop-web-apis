# Web Based API Design and Fundamentals
## Introduction
This is a companion resource to Web Based APIs workshop. This is not required for that, but if you'd like to follow along with a hands on approach please use this resource.

## Tools and Resources
### Tools used
* curl - Probably already on your system. all you need
* telnet - optional - might have to install(mac) or enable(windows)
* jq - Optional. Just formats and prints things nicer
* postman - Not used, but great HTTP client with a GUI and developer tooling

### Apis used
* https://www.weather.gov/documentation/services-web-api
*  https://icanhazdadjoke.com/api
* Other fun & free APIs: NASA, Spotify, Youtube, oMDB, Twitter, GitHub

### Api design links
* [REST white paper](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
* https://restfulapi.net/
* https://www.restapitutorial.com/httpstatuscodes.html
* https://swagger.io/specification/

## HTTP
#### Telnet HTTP
Use telnet to establish a basic network connection on port http port 80.
```
telnet google.com 80
GET /
```
Google is forgiving, but most APIs require headers
```
telnet icanhazdadjoke.com 80
GET /
```
results in a 400 status "Bad Request"

#### cURL and HTTPS
Let's switch to curl, the most common http client there is. It is probably already installed on your command line. Let's request a dad joke with curl and we will tell curl to be verbose so we can see the headers and more.
```
curl --verbose https://icanhazdadjoke.com/
```
Notice the default header of `accept: */*` which means we don't care what kind of response media we get. The server decided to send `Content-type: text/plain` as a result. Let's try some other options. Also instead of verbose let's use `-i` to get just the response body and headers.
```
curl -i -H "Accept: text/html" https://icanhazdadjoke.com/
curl -i -H "Accept: application/json" https://icanhazdadjoke.com/
```

## Media
Previously we negotiated for the content-type we wanted using the accept header. If we ask for structured JSON data then a program like `jq` can easily process the data returned.
```
curl -H "Accept: application/json" https://icanhazdadjoke.com/ | jq .
curl -H "Accept: application/json" https://icanhazdadjoke.com/ | jq .id
```
Imagine a program that saved your favorite dad jokes. You'd want to easily be able to process their ids to save. Extracting the ID seems trivial but most api data is much more complex. For example I can ask weather.gov for my local forecast. It returns a lot of complex data but because it is structured I can easily extract the current temperature.
```
curl https://api.weather.gov/gridpoints/PSR/167,51/forecast
curl https://api.weather.gov/gridpoints/PSR/167,51/forecast | jq .properties.periods[0].temperature
```
The web is filled with all types of media so content negotiation is critical. Let's look at a joke I saved earlier. The documentation says I can fetch images of jokes, but If I try to do that with a program like curl that expects text and bad things happen.
```
curl https://icanhazdadjoke.com/j/kbUv4T0Ddxc
curl --output - https://icanhazdadjoke.com/j/kbUv4T0Ddxc.png
```

## API Design

### History
Making a curl request using SOAP xml. (Wont work with windows CMD, try postman instead)
```
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
To illustrate a RESTful api we will use a simple To Do List example with todoist.com. You can follow along if you like, but you'll need an API key so you can perform token-authentication when making requests. Obtaining one for simple use is free:
1. [Sign up for an account](https://todoist.com/auth/signup)
2. Verify your email and sign in
3. [Navigate to their Developer section](https://todoist.com/app/settings/integrations/developer)
4. Select "Copy API Token"
5. In your (*nix) terminal run `export MYAPITOKEN=PASTE_TOKEN_HERE` or substitute manually below

At this point we are basically follow their [Getting Started](https://developer.todoist.com/rest/v2/#getting-started) tutorial. First, lets get our todo projects and try out our authentication method

```
curl -i -X GET https://api.todoist.com/rest/v2/projects -H "Authorization: Bearer ${MYAPITOKEN}"
```
I'll be using the -i so that curl displays the response headers which are great for debugging, but if you want a more succinct  response its optional. The -X GET specifies the HTTP GET verb. GET is the implicit default so we haven't provided it previously. Because I got a 200 response I know the server accepted my AuthN via the Authorization header. It is common for the header to start with the authentication scheme which in this case is Bearer auth. Bearer auth is typically just a way to say token auth.

The response body above was a collection of project resources. We should be able to request the specific resource by ID. Also, let's see what happens if we do not authenticate at all
```
curl -i -X GET "https://api.todoist.com/rest/v2/projects/2315160149" -H "Authorization: Bearer ${MYAPITOKEN}"
curl -i -X GET "https://api.todoist.com/rest/v2/projects"
```
As expected we received a status 401 error. Next let's CREATE a new project using POST. Notice we're not sending a content-type header as the client. This is required so that server knows to process the JSON we are sending it. Feel free to try it without.
```
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/rest/v2/projects"  --data '{"name": "API Skill Goals"}'
```
This will return a resource representing your new project, take note of it's id. Next we can add some tasks to that project and then list our collection of tasks.
```
export MYPROJECT=ID_FROM_ABOVE
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/rest/v2/tasks"  --data '{"content": "Consume APIs", "project_id": "'"$MYPROJECT"'"}'
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/rest/v2/tasks"  --data '{"content": "Build APIs", "project_id": "'"$MYPROJECT"'"}'
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/rest/v2/tasks"  --data '{"content": "Profit", "project_id": "'"$MYPROJECT"'"}'
curl -i -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/rest/v2/tasks
```
Lets go look at the documentation for tasks: https://developer.todoist.com/rest/v2/#tasks. We can see all the properties for a task and how to interact with their API. Lets try updating a task and see our change take effect. We can also delete and mark a task done with the close api. You'll need to substitute your own task IDs below.
```
export CONSUME=ID_FROM_ABOVE
export BUILD=ID_FROM_ABOVE
export PROFIT=ID_FROM_ABOVE
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/rest/v2/tasks/$PROFIT"  --data '{"priority": 4}'
curl -i -H "Authorization: Bearer ${MYAPITOKEN}"  -H "Content-Type: application/json" "https://api.todoist.com/rest/v2/tasks/$PROFIT"
curl -i -H "Authorization: Bearer ${MYAPITOKEN}"  -X DELETE "https://api.todoist.com/rest/v2/tasks/$BUILD"
curl -i -X POST -H "Authorization: Bearer ${MYAPITOKEN}" "https://api.todoist.com/rest/v2/tasks/$CONSUME/close"
curl -i -X GET -H "Authorization: Bearer ${MYAPITOKEN}" https://api.todoist.com/rest/v2/tasks
```
Here, you will see the Consume APIs task is closed.  The Build APIs task is deleted.  And the Profit task is set to priority=4.
We have successfully tested the CRUD endpoints of this API, but the interface wasn't quite as expected. REST is a pattern which you should know and do you best to follow, but its not a rigid requirement.

