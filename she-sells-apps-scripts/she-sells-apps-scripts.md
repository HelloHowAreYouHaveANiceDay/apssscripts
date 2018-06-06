# she sells apps scripts

a quick guide to a google sheets based webapp. End to end the excercise should take around an hour.

### pros of using google apps scripts

- it's very fast to spin up
- you can use pre-existing gsheets data
- apps script can span multiple gsuite apps

### cons of using google apps scripts

- apps script is mostly javascript but not intuitive what parts of javascript can be used in apps scripts.
- no node module support, adding script libraries is fairly manual
- web IDE is not great.
- support of offline development is bare bones. (may change soon)

### using apps scripts in google business accounts

Apps script is perfect for internal business apps. With webapps and addons, you can restrict access to only those within the organization. Each apps script can be monitored through the dashboard. This takes the usually difficult part of authentication and logging out of the equation.

## the stack

The main logic of this set is using apps scripts as the "backend", using `HtmlService` from the apps scripts api to serve the "frontend", which then loads javascript and css libraries from CDNs. Vuejs is a great complement and handles the front end logic. Since Vue/vue-router/vuex can all be progressively included, your application can grow in complexity.

### google apps script

[guides](https://developers.google.com/apps-script/overview)

[reference](https://developers.google.com/apps-script/reference/calendar/)

- things to look up in the reference
  - apps scripts in general
    - permissions
    - api features
  - webserver
    - `doGet(){}` and reserved functions in apps script
    - `HtmlService.createTemplateFromFile()`
    - `addMetaTag()`
    - `google.script.run`
  - accessing sheets
    - `SpreadsheetApp`
    - `sheet`
    - `getRange() vs getDataRange()`
    - how ranges work and how data is structured when it is programmatically returned from sheets.
- additional reading
  - html template

### vuejs

[website](https://vuejs.org/)
[guides](https://vuejs.org/v2/guide/)

Vue is a big topic that I have no authority to fully cover. We will be using the basic set of features here. Components is supported but we will not be using it here.

- things to look up
  - the vue instance and life cycle
  - template syntax
  - event handling
  - form input binding
  - list rendering
- additional reading
  - component basics
  - global component registration

### bulma css

bulma works great with vue and getting a nice looking site up and running is a breeze. Basic bulma classes are used for style and should be simple to lookup based on the code.

[website](https://bulma.io)


## the app

- what is the problem we are trying to solve?
  - people are increasingly attached to their digital devices, and loosing touch with not only themselves but the emotions of those around them.
- how does this app help?
  - "feel-yo" gives individuals the power to shout their feelings into the world.
  - A limited palette of feelings nudges users towards reflection before committing to a feeling.
  - seeing how other people feel world wide will give the user a sense of "pluged-inness" not to their devices, but to the super concious.


### 1. Creating and Reading entries from google sheets

when you create a new script from sheets, `Code.gs` should already be there. The following apps scripts code will go in there. Rename your blank sheet to `values`. In the first row of the sheet, put `timestamp` in A1 and `feeling` in B1 as the two column titles. You can optionally freeze the first row.

``` js

// we're only building the create and read portion of CRUD. The read will be used in the bonus portion
// internal logic
function _getSheet(){
  // since we are calling this from the webapp, .getActiveSheet() would not work here
  return SpreadsheetApp.openById('yoursheetID').getSheetByName('values');
}

function _getHeaders(sheet){
  // gets the header row. We take the first entry of the 2d array that's returned
  // to get the headers as a single array of values.
  return sheet.getSheetValues(1, 1, 1, sheet.getLastColumn())[0];
}

function _getData(sheet){
  // check if there are any entires in the
  if(sheet.getLastRow() - 1 === 0){
    return false;
  }
  return sheet.getSheetValues(2, 1, (sheet.getLastRow() - 1), sheet.getLastColumn());
}

// these next two functions convert data from sheets format to a more 
// usable object format and vice versa

function _arrayifyObject(headers, payload){
 var values = [];
  for(var i = 0;i < headers.length; i++){
    // headers ensure the returned array is in the correct order
    values.push(payload[headers[i]]);
  }
  return values;
}

function _objectifyArray(headers, data){
  var collection = [];
  for(var i = 0;i < data.length; i++){
    var object = {};
    for(var j = 0; j < headers.length; j++){
      object[headers[j]] = data[i][j];
    }
    collection.push(object);
  }
  return collection;
}

function _addRow(payload){
  // payload should match headers. Since the cells do not verify the data
  // we haave to be kind of careful here.
  var sheet = _getSheet();
  var headers = _getHeaders(sheet);
  // getting headers each time allows us to rearrange columns
  // and maintain the array building logic.
  var values = _arrayifyObject(headers, payload);
  var result = sheet.appendRow(values);
  return result;
}

// These functions are used by the front end through google.script.run

// CREATE

function addValue(payload){
  payload.timestamp = new Date();
  _addRow(payload);
}


// READ
function getRecords(){
  var sheet = _getSheet();
  var headers = _getHeaders(sheet);
  var data = _getData(sheet);
  // IMPORTANT you cannot just return objects to the frontend
  // you have to json stringify it
  return JSON.stringify(_objectifyArray(headers, data));
}

```

These 8 functions should be all that's needed to create and get entries in sheets, allowing us to use it as a basic database. You can run a simple test through all these functions with the following test function.

``` js
function test(){
  var sheet = _getSheet()
  var headers = _getHeaders(sheet);
  var data = _getData(sheet);
  _addRow({
    timestamp: new Date(),
    feeling: 'testy',
  })
  var collect = getRecords();
  Logger.log(sheet);
  Logger.log(headers);
  Logger.log(data);
  Logger.log(collect);
}
```

### 2. serving up the page

Add a new html file. Call it `index.html` and leave it as a blank stub for now. The HTML will be added in the next step.

``` js
// doGet is called whenever someone visits the web link
function doGet() {
  return HtmlService.createTemplateFromFile('index.html')
      .evaluate()
      // responsive meta tag for bulma is added here instead of in the html
      .addMetaTag('viewport', 'width=device-width, initial-scale=1')
      .setSandboxMode(HtmlService.SandboxMode.IFRAME);
}
```

### 3. the webapp

The HTML is reasonably small.

``` html
<html>
  <head>
    <base target="_top">
    <!-- link in bulma -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.7.1/css/bulma.min.css">
    <script defer src="https://use.fontawesome.com/releases/v5.0.7/js/all.js"></script>
  </head>
  <body>
    <div id="app">
      <div class="section">

        <div class="field is-grouped is-grouped-centered">

          <div class="control">
            <div class="field-label is-normal">
              <label class="label"> I am feeling... </label>
            </div>
          </div>
          <div class="control">
            <div class="select is-primary">
              <select v-model="selectedFeeling">
                <option value="" disabled> select a feeling</option>
                <option v-for="feeling in feelings" :value="feeling">{{feeling}}</option>
              </select>
            </div>
          </div>
          <div class="control">
            <div :class="{
              'button': true,
              'is-loading': loading,
            }" @click="addFeeling">feel me</div>
          </div>

        </div>
        <div class="section">
          <table class="table is-fullwidth is-bordered">
            <thead>
              <tr>
                <th>feeling</th>
                <th>times felt</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="record in records">
                <td>{{record.key}}</td>
                <td>{{record.values.length}}</td>
              </tr>
            </tbody>
          </table>
        </div>

      </div>
    </div>

    <!--  add vuejs-->
    <script src="https://cdn.jsdelivr.net/npm/vue"></script>
    <!--  d3 collection helps summarize the data-->
    <script src="https://d3js.org/d3-collection.v1.min.js"></script>

    <script>
      var v = new Vue({
              el: '#app',
              data: {
                message: 'hello world',
                selectedFeeling: '',
                records: [],
                feelings: [
                  'Joy',
                  'Sadness',
                  'Anger',
                  'Fear',
                  'Disgust',
                ],
                loading: false,
              },
              mounted(){
                this.getFeelings();
              },
              methods: {
                getFeelings() {
                  google.script.run.withSuccessHandler(this.setFeelings).getRecords()
                },
                setFeelings(feelings) {
                  this.records = this.aggregateFeelings(JSON.parse(feelings));
                },
                addFeeling() {
                  const payload = {
                    feeling: this.selectedFeeling,
                  }
                  this.loading = true
                  google.script.run.withSuccessHandler(this.valueAddSuccess)
                  .addValue(payload);
                },
                valueAddSuccess(){
                  this.getFeelings();
                  this.loading = false;
                },
                aggregateFeelings(feelings){
                  return d3.nest()
                            .key(d => d.feeling)
                            .rollup()
                            .entries(feelings)
                }
              }
            })
    </script>

  </body>
</html>
```

### 4. deployment

deploy by going to **Publish** > **Deploy as web app**. give the project a new version and a short description for record keeping. Set **who has access to this app** to the setting you would like. Everyone who has access will be able to see the page and submit their feelings. Hit update and your URL should be presented to you.
