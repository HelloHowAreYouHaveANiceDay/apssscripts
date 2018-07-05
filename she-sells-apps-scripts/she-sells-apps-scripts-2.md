# she-sells-apps-scripts part 2

this is a continuation of the previous guide. Here we will be adding update and delete. GAS provides huge library of utilities, we will be using [Utilities.getUuid()](https://developers.google.com/apps-script/reference/utilities/utilities#getUuid) to generate ids for our entries. Then, these IDs are then used to lookup the right entires for update and delete.

## adding unique ids to the backend

### 1. modify the sheet to accommodate ids

add a column to the attached google sheet named `id`. Since our code is agnostic to the order, this column should be able to be added anywhere, as long as the header row has the right key.

### 2. modify the addValue function to add an id to every new entry

with `Utilities.getUuid()` adding this value is really simple. This value should never change for the lifetime of the record. Be careful to not expose this to update/editing.

``` js
function addValue(payload){
  payload.timestamp = new Date();
  //the back end now timestamps and ids each new entry
  payload.id = Utilities.getUuid();
  _addRow(payload);
  return payload.id;
}
```

### 3. adding a findRow helper function

In sheets, manipulating data is dependent on manipulating "ranges" of cells. We need the ability to find the correct "range" of values based on the id, and then delete the row or update the data in the right cells.

``` js
function _findRow(id){
  if(sheet.getLastRow() - 1 === 0){
    return false;
  }
  var ids = sheet.getSheetValues(2, 1, (sheet.getLastRow() - 1), 1);
  // +2 because we are compensating for the header row
  // and the index
  var row = ids.indexOf(id) + 2;
}
```

### 4. adding delete

`Sheet.deleteRow()` takes a `rowPosition`. Our `_findRow()` function compensates for the indexes and will return the right number for us.

``` js
function deleteValue(id){
  var sheet = _getSheet();
  try {
    sheet.deleteRow(_findRow(sheet, id));
  } catch(err) {
    throw err;
  }
}
```

### 5. adding update

update is a little bit more involved, but is mostly `_addRow` with a little help from `_findRow`

``` js
function updateValue(payload) {
  var sheet = _getSheet();
  var headers = _getHeaders(sheet);
  var rowPosition = _findRow(sheet, payload.id);
  var values = sheet.getSheetValues(rowPosition, 1, 1, sheet.getLastColumn());
  for(var i=0;i < headers.length;i++){
    if (headers[i] === 'id' || headers[i] === 'timestamp'){
      // do not modify timestamps or ids
      // there's an opportunity here to do a last edited column
    }else if(values[0][i] === payload[headers[i]] ) {
      // don't update the value if nothing changed
    } else {
      values[0][i] = payload[headers[i]];
    }
  }
  try {
    // shove it back where it came from
    sheet.getRange(rowPosition, 1, 1, sheet.getLastColumn()).setValues(values);
  } catch (err) {
    throw err;
  }
}
```

## business in the front

### 1. remove record aggregation

in example, we aggregated the records data. Now, since we are updating and deleting individual ones, we'll need to remove the logic for aggregating. Go through and remove d3, and the aggregate function. Once done your script section should look something like this:

``` html
<!--  add vuejs-->
<script src="https://cdn.jsdelivr.net/npm/vue"></script>

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
            this.getFeelings()
          },
          methods: {
            getFeelings() {
              google.script.run.withSuccessHandler(this.setFeelings).getRecords()
            },
            setFeelings(feelings) {
              this.records = JSON.parse(feelings);
            },
            addFeeling() {
              const payload = {
                feeling: this.selectedFeeling,
              }
              this.loading = true
              google.script.run.withSuccessHandler(this.valueAddSuccess).addValue(payload);
            },
            valueAddSuccess(){
              this.getFeelings()
              this.loading = false         
            }
          }
        })
</script>
```

### 2. adding a table for the records

to keep things simple, we'll use the table we already built and some buttons. We'll do viewing for individual records first and then attach methods for update and delete. Edit the table we built in the previous exercise into the one below

```html
<div class="section">
  <table class="table is-fullwidth">
    <thead>
      <tr>
        <th>feeling</th>
        <th>timestamp</th>
        <th>edit</th>
        <th>delete</th>
      </tr>
    </thead>
    <tbody>
    <tr v-for="record in records">
      <td>{{record.feeling}}</td>
      <td>{{record.timestamp}}</td>
      <td><button class="button is-info">edit</button></td>
      <td><button class="button is-danger">delete</button></td>
    </tr>
    </tbody>
  </table>
</div>
```

### 3. deleting a record

this is pretty straight-forward. Bind the function to a button and pass it the id.

```html
<td>
<button @click="deleteFeeling(record.id)" :class="{
            'button': true,
            'is-danger' :true,
            'is-loading': loading
          }">delete</button>
</td>
```

``` js
deleteFeeling(id){
    this.loading = true
    // this.refresh is renamed from success function from last excercise.
    google.script.run.withSuccessHandler(this.refresh).deleteValue(id);
  },
```

### 4. updating a record

we'll use v-model, a modal, and some toggles. We will bind this set to the edit button.

``` html
<div :class="{
'modal':true,
'is-active': editing
}">
<div class="modal-background" @click="cancelEdit"></div>
<div class="modal-content">
<!-- Any other Bulma elements you want -->
<div v-if="selectedRecord" class="card">
    <div class="select is-primary">
      <select v-model="selectedRecord.feeling">
        <option value="" disabled> select a feeling</option>
        <option v-for="feeling in feelings" :value="feeling">{{feeling}}</option>
      </select>
    </div>
  
  <button @click="pushEdit" :class="{
      'button': true,
      'is-loading': loading
    }">save edit</button>
</div>
</div>
<button @click="cancelEdit" class="modal-close is-large" aria-label="close"></button>
</div>
```

```js
editFeeling(record){
  this.selectedRecord = record;
  this.editing = true;
},
pushEdit(){
  this.loading = true;
  google.script.run.withSuccessHandler(this.refresh).updateValue(this.selectedRecord);
},
cancelEdit(){
  this.selectedRecord = null;
  this.editing = false;
},
```


with that, the edits should be passing through and working! happy scripting!