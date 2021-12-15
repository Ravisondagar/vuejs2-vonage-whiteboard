### install the package using below command :
```
npm i vuejs2-vonage-whiteboard
```

### import the package in your component using below line :
```
import Vue2VonageWhiteboard from 'vuejs2-vonage-whiteboard'
```

### register it in the component using below syntax :
```
  components: {
    Vue2VonageWhiteboard
  }
```
### How to use Component
### Using OT.initSession this method of @opentok/client, you can pass that object to your component
```
<div v-if="session">
    <Vue2VonageWhiteboard :session="session" />
</div>
```