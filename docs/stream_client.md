# Hyperion Stream Client

!!! warning ""
    Streaming Client for Hyperion History API (v3+)

### 1. Usage

We currently provide libraries for nodejs and prebuilt browser bundle

#### npm package
```
npm install @eosrio/hyperion-stream-client --save
```

Import the client
```
const HyperionSocketClient = require('@eosrio/hyperion-stream-client').default;
```

#### Browser library
```
<script src="https://<ENDPOINT>/stream-client.js"></script>
```
Where `<ENDPOINT>` is the Hyperion API (e.g. `https://wax.hyperion.eosrio.io`)

For other usages the bundle is also available at `dist/bundle.js`

<br>

### 2. Connection

Setup the endpoint that you want to fetch data from and the flow control mode:

```javascript
const client = new HyperionSocketClient(ENDPOINT, {async: false});
```

!!! abstract "Example"
    ```javascript
    const client = new HyperionSocketClient('https://example.com', {async: false});
    ```
    `https://example.com` is the host, from where `https://example.com/v2/history/...` is served.

#### 2.1 Flow control mode

- Async: **true** - The transmission will be asynchronous and you need an acknowledge function. Incoming data will be held in a local queue until ack is called.
- Async: **false** - The transmission will be synchronous. The acknowledge function is not needed.

<br>

### 3. Requests

 - `client.streamActions(request: StreamActionsRequest): void` -- Request action stream
 - `client.streamDeltas(request: StreamDeltasRequest): void` -- Request delta stream (contract rows)
 
to ensure the client is connected, requests should be defined on the `client.onConnect` property, refer to examples below;

#### 3.1 Action Stream - client.streamActions

 - `contract` - contract account
 - `action` - action name
 - `account` - notified account name
 - `start_from` - start reading on block or on a specific date: (0=disabled)
 - `read_until` - stop reading on block  (0=disable) or on a specific date (0=disabled)
 - `filters` - actions filter (more details below)

!!! info "Notes"
    - Block number can be either positive or negative - E.g.: -15000, 700
    - In case of negative block number, it will be subtracted from the HEAD
    - Date format (ISO 8601) - e.g. 2020-01-01T00:00:00.000Z



```javascript
const HyperionSocketClient = require('@eosrio/hyperion-stream-client').default;
const client = new HyperionSocketClient('http://localhost:7000', {async: true});

client.onConnect = () => {
  client.streamActions({
    contract: 'eosio',
    action: 'voteproducer',
    account: '',
    start_from: '2020-03-15T00:00:00.000Z',
    read_until: 0,
    filters: [],
  });
}

// see 3 for handling data
client.onData = async (data, ack) => {
    console.log(data); // process incoming data, replace with your code
    ack(); // ACK when done
}

client.connect(() => {
  console.log('connected!');
});
```

##### 3.1.1 Act Data Filters
You can set up filters to refine your stream.

Filters should use fields following the Hyperion Action Data Structure, such as:

 - `act.data.producers` (on eosio::voteproducer)
 - `@transfer.to` (here the @ prefix is required since transfers have special mappings)
 
 please refer to the [mapping definitions](https://github.com/eosrio/Hyperion-History-API/blob/develop/definitions/index-templates.ts) to know which data fields are available

!!! abstract "Example"
    Filter the stream for
    every transfer made to the `eosio.ramfee` account:
    
    ```javascript
    client.streamActions({
        contract: 'eosio.token',
        action: 'transfer',
        account: 'eosio',
        start_from: 0,
        read_until: 0,
        filters: [
            {field: '@transfer.to', value: 'eosio.ramfee'}
        ],
    });
    ``` 
    
    To refine even more your stream, you could add more filters.  

!!! attention
    Remember that adding more filters will result in AND operations. Currently, it's not possible to make OR operations with filters.

<br>

#### 3.2 Delta Stream - client.streamDeltas

 - `code` - contract account
 - `table` - table name
 - `scope` - table scope
 - `payer` - ram payer
 - `start_from` - start reading on block or on a specific date: (0=disabled)
 - `read_until` - stop reading on block  (0=disable) or on a specific date (0=disabled)

!!! abstract "Example"
    Referring to the same pattern as the action stream example above, one could also include a delta stream request
    ```javascript
    client.streamDeltas({
      code: 'eosio.token',
      table: '*',
      scope: '',
      payer: '',
      start_from: 0,
      read_until: 0,
    });
    ``` 

!!! note
    Delta filters are planned to be implemented soon.

<br>

### 4. Handling Data

Incoming data is handled via the `client.onData` callback

data object is structured as follows:

 - type - _action_ | _delta_
 - mode - _live_ | _history_
 - content - Hyperion Data Structure (see [action index](https://github.com/eosrio/Hyperion-History-API/blob/develop/definitions/index-templates.ts#L40) template)
 
```javascript
client.onData = async (data, ack) => {
    console.log(data); // process incoming data, replace with your code
    ack(); // ACK when done
}
```

On every LIB update, `client.onLIB` is called

```javascript
client.onLIB = async (data) => {
    console.log(data);  
}
```

You can also look for forks, using the `client.onFork` callback 
```javascript
client.onFork = async (data) => {
    console.log(data); // process when a there is a fork
}
```

If you set `async: false` on the connection step, the `ack` must not be used:
```javascript
client.onData = async (data) => {
    //code here
}
```

<br>
 
