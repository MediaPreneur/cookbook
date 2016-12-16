# Recipes for the botpress-subscription module

## Sending a message to all subscribed users given a category

There are many ways you can do this:

### Sending it right now

#### Programmatically

You could query the list of all subscribed users directly from the database:

```js
bp.db.get() // We get a database query instance
.then(knex => {
  // We get all the subscribed users for a given category
  return knex('subscription_users')
  .join('users', 'subscriptions_users.userId', 'users.id')
  .join('subscriptions', 'subscriptions.id', 'subscription_users.subscriptionId')
  .where({ category: '<SUBSCRIPTION_CATEGORY_HERE>' })
})
.then(users => {
  // We send out a message to each of them
  users.forEach(user => {
    bp.messenger.sendMessage(user.id, 'Hello subscribed user :)')
  })
})
```

#### Using broascast

You can schedule a broadcast *in the past*, which has the effect of sending the broadcast right now. You need to leverage the *filters* to filter out the non-subscribed users (see "Sending it at a specific date and time" for an example of how to do that)

### Sending it at a specific date and time

If you would like to send a message at a specific date and time (which is not right now), the easy solution is to use the `botpress-broadcast` module to schedule sending a message. You will then leverage the *filters* to only send the message to subscribed users.

The advantages of using broadcast module is that you don't have to manage user timezones and don't have to worry about your bot rebooting or crashing.

You can create a new broadcast via the UI or using the API.

#### UI

Add a filter with the following condition: `bp.subscription.isSubscribed(userId, subscription_id)`

**Note:** The subscription id is case sensitive

<img src='/assets/mod_subscription_1.jpg'>

#### API

The same can be achieved by calling the broadcast REST API:

```js
// The API is secured, we need to authenticate ourselves
const token = bp.security.login('admin', bp.botfile.login.password, '127.0.0.1')

// We use the "axios" library to make API calls
const api = axios.create({
  baseURL: 'http://localhost:' + bp.botfile.port + '/api/',
  headers: {'Authorization': token}
})

// We plan to send the message tomorrow. We use the "moment" library
const tomorrow = moment().add(1, 'day').format('YYYY-MM-DD')

// We call the API to schedule a broadcast
api.put('botpress-broadcast/broadcasts', {
  date: tomorrow,
  time: '08:00',
  timezone: null, // null means we use the users timezone
  type: 'javascript',
  content: "bp.sendRandomVideo(userId, 'LIFE')", // sendRandomVideo is a function we injected in our bot
  filters: ["bp.subscription.isSubscribed(userId, 'daily')"]
})
```

### Sending it on a recurring basis

To deal with recurring events, we advise to use the `botpress-scheduler` module.

The recipe is the same as on a specific date and time, execept that this time, you want to schedule a broadcast on a recurring basis:

<img src='/assets/mod_subscription_2.jpg'>

### Sending it on a recurring basis on a specific date and time

To achieve this, we easiest way is to:

1. Create a function that schedules a broadcast (see the use of the broadcast API above) and inject it in your bot instance. For example `bp.scheduleBroadcast = function() { ... }`
2. Use the `botpress-scheduler` module to schedule the broadcast on a recurring basis
3. Filter out the people who are not subscribed from the broadcast
