# Content Creation Response

Plaid is releasing a new API that utilizes cursor-based pagination for the /transactions/get endpoint, effective 12/2/2020. Per [Slack’s engineering blog](https://slack.engineering/evolving-api-pagination-at-slack-1c1f644f8e12), “cursor-based pagination works by returning a pointer to a specific item in the dataset. On subsequent requests, the server returns results after the given pointer.” (link) Let’s walk through how we might convert existing code for this endpoint into code that will work with Plaid’s new API.

We’ll start by examining the [existing code](https://plaid.com/docs/#retrieve-transactions-request) to highlight the differences between the current API and the new API. 

```javascript
// Pull transactions for a date range — old API
client.getTransactions(accessToken, '2018-01-01', '2018-02-01', {
  count: 250,
  offset: 0,
}, (err, result) => {
  // Handle err
  const transactions = result.transactions;
});
```

```
// Returns
{
"transactions": [...],
"total_transactions": 1250
}
```

In the above code, the client is retrieving transactions from 2018/01/01 to 2018/02/01 with an access token. The ‘count’ parameter refers to the number of transactions the client is supposed to retrieve, and the ‘offset’ parameter indicates the number of transactions to skip. This code will return a list of transactions (‘transactions’) as well as the total number of transactions within the specified date range (‘total_transactions’).

To make our code compatible with the new API, we need to make some changes. First, we can eliminate the ‘offset’ parameter; second, we need to provide a ‘cursor’ parameter. Example code compatible with the new API, then, looks like:

``` javascript
// Pull transactions for a date range — new API
client.getTransactions(accessToken, '2018-01-01', '2018-02-01', {
  count: 250,
  cursor: “abcd”,
}, (err, result) => {
  // Handle err
  const transactions = result.transactions;
});
```

```
// Returns
{
"transactions": [...],
"next_cursor": "efgh”
}
```

This code will return a list of transactions (‘transactions’) as well as an updated cursor (‘next_cursor’) that provides a pointer to the first transaction not returned in the list of transactions (in other words, the “next transaction”). We can then use this updated cursor — in this case, ‘efgh’ — as the 'cursor' parameter in subsequent calls of the getTransactions function.

We need to provide a null cursor the first time we call this function under the new API: that the server returns results after a given cursor/pointer, so there’s nothing to initialize it to for the first call.

We can retrieve all the transactions within a particular date range with a while loop: while a non-null next cursor is returned (ie. while there are more transactions to be retrieved within the specified date range), retrieve the transactions and update the cursor to the one returned by the current call.
